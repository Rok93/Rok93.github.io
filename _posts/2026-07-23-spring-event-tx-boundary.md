---
layout: post
title: 'Spring 이벤트의 트랜잭션 경계와 load-bearing 상태 — "스멜"이 "정당한 설계"로 뒤집힌 과정'
date: 2026-07-23 14:00:00 +0900
tags: [spring, transaction, event, consistency, code-review]
---

## 어떤 PR에서 출발했나

한 PR이 이런 변경을 했다. 어떤 엔티티를 **생성할 때**가 아니라 **최초 발행할 때** "진행중" 상태가 되도록 상태 흐름을 바꾼 것이다. 예시 도메인(결제/청구)으로 옮겨 설명한다.

- `Invoice`(청구서)를 생성할 때 초기 상태를 `DRAFT`로 둔다. (기존: `ISSUED`)
- 최초 발행 시 도메인 이벤트 `InvoiceIssuedEvent`로 `DRAFT → ISSUED`로 전환한다.

리뷰 스레드의 쟁점은 딱 두 가지였다. **이 후속 처리를 이벤트로 둘까 직접 호출로 둘까? 같은 트랜잭션이냐 별도냐?** 이 한 PR을 파고드니 백엔드 핵심 개념이 줄줄이 엮여 나왔다.

## ① Spring 이벤트는 기본이 "동기"다

`ApplicationEventPublisher.publishEvent()`는 별도 스레드가 아니라 **그 자리에서** 리스너를 실행한다. "이벤트 = 비동기"는 흔한 오해다.

그럼 직접 메서드 호출과 뭐가 다른가? 이벤트를 쓸 명분은 딱 둘이다.

- **디커플링**: 발행자가 "누가 이 일을 처리하는지" 몰라도 된다.
- **팬아웃**: 한 사건에 여러 리액터를 매달 수 있다.

이 둘이 필요 없으면 이벤트는 그저 간접 지시만 늘리는 오버엔지니어링이다. 이 케이스는 같은 이벤트에 리스너가 **2개**(상태전환 / 부수 플래그 갱신) 붙어 있어 팬아웃 명분이 실재했다.

## ② @EventListener vs @TransactionalEventListener = 실행 "시점"

이벤트를 쓰기로 했다면 다음 갈림길은 **리스너가 트랜잭션 생애주기의 어디에서 도느냐**다.

```java
// 상태전환 — 같은 tx, 커밋 전, 동기. 실패하면 발행자까지 롤백
@EventListener
public void syncStatusOnIssued(InvoiceIssuedEvent event) { ... }

// 부수 플래그 — 커밋 후, 새 tx(REQUIRES_NEW)
@TransactionalEventListener
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void markEmailed(InvoiceIssuedEvent event) {
    try {
        invoiceRepository.markEmailedFirstTimeByIds(event.invoiceIds());
    } catch (Exception e) {
        log.error("메일 발송 플래그 갱신 실패: {}", event.invoiceIds(), e);
    }
}
```

- `@EventListener` → 발행 즉시, **커밋 전, 같은 트랜잭션**. 실패 시 원자적 롤백.
- `@TransactionalEventListener` → 기본 **커밋 후**(AFTER_COMMIT). `+REQUIRES_NEW`면 **새 트랜잭션**.

즉 **같은 이벤트인데 두 리스너의 tx 모델이 정반대**였다. 처음엔 이걸 스멜로 의심했다.

## ③ 같은 tx냐 별도냐 = 일관성 모델 선택

| | 강한 일관성 (같은 tx) | 최종 일관성 (별도/async) |
|---|---|---|
| 보장 | 커밋 순간 항상 일치 | 잠깐 어긋났다 수렴 |
| 대가 | 응답 지연·실패 전파 | 유령 구간·재처리 필요 |
| 언제 | 어긋나면 즉시 오판하는 데이터 | 어긋나도 잠깐은 괜찮은 데이터 |

## ④ load-bearing 상태가 일관성 요구를 정한다

판별 기준은 한 문장이다. **"그 값을 읽고 즉시 허용/거부를 분기하는 로직이 있나?"**

```java
public boolean isEditable() {
    return status == Status.DRAFT || status == Status.ISSUED;
}

public void cancel(...) {
    if (!isEditable()) {
        throw new BusinessException("이 상태에서는 취소/수정할 수 없습니다.");
    }
    ...
}
```

- **status**: `isEditable()`이 읽어 수정/취소를 **게이팅**한다 → load-bearing → 강한 일관성(같은 tx) 필요.
- **isEmailed(부수 플래그)**: 조회 필터·화면 표시·타임스탬프용, write를 게이팅하지 않는다 → load-bearing 아님 → 최종 일관성으로 충분.

## ⑤ "한 사건, 다른 tx 모델"은 스멜일 수도, 정당할 수도

두 데이터의 **치명도가 다르면** tx 모델이 다른 게 오히려 옳다.

- load-bearing 상태 → 발행과 원자적으로 같은 tx.
- best-effort 플래그 → 그 실패가 중요한 발행을 롤백시키지 않도록 별도 tx로 **의도적 격리**.

원 설계자가 `REQUIRES_NEW` + 예외 삼킴으로 만든 구조는 "이 플래그는 best-effort"라는 **의도적 설계**였다.

## ⑥ 자가복구 가드가 어느 필드 기준인지 확인하라

부수 플래그 갱신 쿼리를 보자.

```sql
update Invoice i
   set i.isEmailed = true, i.firstIssuedAt = current_timestamp
 where i.id in :ids and (i.isEmailed = false or i.isEmailed is null)
```

가드가 `isEmailed = false`, 즉 **자기 자신** 기준이다. 그래서 첫 갱신이 실패해도 **재발행 시 이 쿼리가 다시 돌아 self-heal** 된다. 나는 이걸 상태 리스너의 가드(`status != DRAFT → no-op`)와 혼동해 "복구 불가"라고 단정했다가, 코드를 보고 정정했다.

## 코드로 검증한 과정 (이 글의 핵심)

"부수 플래그가 정말 안 중요한가?"를 추측하지 않고 사용처를 전수 검색했다. 읽는 곳은 조회 표시 / 검색·다운로드 필터 / 타임스탬프뿐, **write를 게이팅하는 곳은 없었다.** 이 한 번의 검증으로 두 가지가 뒤집혔다.

1. "tx 모델 불일치 = 스멜" → **"치명도에 맞춘 정당한 설계"**
2. "실패 시 복구 불가" → **"재시도로 self-heal"** (남는 건 최초 시각이 재시도 시각으로 오염될 여지 정도)

결론적으로 **must-fix 결함은 없었다.** 리뷰의 진짜 소득은 버그 발견이 아니라, 왜 이 설계가 맞는지 **방어할 수 있게 된 것**이었다.

## 한 줄 교훈

> load-bearing 여부는 상상하지 말고 "누가 그 값을 읽고 무슨 결정을 하는지"를 코드에서 확인해 정하라. 그것이 이벤트/트랜잭션 설계 리뷰의 출발점이다.
