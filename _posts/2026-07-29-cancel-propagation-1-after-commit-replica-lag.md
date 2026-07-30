---
layout: post
title: "커밋했는데 왜 안 읽히지? — AFTER_COMMIT 리스너가 방금 저장한 데이터를 못 찾는 이유 (1/3)"
date: 2026-07-29 09:40:00 +0900
tags: [spring, transaction, jpa, database, replication, event]
series: 커밋 후 외부 전파 설계기
series_part: 1
series_total: 3
---

> **시리즈 "커밋 후 외부 전파 설계기" (1/3)**
> 이 글은 "커밋 직후 조회가 텅 비어 오는" 함정을 다룹니다. 이어지는 (2/3)은 이 함정을 감싸는 큰 그림(로컬 먼저 확정하고 외부는 나중에 전파하는 설계), (3/3)은 그 코드를 어떻게 레이어로 나눴는지를 다룹니다.

## "분명 방금 저장했는데, 조회하면 없다?"

결제 취소 기능을 만들다 이상한 버그를 만났습니다.

흐름은 이랬습니다. 사용자가 결제를 취소하면, **우리 DB에 취소를 저장(커밋)한 뒤**, 그 결제에 딸린 승인 건들을 다시 조회해 외부 결제 게이트웨이(PG사)에 "이 결제 취소해 주세요"라고 전파합니다. (PG = Payment Gateway, 카드사·은행과 우리 서비스 사이에서 결제를 중계하는 외부 시스템)

그런데 이 "다시 조회"가 **가끔 빈 결과**를 돌려줬습니다. 방금 커밋했는데도요. 커밋(트랜잭션을 확정 저장하는 것)이 끝났으면 당연히 읽혀야 하는 것 아닌가? 이 "당연히"가 함정이었습니다.

용어 하나만 먼저 풀고 가겠습니다. **트랜잭션**은 여러 DB 작업을 *전부 성공 아니면 전부 취소*로 묶는 단위이고, 그걸 확정 저장하는 게 **커밋**입니다.

## 개념 ① 커밋 후에 도는 리스너는 "트랜잭션 밖"에 있다

먼저 이 코드가 쓴 도구를 봅시다. 스프링(Spring)에는 `@TransactionalEventListener`라는 게 있습니다. **"트랜잭션이 커밋된 뒤에 실행되는 이벤트 처리기(리스너)"** 를 만드는 어노테이션입니다. (리스너 = 어떤 사건이 발생하면 그에 반응해 실행되는 코드)

기본 설정이 `AFTER_COMMIT`, 즉 **커밋이 끝난 다음**에 실행됩니다. 여기에 함정의 씨앗이 있습니다.

커밋이 끝난 뒤에 실행된다는 건, 리스너 코드가 도는 그 순간에는 **원래 트랜잭션이 이미 종료됐다**는 뜻입니다. 즉 이 리스너는 **활성 트랜잭션이 없는 상태**로 실행됩니다.

## 개념 ② 트랜잭션이 없으면, 조회는 "읽기 전용"으로 열린다

그럼 리스너 안에서 DB를 조회하면 무슨 일이 일어날까요?

스프링 데이터 JPA(DB 접근을 자동화해 주는 라이브러리)의 기본 저장소 구현체인 `SimpleJpaRepository`에는 클래스 전체에 `@Transactional(readOnly = true)`가 붙어 있습니다. **"이 조회는 읽기 전용"** 이라는 표시입니다.

그래서 활성 트랜잭션이 없는 리스너에서 저장소를 **직접** 호출하면, 그 조회를 위해 새로 열리는 트랜잭션은 `readOnly = true`가 됩니다. 여기까지는 그냥 "읽기 전용 조회"일 뿐, 문제없어 보입니다. 다음 개념과 만나기 전까지는요.

## 개념 ③ "읽기 전용"이라는 꼬리표가 조회를 복제 DB로 보낸다

트래픽이 많은 서비스는 보통 DB를 **쓰기 담당(master)** 과 **읽기 담당(replica, 복제본)** 으로 나눕니다. 쓰기는 master 한 곳에서 하고, master의 내용을 replica로 계속 복사해서 읽기 요청을 replica가 나눠 받습니다. 부하를 분산하려는 겁니다.

이때 "이 조회를 master로 보낼까 replica로 보낼까"를 어떻게 정할까요? 스프링에서는 흔히 `AbstractRoutingDataSource`를 써서 **현재 트랜잭션이 읽기 전용인지(readOnly)** 를 보고 정합니다.

아래 코드가 하는 일: **readOnly면 replica로, 아니면 master로 보내라**고 라우팅(경로 결정) 규칙을 정합니다.

```java
public class RoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        // readOnly=true → replica(읽기 복제본), false → master(쓰기 원본)
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                ? "replica" : "master";
    }
}
```

이제 ②와 ③을 이으면 결론이 보입니다.
**커밋 후 리스너에서 저장소를 직접 조회 → readOnly=true → replica로 감.**

## 개념 ④ 그래서 "방금 커밋한 데이터"를 못 볼 수 있다

master에 방금 커밋한 데이터가 replica로 복사되기까지는 시간이 걸립니다. 보통 수 밀리초(ms)에서 상황에 따라 수백 ms. 이 시간차를 **복제 지연(replication lag)** 이라고 합니다.

그런데 `AFTER_COMMIT` 리스너는 커밋 "직후"에 실행됩니다. 하필 복제가 아직 안 끝난 그 짧은 창에 정확히 걸립니다.

- master: 방금 취소 커밋됨 ✅
- replica: 아직 복사 안 됨 ⏳
- 리스너의 조회는 replica로 감 → **빈 결과** ❌

"분명 방금 저장했는데 조회하면 없다"의 정체가 이것이었습니다. 데이터가 사라진 게 아니라, **엉뚱한(아직 안 따라온) DB에 물어본** 겁니다.

## 개념 ⑤ 해결: 조회를 "새 쓰기 트랜잭션"으로 감싸 master로 보낸다

원인이 "readOnly라 replica로 갔다"이니, 해결은 "이 조회만은 readOnly가 아니게 만들어 master로 보내기"입니다.

방법은 조회 메서드를 `@Transactional(propagation = REQUIRES_NEW)`로 감싸는 것입니다. `REQUIRES_NEW`는 **"기존 트랜잭션에 얹히지 말고, 무조건 새 트랜잭션을 시작하라"** 는 뜻입니다. 그리고 `@Transactional`의 readOnly 기본값은 `false`. 따라서 이 조회는 readOnly=false인 새 트랜잭션에서 돌고, 개념 ③의 규칙에 따라 **master로 라우팅**됩니다.

아래 코드가 하는 일: 복제 지연을 피하기 위해, 이 조회를 새 트랜잭션에서 열어 반드시 master(쓰기 원본)에서 읽게 합니다.

```java
@Component
@RequiredArgsConstructor
class CancelPropagationProcessor {

    private final PaymentRepository paymentRepository;

    // 활성 트랜잭션이 없는 AFTER_COMMIT 경로에서 호출된다.
    // REQUIRES_NEW(readOnly=false) → master 조회 → 복제 지연 회피
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public List<Payment> findPayments(String paymentKey) {
        return paymentRepository.findByPaymentKey(paymentKey);
    }
}
```

> 참고: 이 라우팅이 "첫 SQL을 실행하는 시점의 readOnly"를 기준으로 정확히 동작하려면 `LazyConnectionDataSourceProxy`가 필요합니다. 이게 없으면 트랜잭션이 시작되는 순간 DB 커넥션이 먼저 고정돼, readOnly 값을 반영하기 전에 엉뚱한 쪽으로 붙을 수 있습니다.

## 코드로 검증한 과정

이 결론은 "그럴 것 같다"로 낸 게 아닙니다. 처음엔 저도 "커밋 끝났으니 읽히는 게 당연하지"라고 생각했습니다. 그 "당연히"가 **DB가 하나뿐이라는 무의식적 가정**에서 나왔다는 걸 의심하고, 라우팅 코드를 직접 열어 봤습니다.

`RoutingDataSource.determineCurrentLookupKey()`를 확인하니 경로를 가르는 기준이 정확히 `isCurrentTransactionReadOnly()`였습니다. 즉 **읽기 전용 여부가 곧 master/replica를 가른다**는 게 코드로 확정됐고, "저장소 직접 호출 = readOnly = replica = 복제 지연 노출"이라는 사슬이 완성됐습니다. 그래서 `REQUIRES_NEW`로 정정한 겁니다.

## 한 줄 교훈

> **"커밋했으니 읽힌다"는 DB가 하나일 때만 참이다.** 읽기/쓰기 DB를 나눈 환경에서 커밋 직후(AFTER_COMMIT) 조회를 한다면, 그 조회가 *어느 트랜잭션의 readOnly 값으로 어느 DB에 가는지*부터 확인하라.

---

→ **다음 (2/3)**: 이렇게 master에서 읽어온 데이터로, 로컬은 즉시 확정하고 외부 전파는 커밋 후로 미루는 **"로컬 먼저(local-first)" 최종적 일관성 설계**를 다룹니다.
