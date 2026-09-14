---
layout: post
title: 'Redis 자료구조 — "값만 넣는 캐시가 아니다"'
date: 2026-09-14 09:00:00 +0900
image: /assets/img/redis-data-structures/hero.jpg
generated: true
tags: [redis, study]
sanitized: true
---

Redis(레디스 — 데이터를 디스크가 아닌 메모리에 두고 다루는 초고속 저장소)를 처음 쓰면 대개 이렇게만 쓴다. "키 하나에 값 하나 넣고, 나중에 꺼낸다." 캐시(cache — 느린 원본 대신 빠르게 꺼내려고 잠깐 담아두는 임시 저장) 용도로는 그걸로 충분해 보인다. 그런데 조회수 랭킹, 최근 본 상품 목록, 대기열, 태그 교집합 같은 걸 만들려는 순간 벽에 부딪힌다. "이걸 문자열 하나로 어떻게 다루지?" 답은 **Redis가 문자열 말고도 여러 자료구조(data structure — 데이터를 담는 형태·규칙)를 기본 제공한다**는 데 있다. 리스트, 해시, 집합, 정렬 집합… 이 글은 Redis의 대표 자료구조 5가지가 각각 **무엇이고, 언제 쓰며, 어떤 명령으로 다루는지**를 예시와 함께 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·키 이름·명령 예시는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/redis-data-structures/hero.jpg" alt="리스트·해시·정렬 집합·집합을 상징하는 여러 종류의 서랍과 선반이 정리된 도구함, 빠른 메모리를 뜻하는 번개 아이콘" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">Redis는 값 하나 넣는 상자가 아니라, 용도별 서랍이 갖춰진 도구함이다.</figcaption>
</figure>

## 왜 자료구조가 중요한가

**무엇**부터. 자료구조란 데이터를 "어떤 형태로, 어떤 규칙으로" 담느냐다. 같은 데이터라도 형태를 잘 고르면 **원하는 연산이 훨씬 빨라진다.**

예를 들어 "실시간 랭킹 100위"를 뽑는다고 하자. 만약 점수를 그냥 문자열 여러 개로 흩어 저장했다면, 순위를 매길 때마다 **전부 꺼내 정렬**해야 한다. 데이터가 많아질수록 느려진다. 반면 Redis의 정렬 집합(Sorted Set)은 **넣을 때부터 점수 순으로 정돈**해 둔다. 그래서 "상위 100개"를 거의 즉시 꺼낸다.

즉 Redis 자료구조 선택은 곧 **성능과 코드 단순성**의 선택이다. 애플리케이션에서 직접 짜야 할 로직(정렬·중복 제거·순서 유지)을 **Redis가 이미 최적화된 형태로 대신 해준다.**

<!-- diagram: string-vs-structures -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 205" role="img" aria-label="왼쪽은 문자열 하나로만 다뤄 애플리케이션이 직접 정렬·중복제거를 해야 하는 방식, 오른쪽은 Redis 자료구조가 형태별로 그 일을 대신 해주는 방식을 대비한 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="14" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">문자열만 쓸 때 vs 자료구조를 쓸 때</text>
    <!-- left: string only -->
    <rect x="16" y="28" width="140" height="150" rx="8" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="86" y="44" text-anchor="middle" fill="#b23a00" font-size="9" font-weight="700">문자열 하나로</text>
    <rect x="30" y="54" width="112" height="16" rx="3" fill="#fff" stroke="#b23a00"/>
    <text x="86" y="65" text-anchor="middle" fill="#1a1720" font-size="7">"user:1=10,user:2=30…"</text>
    <text x="86" y="88" text-anchor="middle" fill="#5f5a68" font-size="7.5">순위 뽑으려면?</text>
    <text x="86" y="104" text-anchor="middle" fill="#b23a00" font-size="7.5">① 전부 꺼내서</text>
    <text x="86" y="118" text-anchor="middle" fill="#b23a00" font-size="7.5">② 앱에서 파싱하고</text>
    <text x="86" y="132" text-anchor="middle" fill="#b23a00" font-size="7.5">③ 앱에서 정렬</text>
    <text x="86" y="158" text-anchor="middle" fill="#b23a00" font-size="8" font-weight="700">데이터 늘면 느려짐</text>
    <!-- right: structures -->
    <rect x="174" y="28" width="140" height="150" rx="8" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="244" y="44" text-anchor="middle" fill="#1f7a4d" font-size="9" font-weight="700">정렬 집합으로</text>
    <rect x="188" y="54" width="112" height="16" rx="3" fill="#fff" stroke="#1f7a4d"/>
    <text x="244" y="65" text-anchor="middle" fill="#1a1720" font-size="7">넣을 때부터 점수순 정돈</text>
    <text x="244" y="88" text-anchor="middle" fill="#5f5a68" font-size="7.5">순위 뽑으려면?</text>
    <text x="244" y="108" text-anchor="middle" fill="#1f7a4d" font-size="7.5">ZREVRANGE 한 줄</text>
    <text x="244" y="132" text-anchor="middle" fill="#5f5a68" font-size="7.5">Redis가 정렬을 이미 해둠</text>
    <text x="244" y="158" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">상위 N개 거의 즉시</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">형태를 잘 고르면 앱이 하던 일을 Redis가 대신 해준다.</figcaption>
</figure>

## 1) String — 값 하나, 그러나 숫자 연산도 된다

**무엇**: 가장 기본. 키 하나에 값 하나(문자열 또는 숫자)를 담는다. **왜**: 단순 캐시, 세션 토큰 저장, 그리고 **원자적 카운터**에 쓴다. 여기서 "원자적(atomic — 중간에 끼어들 틈 없이 한 번에)"이 핵심이다. 여러 서버가 동시에 조회수를 올려도 값이 꼬이지 않는다.

아래 코드는 상품 조회수를 1씩 올리고, 24시간 뒤 자동 삭제되는 임시 값을 넣는다.

```bash
# 조회수 +1 (없으면 0에서 시작해 1로). 여러 서버가 동시에 해도 안전
INCR product:1001:views        # → 1
INCR product:1001:views        # → 2

# 값 저장 + 만료시간 60초(EX). 60초 뒤 자동 삭제
SET session:abc "user-42" EX 60
```

`INCR`은 "읽고 → 더하고 → 쓰기"를 Redis가 **한 덩어리로** 처리한다. 애플리케이션에서 값을 꺼내 +1 해서 다시 넣으면 그 사이 다른 서버가 끼어들어 값이 어긋날 수 있는데, `INCR`은 그 틈을 없앤다.

## 2) List — 순서가 있는 줄, 양쪽 끝이 빠르다

**무엇**: 값을 **넣은 순서대로** 늘어놓은 목록. 리스트의 **양쪽 끝(머리·꼬리)에서 넣고 빼는 게 매우 빠르다.** **왜**: "최근 본 상품 10개" 같은 최신순 목록, 그리고 간단한 **작업 대기열(queue — 먼저 들어온 게 먼저 처리되는 줄)**에 쓴다.

아래 코드는 사용자가 본 상품을 앞쪽에 쌓고, 목록을 최근 5개로만 유지한다.

```bash
# 왼쪽(머리)에 추가 → 항상 최신이 맨 앞
LPUSH recent:user42 "product:1001"
LPUSH recent:user42 "product:1002"

# 0~4번째만 남기고 잘라내 최근 5개 유지 (오래된 건 버림)
LTRIM recent:user42 0 4

# 앞에서부터 3개 보기
LRANGE recent:user42 0 2   # → product:1002, product:1001, …
```

대기열로 쓸 때는 한쪽 끝(`LPUSH`)으로 넣고 반대쪽 끝(`RPOP`)에서 꺼내면 **먼저 들어온 것이 먼저 나온다(FIFO — First In First Out).** 주의: 리스트 **중간**의 특정 값을 찾거나 빼는 건 느리다. 리스트는 "끝에서 다루는" 자료구조라는 걸 기억하자.

<!-- diagram: list-as-queue -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 150" role="img" aria-label="왼쪽 끝으로 값을 넣고 오른쪽 끝에서 값을 꺼내 먼저 들어온 것이 먼저 나오는 리스트 기반 대기열 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="16" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">List = 양쪽 끝이 빠른 줄 (대기열)</text>
    <text x="40" y="46" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">LPUSH</text>
    <text x="40" y="58" text-anchor="middle" fill="#5f5a68" font-size="7">머리로 넣기</text>
    <line x1="66" y1="52" x2="92" y2="52" stroke="#1f7a4d" stroke-width="1.5" marker-end="url(#ar)"/>
    <rect x="96" y="40" width="42" height="26" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="117" y="57" text-anchor="middle" fill="#1a1720" font-size="8">작업3</text>
    <rect x="142" y="40" width="42" height="26" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="163" y="57" text-anchor="middle" fill="#1a1720" font-size="8">작업2</text>
    <rect x="188" y="40" width="42" height="26" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="209" y="57" text-anchor="middle" fill="#1a1720" font-size="8">작업1</text>
    <line x1="234" y1="52" x2="260" y2="52" stroke="#b23a00" stroke-width="1.5" marker-end="url(#ar2)"/>
    <text x="290" y="46" text-anchor="middle" fill="#b23a00" font-size="8" font-weight="700">RPOP</text>
    <text x="290" y="58" text-anchor="middle" fill="#5f5a68" font-size="7">꼬리서 꺼내기</text>
    <text x="165" y="100" text-anchor="middle" fill="#5f5a68" font-size="8">먼저 들어온 작업1이 먼저 나온다 (FIFO)</text>
    <text x="165" y="122" text-anchor="middle" fill="#b23a00" font-size="7.5">단, 중간 값을 찾고 빼는 건 느리다</text>
    <defs>
      <marker id="ar" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#1f7a4d"/></marker>
      <marker id="ar2" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker>
    </defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">한쪽 끝으로 넣고 반대쪽 끝에서 꺼내면 대기열이 된다.</figcaption>
</figure>

## 3) Hash — 하나의 키 안에 필드-값 여러 쌍

**무엇**: 키 하나 아래에 `필드=값` 쌍을 여러 개 담는다. **작은 객체(사용자 프로필 등)**를 통째로 담기 좋다. **왜**: 사용자 정보를 `user:42:name`, `user:42:age`처럼 키 여러 개로 흩어 두는 대신, **`user:42` 하나에 묶어** 관리하면 키가 깔끔하고 **일부 필드만 골라 읽고 쓰기**가 쉽다.

아래 코드는 사용자 프로필을 한 키에 담고, 나이만 원자적으로 1 올린다.

```bash
# 한 키(user:42)에 필드 여러 개 저장
HSET user:42 name "Rok" age 30 city "Seoul"

# 필드 하나만 읽기 (전체를 안 꺼내도 됨)
HGET user:42 city          # → "Seoul"

# 숫자 필드만 원자적으로 +1
HINCRBY user:42 age 1      # → 31

# 전체 필드-값 한 번에 보기
HGETALL user:42
```

문자열 여러 개로 나눠 담는 것과 무엇이 다른가? **관련 데이터가 한 키에 묶여** 있어 관리가 쉽고, 필드 단위 갱신도 원자적이다. 단, 하나의 해시가 지나치게 커지면(수만 필드) 다루기 부담스러워지니, 그럴 땐 키를 나누는 걸 고려한다.

## 4) Set — 중복 없는 모음, 교집합·합집합이 공짜

**무엇**: **중복을 허용하지 않고 순서도 없는** 값의 모음. **왜**: "이 글에 붙은 태그", "오늘 방문한 유저 ID"처럼 **중복 제거**가 필요할 때, 그리고 **집합 연산(교집합·합집합·차집합)**을 써야 할 때 강력하다.

아래 코드는 두 사용자의 관심 태그를 넣고, **공통 관심사(교집합)**를 뽑는다.

```bash
# 값 추가 (이미 있으면 무시 → 자동 중복 제거)
SADD user:1:tags "redis" "spring" "jpa"
SADD user:2:tags "redis" "kafka" "jpa"

# 두 집합의 교집합 = 공통 태그
SINTER user:1:tags user:2:tags   # → "redis", "jpa"

# 특정 값이 들어 있는지 즉시 확인
SISMEMBER user:1:tags "kafka"    # → 0 (없음)
```

애플리케이션에서 두 목록을 이중 반복문으로 비교해 공통값을 찾는 코드를 짤 필요가 없다. `SINTER` 한 줄이면 Redis가 **최적화된 방식으로** 교집합을 구한다. "공통 친구 찾기", "두 조건을 모두 만족하는 유저" 같은 게 여기에 딱 맞는다.

## 5) Sorted Set — 점수로 자동 정렬되는 집합

**무엇**: Set처럼 중복이 없되, 각 값에 **점수(score)**를 붙여 **항상 점수 순으로 정렬**해 둔 자료구조. **왜**: **랭킹, 리더보드, 우선순위 큐, 시간순 정렬**의 정답이다. 넣을 때 Redis가 알아서 자리를 잡으므로, 순위 조회가 매우 빠르다.

아래 코드는 게임 점수 랭킹을 만들고 상위권을 뽑는다.

```bash
# 값에 점수를 붙여 추가 (Redis가 점수순으로 정렬 유지)
ZADD leaderboard 1500 "player:A"
ZADD leaderboard 2300 "player:B"
ZADD leaderboard 1800 "player:C"

# 점수 높은 순 상위 2명 (WITHSCORES: 점수도 함께)
ZREVRANGE leaderboard 0 1 WITHSCORES
# → player:B 2300, player:C 1800

# 특정 플레이어의 현재 순위 (0부터 시작, 높은 순)
ZREVRANK leaderboard "player:A"   # → 2 (3등)
```

Sorted Set의 진짜 힘은 **"넣는 순간 이미 정렬돼 있다"**는 데 있다. 앞서 String 방식에서 "전부 꺼내 정렬"하던 비용이 사라진다. 조회수 랭킹, 대기 순번, "최근 1시간 인기글"(점수를 타임스탬프로) 등에 두루 쓴다.

<!-- diagram: sorted-set -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 175" role="img" aria-label="ZADD로 점수와 함께 값을 넣으면 Redis가 점수 높은 순으로 자동 정렬해 유지하고 상위권을 즉시 꺼낼 수 있는 정렬 집합 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="16" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">Sorted Set = 점수순 자동 정렬</text>
    <text x="60" y="40" text-anchor="middle" fill="#5f5a68" font-size="8">넣는 순서 (무작위)</text>
    <rect x="20" y="48" width="80" height="18" rx="3" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="60" y="60" text-anchor="middle" fill="#1a1720" font-size="7.5">A · 1500</text>
    <rect x="20" y="70" width="80" height="18" rx="3" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="60" y="82" text-anchor="middle" fill="#1a1720" font-size="7.5">B · 2300</text>
    <rect x="20" y="92" width="80" height="18" rx="3" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="60" y="104" text-anchor="middle" fill="#1a1720" font-size="7.5">C · 1800</text>
    <line x1="112" y1="79" x2="150" y2="79" stroke="#7d5a9e" stroke-width="1.5" marker-end="url(#ar3)"/>
    <text x="131" y="72" text-anchor="middle" fill="#5f5a68" font-size="7">ZADD</text>
    <text x="255" y="40" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">Redis가 유지 (점수순)</text>
    <rect x="205" y="48" width="100" height="18" rx="3" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="255" y="60" text-anchor="middle" fill="#1a1720" font-size="7.5">① B · 2300</text>
    <rect x="205" y="70" width="100" height="18" rx="3" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="255" y="82" text-anchor="middle" fill="#1a1720" font-size="7.5">② C · 1800</text>
    <rect x="205" y="92" width="100" height="18" rx="3" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="255" y="104" text-anchor="middle" fill="#1a1720" font-size="7.5">③ A · 1500</text>
    <text x="165" y="140" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">ZREVRANGE 0 1 → 상위 2명 즉시</text>
    <text x="165" y="158" text-anchor="middle" fill="#5f5a68" font-size="7.5">넣을 때 정렬해두므로 순위 조회가 빠르다</text>
    <defs>
      <marker id="ar3" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#7d5a9e"/></marker>
    </defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">무작위로 넣어도 Redis가 점수순으로 세워둔다. 상위 N개가 거의 즉시.</figcaption>
</figure>

## 상세 예시 — 같은 "실시간 랭킹"을 두 방식으로

개념이 성능으로 어떻게 이어지는지 한 케이스로 묶어 보자. **"상품 조회수 랭킹 상위 3개"**를 뽑는 상황이다.

**방식 A — String만 사용.** 조회수는 `INCR`로 잘 올린다. 그런데 랭킹을 뽑으려면 모든 상품 키를 훑어야 한다.

```bash
INCR product:1001:views   # 각 상품 조회수는 잘 오른다
INCR product:1002:views
# 랭킹? → product:*:views 키를 전부 찾아(KEYS/SCAN),
#         값을 하나씩 GET 해서, 앱에서 정렬해야 한다.
#         상품이 10만 개면 10만 번 읽고 정렬 → 느리고 서버 부담 큼
```

**방식 B — Sorted Set 사용.** 조회수를 올릴 때 `ZINCRBY`로 정렬 집합의 점수를 올린다. 랭킹은 한 줄이다.

```bash
# 조회 발생 때마다 점수 +1 (없으면 0에서 시작). 정렬은 Redis가 유지
ZINCRBY product:views 1 "1001"
ZINCRBY product:views 1 "1002"

# 상위 3개 = 명령 한 번, 이미 정렬돼 있어 빠름
ZREVRANGE product:views 0 2 WITHSCORES
# → 1002 (점수), 1001 (점수), …
```

| 구분 | 방식 A (String) | 방식 B (Sorted Set) |
|---|---|---|
| 조회수 +1 | `INCR` (빠름) | `ZINCRBY` (빠름) |
| 상위 N 조회 | 전체 키 훑고 앱에서 정렬 | `ZREVRANGE` 한 줄 |
| 데이터 증가 시 | 느려짐(전량 스캔) | 영향 적음(정렬 유지) |
| 앱 코드량 | 많음(정렬 로직 직접) | 적음 |

<p style="font-size:13px;color:#5f5a68;">※ 위 수치·상황은 개념 비교를 위한 가상 예시다. 실제 성능은 데이터 크기·서버 사양에 따라 다르다.</p>

같은 목표라도 **자료구조를 바꾸면 코드가 단순해지고 확장에도 견딘다.** 이게 "Redis는 값만 넣는 캐시가 아니다"라는 말의 실체다.

## 자료구조 고르는 한눈 표

| 하고 싶은 것 | 자료구조 | 대표 명령 |
|---|---|---|
| 단순 값 저장·카운터 | String | `SET`, `INCR` |
| 최신순 목록·대기열 | List | `LPUSH`, `RPOP`, `LRANGE` |
| 작은 객체(필드 여러 개) | Hash | `HSET`, `HGETALL`, `HINCRBY` |
| 중복 제거·집합 연산 | Set | `SADD`, `SINTER`, `SISMEMBER` |
| 랭킹·점수순 정렬 | Sorted Set | `ZADD`, `ZREVRANGE`, `ZRANK` |

## 한 줄 교훈

Redis를 잘 쓰는 첫걸음은 "무엇을 캐시할까"가 아니라 **"이 데이터에 어떤 형태가 맞을까"**를 먼저 묻는 것이다 — 형태를 제대로 고르면, 앱이 하던 정렬·중복 제거·순서 유지를 Redis가 대신 해준다.
