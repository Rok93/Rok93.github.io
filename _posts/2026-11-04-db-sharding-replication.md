---
layout: post
title: 'DB 샤딩과 복제 — 한 대로 버티던 데이터베이스를 여러 대로 늘리는 두 가지 방법'
date: 2026-11-04 09:00:00 +0900
image: /assets/img/db-sharding-replication/hero.jpg
generated: true
tags: [study, database]
sanitized: true
---

서비스가 커지면 데이터베이스 한 대가 제일 먼저 비명을 지른다. 읽기 요청이 몰려 CPU가 꽉 차거나, 데이터가 디스크 한 장에 안 들어가거나, 한 대가 죽으면 서비스 전체가 멈춘다. 이 세 문제를 풀려고 DB를 여러 대로 늘리는 방법이 크게 두 가지다. **복제(replication)** 는 *같은 데이터를 여러 대에 똑같이 베껴두는 것*이고, **샤딩(sharding)** 은 *데이터를 조각내서 서로 다른 대에 나눠 담는 것*이다. 둘은 해결하는 문제가 다르고, 실제로는 거의 항상 같이 쓴다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·테이블·쿼리는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/db-sharding-replication/hero.jpg" alt="큰 서랍장 하나가 여러 개의 작은 서랍장으로 나뉘고, 각 서랍장 옆에 똑같은 복사본이 거울처럼 서 있다" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">샤딩은 서랍장을 "나누는" 것, 복제는 서랍장을 "베끼는" 것. 실무에선 나눈 조각마다 복사본을 둔다.</figcaption>
</figure>

## 먼저 — 한 대가 모자란 이유는 세 가지다

DB 한 대로 안 되는 순간은 보통 다음 중 하나다. 어떤 문제냐에 따라 처방이 다르니 먼저 구분해야 한다.

- **읽기 부하**: 같은 데이터를 너무 많이 읽는다. 상품 목록, 피드처럼 쓰기는 적고 읽기는 폭발하는 경우. → **복제**가 답이다. 읽기를 사본들에 나눠 주면 된다.
- **가용성(availability, 장애가 나도 서비스가 계속 되는 성질)**: 한 대가 죽으면 전부 멈춘다. → 역시 **복제**다. 사본이 있으면 그중 하나를 대신 세운다.
- **데이터 크기·쓰기 부하**: 데이터가 수십 TB라 한 디스크에 안 들어가거나, 쓰기 자체가 한 대의 한계를 넘는다. → **샤딩**이 답이다. 복제는 사본마다 *전부* 쓰기를 반복해야 해서 쓰기 부하를 줄여주지 못한다.

<!-- diagram: replication-vs-sharding -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:520px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 380 190" role="img" aria-label="복제는 같은 데이터 전체를 세 대에 똑같이 두고, 샤딩은 데이터를 세 조각으로 나눠 세 대에 하나씩 둔다">
  <g font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" font-size="9">
    <!-- 복제 -->
    <text x="12" y="16" fill="#5F0080" font-weight="700" font-size="10">복제 (replication)</text>
    <text x="12" y="30" fill="#5f5a68" font-size="8">같은 데이터를 똑같이 베낀다</text>
    <g>
      <rect x="12" y="42" width="48" height="56" rx="6" fill="#fff" stroke="#5F0080" stroke-width="1.5"/>
      <text x="36" y="58" text-anchor="middle" fill="#1a1720">A B C</text>
      <text x="36" y="72" text-anchor="middle" fill="#1a1720">D E F</text>
      <text x="36" y="90" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">주(primary)</text>
      <rect x="72" y="42" width="48" height="56" rx="6" fill="#fff" stroke="#b9a9cc" stroke-width="1.5"/>
      <text x="96" y="58" text-anchor="middle" fill="#1a1720">A B C</text>
      <text x="96" y="72" text-anchor="middle" fill="#1a1720">D E F</text>
      <text x="96" y="90" text-anchor="middle" fill="#7d5a9e" font-size="8">복제본</text>
      <rect x="132" y="42" width="48" height="56" rx="6" fill="#fff" stroke="#b9a9cc" stroke-width="1.5"/>
      <text x="156" y="58" text-anchor="middle" fill="#1a1720">A B C</text>
      <text x="156" y="72" text-anchor="middle" fill="#1a1720">D E F</text>
      <text x="156" y="90" text-anchor="middle" fill="#7d5a9e" font-size="8">복제본</text>
      <path d="M60 70 L72 70" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#arr)"/>
      <path d="M120 70 L132 70" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#arr)"/>
    </g>
    <text x="12" y="118" fill="#1f7a4d" font-size="8">+ 읽기 분산, 장애 대비</text>
    <text x="12" y="130" fill="#b23a00" font-size="8">– 쓰기·용량은 그대로</text>
    <!-- 구분선 -->
    <line x1="195" y1="8" x2="195" y2="180" stroke="#e4e0ec" stroke-width="1"/>
    <!-- 샤딩 -->
    <text x="208" y="16" fill="#5F0080" font-weight="700" font-size="10">샤딩 (sharding)</text>
    <text x="208" y="30" fill="#5f5a68" font-size="8">데이터를 조각내 나눠 담는다</text>
    <g>
      <rect x="208" y="42" width="48" height="56" rx="6" fill="#fff" stroke="#5F0080" stroke-width="1.5"/>
      <text x="232" y="66" text-anchor="middle" fill="#1a1720">A B</text>
      <text x="232" y="90" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">샤드 0</text>
      <rect x="268" y="42" width="48" height="56" rx="6" fill="#fff" stroke="#5F0080" stroke-width="1.5"/>
      <text x="292" y="66" text-anchor="middle" fill="#1a1720">C D</text>
      <text x="292" y="90" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">샤드 1</text>
      <rect x="328" y="42" width="48" height="56" rx="6" fill="#fff" stroke="#5F0080" stroke-width="1.5"/>
      <text x="352" y="66" text-anchor="middle" fill="#1a1720">E F</text>
      <text x="352" y="90" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">샤드 2</text>
    </g>
    <text x="208" y="118" fill="#1f7a4d" font-size="8">+ 쓰기·용량 분산</text>
    <text x="208" y="130" fill="#b23a00" font-size="8">– 조각 넘는 조회·트랜잭션이 어려움</text>
    <text x="12" y="166" fill="#5f5a68" font-size="8">실무: 샤드 0·1·2 각각에 복제본을 붙인다 (둘을 겹쳐 쓴다)</text>
  </g>
  <defs><marker id="arr" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 z" fill="#7d5a9e"/></marker></defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">복제는 "전부를 여러 번", 샤딩은 "일부를 한 번". 전자는 읽기와 가용성, 후자는 쓰기와 용량을 푼다.</figcaption>
</figure>

## 복제 — 같은 데이터를 여러 대에 베끼기

### 무엇인가

복제는 한 대(**주 서버**, primary 또는 source. 예전엔 master라 불렀다)에 들어온 쓰기를 다른 대들(**복제본**, replica)에 그대로 전달해 같은 상태를 유지하는 것이다. 쓰기는 주 서버 한 곳에서만 받고, 읽기는 복제본들로 분산한다. 이 구조를 **단일 리더 복제(single-leader replication)** 라 부르고, MySQL·PostgreSQL의 기본 복제가 이 방식이다.

전달하는 내용은 "실행된 SQL 문"이 아니라 보통 **변경 로그**다. MySQL은 **바이너리 로그(binlog, 행 단위 변경 기록)**, PostgreSQL은 **WAL(Write-Ahead Log, 디스크에 반영하기 전에 먼저 적는 변경 일지)** 을 복제본에 흘려 보낸다. SQL 문을 그대로 보내면 `NOW()` 같은 함수가 서버마다 다른 값을 내서 사본이 어긋나기 때문에, 행 단위 결과를 보내는 쪽이 안전하다.

### 동기냐 비동기냐 — 복제의 핵심 선택

주 서버가 쓰기를 받고 클라이언트에 "완료"라고 답하는 시점이 언제냐가 복제의 가장 중요한 갈림길이다.

- **비동기(asynchronous)**: 주 서버가 자기 디스크에 쓰자마자 "완료"라고 답하고, 복제본엔 나중에 보낸다. 빠르지만, 주 서버가 그 사이에 죽으면 **아직 전달 안 된 쓰기는 사라진다**. MySQL·PostgreSQL의 기본값이다.
- **동기(synchronous)**: 복제본이 받았다고 확인한 뒤에야 "완료"라고 답한다. 데이터 유실이 없지만, 복제본이 느리거나 죽으면 주 서버의 쓰기도 같이 멈춘다.
- **반동기(semi-synchronous)**: 복제본 **여러 대 중 최소 N대**(보통 1대)가 받았다고 확인하면 답한다. 유실 방지와 속도의 타협안이다. MySQL의 반동기 복제 플러그인, PostgreSQL의 `synchronous_standby_names` 설정이 이 방식이다.

<!-- diagram: sync-vs-async -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:520px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 380 180" role="img" aria-label="비동기 복제는 주 서버가 쓰자마자 완료를 답하고, 동기 복제는 복제본의 확인을 기다린 뒤 완료를 답한다">
  <g font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" font-size="9">
    <text x="12" y="16" fill="#5F0080" font-weight="700" font-size="10">비동기 (기본값)</text>
    <text x="40" y="36" fill="#5f5a68" font-size="8">클라이언트</text>
    <text x="120" y="36" fill="#5f5a68" font-size="8">주 서버</text>
    <text x="190" y="36" fill="#5f5a68" font-size="8">복제본</text>
    <line x1="58" y1="42" x2="58" y2="90" stroke="#b9a9cc" stroke-dasharray="2 2"/>
    <line x1="134" y1="42" x2="134" y2="90" stroke="#b9a9cc" stroke-dasharray="2 2"/>
    <line x1="204" y1="42" x2="204" y2="90" stroke="#b9a9cc" stroke-dasharray="2 2"/>
    <path d="M58 50 L132 50" stroke="#1a1720" stroke-width="1.2" marker-end="url(#arr2)"/>
    <text x="80" y="47" fill="#1a1720" font-size="7.5">쓰기</text>
    <path d="M134 62 L60 62" stroke="#1f7a4d" stroke-width="1.5" marker-end="url(#arrg)"/>
    <text x="78" y="72" fill="#1f7a4d" font-size="7.5" font-weight="700">완료! (빠름)</text>
    <path d="M134 78 L202 78" stroke="#7d5a9e" stroke-width="1.2" stroke-dasharray="3 2" marker-end="url(#arr2)"/>
    <text x="144" y="88" fill="#b23a00" font-size="7.5">나중에 전달 → 이 사이 죽으면 유실</text>
    <line x1="12" y1="100" x2="368" y2="100" stroke="#e4e0ec"/>
    <text x="12" y="116" fill="#5F0080" font-weight="700" font-size="10">동기</text>
    <line x1="58" y1="122" x2="58" y2="172" stroke="#b9a9cc" stroke-dasharray="2 2"/>
    <line x1="134" y1="122" x2="134" y2="172" stroke="#b9a9cc" stroke-dasharray="2 2"/>
    <line x1="204" y1="122" x2="204" y2="172" stroke="#b9a9cc" stroke-dasharray="2 2"/>
    <path d="M58 130 L132 130" stroke="#1a1720" stroke-width="1.2" marker-end="url(#arr2)"/>
    <text x="80" y="127" fill="#1a1720" font-size="7.5">쓰기</text>
    <path d="M134 142 L202 142" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#arr2)"/>
    <text x="150" y="139" fill="#7d5a9e" font-size="7.5">전달</text>
    <path d="M204 154 L136 154" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#arr2)"/>
    <text x="150" y="164" fill="#7d5a9e" font-size="7.5">받았음</text>
    <path d="M134 166 L60 166" stroke="#1f7a4d" stroke-width="1.5" marker-end="url(#arrg)"/>
    <text x="62" y="176" fill="#1f7a4d" font-size="7.5" font-weight="700">완료 (유실 없음, 느림)</text>
    <text x="236" y="140" fill="#5f5a68" font-size="7.5">반동기 = 여러 복제본 중</text>
    <text x="236" y="152" fill="#5f5a68" font-size="7.5">1대만 "받았음"이면 완료</text>
  </g>
  <defs>
    <marker id="arr2" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 z" fill="#7d5a9e"/></marker>
    <marker id="arrg" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 z" fill="#1f7a4d"/></marker>
  </defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">"완료"를 언제 말하느냐의 차이. 비동기는 빠르지만 유실 가능, 동기는 안전하지만 복제본에 발목 잡힌다.</figcaption>
</figure>

### 복제 지연 — 복제가 만드는 가장 흔한 버그

비동기 복제에서는 주 서버와 복제본 사이에 항상 약간의 시차가 있다. 이를 **복제 지연(replication lag)** 이라 한다. 평소엔 밀리초 단위지만, 대량 업데이트나 복제본의 느린 디스크 때문에 몇 초에서 몇 분까지 벌어질 수 있다.

이 시차가 사용자 눈에 띄는 대표 장면이 있다. 사용자가 닉네임을 바꾸고(주 서버에 쓰기) 바로 내 프로필을 조회했는데(복제본에서 읽기) **옛 닉네임이 보인다**. "방금 저장했는데 왜 안 바뀌어요?"라는 문의가 바로 이것이다. 자기가 쓴 것은 자기가 바로 읽을 수 있어야 한다는 보장을 **쓰기 후 읽기 일관성(read-your-writes consistency)** 이라 하는데, 비동기 복제는 이걸 공짜로 주지 않는다.

흔한 해결책은 다음과 같다.

- **자기 데이터는 주 서버에서 읽기**: "내 프로필"처럼 본인이 방금 수정했을 법한 데이터는 복제본 대신 주 서버로 보낸다.
- **쓰기 직후 일정 시간은 주 서버로**: 쓰기 시각을 세션에 기억해 두고, 그 뒤 몇 초간은 모든 읽기를 주 서버로 보낸다.
- **복제본이 특정 위치까지 따라왔는지 확인**: 쓰기가 남긴 로그 위치(MySQL의 GTID, PostgreSQL의 LSN)를 기억했다가, 복제본이 그 위치를 지났을 때만 거기서 읽는다.

### 장애 조치(failover) — 주 서버가 죽으면

복제의 두 번째 목적인 가용성은 **장애 조치(failover, 주 서버가 죽었을 때 복제본 하나를 새 주 서버로 올리는 것)** 로 실현된다. 말은 간단하지만 함정이 둘 있다.

- **유실된 쓰기**: 비동기였다면 옛 주 서버가 복제본에 미처 보내지 못한 쓰기가 있다. 새 주 서버는 그 쓰기를 모른 채 시작한다.
- **스플릿 브레인(split brain, 두뇌 분열)**: 옛 주 서버가 사실 죽은 게 아니라 네트워크만 잠깐 끊긴 거라면, 복구되는 순간 **주 서버가 두 대**가 된다. 양쪽이 각자 쓰기를 받으면 데이터가 갈라진다. 그래서 장애 조치 도구는 옛 주 서버를 확실히 격리(펜싱, fencing)하고 나서 새 주 서버를 올린다.

이런 판단을 자동으로 안전하게 하려면 "누가 리더인가"에 대한 합의가 필요한데, 그 문제를 푸는 알고리즘이 Raft다. 이 블로그의 Raft 시리즈가 그 내용이다.

## 샤딩 — 데이터를 조각내 나눠 담기

### 무엇인가

샤딩은 하나의 큰 테이블을 **샤드 키(shard key, 어느 조각에 담을지 정하는 기준 컬럼)** 로 여러 조각(샤드)으로 나누고, 각 조각을 다른 서버에 두는 것이다. 수평 분할(horizontal partitioning)이라고도 한다. 사용자 테이블을 `user_id` 기준으로 4조각 내면, 각 서버는 전체의 1/4만 들고 있으니 쓰기 부하와 용량이 1/4로 준다.

핵심 질문은 하나다. **"이 행은 어느 샤드에 가는가?"** 이 규칙을 정하는 방식이 세 가지다.

| 방식 | 규칙 | 장점 | 단점 |
|---|---|---|---|
| **범위(range)** | `user_id 1~100만 → 샤드 0, 100만~200만 → 샤드 1` | 범위 조회가 한 샤드 안에서 끝난다 | 최근 가입자가 몰리는 마지막 샤드만 뜨거워진다(핫스팟) |
| **해시(hash)** | `hash(user_id) % 샤드 수` | 고르게 흩어진다 | 범위 조회가 모든 샤드로 퍼진다. 샤드 수 바꾸면 거의 전부 재배치 |
| **디렉터리(directory)** | 별도 조회 테이블에 `user_id → 샤드` 매핑을 둔다 | 규칙이 유연하고 이동이 쉽다 | 조회 테이블이 단일 장애점·병목이 된다 |

해시 방식의 "샤드 수를 바꾸면 거의 전부 재배치" 문제를 줄이는 기법이 **일관된 해싱(consistent hashing)** 이다. 이 블로그의 직전 글(일관된 해싱)에서 다뤘다. 샤드를 하나 추가해도 옮겨야 할 데이터가 전체의 1/N 수준으로 준다.

### 샤딩이 어려운 진짜 이유

샤딩은 "나누기"는 쉽고 **"나눈 뒤"가 어렵다**. 한 대일 때 공짜였던 것들이 사라지기 때문이다.

- **샤드를 넘는 조인(cross-shard join)**: 주문 테이블은 `user_id`로, 상품 테이블은 `product_id`로 샤딩했다면, "이 상품을 산 사용자 목록"은 여러 샤드를 다 뒤져 애플리케이션에서 합쳐야 한다. 그래서 설계 단계에서 **함께 조회되는 데이터는 같은 샤드 키로 묶는다**(예: 주문도 `user_id`로 샤딩).
- **샤드를 넘는 트랜잭션**: 샤드 0의 A 계좌에서 샤드 1의 B 계좌로 송금하면, 두 서버에 걸친 트랜잭션이 된다. 한쪽만 성공하면 돈이 증발한다. 2단계 커밋(2PC) 같은 분산 트랜잭션이 필요한데, 느리고 복잡하다. 많은 서비스는 이걸 피하려고 **샤드 안에서 끝나게 키를 설계**하거나, 사가(saga) 패턴으로 보상 트랜잭션을 둔다.
- **유일 ID**: `AUTO_INCREMENT`는 서버마다 따로 돌아 샤드 0과 샤드 1이 둘 다 ID 1을 만든다. 샤드 번호를 ID에 섞거나, 시간+서버번호+순번을 조합한 Snowflake 방식 ID를 쓴다.
- **재분배(rebalancing)**: 샤드 하나가 유독 뜨거워지거나 서버를 늘릴 때, 서비스를 멈추지 않고 데이터를 옮겨야 한다. 이게 샤딩 운영에서 가장 손이 많이 가는 작업이다.

<!-- diagram: shard-routing -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:520px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 380 170" role="img" aria-label="애플리케이션이 user_id를 해시해 샤드를 고르고, 각 샤드는 주 서버 하나와 복제본 둘로 이뤄져 있다">
  <g font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" font-size="9">
    <rect x="12" y="60" width="78" height="44" rx="8" fill="#fff" stroke="#5F0080" stroke-width="1.5"/>
    <text x="51" y="78" text-anchor="middle" fill="#1a1720">앱 / 라우터</text>
    <text x="51" y="92" text-anchor="middle" fill="#5f5a68" font-size="7.5">hash(user_id)%3</text>
    <!-- 화살표 -->
    <path d="M90 70 L140 30" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#arr3)"/>
    <path d="M90 82 L140 82" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#arr3)"/>
    <path d="M90 94 L140 134" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#arr3)"/>
    <!-- 샤드 0 -->
    <rect x="144" y="12" width="224" height="40" rx="8" fill="#fff" stroke="#e4e0ec"/>
    <text x="152" y="26" fill="#5F0080" font-weight="700" font-size="8">샤드 0</text>
    <rect x="200" y="18" width="46" height="28" rx="5" fill="#fff" stroke="#5F0080" stroke-width="1.5"/>
    <text x="223" y="36" text-anchor="middle" fill="#5F0080" font-size="8">주(쓰기)</text>
    <rect x="256" y="18" width="46" height="28" rx="5" fill="#fff" stroke="#b9a9cc" stroke-width="1.2"/>
    <text x="279" y="36" text-anchor="middle" fill="#7d5a9e" font-size="8">복제본</text>
    <rect x="312" y="18" width="46" height="28" rx="5" fill="#fff" stroke="#b9a9cc" stroke-width="1.2"/>
    <text x="335" y="36" text-anchor="middle" fill="#7d5a9e" font-size="8">복제본</text>
    <!-- 샤드 1 -->
    <rect x="144" y="62" width="224" height="40" rx="8" fill="#fff" stroke="#e4e0ec"/>
    <text x="152" y="76" fill="#5F0080" font-weight="700" font-size="8">샤드 1</text>
    <rect x="200" y="68" width="46" height="28" rx="5" fill="#fff" stroke="#5F0080" stroke-width="1.5"/>
    <text x="223" y="86" text-anchor="middle" fill="#5F0080" font-size="8">주(쓰기)</text>
    <rect x="256" y="68" width="46" height="28" rx="5" fill="#fff" stroke="#b9a9cc" stroke-width="1.2"/>
    <text x="279" y="86" text-anchor="middle" fill="#7d5a9e" font-size="8">복제본</text>
    <rect x="312" y="68" width="46" height="28" rx="5" fill="#fff" stroke="#b9a9cc" stroke-width="1.2"/>
    <text x="335" y="86" text-anchor="middle" fill="#7d5a9e" font-size="8">복제본</text>
    <!-- 샤드 2 -->
    <rect x="144" y="112" width="224" height="40" rx="8" fill="#fff" stroke="#e4e0ec"/>
    <text x="152" y="126" fill="#5F0080" font-weight="700" font-size="8">샤드 2</text>
    <rect x="200" y="118" width="46" height="28" rx="5" fill="#fff" stroke="#5F0080" stroke-width="1.5"/>
    <text x="223" y="136" text-anchor="middle" fill="#5F0080" font-size="8">주(쓰기)</text>
    <rect x="256" y="118" width="46" height="28" rx="5" fill="#fff" stroke="#b9a9cc" stroke-width="1.2"/>
    <text x="279" y="136" text-anchor="middle" fill="#7d5a9e" font-size="8">복제본</text>
    <rect x="312" y="118" width="46" height="28" rx="5" fill="#fff" stroke="#b9a9cc" stroke-width="1.2"/>
    <text x="335" y="136" text-anchor="middle" fill="#7d5a9e" font-size="8">복제본</text>
    <text x="12" y="164" fill="#5f5a68" font-size="7.5">샤딩(가로로 나눔) × 복제(세로로 베낌) = 실무의 표준 구성</text>
  </g>
  <defs><marker id="arr3" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 z" fill="#7d5a9e"/></marker></defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">라우터가 샤드 키로 조각을 고르고, 각 조각 안에서는 쓰기는 주 서버·읽기는 복제본으로 간다.</figcaption>
</figure>

## 상세 예시 — 주문 서비스를 한 대에서 "3샤드 × 복제"로

가상의 주문 서비스를 따라가 본다. 처음엔 MySQL 한 대였다.

### 1단계: 복제본 추가로 읽기 분산

먼저 복제본 하나를 붙인다. 아래는 주 서버에 바이너리 로그를 켜고 복제본을 연결하는 핵심 설정이다.

주 서버 설정이 하는 일: 행 단위 바이너리 로그를 켜고 서버마다 다른 번호를 준다.

```ini
# 주 서버 my.cnf
[mysqld]
server_id        = 1
log_bin          = mysql-bin
binlog_format    = ROW        # SQL 문이 아니라 행의 변경 결과를 기록
gtid_mode        = ON         # 트랜잭션마다 전역 고유 ID를 붙여 위치 추적을 쉽게
enforce_gtid_consistency = ON
```

복제본에서 실행하는 명령이 하는 일: 주 서버를 가리키고 복제를 시작한다.

```sql
-- 복제본에서 (server_id = 2)
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'primary.internal',
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = '...',
  SOURCE_AUTO_POSITION = 1;   -- GTID 기반으로 "어디까지 받았는지" 자동 추적
START REPLICA;
```

복제가 잘 따라오는지 보는 명령이 하는 일: 복제 스레드 상태와 지연 초를 보여준다.

```sql
SHOW REPLICA STATUS\G
-- Replica_IO_Running:  Yes   ← 주 서버에서 로그를 받는 중
-- Replica_SQL_Running: Yes   ← 받은 로그를 적용하는 중
-- Seconds_Behind_Source: 0   ← 복제 지연(초). 대량 UPDATE 후엔 수십 초로 튈 수 있다
```

`Seconds_Behind_Source`가 핵심 지표다. 이 값이 커지면 앞서 말한 "방금 저장했는데 안 보여요"가 터진다. 그래서 애플리케이션은 읽기 경로를 둘로 나눈다.

이 코드가 하는 일: 쓰기와 "내 것 읽기"는 주 서버로, 그 외 읽기는 복제본으로 보낸다.

```python
def get_order(order_id, *, just_written=False):
    db = primary if just_written else replica   # 방금 쓴 건 주 서버에서 읽는다
    return db.query("SELECT * FROM orders WHERE id = %s", order_id)

def create_order(user_id, items):
    order_id = primary.execute("INSERT INTO orders ...")
    return get_order(order_id, just_written=True)  # 복제 지연에 안 걸린다
```

### 2단계: 쓰기가 한계에 닿아 샤딩

주문이 하루 수천만 건으로 늘어 주 서버 한 대의 쓰기가 한계에 닿았다. 복제본을 더 붙여도 쓰기는 안 줄어드니 샤딩한다.

샤드 키는 `user_id`로 정했다. 이유는 두 가지다. 주문 조회의 대부분이 "내 주문 목록"처럼 사용자 단위이고, 주문 상세(order_items)도 같은 키로 샤딩하면 **조인이 한 샤드 안에서 끝난다**. 반면 `order_id`로 샤딩하면 "내 주문 목록"이 매번 모든 샤드로 퍼진다.

이 코드가 하는 일: `user_id`를 해시해 샤드를 고르고, 그 샤드의 주/복제본으로 쿼리를 보낸다.

```python
import zlib

SHARDS = [shard0, shard1, shard2]     # 각각 {primary, replica} 묶음

def shard_for(user_id):
    return SHARDS[zlib.crc32(str(user_id).encode()) % len(SHARDS)]

def list_my_orders(user_id):
    return shard_for(user_id).replica.query(
        "SELECT * FROM orders WHERE user_id = %s ORDER BY id DESC", user_id)

def create_order(user_id, items):
    s = shard_for(user_id)
    with s.primary.transaction():          # orders + order_items가 같은 샤드 → 한 트랜잭션으로 끝
        order_id = s.primary.execute("INSERT INTO orders ...")
        s.primary.execute("INSERT INTO order_items ...")
    return order_id
```

여기서 `AUTO_INCREMENT`를 그대로 쓰면 세 샤드가 모두 ID 1부터 시작해 충돌한다. 이 예시는 "시간(41비트) + 샤드 번호(10비트) + 순번(12비트)"을 합친 64비트 ID를 애플리케이션에서 만든다. 샤드 번호가 ID에 박혀 있어, 나중에 ID만 보고도 어느 샤드인지 알 수 있다는 덤이 있다.

### 전후 비교

| 항목 | 한 대 | 복제 추가 | 3샤드 × 복제 |
|---|---|---|---|
| 읽기 처리량 | 1× | 약 3× (복제본 2대 기준) | 약 9× |
| 쓰기 처리량 | 1× | **1× (그대로)** | 약 3× |
| 주 서버 장애 | 전체 중단 | 복제본 승격으로 복구 | 해당 샤드만 영향, 승격으로 복구 |
| "내 주문" 조회 | 한 쿼리 | 한 쿼리 | 한 쿼리 (샤드 키 덕분) |
| "이 상품 산 사람들" | 한 쿼리 | 한 쿼리 | **3개 샤드 조회 후 합치기** |
| 송금처럼 두 사용자 걸친 트랜잭션 | 한 트랜잭션 | 한 트랜잭션 | **분산 트랜잭션 필요** |

표의 마지막 두 줄이 샤딩의 대가다. 그래서 샤딩은 "쓰기가 진짜 한계에 닿았을 때" 마지막에 꺼내는 카드이고, 그 전까지는 복제·캐시·인덱스·수직 확장(더 큰 서버)으로 버티는 게 일반적인 순서다.

## 복제·샤딩이 아닌 선택지도 있다

두 기법을 직접 운영하는 부담 때문에, 요즘은 이걸 DB가 내부에서 해 주는 제품을 쓰기도 한다. **Vitess**(MySQL 위에 샤딩 계층을 얹음), **Citus**(PostgreSQL 확장으로 샤딩), **CockroachDB·TiDB·Spanner**(처음부터 분산을 전제로 만든 DB, "NewSQL"이라 부른다)가 그렇다. 이들은 샤드를 넘는 트랜잭션과 재분배를 대신 처리해 주지만, 내부에서는 결국 이 글의 복제와 샤딩을 하고 있다. 원리를 알아야 그 제품의 한계도 읽힌다.

## 한 줄 교훈

**복제는 "같은 것을 여러 번"이라 읽기와 가용성을 주고, 샤딩은 "다른 것을 나눠서"라 쓰기와 용량을 준다.** 복제의 대가는 지연과 장애 조치의 복잡함, 샤딩의 대가는 조각을 넘는 조회·트랜잭션이다. 샤드 키는 "무엇과 함께 조회되느냐"로 정하고, 샤딩은 복제·캐시·인덱스로 버틸 수 없을 때 꺼낸다.
