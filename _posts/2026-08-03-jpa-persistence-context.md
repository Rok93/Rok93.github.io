---
layout: post
title: '영속성 컨텍스트와 1차 캐시 — JPA가 DB 앞에 둔 작업 공간'
date: 2026-08-03 09:00:00 +0900
image: /assets/img/jpa-persistence-context/hero.jpg
generated: true
tags: [jpa, study]
sanitized: true
---

JPA를 처음 쓰면 이상한 장면을 만난다. `find()`로 같은 데이터를 두 번 불렀는데 SQL은 한 번만 나간다. 객체 값만 바꿨을 뿐인데 `UPDATE`가 저절로 실행된다. `save()`를 안 했는데도 DB에 반영된다. 마법 같지만 전부 한 곳에서 벌어지는 일이다 — **영속성 컨텍스트(Persistence Context, 엔티티를 보관·관리하는 논리적 공간)**. JPA는 DB에 바로 쓰지 않고, 이 공간을 앞에 한 겹 두고 거기서 먼저 일한다. 이 글은 그 공간이 무엇을 하는지 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 코드·엔티티·SQL 로그는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/jpa-persistence-context/hero.jpg" alt="개발자와 데이터베이스 사이에 놓인 투명한 유리 상자 안에 엔티티 객체들이 떠 있는 모습" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">영속성 컨텍스트는 애플리케이션과 DB 사이에 놓인 "임시 작업 공간"이다. 엔티티는 여기서 먼저 살고, 정리된 결과만 DB로 나간다.</figcaption>
</figure>

## 영속성 컨텍스트가 뭔가 — DB 앞의 임시 작업 공간

영속성 컨텍스트는 **엔티티(entity, DB 테이블과 매핑된 객체)를 담아 두고 관리하는 메모리 공간**이다. 눈에 보이는 물건이 아니라 논리적인 개념이고, 우리는 `EntityManager`(엔티티를 다루는 창구. 스프링에서는 보통 그 위에 얹은 `Repository`로 쓴다)를 통해 이 공간에 접근한다.

핵심은 **JPA가 DB에 곧바로 쓰지 않는다**는 점이다. 엔티티를 저장·수정하면 일단 이 작업 공간 안에서 상태가 바뀌고, 실제 SQL은 정해진 시점에 몰아서 나간다. 왜 이렇게 할까? 이 한 겹 덕분에 **같은 데이터 재조회를 캐시로 막고, 변경을 자동으로 추적하고, SQL을 모아서 효율적으로 내보내는** 일이 가능해진다. 아래에서 하나씩 본다.

## 엔티티의 4가지 상태 — 어디에 속해 있는가

엔티티 객체는 영속성 컨텍스트와의 관계에 따라 네 가지 상태 중 하나에 있다. 이 상태를 알아야 "왜 저장이 됐는지/안 됐는지"가 설명된다.

- **비영속(new/transient)** — `new`로 막 만든 객체. 아직 영속성 컨텍스트가 모른다. DB와 아무 관련 없다.
- **영속(managed)** — `persist()`했거나 `find()`로 조회한 상태. **영속성 컨텍스트가 관리 중**이다. 여기 있는 동안 캐시·변경 감지 같은 혜택을 받는다.
- **준영속(detached)** — 한때 영속이었지만 컨텍스트에서 분리된 상태. `detach()`, `clear()`, 컨텍스트 종료 등으로 떨어져 나온다. 더는 관리받지 않는다.
- **삭제(removed)** — `remove()`로 삭제 예약된 상태. 커밋 시점에 `DELETE`가 나간다.

<!-- diagram: entity-lifecycle -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:480px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 210" role="img" aria-label="엔티티 생명주기. new로 만든 비영속 객체가 persist로 영속이 되고, detach/clear로 준영속이 되거나 remove로 삭제 상태가 된다">
  <g font-size="10" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <rect x="8" y="86" width="78" height="34" rx="6" fill="none" stroke="#5f5a68" stroke-width="1.6"/>
    <text x="47" y="101" text-anchor="middle" fill="#5f5a68" font-weight="700">비영속</text>
    <text x="47" y="113" text-anchor="middle" fill="#5f5a68" font-size="7">new 객체</text>

    <rect x="130" y="86" width="82" height="34" rx="6" fill="none" stroke="#5F0080" stroke-width="2.2"/>
    <text x="171" y="101" text-anchor="middle" fill="#5F0080" font-weight="700">영속</text>
    <text x="171" y="113" text-anchor="middle" fill="#5f5a68" font-size="7">managed</text>

    <rect x="256" y="20" width="78" height="34" rx="6" fill="none" stroke="#7d5a9e" stroke-width="1.6"/>
    <text x="295" y="35" text-anchor="middle" fill="#4a3a5e" font-weight="700">준영속</text>
    <text x="295" y="47" text-anchor="middle" fill="#5f5a68" font-size="7">detached</text>

    <rect x="256" y="152" width="78" height="34" rx="6" fill="none" stroke="#b23a00" stroke-width="1.8"/>
    <text x="295" y="167" text-anchor="middle" fill="#b23a00" font-weight="700">삭제</text>
    <text x="295" y="179" text-anchor="middle" fill="#5f5a68" font-size="7">removed</text>

    <path d="M88 103 L128 103" stroke="#5F0080" stroke-width="1.6" marker-end="url(#p1)"/>
    <text x="108" y="98" text-anchor="middle" fill="#5F0080" font-size="8">persist()</text>

    <path d="M213 96 L254 46" stroke="#7d5a9e" stroke-width="1.4" marker-end="url(#p2)"/>
    <text x="248" y="78" text-anchor="middle" fill="#7d5a9e" font-size="7.5">detach/clear</text>

    <path d="M213 110 L254 160" stroke="#b23a00" stroke-width="1.4" marker-end="url(#p3)"/>
    <text x="246" y="140" text-anchor="middle" fill="#b23a00" font-size="7.5">remove()</text>
  </g>
  <defs>
    <marker id="p1" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#5F0080"/></marker>
    <marker id="p2" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#7d5a9e"/></marker>
    <marker id="p3" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker>
  </defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">엔티티는 영속성 컨텍스트와의 관계에 따라 상태가 바뀐다. "영속" 상태일 때만 캐시·변경 감지 혜택을 받는다.</figcaption>
</figure>

## 1차 캐시 — 같은 것을 두 번 묻지 않는다

영속성 컨텍스트 안에는 **1차 캐시(first-level cache)**가 있다. 구조는 단순하다. `식별자(@Id) → 엔티티 인스턴스`를 담는 지도(Map) 한 개다. 엔티티가 영속 상태가 되면 이 지도에 등록된다.

이게 왜 쓸모 있나? 조회할 때 JPA는 **DB에 가기 전에 1차 캐시부터 뒤진다.** 찾는 식별자가 캐시에 있으면 DB 쿼리 없이 그 인스턴스를 그대로 돌려준다. 같은 트랜잭션 안에서 같은 데이터를 여러 번 읽어도 SQL은 한 번만 나가는 이유가 이것이다.

이 코드가 하는 일: 같은 id의 회원을 연달아 두 번 조회한다.

```java
Member m1 = em.find(Member.class, 1L);  // ① 캐시에 없음 → SELECT 실행 후 캐시에 저장
Member m2 = em.find(Member.class, 1L);  // ② 캐시에 있음 → SELECT 안 나감

System.out.println(m1 == m2);           // true — 같은 인스턴스
```

SQL 로그를 보면 `SELECT`가 **한 번만** 찍힌다. 두 번째 `find()`는 캐시에서 바로 꺼내기 때문이다. 게다가 `m1`과 `m2`는 **같은 자바 인스턴스**다(`==`가 `true`). 이걸 **동일성 보장(identity guarantee)**이라 한다. 같은 트랜잭션 안에서 같은 식별자의 엔티티는 항상 같은 객체임을 JPA가 보장한다 — 마치 자바 컬렉션에서 같은 키로 꺼낸 값이 같은 객체이듯이.

<!-- diagram: first-level-cache -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 200" role="img" aria-label="1차 캐시 조회 흐름. 첫 조회는 캐시에 없어 DB에서 읽고 캐시에 저장, 두 번째 조회는 캐시에서 바로 반환한다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <rect x="8" y="40" width="66" height="26" rx="5" fill="none" stroke="#5F0080" stroke-width="1.8"/>
    <text x="41" y="52" text-anchor="middle" fill="#5F0080" font-weight="700">find(1L)</text>
    <text x="41" y="62" text-anchor="middle" fill="#5f5a68" font-size="6.5">첫 번째</text>

    <rect x="8" y="118" width="66" height="26" rx="5" fill="none" stroke="#1f7a4d" stroke-width="1.8"/>
    <text x="41" y="130" text-anchor="middle" fill="#1f7a4d" font-weight="700">find(1L)</text>
    <text x="41" y="140" text-anchor="middle" fill="#5f5a68" font-size="6.5">두 번째</text>

    <rect x="120" y="66" width="96" height="52" rx="7" fill="#fff" stroke="#7d5a9e" stroke-width="1.8"/>
    <text x="168" y="84" text-anchor="middle" fill="#4a3a5e" font-weight="700">1차 캐시</text>
    <text x="168" y="98" text-anchor="middle" fill="#5f5a68" font-size="7.5">1L → Member</text>
    <text x="168" y="109" text-anchor="middle" fill="#5f5a68" font-size="7">(식별자 → 엔티티)</text>

    <rect x="256" y="66" width="66" height="52" rx="7" fill="none" stroke="#b23a00" stroke-width="1.6"/>
    <text x="289" y="88" text-anchor="middle" fill="#b23a00" font-weight="700">DB</text>
    <text x="289" y="102" text-anchor="middle" fill="#5f5a68" font-size="7">SELECT</text>

    <path d="M74 53 L118 78" stroke="#5F0080" stroke-width="1.4" marker-end="url(#c1)"/>
    <path d="M216 84 L254 88" stroke="#b23a00" stroke-width="1.4" stroke-dasharray="3 2" marker-end="url(#c2)"/>
    <text x="236" y="72" text-anchor="middle" fill="#b23a00" font-size="7">없음→조회</text>

    <path d="M74 131 L118 106" stroke="#1f7a4d" stroke-width="2" marker-end="url(#c3)"/>
    <text x="150" y="150" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">캐시 적중 · DB 안 감</text>
  </g>
  <defs>
    <marker id="c1" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#5F0080"/></marker>
    <marker id="c2" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker>
    <marker id="c3" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#1f7a4d"/></marker>
  </defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">첫 조회는 DB에서 읽어 캐시에 채우고, 두 번째부터는 캐시에서 바로 꺼낸다. 그래서 같은 식별자를 여러 번 조회해도 SELECT는 한 번뿐이다.</figcaption>
</figure>

주의할 점: 1차 캐시는 **트랜잭션 하나의 수명만큼만** 산다. 트랜잭션이 끝나면 영속성 컨텍스트가 닫히고 캐시도 사라진다. 여러 사용자·여러 요청이 공유하는 캐시가 아니다(그건 2차 캐시의 영역이다). 즉 성능을 위한 전역 캐시라기보다, **한 트랜잭션 안에서의 일관성과 반복 조회 절약**을 위한 장치다.

## 변경 감지 — save를 안 불러도 UPDATE가 나가는 이유

영속 상태의 엔티티는 값만 바꿔도 `UPDATE`가 자동으로 나간다. 이걸 **변경 감지(dirty checking)**라 한다.

원리는 이렇다. 엔티티가 영속 상태가 되는 순간, JPA는 그 시점의 값을 **스냅샷(snapshot, 최초 상태의 사본)**으로 떠 둔다. 그리고 flush(영속성 컨텍스트의 변경 내용을 DB에 반영하는 동작) 시점에 **현재 값과 스냅샷을 비교**한다. 달라진 필드가 있으면 JPA가 알아서 `UPDATE` SQL을 만들어 보낸다.

이 코드가 하는 일: 조회한 회원의 이름을 바꾼다. `update` 메서드 호출은 없다.

```java
@Transactional
public void rename(Long id) {
    Member member = em.find(Member.class, id);  // 영속 상태 → 스냅샷 저장됨
    member.setName("김로키");                     // 값만 변경
    // em.update(member) 같은 호출이 없다!
}   // 트랜잭션 커밋 → flush → 스냅샷과 비교 → UPDATE 자동 실행
```

메서드가 끝나고 트랜잭션이 커밋되면, JPA는 스냅샷의 이름과 현재 이름이 다르다는 걸 감지해 `UPDATE member SET name=? WHERE id=?`를 내보낸다. **개발자가 명시적으로 저장을 호출하지 않아도** 변경이 반영되는 것이다. 반대로 말하면, **준영속·비영속 상태의 객체는 아무리 값을 바꿔도 UPDATE가 나가지 않는다** — 스냅샷도 없고 관리 대상도 아니기 때문이다. JPA 초보가 "값을 바꿨는데 왜 저장이 안 되지?"로 헤맬 때, 답은 대개 "그 객체가 영속 상태가 아니어서"다.

<!-- diagram: dirty-checking -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 180" role="img" aria-label="변경 감지 원리. 영속 시점의 스냅샷과 flush 시점의 현재 값을 비교해 달라진 필드가 있으면 UPDATE를 생성한다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <rect x="14" y="30" width="120" height="46" rx="6" fill="#fff" stroke="#b9a9cc" stroke-width="1.6"/>
    <text x="74" y="46" text-anchor="middle" fill="#5f5a68" font-weight="700" font-size="8.5">스냅샷 (영속 시점)</text>
    <text x="74" y="62" text-anchor="middle" fill="#5f5a68" font-size="8">name = "이전"</text>

    <rect x="14" y="104" width="120" height="46" rx="6" fill="#fff" stroke="#5F0080" stroke-width="1.8"/>
    <text x="74" y="120" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="8.5">현재 값 (flush 시점)</text>
    <text x="74" y="136" text-anchor="middle" fill="#5F0080" font-size="8">name = "김로키"</text>

    <path d="M138 53 Q176 53 176 78 Q176 103 138 127" stroke="#7d5a9e" stroke-width="1.4" fill="none"/>
    <text x="188" y="82" fill="#7d5a9e" font-size="8">비교</text>

    <rect x="210" y="66" width="108" height="46" rx="6" fill="none" stroke="#1f7a4d" stroke-width="2"/>
    <text x="264" y="84" text-anchor="middle" fill="#1f7a4d" font-weight="700" font-size="8.5">달라짐 → UPDATE</text>
    <text x="264" y="100" text-anchor="middle" fill="#5f5a68" font-size="7">SET name=? WHERE id=?</text>

    <path d="M198 89 L208 89" stroke="#1f7a4d" stroke-width="1.6" marker-end="url(#d1)"/>
  </g>
  <defs>
    <marker id="d1" markerWidth="7" markerHeight="7" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#1f7a4d"/></marker>
  </defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">flush 시점에 스냅샷과 현재 값을 비교해, 달라진 필드가 있으면 UPDATE를 자동 생성한다. 그래서 값만 바꿔도 저장이 된다.</figcaption>
</figure>

## flush — 변경이 DB로 나가는 순간

지금까지 "정해진 시점에 SQL이 나간다"고 했는데, 그 시점이 바로 **flush**다. flush는 영속성 컨텍스트에 쌓인 변경(등록·수정·삭제)을 **DB에 SQL로 내보내는 동작**이다. 주의할 점은 flush가 트랜잭션을 커밋하는 게 아니라는 것 — SQL을 DB로 보낼 뿐, 확정(commit)은 별개다.

flush가 자동으로 일어나는 시점은 세 가지다.

- **트랜잭션 커밋 직전** — 가장 흔한 경우. 커밋하려면 그동안의 변경이 DB에 반영돼 있어야 하니 flush가 먼저 실행된다.
- **JPQL 쿼리 실행 직전** — `SELECT ... FROM Member` 같은 쿼리를 날리기 전. 아직 DB에 안 보낸 변경이 있으면 조회 결과가 어긋나므로, JPA가 먼저 flush해 DB와 맞춘다.
- **`em.flush()` 직접 호출** — 개발자가 원하는 시점에 강제로.

여기서 앞서 미룬 이야기가 이어진다. `persist()`를 호출하면 INSERT SQL이 **곧바로 나가지 않고** 영속성 컨텍스트 내부의 "쓰기 지연 SQL 저장소"에 쌓였다가, flush 시점에 모여서 나간다. 이를 **쓰기 지연(transactional write-behind)**이라 한다. SQL을 모아 두면 한 번에 내보내(배치) 네트워크 왕복을 줄일 수 있고, 커밋 직전까지 실제 반영을 미뤄 트랜잭션을 유연하게 다룰 수 있다.

## 정리 — 영속성 컨텍스트가 해주는 네 가지

영속성 컨텍스트는 DB 앞에 놓인 작업 공간 하나로 이 모든 걸 엮는다.

1. **1차 캐시** — 같은 식별자는 캐시에서 꺼내 SELECT를 아낀다.
2. **동일성 보장** — 같은 트랜잭션 안 같은 엔티티는 항상 같은 인스턴스(`==`).
3. **변경 감지** — 값만 바꾸면 스냅샷과 비교해 UPDATE를 자동 생성한다.
4. **쓰기 지연** — SQL을 모았다가 flush 때 한꺼번에 내보낸다.

한 줄 교훈: **JPA의 "마법"은 대부분 영속성 컨텍스트 하나로 설명된다.** 저장이 안 되거나, 쿼리가 예상과 다르게 나가거나, 같은 객체인지 헷갈릴 때 — "이 엔티티가 지금 영속 상태인가?"부터 물으면 답이 보인다.
