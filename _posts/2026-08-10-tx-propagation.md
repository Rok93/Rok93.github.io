---
layout: post
title: '트랜잭션 전파(Propagation) — 안쪽 트랜잭션이 실패하면 바깥도 같이 죽나?'
date: 2026-08-10 09:00:00 +0900
image: /assets/img/tx-propagation/hero.jpg
generated: true
tags: [spring, study]
sanitized: true
---

주문을 저장하면서, 그 과정을 감사 로그(audit log) 테이블에도 남기는 코드를 짰다. 주문 저장 메서드가 로그 저장 메서드를 호출하는 구조다. 그런데 로그 저장이 어쩌다 실패했더니, **멀쩡히 저장됐어야 할 주문까지 통째로 롤백**됐다. 반대로, 주문 저장이 실패했는데 로그만 남아 버리는 경우도 있다. 무엇이 이 차이를 만들까? 답은 **트랜잭션 전파(transaction propagation)** — 이미 진행 중인 트랜잭션이 있을 때, 새로 호출된 `@Transactional` 메서드가 그 트랜잭션에 **합류할지, 따로 놀지**를 정하는 규칙이다. 이 글은 전파의 대표 옵션들과, 가장 많이 데는 롤백 함정을 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 코드·시나리오·예외 상황은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/tx-propagation/hero.jpg" alt="큰 반투명 상자 안에 작은 상자가 중첩돼 있고, 옆에는 분리된 독립 상자가 하나 더 있는 모습" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">전파는 "안쪽 작업을 바깥 트랜잭션에 포함시킬까(왼쪽 중첩 상자), 아니면 완전히 따로 떼어낼까(오른쪽 독립 상자)"를 정하는 일이다.</figcaption>
</figure>

## 트랜잭션 전파가 뭔가 — "합류냐, 분가냐"

먼저 용어부터. **트랜잭션(transaction)**은 여러 DB 작업을 *전부 성공 아니면 전부 취소*로 묶는 단위다. `@Transactional`을 붙인 메서드는 시작할 때 트랜잭션을 열고, 정상 종료하면 **커밋(확정 저장)**, 예외가 터지면 **롤백(통째로 되돌림)**한다.

문제는 **`@Transactional` 메서드가 또 다른 `@Transactional` 메서드를 부를 때** 생긴다. 바깥 메서드가 이미 트랜잭션을 열어 둔 상태에서 안쪽 메서드가 시작하면 — 안쪽은 **바깥 트랜잭션에 얹혀 갈까(합류)**, 아니면 **자기만의 새 트랜잭션을 열까(분가)**? 이 선택을 정하는 게 전파 옵션이다. `@Transactional(propagation = ...)`로 지정한다.

전파가 왜 중요하냐면, **이게 롤백의 경계선을 긋기 때문**이다. 안쪽과 바깥이 한 트랜잭션이면 둘 중 하나만 실패해도 함께 롤백된다. 따로 떨어져 있으면 각자 독립적으로 커밋·롤백한다. 앞의 "주문은 살리고 로그 실패는 무시하고 싶다" 같은 요구는 정확히 이 경계 설정 문제다.

<!-- diagram: required-vs-requires-new -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 210" role="img" aria-label="REQUIRED와 REQUIRES_NEW 비교. REQUIRED는 바깥과 안쪽이 하나의 물리 트랜잭션이고, REQUIRES_NEW는 안쪽이 별개의 트랜잭션이며 바깥은 잠시 중단된다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="82" y="16" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="9">REQUIRED</text>
    <text x="82" y="28" text-anchor="middle" fill="#5f5a68" font-size="7">안쪽이 바깥에 합류</text>
    <rect x="18" y="36" width="128" height="90" rx="8" fill="#f3ecf7" stroke="#5F0080" stroke-width="2"/>
    <text x="30" y="50" fill="#5F0080" font-size="7.5" font-weight="700">물리 트랜잭션 1개</text>
    <rect x="34" y="58" width="96" height="24" rx="4" fill="#fff" stroke="#7d5a9e" stroke-width="1.4"/>
    <text x="82" y="73" text-anchor="middle" fill="#5f5a68" font-size="7.5">바깥: 주문 저장</text>
    <rect x="34" y="90" width="96" height="24" rx="4" fill="#fff" stroke="#7d5a9e" stroke-width="1.4"/>
    <text x="82" y="105" text-anchor="middle" fill="#5f5a68" font-size="7.5">안쪽: 로그 저장</text>
    <text x="82" y="140" text-anchor="middle" fill="#b23a00" font-size="7">하나 실패 → 함께 롤백</text>

    <line x1="165" y1="34" x2="165" y2="150" stroke="#e4e0ec" stroke-width="1.4"/>

    <text x="248" y="16" text-anchor="middle" fill="#1f7a4d" font-weight="700" font-size="9">REQUIRES_NEW</text>
    <text x="248" y="28" text-anchor="middle" fill="#5f5a68" font-size="7">안쪽이 따로 분가</text>
    <rect x="186" y="40" width="124" height="34" rx="6" fill="#fff" stroke="#7d5a9e" stroke-width="1.6" stroke-dasharray="4 3"/>
    <text x="248" y="55" text-anchor="middle" fill="#5f5a68" font-size="7.5">바깥 TX (일시 중단)</text>
    <text x="248" y="67" text-anchor="middle" fill="#b9a9cc" font-size="7">suspend</text>
    <rect x="186" y="82" width="124" height="34" rx="6" fill="#eaf5ef" stroke="#1f7a4d" stroke-width="2"/>
    <text x="248" y="97" text-anchor="middle" fill="#1f7a4d" font-size="7.5" font-weight="700">안쪽 TX (새 트랜잭션)</text>
    <text x="248" y="109" text-anchor="middle" fill="#5f5a68" font-size="7">독립 커밋/롤백</text>
    <text x="248" y="140" text-anchor="middle" fill="#1f7a4d" font-size="7">각자 따로 커밋</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">REQUIRED는 안팎이 한 몸이라 운명을 같이하고, REQUIRES_NEW는 안쪽이 별도 트랜잭션이라 바깥과 무관하게 커밋·롤백된다.</figcaption>
</figure>

## 대표 옵션 두 개 — REQUIRED와 REQUIRES_NEW

전파 옵션은 7개지만, 실무의 90%는 이 둘로 갈린다.

### REQUIRED — 기본값, "있으면 합류, 없으면 새로"

`REQUIRED`는 Spring의 **기본값**이다. 규칙은 단순하다: **진행 중인 트랜잭션이 있으면 거기에 합류하고, 없으면 새로 하나 연다.** 대부분의 서비스 메서드가 이걸 쓴다.

이 코드가 하는 일: 주문을 저장하고, 같은 트랜잭션 안에서 로그도 남긴다.

```java
@Transactional  // propagation = REQUIRED (기본값)
public void placeOrder(Order order) {
    orderRepository.save(order);
    auditService.log("주문 저장됨");  // 아래 메서드 호출
}

@Transactional(propagation = Propagation.REQUIRED)
public void log(String msg) {
    auditRepository.save(new Audit(msg));
}
```

여기서 `log()`는 이미 `placeOrder()`가 연 트랜잭션에 **합류**한다. 즉 둘은 **물리적으로 하나의 트랜잭션**이다. 그래서 로그 저장이 실패하면 주문 저장까지 함께 롤백된다. "전부 아니면 전무"가 필요한 한 덩어리 작업에 알맞다.

### REQUIRES_NEW — "무조건 새 트랜잭션, 바깥은 잠깐 멈춰"

`REQUIRES_NEW`는 **항상 자기만의 새 트랜잭션을 연다.** 진행 중인 바깥 트랜잭션이 있으면 그걸 **일시 중단(suspend, 잠시 보류)**시켜 두고, 안쪽 트랜잭션을 독립적으로 처리한 뒤, 끝나면 바깥을 다시 이어 간다.

핵심은 **독립성**이다. 안쪽이 커밋되면 바깥이 나중에 롤백돼도 안쪽 결과는 남는다. 반대로 안쪽이 롤백돼도 바깥은 멀쩡히 진행될 수 있다(안쪽 예외를 바깥에서 잡아 준다면). 그래서 "본 작업의 성패와 무관하게 반드시 남겨야 하는 기록"에 쓴다. 감사 로그가 대표적이다.

이 코드가 하는 일: 로그를 별도 트랜잭션으로 남겨, 주문이 롤백돼도 로그는 살아남게 한다.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void log(String msg) {
    auditRepository.save(new Audit(msg));
}
```

메서드 시그니처는 그대로고 전파 속성만 바꿨다. 이제 `log()`는 `placeOrder()`와 **별개 트랜잭션**이라, 주문 쪽이 실패해도 로그는 독립적으로 커밋된 상태를 유지한다.

> 주의: `REQUIRES_NEW`는 바깥 트랜잭션을 중단하고 **커넥션(connection, DB 연결)을 하나 더** 잡는다. 한 요청이 여러 개의 `REQUIRES_NEW`를 중첩해 호출하면 커넥션 풀(pool)을 빠르게 소진해 교착에 빠질 수 있다. 남발하지 않는다.

## 가장 많이 데는 함정 — UnexpectedRollbackException

전파를 배우면 반드시 만나는 함정이 있다. **`REQUIRED`로 합류한 안쪽 메서드에서 예외가 나면, 바깥에서 그 예외를 `try-catch`로 잡아도 결국 전체가 롤백된다.** 게다가 커밋하려는 순간 엉뚱하게 `UnexpectedRollbackException`이 튀어나온다.

왜 그럴까? 안쪽 `log()`에서 예외가 터지면 Spring은 그 **물리 트랜잭션에 "이건 롤백해야 함"이라는 표식**을 단다. 이걸 **rollback-only 마킹**이라 한다. 그런데 `REQUIRED`라 안팎이 같은 트랜잭션이므로, 이 표식은 **바깥 트랜잭션에도 그대로 찍힌다.** 바깥에서 아무리 예외를 삼켜도, 트랜잭션은 이미 "롤백 전용"으로 낙인찍힌 상태다. 커밋을 시도하는 순간 Spring이 "롤백하라고 표시된 걸 커밋할 순 없다"며 `UnexpectedRollbackException`을 던진다.

이 코드가 하는 일: 안쪽 예외를 잡아 "무시"하려 하지만, 의도대로 되지 않는다.

```java
@Transactional  // REQUIRED
public void placeOrder(Order order) {
    orderRepository.save(order);
    try {
        auditService.log("주문 저장됨");  // 여기서 예외 발생 (REQUIRED)
    } catch (Exception e) {
        // 삼켜서 주문은 살리려는 의도... 하지만 소용없다
    }
}   // ← 커밋 시점: UnexpectedRollbackException 발생, 주문도 롤백
```

<!-- diagram: rollback-only-flow -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 210" role="img" aria-label="rollback-only 전파 흐름. 안쪽 예외가 물리 트랜잭션을 rollback-only로 마킹하고, 바깥이 예외를 잡아도 커밋 시점에 UnexpectedRollbackException이 발생한다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <rect x="20" y="16" width="290" height="30" rx="6" fill="none" stroke="#5F0080" stroke-width="1.6"/>
    <text x="30" y="30" fill="#5F0080" font-size="8" font-weight="700">① 바깥 placeOrder() — REQUIRED 트랜잭션 시작</text>
    <text x="30" y="41" fill="#5f5a68" font-size="7">주문 save() 정상</text>

    <path d="M165 46 L165 60" stroke="#5f5a68" stroke-width="1.4" marker-end="url(#p0)"/>

    <rect x="20" y="62" width="290" height="30" rx="6" fill="none" stroke="#b23a00" stroke-width="1.6"/>
    <text x="30" y="76" fill="#b23a00" font-size="8" font-weight="700">② 안쪽 log() 합류 → 예외 발생</text>
    <text x="30" y="87" fill="#b23a00" font-size="7">Spring이 물리 TX에 rollback-only 표식</text>

    <path d="M165 92 L165 106" stroke="#b23a00" stroke-width="1.4" marker-end="url(#p1)"/>

    <rect x="20" y="108" width="290" height="30" rx="6" fill="#fdf1eb" stroke="#b23a00" stroke-width="1.4" stroke-dasharray="4 3"/>
    <text x="30" y="122" fill="#5f5a68" font-size="8" font-weight="700">③ 바깥에서 catch로 예외 삼킴</text>
    <text x="30" y="133" fill="#b23a00" font-size="7">하지만 표식은 그대로 (같은 트랜잭션)</text>

    <path d="M165 138 L165 152" stroke="#b23a00" stroke-width="1.4" marker-end="url(#p1)"/>

    <rect x="20" y="154" width="290" height="34" rx="6" fill="#fdf1eb" stroke="#b23a00" stroke-width="2"/>
    <text x="30" y="169" fill="#b23a00" font-size="8" font-weight="700">④ 커밋 시도 → UnexpectedRollbackException</text>
    <text x="30" y="181" fill="#b23a00" font-size="7">주문도 함께 롤백됨</text>
  </g>
  <defs>
    <marker id="p0" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#5f5a68"/></marker>
    <marker id="p1" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker>
  </defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">REQUIRED에서 안쪽 예외는 트랜잭션 전체를 rollback-only로 낙인찍는다. 바깥에서 catch해도 커밋 순간 예외가 터지며 함께 롤백된다.</figcaption>
</figure>

이 함정의 해법이 바로 앞의 `REQUIRES_NEW`다. `log()`를 별도 트랜잭션으로 떼어 내면, 안쪽 롤백이 안쪽에만 갇힌다. 바깥에서 예외를 잡아 주면 주문은 정상 커밋된다. **"안쪽 실패를 바깥이 흡수하고 싶다"면 전파를 분리하라**가 핵심이다.

## 나머지 옵션 한눈에 — NESTED와 조건부 옵션들

`REQUIRED`·`REQUIRES_NEW` 외에 5개가 더 있다. 실무 빈도는 낮지만 개념은 알아 두자.

| 전파 옵션 | 진행 중 트랜잭션이 있으면 | 없으면 | 한 줄 특징 |
|---|---|---|---|
| **REQUIRED** (기본) | 합류 | 새로 생성 | 안팎이 한 몸, 운명 공유 |
| **REQUIRES_NEW** | 바깥 중단 후 새로 생성 | 새로 생성 | 완전 독립, 커넥션 추가 |
| **NESTED** | 세이브포인트(savepoint) 만들어 중첩 | 새로 생성 | 부분 롤백 가능 |
| **SUPPORTS** | 합류 | 트랜잭션 없이 실행 | 있으면 쓰고 없으면 말고 |
| **MANDATORY** | 합류 | 예외 발생 | 반드시 바깥이 있어야 함 |
| **NOT_SUPPORTED** | 바깥 중단, 트랜잭션 없이 | 트랜잭션 없이 | 트랜잭션 밖에서 실행 강제 |
| **NEVER** | 예외 발생 | 트랜잭션 없이 | 트랜잭션 있으면 거부 |

이 중 **NESTED**만 조금 더 짚는다. NESTED는 바깥 트랜잭션 안에 **세이브포인트(savepoint, 트랜잭션 도중 되돌아갈 수 있는 저장 지점)**를 만든다. 안쪽이 실패하면 **그 세이브포인트까지만** 롤백하고 바깥은 계속 진행할 수 있다 — REQUIRES_NEW와 비슷해 보이지만 결정적 차이가 있다: **바깥이 롤백되면 NESTED도 함께 롤백된다**(같은 물리 트랜잭션 안의 부분이라서). REQUIRES_NEW는 바깥과 완전히 무관하다.

주의할 점은 NESTED가 **JDBC 세이브포인트를 지원하는 환경에서만** 동작한다는 것이다. JPA(Hibernate) 위에서는 제약이 있어 실무에서 자주 쓰이진 않는다. "부분 롤백"이 필요하면 먼저 이 제약부터 확인해야 한다.

## 정리 — 전파는 "롤백 경계를 어디에 그을까"의 문제

1. **전파(propagation)** — 진행 중 트랜잭션이 있을 때, 새 `@Transactional` 메서드가 합류할지 분가할지 정하는 규칙.
2. **REQUIRED**(기본) — 안팎이 한 트랜잭션. 하나 실패하면 함께 롤백.
3. **REQUIRES_NEW** — 안쪽이 독립 트랜잭션. 바깥과 무관하게 커밋·롤백(감사 로그 등에 적합, 커넥션 추가 주의).
4. **함정** — REQUIRED에서 안쪽 예외는 트랜잭션을 rollback-only로 마킹한다. 바깥에서 catch해도 커밋 시 `UnexpectedRollbackException`.
5. **NESTED** — 세이브포인트로 부분 롤백. 단 바깥 롤백엔 함께 죽고, JDBC/JPA 지원 제약이 있다.

한 줄 교훈: **`@Transactional` 메서드가 다른 `@Transactional` 메서드를 부르는 순간, "이 둘은 같이 죽어야 하나, 따로 살아야 하나"를 먼저 정하라.** 전파를 기본값에 방치하면, 안쪽의 작은 실패가 바깥 전체를 조용히 끌고 내려간다.
