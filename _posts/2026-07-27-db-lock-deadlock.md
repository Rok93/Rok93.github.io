---
layout: post
title: 'DB 락과 데드락 — 서로가 서로를 기다리다 멈추는 순간'
date: 2026-07-27 11:00:00 +0900
image: /assets/img/db-lock-deadlock/hero.jpg
tags: [database, study]
sanitized: true
---

두 개의 이체가 동시에 일어난다. 하나는 A 계좌 돈을 B로 옮기고, 다른 하나는 B 계좌 돈을 A로 옮긴다. 첫 번째는 A를 잠그고 B를 기다리고, 두 번째는 B를 잠그고 A를 기다린다. 둘 다 상대가 놓아주기를 기다리지만, 상대도 나를 기다린다. 아무도 움직이지 않는다. 이 영원한 정지가 바로 **데드락**(deadlock, 교착 상태)이다. 그리고 그 배경에는 데이터베이스가 동시성을 지키기 위해 거는 **락**(lock, 잠금)이 있다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·테이블·쿼리는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/db-lock-deadlock/hero.jpg" alt="두 사람이 각자 상대가 쥔 열쇠를 기다리며 마주보고 얼어붙어 있는 장면" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">데드락의 직관: 각자 상대가 쥔 것을 기다리므로, 둘 다 영원히 다음으로 못 넘어간다.</figcaption>
</figure>

## 먼저 — 락은 왜 필요한가

트랜잭션(여러 DB 작업을 *전부 성공 아니면 전부 취소*로 묶는 단위)이 여러 개 동시에 같은 데이터를 만지면, 서로 덮어써 값이 깨진다. 이를 막으려고 데이터베이스는 데이터에 **락**을 건다. 락은 "지금 이 데이터는 내가 쓰는 중이니 기다려라"라는 표시다.

락에는 크게 두 종류가 있다.

- **공유 락(shared lock, S 락)**: 읽기용. 여러 트랜잭션이 **동시에** 같은 데이터에 공유 락을 걸 수 있다. 읽기끼리는 서로를 방해하지 않기 때문이다.
- **배타 락(exclusive lock, X 락)**: 쓰기용. 한 트랜잭션이 배타 락을 쥐면 **다른 어떤 락도** 그 데이터에 못 붙는다. 쓰는 동안 남이 읽거나 쓰면 값이 깨지기 때문이다.

두 락이 같은 데이터에 붙을 수 있는지를 정리하면 이렇다. "쓰기가 끼면 무조건 막힌다"가 핵심이다.

<!-- diagram: lock-compat -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:420px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 320 170" role="img" aria-label="공유락과 배타락의 호환성 표. 공유-공유만 허용되고 나머지는 모두 대기">
  <g font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="150" y="24" font-size="9" fill="#5f5a68" text-anchor="middle">이미 걸린 락 →</text>
    <text x="130" y="52" font-size="11" fill="#1a1720" text-anchor="middle" font-weight="700">공유(S)</text>
    <text x="240" y="52" font-size="11" fill="#1a1720" text-anchor="middle" font-weight="700">배타(X)</text>
    <text x="45" y="92" font-size="11" fill="#1a1720" text-anchor="middle" font-weight="700">공유(S)</text>
    <text x="45" y="132" font-size="11" fill="#1a1720" text-anchor="middle" font-weight="700">배타(X)</text>
    <rect x="90" y="72" width="80" height="30" rx="6" fill="#e3f2ea" stroke="#1f7a4d"/>
    <text x="130" y="92" font-size="10" fill="#1f7a4d" text-anchor="middle" font-weight="700">허용</text>
    <rect x="200" y="72" width="80" height="30" rx="6" fill="#fbe9df" stroke="#b23a00"/>
    <text x="240" y="92" font-size="10" fill="#b23a00" text-anchor="middle" font-weight="700">대기</text>
    <rect x="90" y="112" width="80" height="30" rx="6" fill="#fbe9df" stroke="#b23a00"/>
    <text x="130" y="132" font-size="10" fill="#b23a00" text-anchor="middle" font-weight="700">대기</text>
    <rect x="200" y="112" width="80" height="30" rx="6" fill="#fbe9df" stroke="#b23a00"/>
    <text x="240" y="132" font-size="10" fill="#b23a00" text-anchor="middle" font-weight="700">대기</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">공유↔공유만 같이 설 수 있다. 한쪽이라도 배타(쓰기)면 뒤에 온 쪽은 대기한다.</figcaption>
</figure>

## 데드락 — 대기가 원을 그리면 멈춘다

락 자체는 정상이다. 뒤에 온 트랜잭션이 잠깐 기다렸다가 앞이 끝나면 이어받는다. 문제는 이 **대기가 원을 그릴 때** 생긴다.

앞의 이체 예시로 돌아가자. 이 코드는 두 계좌 사이에 서로 반대 방향으로 돈을 옮긴다.

```sql
-- 트랜잭션 1: A → B 로 이체
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 'A';  -- A에 배타 락 획득
UPDATE account SET balance = balance + 100 WHERE id = 'B';  -- B의 락을 기다림

-- 트랜잭션 2: B → A 로 이체 (거의 동시에 실행)
BEGIN;
UPDATE account SET balance = balance - 50 WHERE id = 'B';   -- B에 배타 락 획득
UPDATE account SET balance = balance + 50 WHERE id = 'A';   -- A의 락을 기다림
```

트랜잭션 1은 A를 쥐고 B를 기다린다. 트랜잭션 2는 B를 쥐고 A를 기다린다. **1은 2가 쥔 B를**, **2는 1이 쥔 A를** 기다린다. 대기의 화살표가 원을 그리는 순간, 둘 다 영원히 못 나아간다.

<!-- diagram: deadlock-cycle -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 360 180" role="img" aria-label="트랜잭션 1은 A를 쥐고 B를 기다리고, 트랜잭션 2는 B를 쥐고 A를 기다려 대기가 원을 그리는 그림">
  <defs>
    <marker id="arw" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/>
    </marker>
    <marker id="hold" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L6,3 L0,6 Z" fill="#5F0080"/>
    </marker>
  </defs>
  <g font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" font-size="11">
    <!-- 트랜잭션 노드 -->
    <rect x="20" y="60" width="90" height="40" rx="8" fill="#efe6f5" stroke="#5F0080"/>
    <text x="65" y="85" fill="#5F0080" text-anchor="middle" font-weight="700">Tx 1</text>
    <rect x="250" y="60" width="90" height="40" rx="8" fill="#efe6f5" stroke="#5F0080"/>
    <text x="295" y="85" fill="#5F0080" text-anchor="middle" font-weight="700">Tx 2</text>
    <!-- 자원 노드 -->
    <circle cx="130" cy="30" r="18" fill="#fff" stroke="#5f5a68"/>
    <text x="130" y="34" fill="#1a1720" text-anchor="middle" font-weight="700">A</text>
    <circle cx="230" cy="130" r="18" fill="#fff" stroke="#5f5a68"/>
    <text x="230" y="134" fill="#1a1720" text-anchor="middle" font-weight="700">B</text>
    <!-- 보유(hold): 실선 보라 -->
    <line x1="95" y1="62" x2="118" y2="42" stroke="#5F0080" stroke-width="2" marker-end="url(#hold)"/>
    <line x1="265" y1="98" x2="242" y2="118" stroke="#5F0080" stroke-width="2" marker-end="url(#hold)"/>
    <!-- 대기(wait): 점선 주황 -->
    <line x1="218" y1="118" x2="112" y2="42" stroke="#b23a00" stroke-width="2" stroke-dasharray="4 3" marker-end="url(#arw)"/>
    <line x1="142" y1="42" x2="248" y2="118" stroke="#b23a00" stroke-width="2" stroke-dasharray="4 3" marker-end="url(#arw)"/>
    <text x="70" y="150" fill="#5F0080" font-size="9">— 보유</text>
    <text x="150" y="150" fill="#b23a00" font-size="9">--- 대기</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">보유(보라)와 대기(주황)를 이으면 Tx1→B→Tx2→A→Tx1 순환이 보인다. 이 원이 데드락이다.</figcaption>
</figure>

## 데드락이 성립하는 네 가지 조건

데드락은 아무 때나 생기지 않는다. 다음 **네 조건이 모두** 맞아떨어질 때만 생긴다(코프만 조건, Coffman conditions). 뒤집으면, 이 중 **하나만 깨도** 데드락은 불가능해진다 — 예방의 열쇠다.

- **상호 배제(mutual exclusion)**: 한 자원은 한 번에 한 트랜잭션만 쓸 수 있다(배타 락).
- **점유 대기(hold and wait)**: 이미 뭔가 쥔 채로 다른 것을 더 기다린다.
- **비선점(no preemption)**: 남이 쥔 락을 강제로 뺏을 수 없다. 스스로 놓을 때까지 기다린다.
- **순환 대기(circular wait)**: 대기 관계가 원을 그린다(1→2→1).

## 데이터베이스는 데드락을 어떻게 처리하나

여기서 흔한 오해 하나. **데이터베이스는 데드락을 "예방"하지 않는다.** 대신 생긴 뒤에 **감지해서 푼다.**

대부분의 DB(예: MySQL InnoDB, PostgreSQL)는 내부에 **대기 그래프**(wait-for graph)를 들고 있다. "누가 누구를 기다리는가"를 화살표로 그린 그래프다. 여기에 **순환(cycle)이 생기면** 데드락으로 판정한다.

판정되면 DB는 원을 이루는 트랜잭션 중 하나를 **희생자**(victim)로 골라 강제로 롤백(되돌림)한다. 보통 되돌리는 비용이 가장 적은 쪽(건드린 행이 적은 쪽)을 고른다. 희생당한 트랜잭션은 아래 같은 에러를 받고 끝난다.

```
ERROR 1213 (40001): Deadlock found when trying to get lock;
try restarting transaction
```

메시지의 마지막 문장이 핵심이다 — *"트랜잭션을 다시 시도하라"*. 데드락은 **타이밍 문제**라, 같은 트랜잭션을 잠시 뒤 재시도하면 대개 성공한다. 그래서 실무에서는 데드락 에러(예: MySQL의 1213, SQLSTATE `40001`)를 잡아 **짧게 재시도**하는 코드를 둔다.

## 예방 — 조건 하나를 깨면 된다

감지·재시도는 사후 대응이다. 그보다 애초에 순환이 안 생기게 하는 편이 낫다. 코프만 조건 중 실무에서 가장 깨기 쉬운 건 **순환 대기**다.

**핵심 처방: 모든 트랜잭션이 자원을 같은 순서로 잠근다.** 앞의 이체 예시에서, 두 트랜잭션 모두 "id가 작은 계좌부터" 잠그도록 강제하면 어떻게 될까? 둘 다 A(작은 쪽)를 먼저 노린다. 한쪽이 A를 쥐면 다른 쪽은 A 앞에서 기다린다 — B로 넘어가지 못하므로 **원이 만들어지지 않는다.**

이 코드는 항상 정렬된 순서로 잠가 순환을 원천 차단한다.

```sql
-- 두 트랜잭션 모두 'id가 작은 계좌'부터 잠근다
BEGIN;
SELECT * FROM account WHERE id IN ('A','B') ORDER BY id FOR UPDATE;
-- A 먼저, 그다음 B 순서로 배타 락을 획득 → 순서가 통일돼 순환 불가
UPDATE account SET balance = balance - 100 WHERE id = 'A';
UPDATE account SET balance = balance + 100 WHERE id = 'B';
COMMIT;
```

이 밖에 실무에서 자주 쓰는 예방·완화책은 다음과 같다.

- **트랜잭션을 짧게 유지한다.** 락을 쥐는 시간이 짧으면 겹칠 확률 자체가 준다. 트랜잭션 안에서 외부 API 호출 같은 느린 작업을 하지 않는다.
- **인덱스로 락 범위를 좁힌다.** 인덱스 없이 조건을 걸면 DB가 많은 행(때론 테이블 전체)을 잠글 수 있다. 적절한 인덱스는 딱 필요한 행만 잠그게 해 충돌을 줄인다.
- **락 대기 시간 제한(lock timeout)을 둔다.** 정해진 시간 넘게 못 얻으면 스스로 포기·롤백하게 해, 무한 대기를 짧은 실패로 바꾼다.

## 한 줄 교훈

데드락은 락이 고장 난 게 아니라, **대기가 원을 그린** 것이다. 그러니 해법도 둘 중 하나다 — *원을 못 그리게 순서를 통일하거나*, *생기면 빨리 감지해 재시도하거나*.
