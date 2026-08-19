---
layout: post
title: 'JVM 메모리 구조와 GC — 내가 만든 객체는 어디에 살다가 언제 치워질까?'
date: 2026-08-19 09:00:00 +0900
image: /assets/img/jvm-memory-gc/hero.jpg
generated: true
tags: [jvm, study]
sanitized: true
---

`new`를 한 번도 안 썼는데 메모리가 꽉 차서 서버가 멈춘다. 로그를 보니 몇 초씩 아무 요청도 처리하지 못한 구간이 찍혀 있다. 범인은 대개 **GC(가비지 컬렉션, Garbage Collection — 안 쓰는 객체를 자동으로 치우는 작업)**다. 자바(Java)는 C처럼 메모리를 손으로 반납하지 않는다. 대신 **JVM(자바 가상 머신, Java Virtual Machine)**이 "이제 아무도 안 쓰는 객체"를 알아서 찾아 지운다. 편하지만, 그 청소가 언제·어떻게 일어나는지 모르면 위 같은 멈춤 현상을 못 잡는다. 이 글은 내가 만든 객체가 **어느 메모리 구획에 살다가**, **어떤 규칙으로 치워지는지**를 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·로그·시나리오는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/jvm-memory-gc/hero.jpg" alt="여러 구획으로 나뉜 메모리 공간에서 안 쓰는 객체를 골라 치우는 모습을 형상화한 삽화" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">JVM 메모리는 용도별로 구획이 나뉘어 있고, GC는 그중 힙(heap)에서 안 쓰는 객체를 골라 치운다.</figcaption>
</figure>

## JVM 메모리는 용도별로 나뉜다

먼저 **무엇**부터. 자바 프로그램이 실행되면 JVM은 운영체제로부터 메모리를 한 덩어리 받아, **용도별로 구획을 나눠 쓴다**. 크게 세 곳만 알면 된다.

- **힙(heap)**: `new`로 만든 **객체가 사는 곳**. 모든 스레드가 공유한다. GC의 청소 대상이 바로 여기다.
- **스택(stack)**: 메서드가 호출될 때마다 쌓이는 **지역 변수·매개변수** 공간. 스레드마다 하나씩 따로 있다. 메서드가 끝나면 그 칸(스택 프레임)이 통째로 사라지므로 **GC가 관여하지 않는다**.
- **메타스페이스(metaspace)**: 클래스 정보(어떤 필드·메서드가 있는지 등)를 담는 곳. 자바 8부터 힙 바깥의 **네이티브 메모리(운영체제가 직접 관리하는 메모리)**에 둔다.

**왜** 이렇게 나눌까? 수명이 다르기 때문이다. 지역 변수는 메서드가 끝나면 바로 필요 없어지니 스택에서 즉시 정리하면 된다. 반면 객체는 여러 메서드·스레드가 오래 참조할 수 있어, 함부로 지우면 안 되고 "정말 아무도 안 쓰는지" 따져야 한다. 그 판단과 청소가 GC의 일이다.

<!-- diagram: jvm-memory-regions -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 200" role="img" aria-label="JVM 메모리가 스레드별 스택, 공유되는 힙, 힙 바깥의 메타스페이스로 나뉜 구조">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="165" y="15" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="9.5">JVM 메모리 구획</text>
    <!-- 스택들 -->
    <rect x="16" y="28" width="86" height="72" rx="6" fill="#fff" stroke="#7d5a9e" stroke-width="1.4"/>
    <text x="59" y="42" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">스택 (스레드별)</text>
    <rect x="24" y="50" width="70" height="14" rx="3" fill="#f3ecf7" stroke="#b9a9cc"/>
    <rect x="24" y="68" width="70" height="14" rx="3" fill="#f3ecf7" stroke="#b9a9cc"/>
    <text x="59" y="60" text-anchor="middle" fill="#5f5a68" font-size="7">지역변수·매개변수</text>
    <text x="59" y="78" text-anchor="middle" fill="#5f5a68" font-size="7">메서드 끝나면 소멸</text>
    <!-- 힙 -->
    <rect x="116" y="28" width="120" height="72" rx="6" fill="#f3ecf7" stroke="#5F0080" stroke-width="1.6"/>
    <text x="176" y="42" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">힙 (모든 스레드 공유)</text>
    <circle cx="140" cy="62" r="7" fill="#fff" stroke="#7d5a9e"/>
    <circle cx="162" cy="70" r="7" fill="#fff" stroke="#7d5a9e"/>
    <circle cx="188" cy="58" r="7" fill="#fff" stroke="#7d5a9e"/>
    <circle cx="210" cy="72" r="7" fill="#fff" stroke="#7d5a9e"/>
    <text x="176" y="92" text-anchor="middle" fill="#b23a00" font-size="7" font-weight="700">← GC 청소 대상</text>
    <!-- 메타스페이스 -->
    <rect x="250" y="28" width="66" height="72" rx="6" fill="#fff" stroke="#7d5a9e" stroke-width="1.4"/>
    <text x="283" y="42" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">메타</text>
    <text x="283" y="52" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">스페이스</text>
    <text x="283" y="68" text-anchor="middle" fill="#5f5a68" font-size="6.5">클래스 정보</text>
    <text x="283" y="80" text-anchor="middle" fill="#5f5a68" font-size="6.5">(힙 바깥)</text>
    <text x="165" y="128" text-anchor="middle" fill="#5f5a68" font-size="7.5">스택은 자동으로 비워지고, 힙만 GC가 따로 청소한다.</text>
    <text x="165" y="145" text-anchor="middle" fill="#5f5a68" font-size="7.5">원(●)은 힙에 사는 객체다.</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">GC가 신경 쓰는 곳은 오직 힙이다. 스택은 메서드가 끝날 때 프레임 단위로 자동 정리된다.</figcaption>
</figure>

## GC는 "도달할 수 없는" 객체를 치운다

그럼 GC는 어떤 객체를 "쓰레기"로 볼까? 기준은 **참조 개수가 아니라 도달 가능성(reachability)**이다.

JVM은 **GC 루트(GC root)**라 부르는 확실한 출발점들 — 지금 실행 중인 스택의 지역 변수, 정적(static) 필드 등 — 에서 시작해 참조를 타고 갈 수 있는 객체를 전부 표시한다. 이 과정에서 **한 번도 닿지 못한 객체는 "도달 불가(unreachable)"**, 즉 살아 있는 코드 어디서도 더 못 쓰는 객체이므로 치운다.

**왜** 참조 개수(레퍼런스 카운팅)가 아니라 도달성일까? 서로만 참조하는 두 객체(A→B, B→A)는 참조 개수가 1로 남지만, 바깥에서 아무도 A·B를 안 가리키면 **둘 다 쓰레기**다. 도달성 기준은 이런 "**순환 참조 섬**"도 정확히 걸러낸다.

<!-- diagram: reachability -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 175" role="img" aria-label="GC 루트에서 참조를 타고 닿는 객체는 생존, 서로만 참조하는 순환 섬은 도달 불가로 수거되는 모습">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="70" y="16" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="9">GC 루트</text>
    <rect x="40" y="24" width="60" height="20" rx="4" fill="#f3ecf7" stroke="#5F0080" stroke-width="1.5"/>
    <text x="70" y="38" text-anchor="middle" fill="#5F0080" font-size="7.5">스택·static</text>
    <!-- 생존 체인 -->
    <circle cx="70" cy="78" r="12" fill="#e3f2ea" stroke="#1f7a4d" stroke-width="1.5"/>
    <text x="70" y="81" text-anchor="middle" fill="#1f7a4d" font-size="8">A</text>
    <circle cx="70" cy="128" r="12" fill="#e3f2ea" stroke="#1f7a4d" stroke-width="1.5"/>
    <text x="70" y="131" text-anchor="middle" fill="#1f7a4d" font-size="8">B</text>
    <line x1="70" y1="44" x2="70" y2="66" stroke="#1f7a4d" stroke-width="1.4" marker-end="url(#ah)"/>
    <line x1="70" y1="90" x2="70" y2="116" stroke="#1f7a4d" stroke-width="1.4" marker-end="url(#ah)"/>
    <text x="118" y="103" fill="#1f7a4d" font-size="7.5" font-weight="700">도달 가능 → 생존</text>
    <!-- 순환 섬 -->
    <circle cx="235" cy="70" r="13" fill="#f7e6dc" stroke="#b23a00" stroke-width="1.5"/>
    <text x="235" y="73" text-anchor="middle" fill="#b23a00" font-size="8">C</text>
    <circle cx="290" cy="110" r="13" fill="#f7e6dc" stroke="#b23a00" stroke-width="1.5"/>
    <text x="290" y="113" text-anchor="middle" fill="#b23a00" font-size="8">D</text>
    <line x1="247" y1="79" x2="279" y2="101" stroke="#b23a00" stroke-width="1.3" marker-end="url(#ahr)"/>
    <line x1="281" y1="103" x2="245" y2="79" stroke="#b23a00" stroke-width="1.3" marker-end="url(#ahr)"/>
    <text x="262" y="150" text-anchor="middle" fill="#b23a00" font-size="7.5" font-weight="700">서로만 참조 → 수거</text>
    <text x="262" y="40" text-anchor="middle" fill="#5f5a68" font-size="7">루트에서 안 닿음</text>
    <defs>
      <marker id="ah" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#1f7a4d"/></marker>
      <marker id="ahr" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker>
    </defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">C·D는 서로 참조해 참조 개수가 0이 아니지만, GC 루트에서 도달할 수 없으므로 함께 수거된다.</figcaption>
</figure>

## 힙은 다시 "젊은 곳"과 "늙은 곳"으로 나뉜다

여기서 핵심 아이디어 하나. 대부분의 객체는 **금방 죽는다**(메서드 안에서 잠깐 쓰고 버리는 임시 객체). 이걸 **약한 세대 가설(weak generational hypothesis)**이라 부른다. 이 성질을 이용하려고, 힙을 다시 두 구역으로 나눈다.

- **영 제너레이션(Young Generation, 젊은 영역)**: 새로 만든 객체가 처음 놓이는 곳. 안에서 다시 **에덴(Eden)** 1개 + **서바이버(Survivor)** 2개로 나뉜다.
- **올드 제너레이션(Old Generation, 늙은 영역)**: 영 영역에서 **여러 번 살아남은** 객체가 옮겨(승격, promotion) 오는 곳.

동작은 이렇다. 새 객체는 **에덴**에 태어난다. 에덴이 꽉 차면 **마이너 GC(Minor GC)**가 돌아, 살아남은 객체만 서바이버로 옮기고 에덴을 통째로 비운다. 서바이버를 오가며 **일정 횟수(예: 15회) 이상 살아남으면** "얘는 오래 쓸 객체구나" 판단해 **올드 영역으로 승격**한다. 올드 영역이 차면 더 무거운 **메이저 GC(Major GC, 또는 Full GC)**가 돈다.

**왜** 이렇게 나눌까? "금방 죽는 객체"가 대부분이라면, **좁은 영 영역만 자주 청소**하는 게 힙 전체를 매번 훑는 것보다 훨씬 싸다. 마이너 GC는 작고 빠르고, 메이저 GC는 크고 느리다 — 그래서 메이저 GC를 **덜 일어나게** 하는 게 튜닝의 목표가 된다.

<!-- diagram: heap-generations -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 185" role="img" aria-label="에덴에서 태어난 객체가 마이너 GC를 거쳐 서바이버를 오가다 여러 번 살아남으면 올드 영역으로 승격되는 흐름">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <!-- Young -->
    <rect x="14" y="26" width="180" height="98" rx="6" fill="none" stroke="#7d5a9e" stroke-width="1.4" stroke-dasharray="4 3"/>
    <text x="104" y="20" text-anchor="middle" fill="#5F0080" font-size="8.5" font-weight="700">영 제너레이션 (젊은 영역)</text>
    <rect x="24" y="40" width="70" height="74" rx="5" fill="#f3ecf7" stroke="#5F0080" stroke-width="1.4"/>
    <text x="59" y="54" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">에덴</text>
    <text x="59" y="66" text-anchor="middle" fill="#5f5a68" font-size="6.5">새 객체 탄생</text>
    <circle cx="44" cy="84" r="5" fill="#fff" stroke="#7d5a9e"/>
    <circle cx="60" cy="94" r="5" fill="#fff" stroke="#7d5a9e"/>
    <circle cx="76" cy="82" r="5" fill="#fff" stroke="#7d5a9e"/>
    <rect x="104" y="40" width="36" height="74" rx="5" fill="#fff" stroke="#b9a9cc" stroke-width="1.3"/>
    <text x="122" y="54" text-anchor="middle" fill="#5f5a68" font-size="7">S0</text>
    <rect x="148" y="40" width="36" height="74" rx="5" fill="#fff" stroke="#b9a9cc" stroke-width="1.3"/>
    <text x="166" y="54" text-anchor="middle" fill="#5f5a68" font-size="7">S1</text>
    <text x="144" y="104" text-anchor="middle" fill="#5f5a68" font-size="6.2">서바이버 2칸</text>
    <!-- Old -->
    <rect x="212" y="26" width="104" height="98" rx="6" fill="#efeaf5" stroke="#5F0080" stroke-width="1.6"/>
    <text x="264" y="20" text-anchor="middle" fill="#5F0080" font-size="8.5" font-weight="700">올드 영역</text>
    <circle cx="234" cy="60" r="6" fill="#fff" stroke="#7d5a9e"/>
    <circle cx="256" cy="74" r="6" fill="#fff" stroke="#7d5a9e"/>
    <circle cx="284" cy="62" r="6" fill="#fff" stroke="#7d5a9e"/>
    <circle cx="266" cy="98" r="6" fill="#fff" stroke="#7d5a9e"/>
    <text x="264" y="118" text-anchor="middle" fill="#5f5a68" font-size="6.2">오래 살아남은 객체</text>
    <!-- promotion arrow -->
    <line x1="186" y1="77" x2="210" y2="77" stroke="#1f7a4d" stroke-width="1.6" marker-end="url(#ap)"/>
    <text x="198" y="70" text-anchor="middle" fill="#1f7a4d" font-size="6.5" font-weight="700">승격</text>
    <text x="104" y="150" text-anchor="middle" fill="#5f5a68" font-size="7.2">에덴이 차면 마이너 GC → 생존자만 서바이버로. 여러 번 살아남으면 올드로 승격.</text>
    <text x="104" y="165" text-anchor="middle" fill="#5f5a68" font-size="7.2">올드가 차면 무거운 메이저(Full) GC.</text>
    <defs><marker id="ap" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#1f7a4d"/></marker></defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">"자주 청소하는 좁은 젊은 영역 + 가끔 청소하는 넓은 늙은 영역" 구조가 GC를 싸게 만든다.</figcaption>
</figure>

## 청소 방식 — 표시하고(Mark), 쓸고(Sweep), 다진다(Compact)

GC가 실제로 메모리를 비우는 절차는 보통 세 단계다.

1. **마크(Mark)**: GC 루트에서 도달 가능한 객체에 "살아 있음" 표시를 한다.
2. **스윕(Sweep)**: 표시 안 된 객체가 있던 자리를 "빈 공간"으로 되돌린다.
3. **컴팩트(Compact)**: 살아남은 객체를 한쪽으로 몰아 붙여 **빈틈(단편화, fragmentation)**을 없앤다.

세 번째 컴팩트가 왜 필요할까? 스윕만 하면 살아남은 객체 사이사이에 빈칸이 흩어져, **총 여유 공간은 넉넉해도 큰 객체 하나 넣을 연속 공간이 없는** 상황이 생긴다. 다져 붙이면 그다음 할당이 빠르고 단순해진다.

문제는, 마크·컴팩트를 하는 동안 객체가 움직이면 참조가 꼬이므로 **애플리케이션 스레드를 잠깐 멈춰야** 한다는 점이다. 이 멈춤을 **스톱 더 월드(Stop-The-World, STW)**라 부른다. 서두의 "몇 초간 요청을 못 처리한 구간"이 바로 이 STW다. **현대 GC의 목표는 이 멈춤 시간을 짧게, 예측 가능하게 만드는 것**이다.

대표적인 GC들을 성격만 잡아 보면:

- **Parallel GC**: 여러 스레드로 빠르게 치우지만 STW가 길다. **처리량(throughput)** 우선.
- **G1 GC(Garbage-First)**: 힙을 잘게 쪼갠 **리전(region)** 단위로, "쓰레기가 많은 곳부터" 조금씩 치운다. 자바 9+ 기본값. **멈춤 시간 목표**를 줄 수 있다.
- **ZGC**: 대부분의 작업을 앱과 **동시에(concurrent)** 처리해, 힙이 수십 GB여도 멈춤을 **밀리초 수준**으로 억제한다.

## 상세 예시 — GC 로그로 STW를 읽어 보기

**이 로그가 하는 일**: 서버에 `-Xlog:gc`(자바 9+ GC 로그 옵션)를 켜면, GC가 돌 때마다 어떤 종류가 얼마나 힙을 줄였고 얼마나 멈췄는지 한 줄씩 남는다. 아래는 (지어낸) 예시다.

```text
[3.512s] GC(7) Pause Young (Normal) (G1 Evacuation Pause)
        512M->48M(2048M) 6.1ms
[9.874s] GC(21) Pause Full (G1 Compaction Pause)
        1987M->1420M(2048M) 812.4ms
```

읽는 법:

- `512M->48M(2048M)`: GC **전 힙 사용량 512M → 후 48M**, 전체 힙 크기 2048M. 젊은 영역 청소로 대부분 회수됐다(임시 객체가 많았다는 뜻).
- `6.1ms` vs `812.4ms`: 첫 줄은 마이너 성격이라 **6밀리초**만 멈췄지만, 둘째 줄 **Full GC는 812밀리초** 동안 STW가 걸렸다.
- 둘째 줄에서 힙이 `1987M->1420M`으로 **조금밖에 안 줄었다**. 이건 위험 신호다 — 올드 영역이 살아 있는 객체로 꽉 차 회수가 거의 안 됐다는 뜻이고, 이 상태가 반복되면 곧 `OutOfMemoryError`다.

여기서 얻는 실전 감각: **Full GC의 빈도와 STW 시간, 그리고 "청소 후에도 안 줄어드는 올드 영역"**을 지켜보면, 메모리 누수(leak)나 힙 부족을 조기에 잡을 수 있다. 처리량이 중요하면 Parallel, 지연이 중요하면 G1/ZGC로 **수집기 자체를 바꾸는 것**도 튜닝의 한 축이다.

## 한 줄 교훈

**GC는 "안 쓰는 객체"를 도달 가능성으로 판별해 힙에서만 치우며, "대부분의 객체는 금방 죽는다"는 가정 위에 젊은/늙은 영역을 나눠 청소 비용을 줄인다 — 그래서 우리가 볼 지표는 결국 STW 시간과 Full GC 빈도다.**
