---
layout: post
title: '멱등성 설계 — "한 번 눌러도, 열 번 눌러도 결과는 하나"'
date: 2026-08-31 09:00:00 +0900
image: /assets/img/idempotency/hero.jpg
generated: true
tags: [distributed, study]
sanitized: true
---

결제 버튼을 눌렀는데 화면이 멈춘다. 정말 결제된 건지 알 수 없어 한 번 더 누른다. 잠시 뒤 문자가 두 통 온다 — **같은 금액이 두 번 빠져나갔다.** 분산 시스템(여러 대의 컴퓨터가 협력해 하나의 서비스처럼 동작하는 시스템)에서 이런 일은 버그가 아니라 **기본값에 가깝다.** 네트워크는 끊기고, 응답은 사라지고, 클라이언트는 재시도(retry — 실패한 것 같으면 같은 요청을 다시 보냄)하기 때문이다. **멱등성(idempotency)**은 이 혼란 속에서 "**같은 요청이 몇 번 도착하든 결과는 딱 한 번 처리한 것과 같게** 만든다"는 설계 원칙이다. 이 글은 멱등성이 무엇인지, 왜 재시도가 있는 세상에서 필수인지, 그리고 멱등 키(idempotency key)로 이걸 어떻게 구현하는지를 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·시나리오·코드는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/idempotency/hero.jpg" alt="같은 결제 버튼을 여러 번 누른 여러 손가락이 하나의 영수증으로 수렴하는 모습, 중복 요청이 하나의 결과로 합쳐지는 멱등성 은유" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">여러 번 도착한 같은 요청도, 결과는 하나로 수렴한다 — 이것이 멱등성이다.</figcaption>
</figure>

## 멱등성이란 무엇인가

**무엇**인지부터. 멱등성은 원래 수학 용어다. 어떤 연산 `f`를 **한 번 적용한 것과 여러 번 적용한 것의 결과가 같으면**, 그 연산을 멱등(idempotent)하다고 한다. 식으로는 `f(f(x)) = f(x)`.

일상 비유가 정확하게 맞는다. **엘리베이터 버튼**을 생각해 보자. 5층 버튼을 한 번 누르든 다섯 번 누르든, 엘리베이터는 5층에 **한 번** 온다. 버튼을 더 눌러도 상태(요청된 층)는 바뀌지 않는다. 반대로 **"물 한 잔 따르기"**는 멱등하지 않다 — 다섯 번 하면 물이 다섯 잔 생긴다.

소프트웨어에서 멱등성은 이렇게 다시 쓰인다. **같은 요청을 몇 번 보내든, 서버의 최종 상태와 응답이 한 번 보낸 것과 같아야 한다.** "잔액을 2000으로 **설정**하라"는 몇 번을 해도 잔액은 2000이다(멱등). 하지만 "잔액에서 1000을 **차감**하라"는 두 번 실행되면 2000이 빠진다(멱등 아님). 이 차이가 이 글의 핵심이다.

<!-- diagram: idempotent-vs-not -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 220" role="img" aria-label="같은 요청을 세 번 보냈을 때 멱등 연산은 결과가 하나이고 비멱등 연산은 세 번 누적되는 것을 대비한 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="15" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">같은 요청 3번 — 결과는?</text>
    <!-- idempotent side -->
    <rect x="18" y="30" width="140" height="170" rx="8" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="88" y="46" text-anchor="middle" fill="#1f7a4d" font-size="9" font-weight="700">멱등: "잔액=2000 설정"</text>
    <text x="88" y="66" text-anchor="middle" fill="#5f5a68" font-size="7.5">요청 ①②③ →</text>
    <rect x="43" y="76" width="90" height="20" rx="4" fill="#fff" stroke="#1f7a4d"/>
    <text x="88" y="90" text-anchor="middle" fill="#1a1720" font-size="8">set 2000</text>
    <rect x="43" y="100" width="90" height="20" rx="4" fill="#fff" stroke="#1f7a4d"/>
    <text x="88" y="114" text-anchor="middle" fill="#1a1720" font-size="8">set 2000</text>
    <rect x="43" y="124" width="90" height="20" rx="4" fill="#fff" stroke="#1f7a4d"/>
    <text x="88" y="138" text-anchor="middle" fill="#1a1720" font-size="8">set 2000</text>
    <text x="88" y="170" text-anchor="middle" fill="#1f7a4d" font-size="11" font-weight="700">잔액 = 2000</text>
    <text x="88" y="188" text-anchor="middle" fill="#1f7a4d" font-size="7.5">✓ 안전</text>
    <!-- non-idempotent side -->
    <rect x="172" y="30" width="140" height="170" rx="8" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="242" y="46" text-anchor="middle" fill="#b23a00" font-size="9" font-weight="700">비멱등: "1000 차감"</text>
    <text x="242" y="66" text-anchor="middle" fill="#5f5a68" font-size="7.5">요청 ①②③ →</text>
    <rect x="197" y="76" width="90" height="20" rx="4" fill="#fff" stroke="#b23a00"/>
    <text x="242" y="90" text-anchor="middle" fill="#1a1720" font-size="8">-1000</text>
    <rect x="197" y="100" width="90" height="20" rx="4" fill="#fff" stroke="#b23a00"/>
    <text x="242" y="114" text-anchor="middle" fill="#1a1720" font-size="8">-1000</text>
    <rect x="197" y="124" width="90" height="20" rx="4" fill="#fff" stroke="#b23a00"/>
    <text x="242" y="138" text-anchor="middle" fill="#1a1720" font-size="8">-1000</text>
    <text x="242" y="170" text-anchor="middle" fill="#b23a00" font-size="11" font-weight="700">3000 빠짐</text>
    <text x="242" y="188" text-anchor="middle" fill="#b23a00" font-size="7.5">✗ 중복 사고</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">"설정"은 몇 번 해도 같고, "차감"은 횟수만큼 누적된다. 멱등성은 연산의 성질이다.</figcaption>
</figure>

## 왜 필요한가 — 재시도는 피할 수 없다

멱등성이 없어도 클라이언트가 요청을 딱 한 번만 보낸다면 문제가 없다. **왜** 현실은 그렇지 못할까? 핵심은 **"응답이 안 왔다"와 "처리가 안 됐다"를 구분할 수 없다**는 데 있다.

클라이언트가 결제 요청을 서버에 보냈다고 하자. 응답이 오지 않는다. 여기엔 두 가지 가능성이 있다.

1. 요청이 **서버에 닿기도 전에** 네트워크에서 사라졌다 → 결제는 **안 됐다**.
2. 서버는 결제를 **정상 처리했는데**, 그 응답이 돌아오는 길에 사라졌다 → 결제는 **됐다**.

클라이언트 입장에선 이 둘이 **완전히 똑같이 보인다.** "타임아웃(timeout — 정해진 시간 안에 응답이 없음)"만 남는다. 안전하게 하려면 재시도할 수밖에 없는데, 만약 2번 상황이었다면 재시도는 곧 **이중 결제**다. 재시도를 안 하면 1번 상황에서 결제 누락이다. **재시도는 없앨 수 없고, 그래서 서버가 중복 요청을 흡수해야 한다** — 이것이 멱등성이 필요한 이유다.

<!-- diagram: retry-ambiguity -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 205" role="img" aria-label="응답이 사라졌을 때 요청 유실인지 응답 유실인지 클라이언트가 구분하지 못해 재시도가 이중 처리로 이어지는 흐름 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="14" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">타임아웃 — 무엇이 사라졌나?</text>
    <!-- client / server labels -->
    <rect x="20" y="26" width="56" height="20" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="48" y="40" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">클라이언트</text>
    <rect x="254" y="26" width="56" height="20" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="282" y="40" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">서버</text>
    <!-- case 1: request lost -->
    <text x="20" y="64" fill="#b23a00" font-size="8" font-weight="700">경우1: 요청이 유실</text>
    <line x1="48" y1="70" x2="180" y2="82" stroke="#b23a00" stroke-width="1.3" stroke-dasharray="3 3"/>
    <text x="150" y="76" fill="#b23a00" font-size="9">✕</text>
    <text x="205" y="80" fill="#5f5a68" font-size="7.5">서버 처리 안 함</text>
    <!-- case 2: response lost -->
    <text x="20" y="104" fill="#b23a00" font-size="8" font-weight="700">경우2: 응답이 유실</text>
    <line x1="48" y1="110" x2="282" y2="110" stroke="#1f7a4d" stroke-width="1.3"/>
    <text x="165" y="107" text-anchor="middle" fill="#1f7a4d" font-size="7">요청 도착 → 처리됨</text>
    <line x1="282" y1="120" x2="150" y2="132" stroke="#b23a00" stroke-width="1.3" stroke-dasharray="3 3"/>
    <text x="150" y="128" fill="#b23a00" font-size="9">✕</text>
    <text x="90" y="140" fill="#5f5a68" font-size="7.5">클라이언트는 결과 모름</text>
    <!-- conclusion -->
    <rect x="24" y="152" width="282" height="42" rx="6" fill="#fff" stroke="#7d5a9e"/>
    <text x="165" y="168" text-anchor="middle" fill="#1a1720" font-size="8">둘이 똑같아 보임 → 클라이언트는 <tspan fill="#5F0080" font-weight="700">재시도</tspan></text>
    <text x="165" y="184" text-anchor="middle" fill="#b23a00" font-size="8">경우2에서 재시도 = <tspan font-weight="700">이중 처리</tspan></text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">"응답 없음"은 요청 유실과 응답 유실을 구분해주지 않는다. 그래서 중복은 서버가 막아야 한다.</figcaption>
</figure>

## HTTP 메서드는 이미 멱등성을 나눠 놨다

사실 이 개념은 웹의 기본 규약에 이미 들어 있다. HTTP 메서드(GET·POST·PUT·DELETE 등 — 요청의 종류를 나타내는 동사)는 **멱등한 것과 아닌 것**으로 나뉜다.

- **GET / PUT / DELETE — 멱등하다.** `GET`은 읽기라 상태를 안 바꾼다. `PUT /users/1 {name:"철수"}`는 몇 번 보내도 1번 유저 이름이 "철수"로 **설정**될 뿐이다. `DELETE /users/1`도 두 번째부터는 "이미 없음"이라 최종 상태(없음)가 같다.
- **POST — 멱등하지 않다.** `POST /orders`는 부를 때마다 **새 주문을 하나씩 생성**한다. 두 번 부르면 주문이 둘 생긴다.

여기서 실무의 함정이 나온다. **중복 위험이 가장 큰 "생성·결제" 요청이 하필 비멱등한 POST**라는 점이다. 그래서 멱등성 설계는 대부분 "**POST를 어떻게 멱등하게 만들 것인가**"로 귀결된다. 답이 다음 절의 멱등 키다.

## 핵심 기법 — 멱등 키(idempotency key)

**무엇**이냐면, 클라이언트가 요청마다 **고유한 식별자**를 하나 붙여 보내고, 서버는 **그 키를 이미 처리했는지 기억**하는 방식이다. 같은 키가 다시 오면 새로 처리하지 않고 **처음 결과를 그대로 돌려준다.**

동작 순서는 이렇다.

1. 클라이언트가 요청 전에 고유 키를 하나 만든다(보통 UUID — 사실상 겹치지 않는 랜덤 식별자). 예: `Idempotency-Key: a1b2c3-...`
2. **중요:** 재시도할 때는 **새 키를 만들지 않고 처음 만든 키를 그대로** 다시 보낸다. (그래야 서버가 "아, 아까 그거"라고 알아본다.)
3. 서버는 키를 저장소에서 찾는다. **없으면** 처리하고 `(키 → 결과)`를 저장한 뒤 응답한다. **있으면** 처리를 건너뛰고 저장된 결과를 응답한다.

아래는 이 로직을 옮긴 예시다. **이 코드가 하는 일:** 들어온 멱등 키가 이미 있으면 저장된 결과를 반환하고, 처음이면 결제를 실행한 뒤 결과를 키와 함께 저장한다.

```java
public PaymentResult pay(String idempotencyKey, PayRequest req) {
    // 1) 이미 처리한 키인가?
    Optional<PaymentResult> saved = store.find(idempotencyKey);
    if (saved.isPresent()) {
        return saved.get();           // 재실행 없이 처음 결과 그대로
    }

    // 2) 처음 보는 키 → 실제 결제 실행
    PaymentResult result = charge(req.getAmount());

    // 3) 키와 결과를 함께 저장 (다음 재시도 대비)
    store.save(idempotencyKey, result);
    return result;
}
```

이렇게 하면 클라이언트가 같은 키로 열 번을 보내도 `charge()`는 **딱 한 번**만 실행된다. 나머지 아홉 번은 저장된 결과를 되돌려줄 뿐이다.

<!-- diagram: idempotency-key-flow -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 210" role="img" aria-label="멱등 키를 저장소에서 조회해 처음이면 처리 후 저장하고 재요청이면 저장된 결과를 반환하는 분기 흐름 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="14" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">멱등 키 처리 흐름</text>
    <!-- incoming -->
    <rect x="115" y="26" width="100" height="24" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="165" y="42" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">요청 + 키 도착</text>
    <line x1="165" y1="50" x2="165" y2="64" stroke="#7d5a9e" stroke-width="1.2"/>
    <!-- decision -->
    <polygon points="165,64 225,86 165,108 105,86" fill="#fff" stroke="#7d5a9e" stroke-width="1.2"/>
    <text x="165" y="83" text-anchor="middle" fill="#1a1720" font-size="7.5">저장소에</text>
    <text x="165" y="93" text-anchor="middle" fill="#1a1720" font-size="7.5">키 있나?</text>
    <!-- no branch (left) -->
    <line x1="105" y1="86" x2="60" y2="86" stroke="#1f7a4d" stroke-width="1.2"/>
    <text x="82" y="82" text-anchor="middle" fill="#1f7a4d" font-size="7" font-weight="700">없음</text>
    <rect x="12" y="112" width="96" height="58" rx="6" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="60" y="128" text-anchor="middle" fill="#1f7a4d" font-size="7.5" font-weight="700">처음 요청</text>
    <text x="60" y="142" text-anchor="middle" fill="#5f5a68" font-size="7">① 결제 실행</text>
    <text x="60" y="154" text-anchor="middle" fill="#5f5a68" font-size="7">② 키+결과 저장</text>
    <text x="60" y="166" text-anchor="middle" fill="#5f5a68" font-size="7">③ 결과 응답</text>
    <line x1="60" y1="108" x2="60" y2="112" stroke="#1f7a4d" stroke-width="1.2"/>
    <line x1="60" y1="86" x2="60" y2="108" stroke="#1f7a4d" stroke-width="1.2"/>
    <!-- yes branch (right) -->
    <line x1="225" y1="86" x2="270" y2="86" stroke="#b23a00" stroke-width="1.2"/>
    <text x="248" y="82" text-anchor="middle" fill="#b23a00" font-size="7" font-weight="700">있음</text>
    <line x1="270" y1="86" x2="270" y2="112" stroke="#b23a00" stroke-width="1.2"/>
    <rect x="222" y="112" width="96" height="58" rx="6" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="270" y="130" text-anchor="middle" fill="#b23a00" font-size="7.5" font-weight="700">재요청(중복)</text>
    <text x="270" y="146" text-anchor="middle" fill="#5f5a68" font-size="7">결제 건너뜀</text>
    <text x="270" y="158" text-anchor="middle" fill="#5f5a68" font-size="7">저장된 결과 응답</text>
    <!-- bottom note -->
    <text x="165" y="190" text-anchor="middle" fill="#5F0080" font-size="8">어느 쪽이든 결제는 딱 한 번</text>
    <text x="165" y="203" text-anchor="middle" fill="#5f5a68" font-size="7.5">응답은 항상 같은 결과</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">키가 처음이면 처리 후 저장, 다시 오면 저장된 결과 반환. 실제 처리는 언제나 한 번뿐이다.</figcaption>
</figure>

## 놓치기 쉬운 함정 — 동시성과 저장의 원자성

위 코드는 개념은 맞지만, **동시에 같은 키가 두 번 들이닥치면** 구멍이 난다. 사용자가 버튼을 빠르게 두 번 눌러 요청 A와 B가 **거의 같은 순간** 서버에 도착했다고 하자.

- A가 `store.find()` → 없음
- B가 `store.find()` → (A가 아직 저장 전이라) **역시 없음**
- A가 `charge()` 실행, B도 `charge()` 실행 → **이중 결제**

문제의 뿌리는 "**조회 → 처리 → 저장**"이 하나로 묶이지 않은 것(원자성 부재)이다. 실무에선 보통 이렇게 막는다.

- **DB 유니크 제약(unique constraint):** 멱등 키 컬럼에 "중복 불가" 제약을 걸어, 두 번째 삽입(insert)이 **DB 차원에서 실패**하게 한다. 실패한 쪽은 처리를 멈추고 첫 요청의 결과를 조회해 돌려준다.
- **먼저 선점(claim) 후 처리:** `charge()` 전에 키를 "처리 중" 상태로 **먼저 저장**해 자리를 잡는다. 이미 있으면 다른 요청이 잡은 것이므로 대기하거나 기존 결과를 반환한다.

즉 멱등 키의 진짜 힘은 "저장했다"가 아니라 **"한 키에 대해 처리는 오직 하나만 통과한다"를 저장소가 원자적으로 보장**하는 데서 나온다. 이 보장이 없으면 멱등 키는 순간적인 동시 요청 앞에서 무너진다.

## 한 줄 교훈

멱등성은 "재시도를 없애는 기술"이 아니라 **"재시도가 있어도 안전하도록 서버를 만드는 설계"**다. 네트워크는 반드시 끊기고 클라이언트는 반드시 다시 보낸다 — 그러니 **중복은 막는 게 아니라 흡수하는 것**이고, 그 핵심 도구가 "같은 키는 한 번만 통과"시키는 멱등 키다.
