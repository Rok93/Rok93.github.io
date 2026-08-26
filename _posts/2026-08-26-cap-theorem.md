---
layout: post
title: 'CAP 정리 — 분산 시스템은 왜 "셋 다"를 가질 수 없을까?'
date: 2026-08-26 09:00:00 +0900
image: /assets/img/cap-theorem/hero.jpg
generated: true
tags: [distributed, study]
sanitized: true
---

서버 한 대로 돌던 서비스가 커지면, 데이터를 여러 대에 복제(replication — 같은 데이터를 여러 노드에 나눠 보관)해 나눠 든다. 그런데 그 순간 골치 아픈 질문이 생긴다. "A 서버에 방금 쓴 값을, B 서버에서 바로 읽을 수 있을까?" 그리고 "A와 B 사이 네트워크가 끊기면, 그때도 서비스를 계속할까 아니면 멈출까?" **CAP 정리(CAP theorem)**는 이 두 질문이 사실은 하나로 얽혀 있으며, **셋 다 완벽히 가질 수는 없다**고 말한다. 이 글은 CAP의 세 글자가 각각 무엇인지, 왜 "택일"이 강제되는지, 그리고 현실에서 이걸 어떻게 저울질하는지를 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·시나리오·설정은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/cap-theorem/hero.jpg" alt="세 개의 데이터베이스 노드가 연결되어 있고 그중 한 연결이 번개 모양으로 끊긴 모습, 일관성과 가용성의 균형을 상징" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">노드 사이 연결이 끊기는 순간(P), 시스템은 일관성(C)과 가용성(A) 중 하나를 포기해야 한다.</figcaption>
</figure>

## CAP의 세 글자부터

먼저 **무엇**인지. CAP는 분산 시스템(여러 대의 컴퓨터가 협력해 하나의 서비스처럼 동작하는 시스템)이 가질 수 있는 세 가지 성질의 머리글자다.

- **C — 일관성(Consistency):** 어느 노드에 읽어도 **가장 최근에 쓴 같은 값**이 나온다. A에 `잔액=1000`을 쓰고 곧바로 B에서 읽으면 `1000`이 나와야 한다. (여기서 말하는 일관성은 데이터베이스 ACID의 C가 아니라, **모든 복제본이 같은 값을 보이는가**라는 뜻이다.)
- **A — 가용성(Availability):** **죽지 않은 모든 노드는 요청에 (에러 없이) 응답**한다. 느리더라도 "지금은 못 받아요"라며 거절하지 않는다.
- **P — 분할 내성(Partition tolerance):** 노드 사이 네트워크가 끊겨 **메시지가 오가지 못하는 상황(네트워크 분할, partition)**에서도 시스템이 **통째로 죽지 않고 계속 동작**한다.

**왜** 중요할까? 세 성질은 평소엔 사이좋게 공존하는 것처럼 보인다. 문제는 **네트워크가 끊긴 그 순간**에만 드러난다 — 그리고 분산 시스템에서 네트워크 끊김은 "혹시"가 아니라 "언제나 결국 일어나는" 일이다.

## "셋 중 둘"이 아니라, P는 사실 선택지가 아니다

CAP는 흔히 "세 개 중 두 개만 고를 수 있다"로 소개된다. 틀린 요약은 아니지만, 오해를 부른다. 실제 분산 시스템에서 **P(분할 내성)는 포기할 수 있는 항목이 아니기 때문**이다.

**왜** 그럴까? 네트워크 분할은 우리가 켜고 끄는 옵션이 아니라, 케이블 장애·스위치 재부팅·순간적 지연 등으로 **언젠가 반드시 발생하는 물리 현상**이다. P를 포기한다는 건 "네트워크는 절대 안 끊긴다고 가정한다"는 뜻인데, 그건 여러 대로 나눈 시스템에서 성립하지 않는다. (P를 버려도 되는 건 애초에 노드가 하나뿐인 시스템 — 즉 분산이 아닌 경우다.)

그래서 CAP는 실전에서 이렇게 다시 읽힌다. **"네트워크가 끊겼을 때(P는 이미 주어짐), C와 A 중 무엇을 포기할 것인가?"** 이 택일이 CAP의 진짜 알맹이다.

<!-- diagram: cap-triangle -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 210" role="img" aria-label="CAP 삼각형에서 분할 내성 P는 고정이고 일관성 C와 가용성 A 중 하나를 택해야 함을 보여주는 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="16" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">네트워크 분할이 나면 C와 A 중 택일</text>
    <!-- triangle -->
    <polygon points="165,36 60,168 270,168" fill="#f3ecf7" stroke="#7d5a9e" stroke-width="1.2"/>
    <!-- P vertex (top, fixed) -->
    <circle cx="165" cy="36" r="15" fill="#5F0080"/>
    <text x="165" y="40" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">P</text>
    <text x="165" y="30" text-anchor="middle" fill="#5F0080" font-size="7.5" font-weight="700">고정(못 버림)</text>
    <!-- C vertex -->
    <circle cx="60" cy="168" r="15" fill="#1f7a4d"/>
    <text x="60" y="172" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">C</text>
    <text x="60" y="196" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">일관성</text>
    <!-- A vertex -->
    <circle cx="270" cy="168" r="15" fill="#b23a00"/>
    <text x="270" y="172" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">A</text>
    <text x="270" y="196" text-anchor="middle" fill="#b23a00" font-size="8" font-weight="700">가용성</text>
    <!-- choose one -->
    <text x="165" y="120" text-anchor="middle" fill="#5f5a68" font-size="8">P는 늘 주어진다 →</text>
    <text x="165" y="133" text-anchor="middle" fill="#5f5a68" font-size="8">아래 둘 중 하나만</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">P는 물리적으로 강제되는 조건. 남는 선택은 CP(일관성 지킴)냐 AP(가용성 지킴)냐다.</figcaption>
</figure>

## 택일의 순간을 예시로 — 계좌 잔액

**무엇**을 고르느냐가 실제로 어떻게 갈리는지, 구체적 시나리오로 보자. 노드 두 대 `N1`, `N2`가 같은 계좌 잔액을 복제해 들고 있다. 지금 `잔액 = 1000`으로 둘이 같다. 그때 `N1`과 `N2` 사이 네트워크가 **끊겼다**(분할 발생).

이 상태에서 사용자가 `N1`에 `잔액 = 2000`으로 갱신했다. `N1`은 이 변경을 `N2`에 전파하고 싶지만, 선이 끊겨 **전달하지 못한다**. 잠시 뒤 다른 사용자가 `N2`에 잔액을 **읽으러** 온다. 시스템은 여기서 갈림길에 선다.

**선택 1 — CP (일관성 우선, 가용성 포기):** `N2`는 "나는 지금 최신 값인지 확신할 수 없다"며 **읽기를 거절하거나 대기**시킨다. 틀린 값(옛 `1000`)을 주느니, **응답하지 않는 쪽**을 택한다. 데이터는 절대 어긋나지 않지만, 그 시간 동안 `N2`는 **일부 요청에 답을 못 준다**(가용성 손실).

**선택 2 — AP (가용성 우선, 일관성 포기):** `N2`는 자기가 가진 값 `1000`을 **일단 그대로 응답**한다. 서비스는 안 멈추지만, 이 순간 사용자는 **낡은 값(stale)**을 본다 — `N1`엔 이미 `2000`인데 `N2`는 `1000`을 준다(일관성 손실). 나중에 네트워크가 복구되면 두 값을 맞춘다.

<!-- diagram: cp-vs-ap -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 260" role="img" aria-label="네트워크 분할 시 CP는 읽기를 거절하고 AP는 낡은 값을 반환하는 두 선택을 대비한 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="15" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">같은 분할, 두 가지 선택</text>
    <!-- nodes with partition -->
    <rect x="40" y="28" width="70" height="26" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="75" y="41" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">N1</text>
    <text x="75" y="50" text-anchor="middle" fill="#1f7a4d" font-size="7.5">잔액=2000(새 값)</text>
    <rect x="220" y="28" width="70" height="26" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="255" y="41" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">N2</text>
    <text x="255" y="50" text-anchor="middle" fill="#5f5a68" font-size="7.5">잔액=1000(옛 값)</text>
    <!-- broken link -->
    <line x1="110" y1="41" x2="150" y2="41" stroke="#b23a00" stroke-width="1.5"/>
    <line x1="180" y1="41" x2="220" y2="41" stroke="#b23a00" stroke-width="1.5"/>
    <text x="165" y="38" text-anchor="middle" fill="#b23a00" font-size="12" font-weight="700">⚡</text>
    <text x="165" y="53" text-anchor="middle" fill="#b23a00" font-size="7">전파 못함</text>
    <!-- CP box -->
    <rect x="24" y="78" width="130" height="80" rx="8" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="89" y="94" text-anchor="middle" fill="#1f7a4d" font-size="9" font-weight="700">CP: 일관성 지킴</text>
    <text x="89" y="112" text-anchor="middle" fill="#5f5a68" font-size="7.5">N2 → "지금 못 줍니다"</text>
    <text x="89" y="126" text-anchor="middle" fill="#5f5a68" font-size="7.5">읽기 거절/대기</text>
    <text x="89" y="144" text-anchor="middle" fill="#1f7a4d" font-size="7.5">✓ 틀린 값 없음</text>
    <text x="89" y="156" text-anchor="middle" fill="#b23a00" font-size="7.5">✗ 잠시 응답 불가</text>
    <!-- AP box -->
    <rect x="176" y="78" width="130" height="80" rx="8" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="241" y="94" text-anchor="middle" fill="#b23a00" font-size="9" font-weight="700">AP: 가용성 지킴</text>
    <text x="241" y="112" text-anchor="middle" fill="#5f5a68" font-size="7.5">N2 → "1000입니다"</text>
    <text x="241" y="126" text-anchor="middle" fill="#5f5a68" font-size="7.5">일단 응답</text>
    <text x="241" y="144" text-anchor="middle" fill="#1f7a4d" font-size="7.5">✓ 안 멈춤</text>
    <text x="241" y="156" text-anchor="middle" fill="#b23a00" font-size="7.5">✗ 낡은 값 반환</text>
    <!-- recover note -->
    <text x="165" y="182" text-anchor="middle" fill="#5f5a68" font-size="8" font-weight="700">네트워크 복구 후</text>
    <text x="165" y="197" text-anchor="middle" fill="#5f5a68" font-size="7.5">CP: 이미 맞음 · AP: 지금부터 값 동기화(수렴)</text>
    <text x="165" y="218" text-anchor="middle" fill="#5F0080" font-size="8">돈이면 CP, 좋아요 수면 AP —</text>
    <text x="165" y="231" text-anchor="middle" fill="#5F0080" font-size="8">"틀린 값의 대가"로 고른다</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">같은 분할 상황에서도 무엇을 포기하느냐로 시스템의 성격이 갈린다.</figcaption>
</figure>

## 어느 쪽을 골라야 할까 — 도메인이 정한다

CP와 AP 중 **정답은 없다.** "틀린 값을 잠깐 보여주는 비용"과 "잠깐 응답을 못 하는 비용" 중 **어느 쪽이 더 아픈지**로 고른다.

- **CP가 맞는 경우:** 값이 틀리면 안 되는 도메인. 계좌 잔액·재고 수량·좌석 예약처럼 **잘못된 값이 곧 사고**인 곳. 은행은 "잠깐 안 되는" 게 "이중 출금"보다 낫다.
- **AP가 맞는 경우:** 잠깐 낡아도 괜찮은 도메인. 게시글 좋아요 수·조회수·추천 피드처럼 **몇 초 뒤 맞으면 그만**인 곳. 쇼핑몰 상품 페이지는 "잠깐 멈춤"보다 "낡은 재고 표시 후 결제 때 재확인"이 낫다.

실제 데이터베이스도 이 성향으로 갈린다. 예를 들어 여러 노드에 걸친 강한 일관성을 우선하는 제품군은 분할 시 일부 요청을 막는 **CP 성향**을, 항상 응답을 우선하는 제품군은 낡은 값을 잠깐 허용하는 **AP 성향**을 띤다. (같은 제품도 설정으로 성향을 조절할 수 있어, "이 DB는 무조건 CP"라고 단정하긴 어렵다.)

## AP의 뒷정리 — 결과적 일관성(eventual consistency)

AP를 골랐다고 값이 **영원히** 어긋난 채 방치되는 건 아니다. AP 시스템은 대개 **결과적 일관성(eventual consistency)**을 목표로 한다. **무엇**이냐면, "새 쓰기가 멈추고 네트워크가 복구되면, 시간이 지나 **결국 모든 노드가 같은 값으로 수렴**한다"는 약속이다.

즉 AP는 "일관성을 버린다"기보다 **"지금 당장의 일관성(강한 일관성, strong consistency)을 미루고, 나중의 일관성을 택한다"**에 가깝다. 그래서 CAP의 C(강한 일관성)를 포기한 시스템도, 결과적 일관성이라는 **약한 형태의 일관성**은 유지하는 경우가 많다.

여기서 실무의 핵심 감각이 나온다. **"강하냐 없냐"의 이분법이 아니라, 일관성에는 스펙트럼이 있다** — 강한 일관성 ↔ 읽은 값이 최소한 뒤로 가진 않는 보장 ↔ 결과적 일관성. 시스템을 설계할 때 우리가 실제로 고르는 건 CAP의 극단이 아니라, 이 스펙트럼 위의 **한 점**이다.

## 한 줄 교훈

CAP는 "무엇을 못 갖느냐"의 정리가 아니라 **"네트워크가 끊긴 순간, 무엇을 포기할지 미리 정해두라"**는 설계 강제 규칙이다. P는 주어지고, 남는 건 C냐 A냐 — 그리고 그 답은 기술이 아니라 **도메인이 감당할 수 있는 대가**가 정한다.
