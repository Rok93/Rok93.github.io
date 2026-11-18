---
layout: post
title: '로그·메트릭·트레이싱 — "언제, 어디서, 왜" 를 나눠 묻는 법'
date: 2026-11-18 09:00:00 +0900
image: /assets/img/observability-pillars/hero.jpg
generated: true
tags: [observability, study]
sanitized: true
---

"주문이 좀 느린 것 같아요." 장애 문의는 대개 이렇게 막연하게 온다. 이때 바로 서버 로그를 열어 `grep`부터 하면, 수백만 줄 속에서 길을 잃기 쉽다. 사실 이 문장 안에는 질문이 세 개 숨어 있다. **언제부터, 얼마나 느린가?** **요청이 거치는 여러 서비스 중 어디가 느린가?** **그 지점에서 왜 느린가?** 관측성(observability)의 세 기둥인 **메트릭·트레이싱·로그**는 각각 이 세 질문에 답하도록 만들어진 도구다. 이 글은 셋이 무엇이고 왜 따로 필요한지 잡은 뒤, 느려진 주문 API 하나를 셋으로 차례로 좁혀 가는 과정을 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 서비스·수치·로그·쿼리는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/observability-pillars/hero.jpg" alt="어두운 서버실에서 한 손이 손전등으로 서버 랙을 비추고, 벽에는 계기판이 걸려 있으며, 바닥에는 장비 사이를 잇는 빛나는 선이 뻗어 있다" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">벽의 계기판(메트릭)으로 이상을 알아채고, 바닥의 실(트레이스)을 따라 위치를 찾고, 손전등(로그)으로 그 자리를 자세히 비춘다.</figcaption>
</figure>

## 먼저 — 모니터링과 관측성은 뭐가 다른가

**모니터링(monitoring)**은 "미리 정해 둔 질문"을 계속 확인하는 일이다. "CPU가 90%를 넘었나?", "에러율이 1%를 넘었나?" 같은 것. 알고 있는 고장(known unknowns)을 잡는 데 강하다.

**관측성(observability)**은 원래 제어 이론에서 온 말로, "**바깥으로 나오는 출력만 보고 시스템 내부 상태를 얼마나 알아낼 수 있나**"를 뜻한다. 소프트웨어에서는 "**미리 예상하지 못한 질문**에도 답할 수 있을 만큼 데이터를 남겨 두었나"로 쓰인다. 예를 들어 "특정 결제수단 + 특정 앱 버전 사용자만 느린가?" 같은 질문은 미리 대시보드를 만들어 둘 수 없다.

둘은 대립하지 않는다. 모니터링은 관측 데이터 위에서 돌아가는 "알람과 대시보드"이고, 관측성은 그 아래 데이터를 **얼마나 풍부하고 서로 연결되게** 남기느냐의 문제다. 그 데이터의 대표 형태가 세 기둥(three pillars)이다.

<!-- diagram: three-pillars-questions -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 200" role="img" aria-label="세 기둥과 각각이 답하는 질문. 메트릭은 언제, 얼마나에 답하고 숫자를 시간 순으로 집계한다. 트레이스는 어디서에 답하고 요청 하나의 경로를 기록한다. 로그는 왜에 답하고 사건 하나하나를 자세히 기록한다.">
<rect x="10" y="10" width="100" height="150" rx="8" fill="#fff" stroke="#5F0080" stroke-width="1.5"/>
<rect x="120" y="10" width="100" height="150" rx="8" fill="#fff" stroke="#7d5a9e" stroke-width="1.5"/>
<rect x="230" y="10" width="100" height="150" rx="8" fill="#fff" stroke="#b9a9cc" stroke-width="1.5"/>
<text x="60" y="32" text-anchor="middle" font-size="13" font-weight="bold" fill="#5F0080">메트릭</text>
<text x="170" y="32" text-anchor="middle" font-size="13" font-weight="bold" fill="#7d5a9e">트레이스</text>
<text x="280" y="32" text-anchor="middle" font-size="13" font-weight="bold" fill="#5f5a68">로그</text>
<text x="60" y="52" text-anchor="middle" font-size="11" fill="#1a1720">"언제·얼마나?"</text>
<text x="170" y="52" text-anchor="middle" font-size="11" fill="#1a1720">"어디서?"</text>
<text x="280" y="52" text-anchor="middle" font-size="11" fill="#1a1720">"왜?"</text>
<polyline points="22,128 36,120 50,124 64,110 78,116 92,84 100,80" fill="none" stroke="#5F0080" stroke-width="2"/>
<line x1="20" y1="138" x2="100" y2="138" stroke="#b9a9cc" stroke-width="1"/>
<rect x="132" y="76" width="76" height="9" rx="2" fill="#7d5a9e"/>
<rect x="142" y="92" width="22" height="9" rx="2" fill="#b9a9cc"/>
<rect x="168" y="108" width="38" height="9" rx="2" fill="#b23a00"/>
<rect x="172" y="124" width="30" height="9" rx="2" fill="#b9a9cc"/>
<text x="242" y="84" font-size="8" fill="#5f5a68" font-family="monospace">10:02 INFO ..</text>
<text x="242" y="98" font-size="8" fill="#5f5a68" font-family="monospace">10:02 INFO ..</text>
<text x="242" y="112" font-size="8" fill="#b23a00" font-family="monospace">10:03 WARN ..</text>
<text x="242" y="126" font-size="8" fill="#5f5a68" font-family="monospace">10:03 INFO ..</text>
<text x="242" y="140" font-size="8" fill="#5f5a68" font-family="monospace">10:03 INFO ..</text>
<text x="60" y="180" text-anchor="middle" font-size="10" fill="#5f5a68">집계된 숫자</text>
<text x="170" y="180" text-anchor="middle" font-size="10" fill="#5f5a68">요청 하나의 경로</text>
<text x="280" y="180" text-anchor="middle" font-size="10" fill="#5f5a68">사건 하나의 기록</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">세 기둥은 "같은 데이터를 다르게 본 것"이 아니라, 서로 다른 질문에 답하도록 다르게 생긴 데이터다.</figcaption>
</figure>

## 예시 무대 — 주문 서비스

무대를 하나 정한다. 사용자가 "주문하기"를 누르면 요청이 이렇게 흐른다.

- `order-api` — 주문 요청을 받는 서비스
- `inventory` — 재고를 확인·차감하는 서비스
- `payment` — 결제를 처리하는 서비스 (내부에서 외부 PG사 API 호출)
- `order-db` — 주문을 저장하는 데이터베이스

**마이크로서비스(microservice — 기능별로 쪼개 따로 배포하는 작은 서비스들)** 구조라서, 요청 하나가 서비스 4개를 거친다. 바로 이 "여러 곳을 거친다"가 세 기둥이 모두 필요한 이유다.

## 1. 로그 — 사건 하나하나의 일기장

### 무엇인가

**로그(log)**는 "어떤 시각에, 어떤 일이 있었다"를 한 줄씩 남긴 **사건 기록(event record)**이다. 가장 오래되고 가장 익숙한 기둥이다.

예전 로그는 사람이 읽기 좋은 문장이었다.

```text
2026-11-18 10:03:12 WARN 결제 지연: 주문 8812, PG 응답 2830ms
```

요즘은 **구조화 로그(structured log — 정해진 필드 이름과 값의 묶음, 주로 JSON)**를 권장한다. 아래는 위와 같은 내용을 구조화 로그로 남긴 모습이다.

```json
{"ts":"2026-11-18T10:03:12.418+09:00","level":"WARN","service":"payment",
 "msg":"pg response slow","order_id":8812,"pg":"pg-a","elapsed_ms":2830,
 "trace_id":"4bf92f3577b34da6a3ce929d0e0e4736"}
```

### 왜 구조화하나

문장형 로그로 "PG 응답이 2초 넘은 건만 찾기"를 하려면 정규식으로 숫자를 뽑아야 한다. 문장 표현이 조금만 바뀌어도 검색이 깨진다. 구조화 로그면 로그 검색 도구에서 `elapsed_ms > 2000 AND pg = "pg-a"`처럼 **필드로 바로 거를 수 있다.** 마지막 필드 `trace_id`는 뒤에서 트레이스와 로그를 잇는 열쇠가 된다.

### 강점과 약점

- **강점**: 가장 자세하다. 에러 메시지, 스택 트레이스(stack trace — 에러가 난 지점까지의 함수 호출 경로), 입력값 같은 "**왜**"의 단서가 여기 있다.
- **약점**: 양이 많다. 요청 1건에 로그 10줄, 초당 1,000건이면 **초당 1만 줄**이다. 저장·검색 비용이 크고, 그래서 "**지금 전체적으로 어떤 상태인가**"를 로그로 답하려 하면 느리고 비싸다.

## 2. 메트릭 — 시간에 따라 집계한 숫자

### 무엇인가

**메트릭(metric)**은 **숫자를 일정 간격으로 모아 시간 순으로 쌓은 것**이다. 이렇게 시간 축에 놓인 숫자열을 **시계열(time series)**이라 부른다. 사건 하나하나를 남기지 않고, "지난 15초 동안 요청 몇 건", "지금 메모리 몇 MB"처럼 **미리 합치거나 재서** 남긴다.

자주 쓰는 종류는 세 가지다. (Prometheus·OpenTelemetry 등 대부분 도구가 비슷하게 나눈다.)

| 종류 | 뜻 | 예 |
|---|---|---|
| **카운터(counter)** | 계속 늘어나기만 하는 누적 값 | 처리한 요청 수, 에러 수 |
| **게이지(gauge)** | 오르내리는 현재 값 | 메모리 사용량, 대기 중인 작업 수 |
| **히스토그램(histogram)** | 값을 구간(bucket)별로 세어 둔 분포 | 응답 시간 분포 |

### 왜 필요한가

메트릭은 **작고 싸다.** 요청이 초당 1,000건이든 10만 건이든, "요청 수" 카운터는 숫자 하나가 늘어날 뿐이다. 그래서 오래 보관하고, 대시보드로 그리고, **알람(alert)**을 걸기 좋다. "언제부터 이상해졌나", "얼마나 나쁜가"에 가장 빨리 답한다.

### 평균 말고 백분위수

응답 시간은 평균으로 보면 속는다. 요청 1,000건 중 980건이 50ms, 20건이 5,000ms라고 하자. 평균은 (980×50 + 20×5,000) / 1,000 = **149ms**로 "괜찮아" 보인다. 하지만 20명은 5초를 기다렸다.

그래서 **백분위수(percentile)**를 본다. **p99**는 "요청을 빠른 순으로 줄 세웠을 때 99% 지점의 값", 즉 "**100건 중 가장 느린 1건 바로 앞**" 수준이다. 위 예에서 p50(중앙값)은 50ms지만, 빠른 순 990번째 요청은 느린 20건 안에 들어가므로 p99는 **5,000ms**다. 평균이 감춘 꼬리 지연(tail latency)이 그대로 드러난다. 히스토그램은 이 백분위수를 계산하려고 존재한다.

아래는 Prometheus 쿼리 언어(PromQL)로 "최근 5분간 `/orders` 경로의 p99 응답 시간"을 구하는 식이다.

```promql
histogram_quantile(
  0.99,
  sum by (le) (rate(http_server_request_duration_seconds_bucket{route="/orders"}[5m]))
)
```

`rate(...[5m])`은 "5분 동안 구간별 카운트가 초당 얼마씩 늘었나", `sum by (le)`는 서버 여러 대를 구간 경계(`le`, less-or-equal)별로 합치기, `histogram_quantile(0.99, ...)`은 그 분포에서 99% 지점 추정이다. 구간 안에서는 값이 고르게 퍼져 있다고 가정해 **보간(interpolation)**하므로, 결과는 정확한 값이 아니라 **구간 경계에 좌우되는 근삿값**이다.

### 함정 — 카디널리티

메트릭에는 **레이블(label — 시계열을 나누는 꼬리표)**을 붙인다. `route="/orders"`, `status="500"` 같은 것이다. 메트릭 이름 + 레이블 값의 **조합 하나가 시계열 하나**다.

- `route` 3종 × `method` 4종 × `status` 5종 = 시계열 **60개**. 문제없다.
- 여기에 `user_id` 레이블을 붙이면? 사용자 100만 명이면 시계열이 **최대 6,000만 개**로 불어난다.

이 "레이블 값 조합의 가짓수"를 **카디널리티(cardinality)**라 부른다. 메트릭 저장소는 시계열 개수에 비례해 메모리를 쓰므로, 사용자 ID·주문 ID·요청 ID처럼 **값이 끝없이 많은 것은 메트릭 레이블에 넣지 않는다.** 그런 건 로그나 트레이스의 몫이다.

## 3. 트레이싱 — 요청 하나가 지나간 길

### 무엇인가

**분산 트레이싱(distributed tracing)**은 **요청 하나가 여러 서비스를 거치는 경로와 각 구간의 소요 시간**을 기록한다.

- **스팬(span)**: 작업 한 덩어리. "order-api가 주문을 처리함", "payment가 PG를 호출함"처럼 시작 시각·종료 시각·이름·속성을 가진다.
- **트레이스(trace)**: 한 요청에 속한 스팬들의 나무(tree). 모든 스팬이 같은 **trace_id**를 공유하고, 각 스팬은 자기를 부른 **부모 스팬 ID(parent span id)**를 기억한다.

이걸 시간 축에 펼치면 **워터폴(waterfall)** 그림이 된다.

<!-- diagram: trace-waterfall -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 200" role="img" aria-label="주문 요청 하나의 트레이스 워터폴. 전체 POST /orders 스팬은 3100ms이다. 그 아래 inventory 재고 확인 스팬 120ms, payment 결제 스팬 2900ms, 그 안의 PG사 호출 스팬 2830ms, 마지막으로 order-db INSERT 스팬 40ms가 있다. PG사 호출이 전체 시간의 대부분을 차지한다.">
<line x1="110" y1="22" x2="330" y2="22" stroke="#b9a9cc" stroke-width="1"/>
<text x="110" y="16" font-size="8" fill="#5f5a68">0ms</text>
<text x="330" y="16" text-anchor="end" font-size="8" fill="#5f5a68">3100ms</text>
<text x="8" y="42" font-size="9" fill="#1a1720">order-api</text>
<text x="8" y="52" font-size="8" fill="#5f5a68">POST /orders</text>
<rect x="110" y="36" width="220" height="12" rx="2" fill="#5F0080"/>
<text x="16" y="72" font-size="9" fill="#1a1720">inventory</text>
<rect x="114" y="64" width="9" height="12" rx="2" fill="#b9a9cc"/>
<text x="128" y="74" font-size="8" fill="#5f5a68">120ms</text>
<text x="16" y="100" font-size="9" fill="#1a1720">payment</text>
<rect x="119" y="92" width="205" height="12" rx="2" fill="#7d5a9e"/>
<text x="24" y="128" font-size="9" fill="#b23a00">PG사 호출</text>
<rect x="122" y="120" width="200" height="12" rx="2" fill="#b23a00"/>
<text x="222" y="130" text-anchor="middle" font-size="8" fill="#fff">2830ms</text>
<text x="16" y="156" font-size="9" fill="#1a1720">order-db</text>
<rect x="325" y="148" width="4" height="12" rx="1" fill="#b9a9cc"/>
<text x="322" y="158" text-anchor="end" font-size="8" fill="#5f5a68">INSERT 40ms</text>
<line x1="8" y1="174" x2="332" y2="174" stroke="#e4e0ec" stroke-width="1"/>
<text x="8" y="190" font-size="9" fill="#5f5a68">trace_id = 4bf92f35…4736 (모든 스팬이 공유)</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">가로 막대 길이가 소요 시간이다. 들여쓰기가 부모-자식 관계다. 3.1초 중 2.8초가 PG사 호출에 있다는 것이 한눈에 보인다.</figcaption>
</figure>

### 어떻게 서비스 사이를 잇나 — 컨텍스트 전파

서비스가 다르면 프로세스도, 서버도 다르다. 그런데 어떻게 같은 trace_id를 공유할까? 요청을 다음 서비스로 보낼 때 **HTTP 헤더에 trace_id를 실어 보낸다.** 이걸 **컨텍스트 전파(context propagation)**라 한다.

표준은 W3C의 **Trace Context** 규격이고, 헤더 이름은 `traceparent`다. 아래는 order-api가 payment를 부를 때 붙이는 헤더 예시다.

```http
POST /payments HTTP/1.1
Host: payment
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

`-`로 나눈 네 칸의 뜻은 이렇다.

| 칸 | 값 | 뜻 |
|---|---|---|
| version | `00` | 규격 버전 |
| trace-id | `4bf92f35…4736` (16바이트, 16진수 32자) | 이 요청 전체의 ID |
| parent-id | `00f067aa0ba902b7` (8바이트, 16진수 16자) | 이 요청을 보낸 **호출자 쪽 스팬** ID |
| trace-flags | `01` | 샘플링 여부 (`01` = 기록 대상) |

payment는 이 헤더를 읽어 **같은 trace-id로** 자기 스팬을 만들고, 받은 parent-id를 부모로 기록한다. 이걸 모든 서비스가 지키면 흩어진 스팬들이 하나의 나무로 모인다. 직접 구현하기보다 **OpenTelemetry**(로그·메트릭·트레이스 수집 방식을 통일한 오픈소스 표준과 SDK 모음) 같은 라이브러리가 HTTP 클라이언트·서버에 자동으로 붙여 주는 경우가 많다.

### 샘플링 — 다 남기면 비싸다

요청마다 스팬 수십 개를 남기면 로그만큼 양이 많아진다. 그래서 일부만 남기는 **샘플링(sampling)**을 한다.

- **헤드 기반(head-based)**: 요청이 **시작될 때** "이번 건은 기록할까?"를 정한다(예: 10%). 단순하고 싸다. 대신 결정 시점엔 이 요청이 느릴지 에러일지 모르므로, **정작 보고 싶은 느린 요청을 놓칠 수 있다.** 위 헤더의 `trace-flags`가 이 결정을 다음 서비스로 전달한다.
- **테일 기반(tail-based)**: 요청이 **끝난 뒤** 전체 스팬을 보고 정한다("에러거나 1초 넘으면 무조건 보관, 나머지는 1%"). 중요한 건 놓치지 않지만, 결정 전까지 스팬을 수집기에 **모아 두어야 해서** 메모리와 운영 부담이 크다.

## 셋을 잇는 열쇠 — trace_id

세 기둥이 따로 놀면 결국 사람이 시각을 맞춰 가며 눈으로 대조해야 한다. 그래서 **연결 고리**를 심는다.

- **로그 ↔ 트레이스**: 로그 한 줄마다 `trace_id`를 넣는다(위 JSON 로그의 마지막 필드). 트레이스에서 문제 스팬을 찾으면, 그 trace_id로 **그 요청의 로그만** 정확히 뽑을 수 있다. Java라면 보통 **MDC(Mapped Diagnostic Context — 현재 스레드에 꼬리표를 달아 두면 로그마다 자동으로 찍어 주는 기능)**에 trace_id를 넣어 처리한다.
- **메트릭 ↔ 트레이스**: 메트릭 자체엔 trace_id를 레이블로 넣지 않는다(카디널리티 폭발). 대신 **엑셈플러(exemplar — 히스토그램 구간에 "이 구간에 해당한 대표 요청 하나의 trace_id"를 샘플로 붙여 두는 기능)**를 쓰면, 그래프의 튀는 점을 눌러 해당 트레이스로 바로 건너갈 수 있다.

<!-- diagram: drill-down-flow -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 190" role="img" aria-label="조사 흐름. 1단계 메트릭 알람으로 언제부터 얼마나 나빠졌는지 안다. 엑셈플러를 통해 2단계 트레이스로 가서 어느 구간이 느린지 찾는다. trace_id로 3단계 로그를 검색해 왜 느린지 원인을 찾는다. 범위는 전체 요청에서 요청 한 건, 그리고 그 요청의 로그 몇 줄로 좁아진다.">
<defs>
<marker id="obs-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5F0080"/></marker>
</defs>
<rect x="10" y="20" width="90" height="70" rx="8" fill="#fff" stroke="#5F0080" stroke-width="1.5"/>
<text x="55" y="42" text-anchor="middle" font-size="12" font-weight="bold" fill="#5F0080">① 메트릭</text>
<text x="55" y="60" text-anchor="middle" font-size="9" fill="#1a1720">p99 3초로 급등</text>
<text x="55" y="76" text-anchor="middle" font-size="9" fill="#5f5a68">언제·얼마나</text>
<rect x="125" y="20" width="90" height="70" rx="8" fill="#fff" stroke="#7d5a9e" stroke-width="1.5"/>
<text x="170" y="42" text-anchor="middle" font-size="12" font-weight="bold" fill="#7d5a9e">② 트레이스</text>
<text x="170" y="60" text-anchor="middle" font-size="9" fill="#1a1720">PG 호출 2.8초</text>
<text x="170" y="76" text-anchor="middle" font-size="9" fill="#5f5a68">어디서</text>
<rect x="240" y="20" width="90" height="70" rx="8" fill="#fff" stroke="#b9a9cc" stroke-width="1.5"/>
<text x="285" y="42" text-anchor="middle" font-size="12" font-weight="bold" fill="#5f5a68">③ 로그</text>
<text x="285" y="60" text-anchor="middle" font-size="9" fill="#b23a00">재시도 3회</text>
<text x="285" y="76" text-anchor="middle" font-size="9" fill="#5f5a68">왜</text>
<line x1="100" y1="55" x2="123" y2="55" stroke="#5F0080" stroke-width="1.5" marker-end="url(#obs-arrow)"/>
<line x1="215" y1="55" x2="238" y2="55" stroke="#5F0080" stroke-width="1.5" marker-end="url(#obs-arrow)"/>
<text x="112" y="108" text-anchor="middle" font-size="8" fill="#5F0080">exemplar</text>
<text x="227" y="108" text-anchor="middle" font-size="8" fill="#5F0080">trace_id</text>
<polygon points="10,130 330,130 300,160 40,160" fill="#efe9f5"/>
<text x="55" y="149" text-anchor="middle" font-size="9" fill="#1a1720">요청 전체</text>
<text x="170" y="149" text-anchor="middle" font-size="9" fill="#1a1720">요청 1건</text>
<text x="285" y="149" text-anchor="middle" font-size="9" fill="#1a1720">로그 몇 줄</text>
<text x="170" y="180" text-anchor="middle" font-size="9" fill="#5f5a68">→ 넓게 보고, 점점 좁혀 간다</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">조사 순서는 대개 넓은 것(메트릭)에서 좁은 것(로그)으로 간다. 각 단계를 건너가는 다리가 exemplar와 trace_id다.</figcaption>
</figure>

## 상세 예시 — 느려진 주문 API를 셋으로 좁히기

이제 처음의 "주문이 좀 느린 것 같아요"를 실제로 따라가 본다.

### ① 메트릭: 언제부터, 얼마나?

알람이 먼저 울린다. 알람 규칙은 "`/orders`의 p99가 5분 넘게 1초를 넘으면 알림"이다. 대시보드에서 p50·p99를 나란히 본다.

| 시각 | 요청 수(/s) | p50 | p99 | 에러율 |
|---|---|---|---|---|
| 09:55 | 120 | 85ms | 310ms | 0.1% |
| 10:00 | 125 | 88ms | 340ms | 0.1% |
| **10:05** | 122 | 92ms | **3,050ms** | 0.2% |
| 10:10 | 118 | 95ms | **3,120ms** | 0.3% |

여기서 얻는 정보:
- **10:00~10:05 사이**에 시작됐다.
- **p50은 거의 그대로**이고 **p99만 10배** 뛰었다. → "모든 요청이 느린" 게 아니라 "**일부 요청만** 아주 느리다."
- 요청 수는 그대로다. → 트래픽 폭증 때문은 아니다.

메트릭은 여기까지다. "**어느 서비스**가 느린지"는 숫자만으로는 모른다. 주문 API가 느린 건지, 그 뒤의 재고·결제·DB가 느린 건지 구분이 안 된다.

### ② 트레이스: 어디서?

10:05 이후의 트레이스 중 **소요 시간 2초 이상**인 것만 걸러 하나를 연다(엑셈플러가 있으면 그래프 점을 눌러 바로 간다). 앞의 워터폴 그림이 그 결과다.

- 전체 3,100ms 중 **payment → PG사 호출 스팬이 2,830ms**.
- inventory(120ms), order-db(40ms)는 평소 수준.

느린 트레이스 몇 개를 더 열어 보니 모두 같은 패턴이고, 스팬 속성을 보면 공통적으로 `pg = "pg-a"`다. 다른 PG(`pg-b`)를 쓴 요청은 빠르다. 이제 범위가 "**pg-a를 호출하는 구간**"으로 좁혀졌다. p99만 튄 이유도 설명된다. 전체 결제 중 pg-a를 쓰는 비율이 일부라서다.

### ③ 로그: 왜?

트레이스의 trace_id로 로그를 검색한다. 이 한 요청의 로그만 시간 순으로 나온다.

```text
10:06:41.102 INFO  payment  pg request sent         pg=pg-a order_id=9134 attempt=1
10:06:42.105 WARN  payment  pg timeout, retrying    pg=pg-a order_id=9134 attempt=1 timeout_ms=1000
10:06:43.108 WARN  payment  pg timeout, retrying    pg=pg-a order_id=9134 attempt=2 timeout_ms=1000
10:06:43.930 INFO  payment  pg response ok          pg=pg-a order_id=9134 attempt=3 elapsed_ms=822
```

원인이 보인다. pg-a가 평소보다 느려져 **1초 타임아웃(timeout — 이 시간 안에 응답이 없으면 포기)**에 두 번 걸렸고, 클라이언트가 **재시도(retry)**를 거듭해 세 번째에 성공했다. 1,000 + 1,000 + 822 ≈ **2.8초**. 트레이스의 2,830ms와 맞아떨어진다.

로그가 없었다면 트레이스만으로는 "PG 호출이 2.8초 걸렸다"까지만 알고, **그 안에서 재시도가 세 번 있었다**는 사실은 몰랐을 것이다(재시도마다 자식 스팬을 따로 남기도록 계측했다면 트레이스에서도 보였을 것이다 — 계측을 얼마나 촘촘히 하느냐의 문제다).

### 정리 — 각 기둥이 맡은 몫

| 단계 | 쓴 기둥 | 알아낸 것 | 그 기둥만으론 몰랐던 것 |
|---|---|---|---|
| ① | 메트릭 | 10:05부터, p99만 10배, 트래픽은 정상 | 어느 서비스인지 |
| ② | 트레이스 | payment의 pg-a 호출 구간 | 그 구간 안에서 무슨 일이 있었는지 |
| ③ | 로그 | 타임아웃 2회 + 재시도 | (여기서 원인 확정) |

## 셋을 한 장으로

| | 메트릭 | 트레이스 | 로그 |
|---|---|---|---|
| 단위 | 집계된 숫자(시계열) | 요청 1건의 스팬 나무 | 사건 1건 |
| 답하는 질문 | 언제·얼마나? | 어디서? | 왜? |
| 비용 | 낮음 (요청 수와 거의 무관) | 중간 (샘플링으로 조절) | 높음 (요청 수에 비례) |
| 잘 하는 일 | 알람, 추세, 대시보드 | 서비스 간 병목 찾기 | 원인·맥락·에러 상세 |
| 조심할 것 | 높은 카디널리티 레이블 | 샘플링에 느린 요청이 빠지는 것 | 양·비용, 비구조화 문장 |
| 잇는 고리 | exemplar → trace_id | trace_id | 로그마다 trace_id |

참고로 "세 기둥"은 정해진 표준이라기보다 널리 퍼진 **분류 방식**이다. 최근에는 **프로파일링(profiling — 코드의 어느 함수가 CPU·메모리를 얼마나 쓰는지 측정)**을 네 번째 신호로 꼽기도 하고, "기둥을 따로 세우기보다 하나의 풍부한 이벤트에서 셋을 뽑아내자"는 관점도 있다. 어느 쪽이든 핵심은 같다. **서로 다른 질문에 답하는 데이터를, 서로 건너갈 수 있게 남겨 두는 것.**

## 한 줄 교훈

**메트릭으로 "언제·얼마나"를 알아채고, 트레이스로 "어디서"를 좁히고, 로그로 "왜"를 확인한다** — 그리고 이 세 단계를 건너가는 다리는 모든 데이터에 심어 둔 trace_id 하나다.
