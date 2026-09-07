---
layout: post
title: 'Kafka 파티셔닝·컨슈머 그룹 — "순서와 병렬 처리를 동시에"'
date: 2026-09-07 09:00:00 +0900
image: /assets/img/kafka-partition-consumer/hero.jpg
generated: true
tags: [kafka, study]
sanitized: true
---

주문이 초당 1만 건씩 쏟아진다. 이 주문들을 처리기(consumer) 한 대가 한 줄로 처리하면 절대 못 따라간다. 그렇다고 처리기를 여러 대로 늘려 아무렇게나 나눠 먹게 하면, **같은 주문의 "결제 완료"와 "취소"가 뒤바뀐 순서로 처리**되는 사고가 난다. Kafka(카프카 — 대용량 메시지를 안정적으로 흘려보내는 분산 메시지 큐)는 이 두 요구 — **빠른 병렬 처리**와 **순서 보장** — 를 **파티션(partition)**과 **컨슈머 그룹(consumer group)**이라는 두 장치로 동시에 푼다. 이 글은 토픽이 파티션으로 어떻게 쪼개지는지, 컨슈머 그룹이 그 파티션을 어떻게 나눠 갖는지, 그리고 "왜 순서는 파티션 안에서만 지켜지는지"를 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·메시지·설정은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/kafka-partition-consumer/hero.jpg" alt="여러 우편함(파티션)에 편지가 순서대로 쌓이고 서로 다른 집배원(컨슈머)이 우편함을 하나씩 나눠 맡는 카프카 은유" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">우편함(파티션)마다 편지는 순서대로 쌓이고, 집배원(컨슈머)은 우편함을 하나씩 나눠 맡는다.</figcaption>
</figure>

## 토픽과 파티션 — 하나의 흐름을 여러 줄로 쪼갠다

**무엇**부터. Kafka에서 메시지는 **토픽(topic — 메시지를 주제별로 담는 이름 붙은 통로)**으로 흘러 들어간다. 예를 들어 `orders`라는 토픽에 주문 이벤트를 모두 보낸다.

그런데 토픽 하나를 물리적으로 **한 줄**로만 두면, 그 줄을 읽는 처리기도 한 대뿐이라 병렬 처리가 안 된다. 그래서 Kafka는 토픽을 **파티션 여러 개**로 쪼갠다. 파티션은 **메시지가 들어온 순서대로 차곡차곡 쌓이는 하나의 로그(append-only log — 뒤에만 덧붙이는 기록)**다. `orders` 토픽을 파티션 3개로 만들면, 주문 이벤트는 이 3개의 줄 중 하나에 나뉘어 쌓인다.

여기서 가장 중요한 성질 하나. **순서는 "파티션 하나 안에서만" 보장된다.** 같은 파티션에 들어간 메시지는 들어온 순서대로 읽히지만, **서로 다른 파티션 사이에는 순서 개념이 없다.** 파티션 0의 3번째 메시지와 파티션 1의 1번째 메시지 중 뭐가 먼저인지는 Kafka가 보장하지 않는다.

<!-- diagram: topic-partitions -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 210" role="img" aria-label="하나의 orders 토픽이 세 개의 파티션으로 쪼개지고 각 파티션 안에서는 메시지가 오프셋 순서대로 쌓이지만 파티션 사이에는 순서가 없음을 보여주는 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="14" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">토픽 orders = 파티션 3줄</text>
    <!-- partition 0 -->
    <text x="16" y="46" fill="#5f5a68" font-size="8">P0</text>
    <rect x="34" y="34" width="26" height="18" rx="3" fill="#f3ecf7" stroke="#7d5a9e"/><text x="47" y="46" text-anchor="middle" fill="#1a1720" font-size="7">0</text>
    <rect x="64" y="34" width="26" height="18" rx="3" fill="#f3ecf7" stroke="#7d5a9e"/><text x="77" y="46" text-anchor="middle" fill="#1a1720" font-size="7">1</text>
    <rect x="94" y="34" width="26" height="18" rx="3" fill="#f3ecf7" stroke="#7d5a9e"/><text x="107" y="46" text-anchor="middle" fill="#1a1720" font-size="7">2</text>
    <rect x="124" y="34" width="26" height="18" rx="3" fill="#fff" stroke="#b9a9cc" stroke-dasharray="2 2"/><text x="137" y="46" text-anchor="middle" fill="#5f5a68" font-size="7">3</text>
    <text x="230" y="46" fill="#5f5a68" font-size="7">순서대로 쌓임 →</text>
    <!-- partition 1 -->
    <text x="16" y="88" fill="#5f5a68" font-size="8">P1</text>
    <rect x="34" y="76" width="26" height="18" rx="3" fill="#f3ecf7" stroke="#7d5a9e"/><text x="47" y="88" text-anchor="middle" fill="#1a1720" font-size="7">0</text>
    <rect x="64" y="76" width="26" height="18" rx="3" fill="#f3ecf7" stroke="#7d5a9e"/><text x="77" y="88" text-anchor="middle" fill="#1a1720" font-size="7">1</text>
    <rect x="94" y="76" width="26" height="18" rx="3" fill="#fff" stroke="#b9a9cc" stroke-dasharray="2 2"/><text x="107" y="88" text-anchor="middle" fill="#5f5a68" font-size="7">2</text>
    <!-- partition 2 -->
    <text x="16" y="130" fill="#5f5a68" font-size="8">P2</text>
    <rect x="34" y="118" width="26" height="18" rx="3" fill="#f3ecf7" stroke="#7d5a9e"/><text x="47" y="130" text-anchor="middle" fill="#1a1720" font-size="7">0</text>
    <rect x="64" y="118" width="26" height="18" rx="3" fill="#f3ecf7" stroke="#7d5a9e"/><text x="77" y="130" text-anchor="middle" fill="#1a1720" font-size="7">1</text>
    <rect x="94" y="118" width="26" height="18" rx="3" fill="#f3ecf7" stroke="#7d5a9e"/><text x="107" y="130" text-anchor="middle" fill="#1a1720" font-size="7">2</text>
    <rect x="124" y="118" width="26" height="18" rx="3" fill="#f3ecf7" stroke="#7d5a9e"/><text x="137" y="130" text-anchor="middle" fill="#1a1720" font-size="7">3</text>
    <!-- notes -->
    <text x="165" y="168" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">파티션 안 = 순서 보장 (숫자=offset)</text>
    <text x="165" y="188" text-anchor="middle" fill="#b23a00" font-size="8" font-weight="700">파티션 사이 = 순서 없음</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">각 파티션은 오프셋(offset — 파티션 안 순번) 순서로 쌓인다. 순서 보장은 파티션 하나 안에서만 유효하다.</figcaption>
</figure>

## 메시지는 어느 파티션으로 가는가 — 키(key) 해싱

그럼 "결제 완료"와 "취소"가 뒤바뀌면 안 되는 **같은 주문의 이벤트들**은 어떻게 순서를 지킬까? 답은 간단하다. **같은 것들을 같은 파티션에 몰아넣으면** 된다.

Kafka는 메시지를 보낼 때 **키(key)**를 함께 붙일 수 있다. 프로듀서(producer — 메시지를 보내는 쪽)는 이 키를 해시(hash — 값을 고정 길이 숫자로 바꾸는 계산)한 뒤, 그 숫자를 파티션 개수로 나눈 나머지로 파티션을 정한다. 공식은 이렇다: `파티션 번호 = hash(key) % 파티션수`.

이 규칙의 핵심은 **같은 키는 항상 같은 파티션으로 간다**는 점이다. 주문 ID를 키로 쓰면, `order-42`의 모든 이벤트(생성 → 결제 → 취소)는 **전부 같은 파티션**에 순서대로 쌓인다. 파티션 안에서는 순서가 보장되니, 이 주문의 이벤트 순서는 안전하다. 반면 `order-42`와 `order-99`는 서로 다른 파티션에 흩어질 수 있어 **병렬로** 처리된다 — 순서와 병렬을 동시에 얻는 지점이 바로 여기다.

**이 코드가 하는 일:** 주문 이벤트를 보낼 때 주문 ID를 키로 지정해, 같은 주문의 이벤트가 항상 같은 파티션에 순서대로 쌓이게 한다.

```java
// 2번째 인자가 key. 같은 orderId는 항상 같은 파티션으로 간다
producer.send(new ProducerRecord<>(
    "orders",          // 토픽
    order.getId(),     // key = 주문 ID → 이 주문의 이벤트는 한 파티션에 모임
    event              // value = 실제 메시지(결제완료/취소 등)
));
```

키를 **주지 않으면**(`null`) Kafka는 메시지를 파티션에 고르게 흩뿌린다(라운드 로빈 등). 병렬 처리량은 최대지만 **순서 보장은 포기**하는 셈이다. 그래서 "순서가 중요한 단위"를 키로 잡는 설계가 핵심이다.

## 컨슈머 그룹 — 파티션을 나눠 맡는다

이제 읽는 쪽이다. **컨슈머 그룹은 "같은 일을 하는 처리기 여러 대를 하나로 묶은 팀"**이다. 같은 `group.id`를 가진 컨슈머들이 한 그룹을 이룬다. Kafka는 이 그룹에게 토픽의 파티션들을 **겹치지 않게 나눠준다.**

**규칙: 한 파티션은 그룹 안에서 딱 한 컨슈머에게만 배정된다.** 파티션이 3개이고 컨슈머가 3대면 1:1로 하나씩 맡는다. 이렇게 나눠 맡기 때문에 **각 파티션의 순서를 지키면서도 3배 병렬**로 처리된다. "결제 완료 → 취소"는 같은 파티션 = 같은 컨슈머가 순서대로 처리하니 안전하고, 서로 다른 주문은 다른 컨슈머가 동시에 처리하니 빠르다.

<!-- diagram: consumer-group-assign -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 200" role="img" aria-label="파티션 세 개를 컨슈머 그룹의 컨슈머 세 대가 하나씩 나눠 맡아 한 파티션당 한 컨슈머로 배정되는 것을 보여주는 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="14" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">파티션 3 → 컨슈머 3 (1:1)</text>
    <!-- partitions -->
    <rect x="24" y="34" width="80" height="22" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/><text x="64" y="49" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">P0</text>
    <rect x="24" y="86" width="80" height="22" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/><text x="64" y="101" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">P1</text>
    <rect x="24" y="138" width="80" height="22" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/><text x="64" y="153" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">P2</text>
    <!-- consumers -->
    <rect x="226" y="34" width="80" height="22" rx="5" fill="#eaf5ef" stroke="#1f7a4d"/><text x="266" y="49" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">컨슈머 1</text>
    <rect x="226" y="86" width="80" height="22" rx="5" fill="#eaf5ef" stroke="#1f7a4d"/><text x="266" y="101" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">컨슈머 2</text>
    <rect x="226" y="138" width="80" height="22" rx="5" fill="#eaf5ef" stroke="#1f7a4d"/><text x="266" y="153" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">컨슈머 3</text>
    <!-- links -->
    <line x1="104" y1="45" x2="226" y2="45" stroke="#1f7a4d" stroke-width="1.4"/>
    <line x1="104" y1="97" x2="226" y2="97" stroke="#1f7a4d" stroke-width="1.4"/>
    <line x1="104" y1="149" x2="226" y2="149" stroke="#1f7a4d" stroke-width="1.4"/>
    <text x="165" y="186" text-anchor="middle" fill="#5f5a68" font-size="7.5">한 파티션 = 한 컨슈머. 겹치지 않게 배정된다.</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">컨슈머 그룹은 파티션을 겹치지 않게 나눠 맡는다. 그래서 순서를 지키면서도 병렬이 된다.</figcaption>
</figure>

## 병렬성의 한계는 파티션 수 — 남는 컨슈머는 논다

여기서 초보자가 자주 놓치는 지점. **한 그룹의 병렬 처리량 상한은 파티션 개수**다. 파티션이 3개인데 컨슈머를 4대로 늘리면, 3대가 하나씩 맡고 **4번째 컨슈머는 배정받을 파티션이 없어 놀게** 된다(idle). 처리 속도가 안 오르는데 왜 그런지 몰라 헤매는 흔한 함정이다.

- **컨슈머 < 파티션**: 한 컨슈머가 파티션 여러 개를 맡는다. (예: 파티션 3, 컨슈머 2 → 한 대가 2개, 한 대가 1개)
- **컨슈머 = 파티션**: 1:1, 가장 균형 잡힌 상태.
- **컨슈머 > 파티션**: 남는 컨슈머는 논다. 늘려도 처리량이 안 는다.

그래서 "나중에 컨슈머를 몇 대까지 늘려 처리량을 키울지"를 내다보고 **파티션 수를 넉넉히** 잡는 게 실무의 기본이다. 파티션은 나중에 늘릴 수는 있지만 **줄일 수 없고**, 늘리면 `hash(key) % 파티션수`의 나머지가 바뀌어 **같은 키가 다른 파티션으로 갈 수 있다** — 즉 그 시점부터 순서 보장이 흔들릴 수 있으니 신중해야 한다.

<!-- diagram: partition-limit -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 180" role="img" aria-label="파티션이 세 개인데 컨슈머가 네 대일 때 세 대만 파티션을 맡고 네 번째 컨슈머는 놀게 되는 상황을 보여주는 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="14" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">파티션 3 &lt; 컨슈머 4 → 1대는 논다</text>
    <rect x="24" y="30" width="70" height="20" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/><text x="59" y="44" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">P0</text>
    <rect x="24" y="66" width="70" height="20" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/><text x="59" y="80" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">P1</text>
    <rect x="24" y="102" width="70" height="20" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/><text x="59" y="116" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">P2</text>
    <rect x="236" y="30" width="70" height="20" rx="4" fill="#eaf5ef" stroke="#1f7a4d"/><text x="271" y="44" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">C1</text>
    <rect x="236" y="66" width="70" height="20" rx="4" fill="#eaf5ef" stroke="#1f7a4d"/><text x="271" y="80" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">C2</text>
    <rect x="236" y="102" width="70" height="20" rx="4" fill="#eaf5ef" stroke="#1f7a4d"/><text x="271" y="116" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">C3</text>
    <rect x="236" y="138" width="70" height="20" rx="4" fill="#f7e6dc" stroke="#b23a00" stroke-dasharray="3 2"/><text x="271" y="152" text-anchor="middle" fill="#b23a00" font-size="8" font-weight="700">C4 (놀음)</text>
    <line x1="94" y1="40" x2="236" y2="40" stroke="#1f7a4d" stroke-width="1.3"/>
    <line x1="94" y1="76" x2="236" y2="76" stroke="#1f7a4d" stroke-width="1.3"/>
    <line x1="94" y1="112" x2="236" y2="112" stroke="#1f7a4d" stroke-width="1.3"/>
    <text x="271" y="172" text-anchor="middle" fill="#b23a00" font-size="7">맡을 파티션 없음</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">병렬성 상한 = 파티션 수. 컨슈머를 그보다 많이 두면 초과분은 놀게 된다.</figcaption>
</figure>

## 리밸런싱 — 멤버가 바뀌면 파티션을 다시 나눈다

컨슈머 그룹은 고정이 아니다. 컨슈머가 **새로 들어오거나(배포·스케일 아웃), 죽거나(장애), 응답이 끊기면** Kafka는 파티션 배정을 **다시 계산**한다. 이걸 **리밸런싱(rebalancing)**이라 한다.

예를 들어 컨슈머 3대(파티션 3개, 1:1)에서 컨슈머 1대가 죽으면, 그 컨슈머가 맡던 파티션이 **남은 2대에게 재배정**된다. 반대로 트래픽이 늘어 컨슈머를 추가하면(파티션 수 이내에서) 파티션이 새 멤버에게 나뉜다. 덕분에 그룹은 **자동으로 장애를 견디고(fault tolerance) 확장**된다.

주의할 점: 리밸런싱이 진행되는 **짧은 순간에는 처리가 잠시 멈춘다**(stop-the-world 형태의 지연). 컨슈머가 자주 들락거리거나 처리가 오래 걸려 "죽은 걸로 오해"받으면 리밸런싱이 반복돼 처리량이 떨어진다. 그래서 실무에서는 컨슈머가 "나 살아있다"를 주기적으로 알리는 하트비트(heartbeat) 설정과, 한 번에 가져오는 메시지 수를 처리 능력에 맞게 조절하는 것이 중요하다.

## 오프셋 커밋 — "어디까지 읽었나"를 기억한다

마지막 조각. 컨슈머는 파티션을 읽으며 **오프셋(offset — 파티션 안에서 몇 번째 메시지까지 읽었는지 가리키는 순번)**을 **커밋(commit)**해 기록한다. 이 기록 덕분에, 컨슈머가 죽었다 살아나거나 리밸런싱으로 파티션이 다른 컨슈머에게 넘어가도 **"이어서 읽기"**가 된다.

여기서 **처리와 커밋의 순서**가 사고를 가른다.

- **처리 먼저 → 커밋 나중(at-least-once, 최소 한 번):** 메시지를 처리한 뒤 오프셋을 커밋한다. 처리 후 커밋 전에 죽으면, 재시작 시 **그 메시지를 다시 처리**한다 → **중복 처리** 가능. 그래서 처리는 **멱등(idempotent — 여러 번 해도 결과가 같음)**하게 짜야 안전하다.
- **커밋 먼저 → 처리 나중(at-most-once, 최대 한 번):** 오프셋을 먼저 커밋하고 처리한다. 커밋 후 처리 전에 죽으면 그 메시지는 **영영 유실**된다.

대부분의 실무는 **at-least-once + 멱등 처리**를 택한다 — "빠뜨리는 것보다 중복이 낫고, 중복은 멱등으로 지운다"는 판단이다. (멱등성 자체는 이 블로그의 별도 글에서 다뤘다.)

## 한 줄 교훈

Kafka의 파티션과 컨슈머 그룹은 **"순서 보장"과 "병렬 처리"라는 상충하는 두 요구를 쪼개기(파티션)와 나눠 맡기(컨슈머 그룹)로 동시에 만족**시키는 설계다. 기억할 축은 셋이다: **① 순서는 파티션 하나 안에서만 지켜진다 → 순서가 중요한 단위를 키로 잡아라. ② 병렬성 상한은 파티션 수다 → 파티션을 넉넉히 잡아라. ③ 어디까지 읽었나는 오프셋 커밋이 기억한다 → at-least-once면 처리를 멱등하게 짜라.**
