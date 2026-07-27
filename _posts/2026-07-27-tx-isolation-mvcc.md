---
layout: post
title: '트랜잭션 격리수준과 MVCC — 읽기와 쓰기는 어떻게 서로를 안 막을까'
date: 2026-07-27 09:00:00 +0900
tags: [database, transaction, isolation, mvcc, study]
sanitized: true
---

두 사람이 같은 계좌를 동시에 만진다. 한 명은 잔액을 읽는 중이고, 다른 한 명은 그 잔액을 바꾸는 중이다. 읽는 쪽은 **바뀌기 전 값**을 봐야 할까, **바뀌는 중인 값**을 봐야 할까? 이 질문에 대한 데이터베이스의 답이 바로 **트랜잭션 격리수준**(transaction isolation level, 동시에 도는 트랜잭션들이 서로를 얼마나 볼 수 있는지 정하는 등급)이다. 그리고 그걸 성능 손해 없이 구현하는 기술이 **MVCC**다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·테이블·쿼리는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/tx-isolation-mvcc/hero.jpg" alt="여러 사람이 각자 다른 시점의 문서 사본을 서로 방해 없이 읽고, 뒤로 문서 버전들이 층층이 쌓여 있다" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">MVCC의 직관: 각자 "자기 시점의 사본"을 읽으므로, 남이 원본을 고쳐도 내 읽기는 방해받지 않는다.</figcaption>
</figure>

## 먼저 — 격리를 안 하면 생기는 세 가지 사고

트랜잭션(여러 DB 작업을 *전부 성공 아니면 전부 취소*로 묶는 단위)을 여러 개 동시에 돌리면, 서로 간섭해 이상한 값을 읽는 일이 생긴다. 대표적인 세 가지를 **이상 현상**(anomaly)이라 부른다.

- **더티 리드(dirty read, 더러운 읽기)**: 다른 트랜잭션이 바꿨지만 **아직 커밋(확정 저장)하지 않은** 값을 읽는 것. 그 트랜잭션이 롤백(되돌림)하면, 나는 존재한 적 없는 값을 읽은 셈이 된다.
- **반복 불가능한 읽기(non-repeatable read)**: 같은 행을 **두 번 읽었는데 값이 다른** 것. 두 읽기 사이에 다른 트랜잭션이 그 행을 수정하고 커밋했기 때문.
- **팬텀 리드(phantom read, 유령 읽기)**: 같은 조건으로 **두 번 조회했는데 행의 개수가 다른** 것. 사이에 다른 트랜잭션이 조건에 맞는 행을 새로 추가(insert)하고 커밋했기 때문. 없던 행이 유령처럼 나타난다.

세 현상은 "무엇이 흔들리느냐"가 다르다. 더티 리드는 *커밋 여부*, 반복 불가능한 읽기는 *한 행의 값*, 팬텀은 *행의 개수*다.

## 격리수준 4단계 — 무엇을 막아주느냐의 등급

SQL 표준은 격리수준을 네 단계로 정의한다. 위로 갈수록 덜 막고 빠르며, 아래로 갈수록 많이 막고 느리다. **막는 것과 성능은 맞바꿈(trade-off)** 관계다.

<!-- diagram: isolation-matrix -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:520px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 380 200" role="img" aria-label="격리수준 4단계와 세 이상현상의 허용/차단 표. 위로 갈수록 덜 막고, 아래로 갈수록 많이 막는다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="120" y="14" fill="#5f5a68" font-size="8">더티</text>
    <text x="200" y="14" fill="#5f5a68" font-size="8">반복불가</text>
    <text x="290" y="14" fill="#5f5a68" font-size="8">팬텀</text>
    <g text-anchor="start">
      <text x="6" y="42" fill="#1a1720">READ UNCOMMITTED</text>
      <text x="6" y="80" fill="#1a1720">READ COMMITTED</text>
      <text x="6" y="118" fill="#1a1720">REPEATABLE READ</text>
      <text x="6" y="156" fill="#5F0080" font-weight="700">SERIALIZABLE</text>
    </g>
    <g text-anchor="middle" font-weight="700">
      <!-- READ UNCOMMITTED: 다 허용 -->
      <text x="122" y="42" fill="#b23a00">허용</text><text x="205" y="42" fill="#b23a00">허용</text><text x="292" y="42" fill="#b23a00">허용</text>
      <!-- READ COMMITTED -->
      <text x="122" y="80" fill="#1f7a4d">차단</text><text x="205" y="80" fill="#b23a00">허용</text><text x="292" y="80" fill="#b23a00">허용</text>
      <!-- REPEATABLE READ -->
      <text x="122" y="118" fill="#1f7a4d">차단</text><text x="205" y="118" fill="#1f7a4d">차단</text><text x="292" y="118" fill="#b6892a">허용*</text>
      <!-- SERIALIZABLE -->
      <text x="122" y="156" fill="#1f7a4d">차단</text><text x="205" y="156" fill="#1f7a4d">차단</text><text x="292" y="156" fill="#1f7a4d">차단</text>
    </g>
    <g stroke="#e4e0ec" stroke-width="1">
      <line x1="0" y1="52" x2="360" y2="52"/><line x1="0" y1="90" x2="360" y2="90"/><line x1="0" y1="128" x2="360" y2="128"/>
    </g>
    <text x="6" y="180" fill="#b6892a" font-size="7.5">* SQL 표준은 REPEATABLE READ에서 팬텀 허용. 단 MySQL InnoDB는 갭 락으로 대체로 막는다(뒤에서 설명).</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">아래 단계는 위 단계가 막는 것 + 하나를 더 막는 누적 구조다. SERIALIZABLE은 셋 다 막지만 가장 느리다.</figcaption>
</figure>

참고로 **기본값은 DB마다 다르다.** MySQL의 InnoDB는 `REPEATABLE READ`가 기본이고, PostgreSQL·Oracle은 `READ COMMITTED`가 기본이다. "내 DB가 몇 단계인지"를 모르면 동시성 버그를 못 잡는다.

## MVCC — 버전을 여러 개 두어 읽기와 쓰기를 떼어놓는다

여기서 자연스러운 걱정이 생긴다. "반복 불가능한 읽기를 막으려면, 내가 읽는 동안 남이 못 고치게 **잠가야(lock)** 하는 것 아닌가? 그럼 느리잖아." 실제로 옛날 방식(순수 락 기반)은 그랬다.

**MVCC**(Multi-Version Concurrency Control, 다중 버전 동시성 제어)는 이 문제를 다르게 푼다. 데이터를 덮어쓰지 않고 **여러 버전으로 남긴다.** 그래서 읽는 쪽은 잠그지 않고 "내 시점에 맞는 과거 버전"을 보고, 쓰는 쪽은 새 버전을 만든다. **읽기는 쓰기를 막지 않고, 쓰기는 읽기를 막지 않는다.**

InnoDB가 이걸 구현하는 방식은 이렇다. 각 행에는 숨은 컬럼이 붙는다.

- `DB_TRX_ID`: 이 행을 **마지막으로 바꾼 트랜잭션의 번호**.
- `DB_ROLL_PTR`: **이전 버전이 저장된 위치**(언두 로그, undo log)를 가리키는 포인터.

수정이 일어나면 옛 값은 언두 로그에 남고, 포인터로 사슬처럼 연결된다. 읽을 때는 내 **스냅샷(snapshot, 특정 시점의 상태 사진)** 기준으로 "이 버전이 나에게 보여도 되는지"를 판단해, 안 되면 포인터를 타고 과거로 내려간다.

<!-- diagram: mvcc-version-chain -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:520px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 380 190" role="img" aria-label="한 행의 여러 버전이 언두 로그로 사슬처럼 연결되고, 두 트랜잭션이 각자 시점에 맞는 버전을 읽는다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" text-anchor="middle">
    <text x="90" y="14" fill="#5F0080" font-weight="700" font-size="8">현재 버전 (테이블)</text>
    <rect x="40" y="22" width="100" height="34" rx="6" fill="none" stroke="#5F0080" stroke-width="2"/>
    <text x="90" y="38" fill="#1a1720">잔액=8000</text>
    <text x="90" y="50" fill="#5f5a68" font-size="7.5">trx_id=50</text>
    <text x="230" y="14" fill="#5f5a68" font-size="8">← 언두 로그(과거 버전) →</text>
    <rect x="180" y="22" width="90" height="34" rx="6" fill="none" stroke="#7d5a9e" stroke-width="1.6" stroke-dasharray="4 2"/>
    <text x="225" y="38" fill="#4a3a5e">잔액=5000</text>
    <text x="225" y="50" fill="#5f5a68" font-size="7.5">trx_id=32</text>
    <rect x="300" y="22" width="70" height="34" rx="6" fill="none" stroke="#b9a9cc" stroke-width="1.4" stroke-dasharray="4 2"/>
    <text x="335" y="38" fill="#6a5c7e">잔액=3000</text>
    <text x="335" y="50" fill="#5f5a68" font-size="7.5">trx_id=20</text>
    <g stroke="#b9a9cc" stroke-width="1.4" fill="none" marker-end="url(#pa)">
      <path d="M140 39 L178 39"/><path d="M270 39 L298 39"/>
    </g>
    <text x="60" y="72" fill="#5f5a68" font-size="7" text-anchor="start">roll_ptr 로 과거와 연결</text>
    <!-- 두 트랜잭션의 시선 -->
    <circle cx="90" cy="120" r="16" fill="none" stroke="#1f7a4d" stroke-width="2"/>
    <text x="90" y="118" fill="#1f7a4d" font-size="8">T-A</text>
    <text x="90" y="129" fill="#5f5a68" font-size="7">최신 봄</text>
    <path d="M90 104 L90 58" stroke="#1f7a4d" stroke-width="1.4" stroke-dasharray="3 2" marker-end="url(#pg)"/>
    <circle cx="225" cy="120" r="16" fill="none" stroke="#b23a00" stroke-width="2"/>
    <text x="225" y="118" fill="#b23a00" font-size="8">T-B</text>
    <text x="225" y="129" fill="#5f5a68" font-size="7">옛 시점</text>
    <path d="M225 104 L225 58" stroke="#b23a00" stroke-width="1.4" stroke-dasharray="3 2" marker-end="url(#po)"/>
    <text x="190" y="160" fill="#5f5a68" font-size="8">같은 행이라도 트랜잭션마다 "보이는 버전"이 다르다 — 그래서 서로 안 막는다.</text>
  </g>
  <defs>
    <marker id="pa" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b9a9cc"/></marker>
    <marker id="pg" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#1f7a4d"/></marker>
    <marker id="po" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker>
  </defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">한 행의 과거 버전들이 언두 로그로 이어진다. 트랜잭션은 자기 스냅샷에 맞는 버전을 골라 읽으므로 락이 필요 없다.</figcaption>
</figure>

### 스냅샷을 언제 찍느냐가 READ COMMITTED와 REPEATABLE READ를 가른다

MVCC의 스냅샷(정확히는 InnoDB의 **Read View**)을 **언제 찍는가**가 두 격리수준의 차이를 만든다. 이 한 줄이 핵심이다.

- **READ COMMITTED**: **매 SELECT마다 새로** 스냅샷을 찍는다. 그래서 앞 SELECT 뒤에 남이 커밋하면, 다음 SELECT는 바뀐 값을 본다 → 반복 불가능한 읽기가 생긴다.
- **REPEATABLE READ**: **트랜잭션의 첫 읽기에서 한 번** 찍고 끝까지 고정한다. 그래서 중간에 남이 뭘 커밋하든 나는 같은 스냅샷을 보므로, 같은 행을 몇 번 읽어도 값이 같다.

## 스냅샷 읽기 vs 락 읽기 — MVCC가 항상 쓰이는 건 아니다

주의할 점. MVCC의 과거 버전 읽기(스냅샷 읽기, consistent read)는 **평범한 `SELECT`**에만 적용된다. 다음처럼 **락을 함께 거는 읽기**(locking read)는 스냅샷이 아니라 **현재 커밋된 최신 값**을 읽고 그 행을 잠근다.

이 코드는 락을 걸어 재고를 읽는다 — 다른 트랜잭션이 이 행을 못 바꾸게 막는다.

```sql
SELECT stock FROM product WHERE id = 10 FOR UPDATE;   -- 쓰기 락
SELECT stock FROM product WHERE id = 10 LOCK IN SHARE MODE;  -- 읽기(공유) 락
```

즉 "재고를 확인하고 차감"처럼 **읽은 값을 근거로 바로 쓰는** 로직은 스냅샷 읽기로는 위험하고, `FOR UPDATE`로 잠가야 안전하다. 스냅샷은 과거를 보여줄 뿐, 그 사이 남이 바꾼 걸 반영하지 않기 때문이다.

그리고 앞 도식의 각주 — InnoDB의 `REPEATABLE READ`가 표준과 달리 팬텀을 대체로 막는 이유가 여기 있다. InnoDB는 락 읽기·수정 시 **갭 락**(gap lock, 행과 행 *사이 빈 구간*을 잠가 새 행 삽입을 막는 락)을 걸어, 조건에 맞는 새 행이 끼어드는 걸 방지한다.

## 상세 예시 케이스 — 두 세션으로 눈으로 확인

말보다 세션 두 개를 나란히 돌리는 게 확실하다. 계좌 잔액을 세션 A가 두 번 읽는 사이, 세션 B가 값을 바꾸고 커밋한다.

먼저 **READ COMMITTED**에서. (`SET SESSION ...`은 이 접속의 격리수준을 바꾼다.)

```sql
-- 세션 A
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
SELECT balance FROM account WHERE id = 1;   -- (1) 결과: 5000

                          -- 세션 B (그 사이)
                          BEGIN;
                          UPDATE account SET balance = 8000 WHERE id = 1;
                          COMMIT;            -- 커밋

SELECT balance FROM account WHERE id = 1;   -- (2) 결과: 8000  ← 값이 바뀜!
COMMIT;
```

`(1)`과 `(2)`가 **5000 → 8000**으로 달라졌다. 같은 트랜잭션 안에서 같은 행을 읽었는데 값이 흔들린다 — **반복 불가능한 읽기**다.

이제 세션 A의 격리수준만 **REPEATABLE READ**로 바꾸고 똑같이 해보자.

```sql
-- 세션 A
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
SELECT balance FROM account WHERE id = 1;   -- (1) 결과: 5000  ← 여기서 스냅샷 고정

                          -- 세션 B: 위와 똑같이 8000으로 UPDATE + COMMIT

SELECT balance FROM account WHERE id = 1;   -- (2) 결과: 5000  ← 여전히 5000!
COMMIT;
```

이번엔 `(2)`도 **5000**이다. 세션 B가 분명히 커밋했는데도, 세션 A는 자기 트랜잭션 시작 시점의 스냅샷을 끝까지 본다. 바로 MVCC가 REPEATABLE READ를 만들어내는 장면이다.

> 한 가지 헷갈림 주의: REPEATABLE READ의 세션 A가 `COMMIT` 후 다시 `SELECT` 하면 그제야 8000을 본다. 스냅샷은 **트랜잭션 단위**로 고정되는 것이지 영원한 게 아니다.

## 한 줄 교훈

> 격리수준은 "동시에 도는 트랜잭션이 서로를 얼마나 보느냐"의 등급이고, MVCC는 그걸 **잠그는 대신 버전을 여러 개 두어** 싸게 구현하는 기술이다. "내 스냅샷은 언제 찍혔나?"와 "이건 스냅샷 읽기인가 락 읽기인가?" 두 질문이면 동시성 동작의 대부분이 설명된다.
