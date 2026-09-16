---
layout: post
title: '캐시 전략 — "빠르게"보다 먼저 정하는 "누가 언제 채우고 지우나"'
date: 2026-09-16 09:00:00 +0900
image: /assets/img/cache-strategy/hero.jpg
generated: true
tags: [cache, study]
sanitized: true
---

캐시(cache — 느린 원본 대신 빠르게 꺼내려고 잠깐 담아두는 임시 저장)를 붙이는 이유는 단순하다. "느린 DB 대신 빠른 메모리에서 꺼내자." 그런데 캐시를 붙이는 순간 원래 없던 골칫거리가 생긴다. **"이 값을 캐시에 언제 넣고, 원본이 바뀌면 캐시는 어떻게 갱신하나?"** 이걸 대충 정하면, 화면에는 옛날 값이 뜨는데 DB에는 새 값이 들어 있는 **불일치(stale — 원본과 어긋난 낡은 값)**가 생긴다. 캐시 전략(caching strategy)은 바로 이 질문 — **읽기·쓰기 경로에서 누가 캐시를 채우고, 누가 지우느냐**를 정하는 규칙이다. 이 글은 대표 전략인 Cache-Aside부터 Read/Write-Through, Write-Behind, Write-Around까지 **각각 무엇이고, 언제 쓰며, 어떤 함정이 있는지**를 흐름도와 함께 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·코드·시나리오는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/cache-strategy/hero.jpg" alt="느린 데이터베이스 창고 앞에 놓인 빠른 임시 선반을 사람이 먼저 확인하는 모습, 지름길을 상징하는 화살표" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">캐시는 창고(DB) 앞에 둔 임시 선반이다. 문제는 "선반을 누가 채우고 언제 비우나"다.</figcaption>
</figure>

## 왜 "전략"이 필요한가

**무엇**부터. 캐시 전략이란 데이터를 읽고 쓸 때 **애플리케이션·캐시·DB 셋이 어떤 순서로 주고받느냐**를 정한 규칙이다. **왜** 중요한가? 캐시는 원본의 **복사본**이다. 복사본이 있으면 항상 같은 문제가 따라온다 — **원본이 바뀌면 복사본은 낡는다.** 이 낡음을 언제, 어떻게 처리할지 정하지 않으면 두 가지 사고가 난다.

- **낡은 값(stale) 노출**: DB는 바뀌었는데 캐시가 옛 값을 계속 돌려준다.
- **캐시 쏠림(cache stampede — 캐시가 비는 순간 요청이 한꺼번에 DB로 몰리는 현상)**: 인기 데이터의 캐시가 만료되면 수많은 요청이 동시에 DB를 때린다.

그래서 캐시는 "붙이면 빨라진다"로 끝나지 않는다. **읽기 경로와 쓰기 경로 각각에서 캐시를 어떻게 다룰지**를 먼저 정해야 한다. 아래 그림은 캐시가 앱과 DB 사이에 끼어드는 기본 구도다.

<!-- diagram: cache-position -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 175" role="img" aria-label="애플리케이션이 먼저 빠른 캐시를 확인하고, 캐시에 값이 있으면 즉시 응답하고 없으면 느린 DB에서 읽어오는 캐시의 기본 위치 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="16" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">캐시는 앱과 DB 사이의 지름길</text>
    <rect x="16" y="60" width="70" height="40" rx="6" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="51" y="84" text-anchor="middle" fill="#1a1720" font-size="8.5">애플리케이션</text>
    <rect x="130" y="60" width="70" height="40" rx="6" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="165" y="80" text-anchor="middle" fill="#1a1720" font-size="8.5">캐시</text>
    <text x="165" y="92" text-anchor="middle" fill="#5f5a68" font-size="7">(메모리·빠름)</text>
    <rect x="244" y="60" width="70" height="40" rx="6" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="279" y="80" text-anchor="middle" fill="#1a1720" font-size="8.5">DB</text>
    <text x="279" y="92" text-anchor="middle" fill="#5f5a68" font-size="7">(디스크·느림)</text>
    <line x1="86" y1="72" x2="128" y2="72" stroke="#1f7a4d" stroke-width="1.5" marker-end="url(#ca)"/>
    <text x="107" y="66" text-anchor="middle" fill="#1f7a4d" font-size="7">① 먼저 확인</text>
    <line x1="128" y1="88" x2="88" y2="88" stroke="#1f7a4d" stroke-width="1.5" marker-end="url(#ca)"/>
    <text x="107" y="112" text-anchor="middle" fill="#1f7a4d" font-size="7">있으면(hit) 즉시 응답</text>
    <line x1="200" y1="80" x2="242" y2="80" stroke="#b23a00" stroke-width="1.5" stroke-dasharray="3 2" marker-end="url(#cb)"/>
    <text x="222" y="128" text-anchor="middle" fill="#b23a00" font-size="7">없으면(miss) DB로</text>
    <text x="165" y="150" text-anchor="middle" fill="#5f5a68" font-size="7.5">핵심 질문: 캐시를 누가 채우고, 원본이 바뀌면 누가 지우나?</text>
    <defs>
      <marker id="ca" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#1f7a4d"/></marker>
      <marker id="cb" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker>
    </defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">캐시에 있으면 빠른 길로, 없으면 느린 DB로. 전략은 이 화살표의 순서를 정한다.</figcaption>
</figure>

전략은 크게 **읽기 전략**과 **쓰기 전략**으로 나뉜다. 읽기부터 보자.

## 읽기 전략 1) Cache-Aside — 앱이 직접 챙긴다 (가장 흔함)

**무엇**: 애플리케이션이 캐시를 **옆에 두고(aside)** 직접 관리한다. 읽을 때 **먼저 캐시를 보고, 없으면 DB에서 읽어 캐시에 채운다.** 캐시가 비어 있을 때만 채우므로 **Lazy Loading(게으른 적재 — 필요할 때만 채움)**이라고도 부른다. **왜** 가장 흔한가? 캐시 라이브러리에 기대지 않고 앱 코드로 제어하므로 유연하고, 어떤 저장소·캐시 조합에도 붙는다.

아래 코드는 상품을 읽을 때 캐시를 먼저 확인하고, 없으면 DB에서 읽어 캐시에 채우는 전형적인 Cache-Aside 흐름이다.

```java
Product getProduct(Long id) {
    String key = "product:" + id;

    // ① 캐시 먼저 확인
    Product cached = cache.get(key);
    if (cached != null) {
        return cached;              // 캐시 히트(hit) → 바로 반환
    }

    // ② 캐시 미스(miss) → DB에서 읽고
    Product product = db.findById(id);

    // ③ 캐시에 채운 뒤 반환 (TTL 5분: 5분 뒤 자동 만료)
    cache.set(key, product, Duration.ofMinutes(5));
    return product;
}
```

여기서 핵심은 **채우는 주체가 애플리케이션**이라는 점이다. 캐시는 그저 값을 보관할 뿐, "DB에서 읽어와 채워라"는 판단은 앱이 한다.

<!-- diagram: cache-aside-read -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 200" role="img" aria-label="Cache-Aside 읽기 흐름. 히트면 캐시에서 바로 반환하고, 미스면 DB에서 읽어 캐시에 채운 뒤 반환하는 두 갈래 흐름 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="15" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">Cache-Aside 읽기 — 두 갈래</text>
    <rect x="16" y="30" width="66" height="30" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="49" y="49" text-anchor="middle" fill="#1a1720" font-size="8">앱: 조회</text>
    <line x1="82" y1="45" x2="116" y2="45" stroke="#5f5a68" stroke-width="1.3" marker-end="url(#cc)"/>
    <rect x="118" y="30" width="80" height="30" rx="5" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="158" y="44" text-anchor="middle" fill="#1a1720" font-size="8">캐시 확인</text>
    <text x="158" y="55" text-anchor="middle" fill="#5f5a68" font-size="7">있나?</text>
    <!-- hit branch -->
    <line x1="198" y1="40" x2="250" y2="40" stroke="#1f7a4d" stroke-width="1.3" marker-end="url(#cc2)"/>
    <text x="224" y="34" text-anchor="middle" fill="#1f7a4d" font-size="7">히트</text>
    <rect x="252" y="28" width="62" height="26" rx="5" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="283" y="45" text-anchor="middle" fill="#1f7a4d" font-size="7.5">즉시 반환</text>
    <!-- miss branch -->
    <line x1="158" y1="60" x2="158" y2="86" stroke="#b23a00" stroke-width="1.3" marker-end="url(#cd)"/>
    <text x="176" y="76" text-anchor="middle" fill="#b23a00" font-size="7">미스</text>
    <rect x="118" y="88" width="80" height="28" rx="5" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="158" y="105" text-anchor="middle" fill="#1a1720" font-size="8">DB에서 읽기</text>
    <line x1="118" y1="102" x2="84" y2="102" stroke="#b23a00" stroke-width="1.3" marker-end="url(#cd2)"/>
    <rect x="16" y="122" width="80" height="28" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="56" y="139" text-anchor="middle" fill="#1a1720" font-size="7.5">캐시에 채움</text>
    <line x1="98" y1="100" x2="112" y2="100" stroke="#b23a00" stroke-width="0" />
    <path d="M158,116 L158,124 L100,124" fill="none" stroke="#7d5a9e" stroke-width="1.3" marker-end="url(#ce)"/>
    <text x="165" y="176" text-anchor="middle" fill="#5f5a68" font-size="7.5">미스일 때만 DB→캐시 적재 (Lazy Loading)</text>
    <text x="165" y="192" text-anchor="middle" fill="#b23a00" font-size="7">약점: 첫 요청은 항상 미스, 캐시가 비면 요청이 DB로 몰림</text>
    <defs>
      <marker id="cc" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#5f5a68"/></marker>
      <marker id="cc2" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#1f7a4d"/></marker>
      <marker id="cd" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker>
      <marker id="cd2" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker>
      <marker id="ce" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#7d5a9e"/></marker>
    </defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">히트는 빠른 길, 미스는 DB를 거쳐 캐시를 채운다. 채우는 판단은 앱이 한다.</figcaption>
</figure>

Cache-Aside의 약점도 분명하다. **첫 요청은 항상 미스**라 느리고(콜드 스타트), 인기 키의 캐시가 만료되는 순간 요청이 한꺼번에 DB로 몰리는 **캐시 쏠림**이 생길 수 있다. 이를 막으려고 만료 시각을 조금씩 흩뿌리거나(TTL 지터), 캐시를 채우는 동안 락을 잡는 기법을 쓴다.

## 읽기 전략 2) Read-Through — 캐시가 대신 DB를 읽어준다

**무엇**: Cache-Aside와 목적은 같지만, **DB에서 읽어 채우는 일을 앱이 아니라 캐시 계층(라이브러리)이 대신** 한다. 앱은 캐시에만 "이 값 줘"라고 요청하고, 캐시가 없으면 알아서 DB에서 읽어 채워 돌려준다. **왜** 쓰나? 앱 코드에서 미스 처리 로직이 사라져 **읽기 코드가 단순**해진다.

Cache-Aside와의 차이는 **"누가 DB를 읽느냐"** 하나다. Cache-Aside는 앱이, Read-Through는 캐시가 읽는다. 대신 Read-Through는 그런 기능을 지원하는 캐시 라이브러리·구성이 필요하다.

## 쓰기 전략 — 원본이 바뀔 때 캐시를 어떻게 하나

읽기만 있으면 캐시는 편하다. 문제는 **쓰기**다. DB가 바뀌면 캐시의 복사본은 그 순간 낡는다. 쓰기 전략은 이 낡음을 처리하는 세 갈래다.

<!-- diagram: write-strategies -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 230" role="img" aria-label="세 가지 쓰기 전략 비교. Write-Through는 캐시와 DB를 동시에 쓰고, Write-Behind는 캐시 먼저 쓰고 DB는 나중에 모아 쓰며, Write-Around는 DB만 쓰고 캐시는 비운다">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="15" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">쓰기 전략 3가지</text>
    <!-- write-through -->
    <text x="60" y="38" text-anchor="middle" fill="#1f7a4d" font-size="8.5" font-weight="700">Write-Through</text>
    <rect x="30" y="46" width="60" height="20" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="60" y="60" text-anchor="middle" fill="#1a1720" font-size="7">쓰기</text>
    <line x1="60" y1="66" x2="60" y2="78" stroke="#1f7a4d" stroke-width="1.3"/>
    <rect x="18" y="80" width="40" height="18" rx="3" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="38" y="92" text-anchor="middle" fill="#1a1720" font-size="7">캐시</text>
    <rect x="62" y="80" width="40" height="18" rx="3" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="82" y="92" text-anchor="middle" fill="#1a1720" font-size="7">DB</text>
    <text x="60" y="114" text-anchor="middle" fill="#5f5a68" font-size="6.5">둘 다 즉시(동기)</text>
    <text x="60" y="124" text-anchor="middle" fill="#5f5a68" font-size="6.5">→ 항상 일치, 쓰기 느림</text>
    <!-- write-behind -->
    <text x="165" y="38" text-anchor="middle" fill="#5F0080" font-size="8.5" font-weight="700">Write-Behind</text>
    <rect x="135" y="46" width="60" height="20" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="165" y="60" text-anchor="middle" fill="#1a1720" font-size="7">쓰기</text>
    <line x1="165" y1="66" x2="165" y2="78" stroke="#5F0080" stroke-width="1.3"/>
    <rect x="145" y="80" width="40" height="18" rx="3" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="165" y="92" text-anchor="middle" fill="#1a1720" font-size="7">캐시</text>
    <line x1="165" y1="98" x2="165" y2="110" stroke="#5f5a68" stroke-width="1.1" stroke-dasharray="2 2" marker-end="url(#cf)"/>
    <rect x="145" y="112" width="40" height="18" rx="3" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="165" y="124" text-anchor="middle" fill="#1a1720" font-size="7">DB</text>
    <text x="165" y="146" text-anchor="middle" fill="#5f5a68" font-size="6.5">캐시 먼저, DB는 나중에</text>
    <text x="165" y="156" text-anchor="middle" fill="#b23a00" font-size="6.5">→ 빠름, 유실 위험</text>
    <!-- write-around -->
    <text x="270" y="38" text-anchor="middle" fill="#b23a00" font-size="8.5" font-weight="700">Write-Around</text>
    <rect x="240" y="46" width="60" height="20" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="270" y="60" text-anchor="middle" fill="#1a1720" font-size="7">쓰기</text>
    <line x1="270" y1="66" x2="270" y2="110" stroke="#b23a00" stroke-width="1.3"/>
    <rect x="248" y="80" width="44" height="18" rx="3" fill="#f0eef2" stroke="#b9a9cc" stroke-dasharray="3 2"/>
    <text x="270" y="92" text-anchor="middle" fill="#8a8594" font-size="6.5">캐시 건너뜀</text>
    <rect x="250" y="112" width="40" height="18" rx="3" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="270" y="124" text-anchor="middle" fill="#1a1720" font-size="7">DB</text>
    <text x="270" y="146" text-anchor="middle" fill="#5f5a68" font-size="6.5">DB만 쓰고 캐시는 무효화</text>
    <text x="270" y="156" text-anchor="middle" fill="#5f5a68" font-size="6.5">→ 안 읽힐 값에 유리</text>
    <line x1="20" y1="178" x2="310" y2="178" stroke="#e4e0ec" stroke-width="1"/>
    <text x="165" y="196" text-anchor="middle" fill="#1a1720" font-size="7.5" font-weight="700">공통 목표: DB와 캐시의 불일치를 언제·어떻게 줄일까</text>
    <text x="165" y="212" text-anchor="middle" fill="#5f5a68" font-size="7">일치 강할수록 쓰기 느리고, 빠를수록 낡을 위험 커짐 (트레이드오프)</text>
    <defs>
      <marker id="cf" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#5f5a68"/></marker>
    </defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">일치를 강하게 하면 쓰기가 느려지고, 쓰기를 빠르게 하면 낡을 위험이 커진다.</figcaption>
</figure>

### Write-Through — 캐시와 DB를 동시에 쓴다

**무엇**: 쓸 때 **캐시와 DB에 함께(동기적으로) 쓴다.** **왜**: 쓰기가 끝나면 캐시와 DB가 **항상 일치**하므로, 읽을 때 낡은 값을 볼 일이 없다. **단점**: 매 쓰기마다 두 곳에 써야 해 **쓰기가 느리다.** 또 잘 안 읽히는 데이터까지 전부 캐시에 올라가 메모리를 낭비할 수 있다. 그래서 흔히 Read-Through와 짝지어, **읽기도 쓰기도 캐시를 거치게** 구성한다.

### Write-Behind (Write-Back) — 캐시 먼저, DB는 나중에 모아서

**무엇**: 쓸 때 **캐시에만 먼저 쓰고**, DB에는 잠시 뒤 **모아서 비동기로** 반영한다. **왜**: 쓰기 응답이 매우 빠르고, 여러 변경을 묶어 DB 부하를 줄인다. **치명적 단점**: 캐시에만 있고 아직 DB에 안 내려간 값이 있는 상태에서 **캐시가 죽으면 그 쓰기가 사라진다(데이터 유실).** 그래서 유실을 감당할 수 있는 데이터(조회수 집계 등)나, 별도 내구성 장치가 있을 때만 쓴다.

### Write-Around — DB만 쓰고 캐시는 비운다

**무엇**: 쓸 때 **DB에만 쓰고 캐시는 건드리지 않거나, 해당 키를 지운다(무효화).** 그 값은 다음에 읽힐 때 Cache-Aside 방식으로 다시 채워진다. **왜**: **한 번 쓰고 잘 안 읽는 데이터**에 좋다. 어차피 안 읽힐 값을 캐시에 올려 자리를 차지하지 않는다. **단점**: 방금 쓴 값을 곧바로 읽으면 미스가 나 DB를 한 번 더 친다.

## 상세 예시 — Cache-Aside에서 "수정 후 옛 값"이 뜨는 함정

가장 흔한 Cache-Aside에서 실제로 자주 겪는 불일치를 재현해 보자. 상품 가격을 수정하는 상황이다. **잘못된 코드**부터.

이 코드는 DB만 갱신하고 캐시는 그대로 둔다.

```java
void updatePrice(Long id, int newPrice) {
    db.updatePrice(id, newPrice);   // ① DB만 바꿈
    // ② 캐시(product:id)는 옛 가격 그대로 남아 있음!
}
```

문제 시나리오(가상):

| 시각 | 동작 | 캐시 값 | DB 값 |
|---|---|---|---|
| 10:00 | 상품 조회 → 캐시에 적재 | 1,000원 | 1,000원 |
| 10:01 | 가격 수정(위 코드) | **1,000원(낡음)** | 1,500원 |
| 10:02 | 상품 조회 → 캐시 히트 | 1,000원 | 1,500원 |

10:02에 사용자는 **1,000원**을 본다. TTL 5분이 지나기 전까지 계속 옛 가격이 노출된다. **해결**은 쓸 때 캐시를 **무효화(invalidate — 낡을 값을 지워 다음 읽기에서 다시 채우게 함)**하는 것이다.

이 코드는 DB를 바꾼 뒤 캐시 키를 지워, 다음 조회 때 새 값으로 다시 채워지게 한다.

```java
void updatePrice(Long id, int newPrice) {
    db.updatePrice(id, newPrice);       // ① DB 갱신
    cache.delete("product:" + id);      // ② 캐시 무효화 → 다음 읽기에서 재적재
}
```

여기서 "갱신(update)"이 아니라 **"삭제(delete)"**를 쓰는 게 요령이다. 캐시에 새 값을 직접 써넣으면, 여러 요청이 동시에 수정할 때 **어떤 값이 최종으로 남을지 경쟁**이 생긴다. 반면 지워 두면 다음 읽기가 DB의 확정된 값을 가져와 채우므로 더 안전하다. (완벽하진 않다 — 삭제와 재적재 사이의 짧은 틈에도 경쟁이 있을 수 있어, 민감한 데이터는 짧은 TTL이나 버전 검증을 함께 쓴다.)

## 어떤 전략을 언제 — 한눈 표

| 상황 | 추천 전략 | 이유 |
|---|---|---|
| 읽기 많고 일반적 | Cache-Aside | 유연·범용, 앱이 제어 |
| 읽기 코드 단순화 | Read-Through | 미스 처리를 캐시가 대신 |
| 항상 최신 일치 필요 | Write-Through | 쓸 때 캐시·DB 동시 갱신 |
| 쓰기 폭주·유실 감당 가능 | Write-Behind | 캐시 먼저, DB는 모아서 |
| 쓰고 잘 안 읽는 데이터 | Write-Around | 캐시 자리 낭비 방지 |

## 한 줄 교훈

캐시 전략의 본질은 "얼마나 빠르냐"가 아니라 **"원본과 복사본의 불일치를 언제·누가 메우느냐"**다 — 일치를 강하게 할수록 쓰기가 느려지고, 쓰기를 빠르게 할수록 낡을 위험이 커지는 트레이드오프 위에서, 데이터 성격에 맞는 지점을 고르는 일이다.
