---
layout: post
title: 'DB 인덱스와 B-Tree — 왜 인덱스 하나로 쿼리가 100배 빨라질까'
date: 2026-07-27 09:00:00 +0900
tags: [database, index, b-tree, mysql, study]
sanitized: true
---

주문 100만 건이 쌓인 테이블에서 특정 회원의 주문을 찾는 쿼리가 **1.2초**나 걸렸다. 인덱스(index, 데이터를 빨리 찾기 위한 "찾아보기" 자료구조) 한 줄을 추가하자 **8밀리초**로 줄었다. 150배다. 무슨 일이 벌어진 걸까? 이 글은 그 원리를 코드로 확인하며 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·테이블·쿼리는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/db-index-btree/hero.jpg" alt="왼쪽은 색인으로 원하는 책을 바로 찾는 사람, 오른쪽은 책 더미를 처음부터 하나씩 뒤지는 사람" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">인덱스가 있고 없고의 차이. 왼쪽은 색인으로 곧장 찾고, 오른쪽은 쌓인 걸 처음부터 다 뒤진다.</figcaption>
</figure>

## 인덱스가 없으면 — 처음부터 끝까지 다 읽는다

인덱스가 없는 테이블에서 `WHERE member_id = 8234`를 찾으면, DB는 **첫 행부터 마지막 행까지 전부 비교**한다. 이걸 풀 스캔(full scan, 전체 훑기)이라고 부른다. 100만 행이면 100만 번 비교다.

책으로 비유하면, 색인(찾아보기)이 없는 책에서 특정 단어를 찾으려고 1페이지부터 끝까지 넘기는 것과 같다. 인덱스는 책 뒤의 **찾아보기 페이지**다. "트랜잭션 … 152쪽"처럼 미리 정렬해두면, 몇 번의 점프로 원하는 곳에 닿는다.

아래는 풀 스캔과 인덱스 탐색의 차이를 그린 그림이다.

<!-- diagram: fullscan-vs-index -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:460px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 320 200" role="img" aria-label="풀 스캔은 모든 행을 순서대로 훑고, 인덱스는 정렬된 트리로 몇 단계만에 도달한다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="70" y="16" text-anchor="middle" fill="#b23a00" font-weight="700">풀 스캔</text>
    <text x="70" y="28" text-anchor="middle" fill="#5f5a68" font-size="8">100만 번 비교</text>
    <g fill="none" stroke="#b23a00" stroke-width="1.5">
      <rect x="45" y="36" width="50" height="14" rx="3"/>
      <rect x="45" y="54" width="50" height="14" rx="3"/>
      <rect x="45" y="72" width="50" height="14" rx="3"/>
      <rect x="45" y="90" width="50" height="14" rx="3"/>
      <rect x="45" y="108" width="50" height="14" rx="3"/>
      <rect x="45" y="126" width="50" height="14" rx="3" stroke="#5F0080" stroke-width="2.5"/>
      <rect x="45" y="144" width="50" height="14" rx="3" stroke-dasharray="3 2"/>
    </g>
    <text x="70" y="170" text-anchor="middle" fill="#5F0080" font-size="8">6번째서 발견 (운 나쁘면 끝까지)</text>
    <path d="M120 100 L150 100" stroke="#c9c4d4" stroke-width="1.5" marker-end="url(#arr)"/>
    <text x="245" y="16" text-anchor="middle" fill="#1f7a4d" font-weight="700">인덱스 (B-Tree)</text>
    <text x="245" y="28" text-anchor="middle" fill="#5f5a68" font-size="8">약 3단계 점프</text>
    <g fill="none" stroke="#1f7a4d" stroke-width="1.8">
      <rect x="220" y="42" width="50" height="16" rx="4"/>
      <rect x="185" y="82" width="42" height="16" rx="4"/>
      <rect x="262" y="82" width="42" height="16" rx="4"/>
      <rect x="170" y="122" width="34" height="16" rx="4"/>
      <rect x="212" y="122" width="34" height="16" rx="4" stroke="#5F0080" stroke-width="2.5"/>
      <rect x="278" y="122" width="34" height="16" rx="4"/>
    </g>
    <g stroke="#1f7a4d" stroke-width="1.2" fill="none">
      <path d="M235 58 L206 82"/><path d="M255 58 L283 82"/>
      <path d="M198 98 L187 122"/><path d="M214 98 L229 122"/>
    </g>
    <text x="245" y="112" text-anchor="middle" fill="#5f5a68" font-size="7">정렬돼 있어 방향만 고름</text>
  </g>
  <defs><marker id="arr" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#c9c4d4"/></marker></defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">풀 스캔은 앞에서부터 전부 비교한다. B-Tree 인덱스는 정렬된 트리라 "왼쪽이냐 오른쪽이냐"만 몇 번 고르면 도달한다.</figcaption>
</figure>

## 왜 하필 B-Tree인가 — 균형과 범위

인덱스를 그냥 "정렬된 목록"으로 두면 값 하나 넣을 때마다 뒤 전체를 밀어야 해서 느리다. 그래서 대부분의 관계형 DB는 인덱스를 **B-Tree**라는 트리 구조로 만든다. (정확히는 InnoDB·대다수 DB가 쓰는 건 그 변형인 **B+Tree**다. 아래에서 구분한다.)

B-Tree의 핵심 성질 세 가지다.

- **균형(balanced)**: 어느 잎(leaf, 트리의 맨 아래 마디)까지 가든 깊이가 같다. 100만 행이어도 보통 3~4단계면 닿는다. 비교 횟수가 100만이 아니라 `log`에 비례해 확 준다.
- **정렬**: 각 마디 안의 키가 정렬돼 있어, 찾을 값이 "이 마디의 왼쪽 갈래냐 오른쪽 갈래냐"를 바로 정한다.
- **범위 검색에 강함**: 시작점을 찾은 뒤 옆으로 훑으면 `BETWEEN`, `>`, `ORDER BY`가 값싸게 처리된다.

<!-- diagram: bplus-tree -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:480px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 190" role="img" aria-label="B+Tree는 실제 데이터가 맨 아래 리프에만 있고, 리프끼리 옆으로 연결돼 범위 검색이 빠르다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" text-anchor="middle">
    <rect x="130" y="20" width="80" height="22" rx="5" fill="none" stroke="#5F0080" stroke-width="2"/>
    <text x="170" y="35" fill="#5F0080" font-weight="700">30 | 60</text>
    <rect x="35" y="78" width="70" height="22" rx="5" fill="none" stroke="#7d5a9e" stroke-width="1.6"/>
    <text x="70" y="93" fill="#4a3a5e">10 | 20</text>
    <rect x="135" y="78" width="70" height="22" rx="5" fill="none" stroke="#7d5a9e" stroke-width="1.6"/>
    <text x="170" y="93" fill="#4a3a5e">40 | 50</text>
    <rect x="235" y="78" width="70" height="22" rx="5" fill="none" stroke="#7d5a9e" stroke-width="1.6"/>
    <text x="270" y="93" fill="#4a3a5e">70 | 90</text>
    <g stroke="#b9a9cc" stroke-width="1.2" fill="none">
      <path d="M150 42 L70 78"/><path d="M170 42 L170 78"/><path d="M190 42 L270 78"/>
    </g>
    <text x="170" y="128" fill="#1f7a4d" font-size="8" font-weight="700">리프 (실제 행 데이터, 옆으로 연결)</text>
    <g fill="none" stroke="#1f7a4d" stroke-width="1.6">
      <rect x="30" y="138" width="80" height="20" rx="4"/>
      <rect x="130" y="138" width="80" height="20" rx="4"/>
      <rect x="230" y="138" width="80" height="20" rx="4"/>
    </g>
    <text x="70" y="152" fill="#1a1720" font-size="8">10 20 → row</text>
    <text x="170" y="152" fill="#1a1720" font-size="8">40 50 → row</text>
    <text x="270" y="152" fill="#1a1720" font-size="8">70 90 → row</text>
    <g stroke="#1f7a4d" stroke-width="1.4" stroke-dasharray="3 2">
      <path d="M110 148 L130 148"/><path d="M210 148 L230 148"/>
    </g>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">B+Tree: 위 마디는 이정표(어디로 갈지)만 담고, 실제 행은 맨 아래 리프에만 있다. 리프끼리 점선으로 이어져 범위 스캔은 옆으로 훑기만 하면 된다.</figcaption>
</figure>

## 클러스터드 vs 세컨더리 — 리프에 뭐가 들었나

InnoDB(MySQL의 기본 저장 엔진)에서 인덱스는 두 종류다. 이 구분을 모르면 "인덱스를 탔는데 왜 여전히 느리지?"를 설명하지 못한다.

- **클러스터드 인덱스(clustered index)**: 기본키(PK) 기준으로 만들어지며, **리프에 행 데이터 전체**가 들어 있다. 즉 PK로 찾으면 한 번에 데이터에 닿는다. 테이블당 1개뿐이다(데이터 자체의 물리 정렬이라서).
- **세컨더리 인덱스(secondary index)**: PK가 아닌 컬럼에 만든 인덱스. 리프에는 행 전체가 아니라 **그 행의 PK 값**만 들어 있다. 그래서 `member_id`로 찾으면 → PK를 얻고 → 그 PK로 클러스터드 인덱스를 **다시 탐색**해 실제 데이터를 가져온다. 이 2단계를 **북마크 조회**(bookmark lookup) 또는 back-to-primary라 부른다.

이 코드는 세컨더리 인덱스가 실제로 PK를 거쳐 데이터를 찾는 과정을 의사코드로 보여준다.

```text
-- SELECT * FROM orders WHERE member_id = 8234
1) member_id 세컨더리 인덱스 탐색  → member_id=8234 인 행들의 PK 목록 [1021, 5540, ...]
2) 각 PK로 클러스터드 인덱스 재탐색 → 실제 행 데이터 읽기   (행 수만큼 반복)
```

여기서 중요한 함의: **찾는 행이 많으면 2단계 재탐색 비용이 커진다.** 그래서 다음에 나오는 커버링 인덱스가 강력하다.

## 복합 인덱스와 "최좌측 접두사" 규칙

여러 컬럼을 묶은 인덱스를 복합 인덱스(composite index)라 한다. 예: `INDEX (member_id, status, created_at)`. 이건 세 컬럼을 **이 순서로 정렬**해 저장한다 — 먼저 `member_id`로, 같으면 `status`로, 또 같으면 `created_at`으로.

핵심 규칙은 **최좌측 접두사(leftmost prefix)**다. 왼쪽부터 연속으로 쓰는 조건만 인덱스를 탄다.

<!-- diagram: leftmost-prefix -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 150" role="img" aria-label="복합 인덱스는 왼쪽 컬럼부터 연속으로 쓰는 조건만 사용할 수 있다">
  <g font-size="9.5" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="10" y="18" fill="#5F0080" font-weight="700">INDEX (member_id, status, created_at)</text>
    <g>
      <text x="14" y="46" fill="#1f7a4d">✓</text><text x="30" y="46" fill="#1a1720">WHERE member_id = 8234</text><text x="230" y="46" fill="#1f7a4d" font-size="8">탄다</text>
      <text x="14" y="70" fill="#1f7a4d">✓</text><text x="30" y="70" fill="#1a1720">member_id = 8234 AND status = 'PAID'</text><text x="290" y="70" fill="#1f7a4d" font-size="8">탄다</text>
      <text x="14" y="94" fill="#b23a00">✗</text><text x="30" y="94" fill="#1a1720">WHERE status = 'PAID'</text><text x="230" y="94" fill="#b23a00" font-size="8">못 탄다 (member_id 건너뜀)</text>
      <text x="14" y="118" fill="#b23a00">✗</text><text x="30" y="118" fill="#1a1720">WHERE created_at &gt; '2026-01-01'</text><text x="250" y="118" fill="#b23a00" font-size="8">못 탄다</text>
    </g>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">전화번호부가 (성 → 이름) 순으로 정렬돼 있으면 "성이 김"은 빨리 찾지만, "이름이 영희"만으로는 처음부터 훑어야 한다. 복합 인덱스도 같다.</figcaption>
</figure>

전화번호부 비유가 정확하다. (성, 이름) 순 정렬이면 "김"으로 시작하는 사람은 빨리 찾지만, 성을 모른 채 "영희"만으로는 전체를 훑어야 한다. 그래서 복합 인덱스는 **컬럼 순서가 곧 성능**이다.

## 커버링 인덱스 — 테이블을 아예 안 읽는다

앞서 세컨더리 인덱스는 PK로 데이터를 다시 찾아온다고 했다. 그런데 **SELECT가 필요로 하는 컬럼이 전부 인덱스 안에 있으면**, DB는 테이블 데이터를 읽지 않고 인덱스만으로 답한다. 이걸 커버링 인덱스(covering index)라 하고, 실행계획에 `Using index`로 표시된다.

이 쿼리는 `member_id`와 `status`가 모두 인덱스에 있으므로 테이블 접근이 사라진다.

```sql
-- INDEX (member_id, status) 가 있을 때
SELECT status FROM orders WHERE member_id = 8234;
-- 필요한 값(status)이 인덱스에 이미 있음 → 북마크 조회 생략 → 'Using index'
```

## 인덱스를 만들어도 안 타는 흔한 경우

인덱스를 걸었는데 여전히 풀 스캔이면, 대개 아래 중 하나다. 원리는 하나다 — **인덱스에 저장된 "원래 값"의 정렬을 못 쓰게 만들면** 인덱스는 무력화된다.

- **컬럼에 함수·연산을 씌움**: `WHERE YEAR(created_at) = 2026`은 못 탄다. `created_at`을 가공한 값은 인덱스에 없기 때문. → `WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'`로 바꾸면 탄다.
- **암묵적 형변환**: `phone` 컬럼이 문자열인데 `WHERE phone = 01012345678`(숫자)로 비교하면 형변환이 일어나 인덱스를 못 쓴다.
- **앞이 열린 LIKE**: `LIKE '%김%'`이나 `LIKE '%김'`은 못 탄다. 정렬은 앞글자 기준이라 뒤가 고정돼야 의미가 있다. `LIKE '김%'`(뒤가 열림)는 탄다.
- **낮은 카디널리티**: 카디널리티(cardinality, 서로 다른 값의 개수)가 낮은 컬럼(예: 성별 2종, 상태 3종)은 인덱스로 걸러도 절반이 남아 풀 스캔이 더 빠를 수 있다. 옵티마이저가 일부러 인덱스를 안 쓴다.

## 상세 예시 케이스 — EXPLAIN으로 눈으로 확인

말로만 하면 안 믿긴다. 실제 테이블로 확인해보자. 주문 100만 건, `member_id`에 인덱스가 없는 상태다.

이 명령은 쿼리를 실행하지 않고 **DB가 세운 실행 계획**만 보여준다.

```sql
EXPLAIN SELECT * FROM orders WHERE member_id = 8234;
```

인덱스 **없을 때** 결과 (핵심 열만):

```text
 type | key  | rows    | Extra
------+------+---------+-------------
 ALL  | NULL | 1002340 | Using where     ← type=ALL: 풀 스캔, 100만 행 다 읽음
```

이제 인덱스를 만든다.

```sql
CREATE INDEX idx_orders_member ON orders (member_id);
```

같은 `EXPLAIN`을 다시 돌리면:

```text
 type | key               | rows | Extra
------+-------------------+------+-------------
 ref  | idx_orders_member |   12 | Using where  ← type=ref: 인덱스로 12행만 조회
```

`rows`가 **1,002,340 → 12**로 줄었다. 읽는 양이 8만 분의 1이다. 도입부의 1.2초 → 8밀리초가 바로 이 차이다.

한 걸음 더 — 이 회원의 주문 중 `status`만 필요하다면 복합 인덱스로 커버링까지 노린다.

```sql
CREATE INDEX idx_orders_member_status ON orders (member_id, status);
EXPLAIN SELECT status FROM orders WHERE member_id = 8234;
-- Extra 에 'Using index' 등장 → 테이블 접근조차 사라짐
```

## 한 줄 교훈

> 인덱스는 마법이 아니라 **정렬된 사본**이다. "이 조건이 인덱스의 정렬을 그대로 쓸 수 있는가?"를 물으면, 탈지 안 탈지·컬럼 순서를 어떻게 둘지가 전부 설명된다. 그리고 확신 대신 `EXPLAIN`으로 확인하라.
