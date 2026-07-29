---
layout: post
title: '실행계획(EXPLAIN) 읽는 법 — 쿼리가 느린 이유를 눈으로 확인하기'
date: 2026-07-29 09:00:00 +0900
image: /assets/img/explain-query-plan/hero.jpg
generated: true
tags: [database, study]
sanitized: true
---

"이 쿼리 왜 이렇게 느려요?"라는 질문에 감으로 답하면 자주 틀린다. DB는 쿼리를 실행하기 전에 **"어떤 순서로, 어떤 길로 데이터를 찾을지"**를 스스로 정하는데, 그 계획을 그대로 보여주는 명령이 `EXPLAIN`이다. 인덱스를 안 타는지, 몇 행을 훑을 셈인지, 정렬을 따로 하는지 — 느림의 원인이 여기 다 적혀 있다. 이 글은 그 계획서를 한 줄씩 읽는 법을 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·테이블·실행계획은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/explain-query-plan/hero.jpg" alt="지도 위에서 돋보기와 이정표로 최적 경로를 미리 찾는 모습" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">EXPLAIN은 DB가 데이터를 찾아가기 전에 그려둔 "경로 지도"를 펼쳐 보여준다.</figcaption>
</figure>

## EXPLAIN이 뭘 보여주나 — 실행이 아니라 "계획"

`EXPLAIN`은 쿼리 앞에 붙이는 키워드다. 예를 들어 `EXPLAIN SELECT ...`처럼 쓴다. 이걸 붙이면 DB는 **쿼리를 실제로 실행하지 않고**, 대신 옵티마이저(optimizer, 쿼리를 어떻게 처리할지 결정하는 DB 내부 엔진)가 세운 **실행계획(execution plan)**을 표로 돌려준다.

왜 중요한가? 같은 결과를 내는 쿼리라도 DB가 택하는 "길"은 여러 개다. 인덱스를 타고 몇 행만 콕 집을 수도 있고, 테이블 전체를 처음부터 끝까지 훑을 수도 있다. 옵티마이저는 비용(cost, 예상 처리량)이 가장 싸다고 판단한 길을 고른다. EXPLAIN은 그 **선택 결과**를 보여준다. 느리다면, 대개 옵티마이저가 나쁜 길을 골랐거나 좋은 길(인덱스)이 아예 없어서다.

<!-- diagram: optimizer-picks-path -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 200" role="img" aria-label="옵티마이저가 여러 실행 경로 후보 중 비용이 가장 싼 하나를 고르고, EXPLAIN이 그 선택을 보여준다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <rect x="10" y="86" width="66" height="26" rx="5" fill="none" stroke="#5F0080" stroke-width="2"/>
    <text x="43" y="98" text-anchor="middle" fill="#5F0080" font-weight="700">SQL</text>
    <text x="43" y="108" text-anchor="middle" fill="#5f5a68" font-size="7">한 개의 질의</text>
    <path d="M78 99 L104 99" stroke="#c9c4d4" stroke-width="1.5" marker-end="url(#a2)"/>
    <rect x="106" y="80" width="70" height="38" rx="6" fill="none" stroke="#7d5a9e" stroke-width="1.8"/>
    <text x="141" y="96" text-anchor="middle" fill="#4a3a5e" font-weight="700">옵티마이저</text>
    <text x="141" y="108" text-anchor="middle" fill="#5f5a68" font-size="7">비용 비교</text>
    <g fill="none" stroke-width="1.4">
      <path d="M178 90 L206 50" stroke="#b23a00" marker-end="url(#a3)"/>
      <path d="M178 99 L206 99" stroke="#b9a9cc" marker-end="url(#a4)"/>
      <path d="M178 108 L206 150" stroke="#1f7a4d" stroke-width="2.4" marker-end="url(#a5)"/>
    </g>
    <text x="210" y="46" fill="#b23a00" font-size="8">풀 스캔 · 비용 90</text>
    <text x="210" y="102" fill="#5f5a68" font-size="8">인덱스 A · 비용 40</text>
    <text x="210" y="146" fill="#1f7a4d" font-size="8" font-weight="700">인덱스 B · 비용 8 ✓</text>
    <text x="210" y="168" fill="#5f5a68" font-size="7">← 가장 싼 길 선택</text>
    <text x="165" y="185" text-anchor="middle" fill="#5F0080" font-size="8">EXPLAIN = 이 선택을 그대로 출력</text>
  </g>
  <defs>
    <marker id="a2" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#c9c4d4"/></marker>
    <marker id="a3" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker>
    <marker id="a4" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b9a9cc"/></marker>
    <marker id="a5" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#1f7a4d"/></marker>
  </defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">옵티마이저는 여러 실행 경로 후보의 예상 비용을 견줘 가장 싼 하나를 고른다. EXPLAIN은 그 결정을 표로 보여줄 뿐, 쿼리를 실행하진 않는다.</figcaption>
</figure>

## 표의 핵심 열 — 어디부터 봐야 하나

MySQL 기준 `EXPLAIN`은 여러 열을 준다. 전부 외울 필요는 없다. **느림 진단에 결정적인 세 열**만 먼저 익히면 된다.

- **`type`** — 이 테이블에 어떻게 접근하는가. **가장 먼저 봐야 할 열.** 뒤에서 자세히 다룬다.
- **`key`** — 실제로 사용한 인덱스 이름. `NULL`이면 인덱스를 하나도 안 썼다는 뜻(위험 신호).
- **`rows`** — 이 단계에서 **훑을 것으로 예상하는 행 수**. 작을수록 좋다. 100만 행 테이블에서 `rows`가 100만이면 사실상 전체를 읽는다는 뜻이다.

그 밖에 자주 보는 열:

- **`possible_keys`** — 쓸 수 있었던 인덱스 후보. 여기엔 있는데 `key`가 `NULL`이면, 인덱스가 있는데도 안 썼다는 뜻이라 조사 대상이다.
- **`Extra`** — 추가 동작 메모. `Using index`(커버링 인덱스, 좋음), `Using filesort`·`Using temporary`(별도 정렬·임시 테이블, 대개 나쁨) 같은 힌트가 여기 뜬다.

## `type` 읽는 법 — 접근 방식의 등급

`type`은 "이 테이블의 행을 **어떻게 찾아가는가**"를 나타낸다. 성능 등급이라고 봐도 된다. 자주 나오는 값을 **좋음 → 나쁨** 순으로 정리하면 이렇다.

- **`const` / `eq_ref`** — 기본키나 유니크 키로 딱 1행만 집는다. 가장 빠르다.
- **`ref`** — 인덱스로 특정 값에 매칭되는 여러 행을 찾는다. `WHERE member_id = 8234`처럼 일반 인덱스를 탈 때. 실무에서 목표로 삼는 등급.
- **`range`** — 인덱스로 범위를 훑는다. `BETWEEN`, `>`, `IN` 등. 양호하다.
- **`index`** — 인덱스 **전체**를 처음부터 끝까지 훑는다. 이름과 달리 좋은 게 아니다. 인덱스 크기만큼 다 읽는다.
- **`ALL`** — 풀 테이블 스캔(full table scan). 테이블 전 행을 다 읽는다. **가장 느린 등급.** 큰 테이블에서 `ALL`이 뜨면 우선 손봐야 할 대상이다.

핵심 감각: **큰 테이블에서 `ALL`이나 `index`가 보이면 경고**, `ref`·`range`로 내려오면 대개 정상이다.

<!-- diagram: type-ladder -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:430px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 210" role="img" aria-label="type 값 등급 사다리. 위에서 아래로 const, eq_ref, ref, range, index, ALL 순이며 위가 빠르고 아래가 느리다">
  <g font-size="10" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="150" y="16" text-anchor="middle" fill="#1f7a4d" font-size="9" font-weight="700">▲ 빠름 (몇 행만 집음)</text>
    <g text-anchor="middle" font-weight="700">
      <rect x="70" y="24" width="160" height="22" rx="4" fill="none" stroke="#1f7a4d" stroke-width="2"/>
      <text x="150" y="39" fill="#1f7a4d">const / eq_ref</text>
      <rect x="70" y="52" width="160" height="22" rx="4" fill="none" stroke="#1f7a4d" stroke-width="1.8"/>
      <text x="150" y="67" fill="#1f7a4d">ref</text>
      <rect x="70" y="80" width="160" height="22" rx="4" fill="none" stroke="#7d5a9e" stroke-width="1.6"/>
      <text x="150" y="95" fill="#4a3a5e">range</text>
      <rect x="70" y="108" width="160" height="22" rx="4" fill="none" stroke="#b23a00" stroke-width="1.6"/>
      <text x="150" y="123" fill="#b23a00">index</text>
      <rect x="70" y="136" width="160" height="22" rx="4" fill="none" stroke="#b23a00" stroke-width="2.4"/>
      <text x="150" y="151" fill="#b23a00">ALL</text>
    </g>
    <text x="150" y="176" text-anchor="middle" fill="#b23a00" font-size="9" font-weight="700">▼ 느림 (전부 훑음)</text>
    <text x="150" y="196" text-anchor="middle" fill="#5f5a68" font-size="8">큰 테이블에서 index·ALL 이면 인덱스를 의심</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">type 값의 성능 등급. 위로 갈수록 적은 행만 집고, 아래로 갈수록 많이 훑는다. `index`는 이름과 달리 "인덱스 전체 훑기"라 느린 축이다.</figcaption>
</figure>

## 상세 예시 — 인덱스 추가 전후, EXPLAIN이 어떻게 바뀌나

말보다 실제 출력이 빠르다. 주문 100만 건이 든 `orders` 테이블에서 특정 회원의 주문을 찾는 상황을 가정하자. (아래 수치·출력은 개념 설명용으로 지어낸 예시다.)

이 쿼리가 하는 일: `member_id`가 8234인 주문을 모두 가져온다.

```sql
EXPLAIN
SELECT * FROM orders WHERE member_id = 8234;
```

`member_id`에 인덱스가 **없을 때** 결과:

```
+----+-------+------+---------------+------+---------+------+---------+-------------+
| id | table | type | possible_keys | key  | key_len | ref  | rows    | Extra       |
+----+-------+------+---------------+------+---------+------+---------+-------------+
|  1 | orders| ALL  | NULL          | NULL | NULL    | NULL | 1000000 | Using where |
+----+-------+------+---------------+------+---------+------+---------+-------------+
```

읽는 법: `type=ALL`(풀 스캔) + `key=NULL`(인덱스 안 씀) + `rows=1000000`(백만 행 다 훑을 예정). **세 신호가 전부 나쁜 쪽**이다. `Extra=Using where`는 "훑으면서 조건으로 걸러낸다"는 뜻 — 즉 미리 좁히지 못하고 다 읽은 뒤 필터한다.

이 쿼리가 하는 일: `member_id` 열에 인덱스를 만든다.

```sql
CREATE INDEX idx_member ON orders (member_id);
```

같은 EXPLAIN을 다시 돌리면:

```
+----+-------+------+---------------+------------+---------+-------+------+-------+
| id | table | type | possible_keys | key        | key_len | ref   | rows | Extra |
+----+-------+------+---------------+------------+---------+-------+------+-------+
|  1 | orders| ref  | idx_member    | idx_member | 4       | const |   12 | NULL  |
+----+-------+------+---------------+------------+---------+-------+------+-------+
```

무엇이 달라졌나:

- `type`: `ALL` → **`ref`** (풀 스캔에서 인덱스 매칭으로)
- `key`: `NULL` → **`idx_member`** (만든 인덱스를 실제로 사용)
- `rows`: `1000000` → **`12`** (백만 행 훑을 예정에서 12행으로)

`rows`가 100만에서 12로 줄었다는 게 핵심이다. DB가 읽어야 할 양 자체가 8만 배 가까이 줄었으니 쿼리가 빨라진다. **인덱스 추가의 효과를 실행 없이 숫자로 미리 확인**한 셈이다.

## EXPLAIN vs EXPLAIN ANALYZE — 예상값과 실제값

지금까지의 `rows`는 옵티마이저의 **예상치**다. 통계에 기반한 추정이라 실제와 어긋날 수 있다. 실제로 실행해서 **진짜 걸린 시간과 행 수**를 보고 싶으면 `EXPLAIN ANALYZE`를 쓴다(MySQL 8.0.18+, PostgreSQL은 예전부터 지원).

차이를 한 줄로:

- `EXPLAIN` — 실행 안 함. 옵티마이저의 **계획과 예상 행 수**만.
- `EXPLAIN ANALYZE` — **실제로 실행**하고, 단계별 실제 시간·실제 행 수까지.

주의: `EXPLAIN ANALYZE`는 쿼리를 진짜 돌린다. `UPDATE`·`DELETE`에 붙이면 데이터가 바뀌니, 조회가 아닌 쿼리엔 함부로 쓰지 않는다. 예상 `rows`와 실제 행 수가 크게 다르면, 통계가 오래됐다는 신호일 수 있다(`ANALYZE TABLE`로 통계 갱신을 검토).

## 정리 — EXPLAIN을 볼 때 3초 체크리스트

느린 쿼리를 만나면 EXPLAIN을 붙이고 이 순서로 본다.

1. **`type`** — 큰 테이블인데 `ALL`·`index`면 인덱스부터 의심.
2. **`key`** — `NULL`이면 인덱스를 안 탄 것. `possible_keys`에 후보가 있는데도 `NULL`이면 왜 안 골랐는지 조사.
3. **`rows`** — 예상 훑을 행 수. 결과 건수에 비해 지나치게 크면 낭비가 있다는 뜻.
4. **`Extra`** — `Using filesort`·`Using temporary`가 보이면 정렬·그룹핑을 인덱스로 못 받쳐준 것.

한 줄 교훈: **쿼리가 느린 이유는 감으로 맞히지 말고 EXPLAIN으로 읽자.** DB는 자기가 어떤 길로 갈지 이미 표에 적어두었다.
