---
layout: post
title: 'Java 동시성 — 스레드는 왜 서로를 밟고, 락·CAS는 그걸 어떻게 막을까?'
date: 2026-08-24 09:00:00 +0900
image: /assets/img/java-concurrency/hero.jpg
generated: true
tags: [java, study]
sanitized: true
---

카운터 하나를 두 스레드가 각각 100만 번씩 올렸다. 기대값은 200만인데, 찍힌 값은 137만이다. 코드는 `count++` 한 줄뿐이고 버그처럼 보이는 곳이 없다. 범인은 **동시성(concurrency — 여러 작업이 시간 위에서 겹쳐 실행되는 것)**이다. 자바(Java)는 여러 **스레드(thread — 하나의 프로그램 안에서 동시에 도는 실행 흐름)**를 손쉽게 띄울 수 있지만, 그 스레드들이 **같은 데이터를 함께 만지는 순간** 위 같은 "숫자가 새는" 일이 생긴다. 이 글은 왜 `count++`가 안전하지 않은지부터, 그걸 막는 **락(lock)**과 **CAS(Compare-And-Swap)**가 각각 어떤 원리로 동작하는지를 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·코드·시나리오는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/java-concurrency/hero.jpg" alt="여러 스레드가 하나의 공유 자원에 동시에 접근하고 자물쇠가 그것을 지키는 모습을 형상화한 삽화" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">여러 스레드가 같은 자원을 동시에 만지면 값이 깨진다. 락·CAS는 그 접근을 정리하는 두 가지 방식이다.</figcaption>
</figure>

## `count++`은 한 동작이 아니다

먼저 **무엇**부터. `count++`은 소스에선 한 줄이지만, CPU가 실제로 하는 일은 **세 단계**다.

1. **읽기(read)**: 메모리에서 현재 `count` 값을 가져온다.
2. **더하기(modify)**: 그 값에 1을 더한다.
3. **쓰기(write)**: 결과를 다시 메모리에 저장한다.

문제는 이 세 단계 **사이사이에 다른 스레드가 끼어들 수 있다**는 것이다. 스레드 A가 `count`(=10)를 읽고 아직 저장하기 전에, 스레드 B도 같은 10을 읽는다. 둘 다 11을 계산해 저장하면, 두 번 더했는데 결과는 11 — **한 번의 증가가 사라진다**. 이렇게 **실행 순서(타이밍)에 따라 결과가 달라지는 결함**을 **경쟁 상태(race condition)**라 부른다.

**왜** 중요할까? 이 버그는 재현이 어렵다. 타이밍이 맞아야만 터지니, 테스트에선 멀쩡하다가 트래픽이 몰리는 운영에서만 값이 샌다. 서두의 "137만"이 바로 63만 번의 증가가 이렇게 증발한 결과다.

<!-- diagram: lost-update -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 190" role="img" aria-label="두 스레드가 같은 값 10을 읽고 각자 11을 써서 한 번의 증가가 사라지는 경쟁 상태">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="165" y="15" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="9.5">경쟁 상태: 사라진 증가</text>
    <!-- Thread A column -->
    <text x="66" y="34" text-anchor="middle" fill="#5F0080" font-size="8.5" font-weight="700">스레드 A</text>
    <rect x="24" y="42" width="84" height="20" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="66" y="55" text-anchor="middle" fill="#5f5a68" font-size="7.5">읽기: count=10</text>
    <rect x="24" y="92" width="84" height="20" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="66" y="105" text-anchor="middle" fill="#5f5a68" font-size="7.5">쓰기: count=11</text>
    <!-- Thread B column -->
    <text x="264" y="34" text-anchor="middle" fill="#5F0080" font-size="8.5" font-weight="700">스레드 B</text>
    <rect x="222" y="66" width="84" height="20" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="264" y="79" text-anchor="middle" fill="#5f5a68" font-size="7.5">읽기: count=10</text>
    <rect x="222" y="116" width="84" height="20" rx="4" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="264" y="129" text-anchor="middle" fill="#b23a00" font-size="7.5">쓰기: count=11</text>
    <!-- shared value -->
    <text x="165" y="160" text-anchor="middle" fill="#b23a00" font-size="8" font-weight="700">두 번 더했는데 count = 11 (하나 증발)</text>
    <text x="165" y="176" text-anchor="middle" fill="#5f5a68" font-size="7">B가 A의 쓰기 전에 같은 10을 읽은 게 원인</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">읽기와 쓰기 사이에 다른 스레드가 끼어들면, 마지막 쓰기가 앞선 쓰기를 덮어써 증가가 사라진다.</figcaption>
</figure>

## 눈에 안 보이는 또 하나의 문제 — 가시성(visibility)

경쟁 상태만이 아니다. 스레드는 성능을 위해 값을 **CPU 캐시나 레지스터에 잠깐 들고** 있기도 한다. 그러면 한 스레드가 바꾼 값이 **다른 스레드에는 한동안 안 보이는** 일이 생긴다. 이걸 **가시성(visibility) 문제**라 한다.

전형적 예로, 한 스레드가 `while (running) { ... }`로 도는데 다른 스레드가 `running = false`로 바꿔도 **루프가 안 멈추는** 경우가 있다. 도는 스레드가 옛 `running` 값을 캐시에 쥐고 있어서다.

자바는 이를 위해 **`volatile`** 키워드를 준다. `volatile`을 붙인 변수는 항상 **주 메모리(main memory)에서 읽고 쓰도록** 강제해, 한 스레드의 변경이 즉시 다른 스레드에 보인다. 단, **주의**: `volatile`은 **가시성만** 보장하지 원자성(한 덩어리로 실행됨)은 보장하지 않는다. 그래서 `volatile int count; count++`는 여전히 안전하지 않다 — 읽기·쓰기 사이 끼어듦은 그대로 막지 못하기 때문이다.

## 락(lock) — 한 번에 한 스레드만 들어가게 막는다

경쟁 상태의 근본 원인은 "읽기~쓰기 세 단계가 쪼개질 수 있다"는 것이다. 그러니 **그 세 단계를 통째로 묶어, 한 번에 한 스레드만 실행하게** 만들면 된다. 이 "한 스레드만 들어갈 수 있는 구역"을 **임계 영역(critical section)**, 그 출입을 통제하는 장치를 **락(lock, 자물쇠)**이라 한다.

자바의 가장 기본 도구는 **`synchronized`**다. **이 코드가 하는 일**: 메서드나 블록에 `synchronized`를 붙이면, 그 구역에 들어가려는 스레드는 먼저 **모니터 락(monitor lock)**을 얻어야 하고, 못 얻으면 앞 스레드가 나올 때까지 **기다린다(blocking)**.

```java
// 이 코드가 하는 일: 한 번에 한 스레드만 increment 본문을 실행하게 막는다
private int count = 0;

public synchronized void increment() {
    count++;          // 읽기-더하기-쓰기가 하나의 덩어리로 보호됨
}
```

이제 A가 `increment()` 안에 있는 동안 B는 문 앞에서 대기하므로, 서두의 "사라진 증가"가 생기지 않는다. `synchronized`는 **원자성과 가시성을 함께** 보장한다(락을 풀 때 변경이 주 메모리로 반영됨).

더 유연한 도구로 **`ReentrantLock`**도 있다. `lock()` / `unlock()`을 직접 호출해, "일정 시간만 기다리기(`tryLock`)"나 "여러 조건으로 나눠 대기하기" 같은 세밀한 제어가 된다. 대신 **`unlock()`을 `finally`에서 반드시 호출**해야 한다 — 안 그러면 예외가 났을 때 락이 안 풀려 다른 스레드가 영영 못 들어간다.

<!-- diagram: lock-mutex -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 175" role="img" aria-label="스레드 A가 락을 쥐고 임계 영역 안에서 실행하는 동안 스레드 B는 문 앞에서 대기하는 모습">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="165" y="16" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="9.5">락: 한 번에 한 스레드만</text>
    <!-- critical section box -->
    <rect x="118" y="34" width="120" height="86" rx="8" fill="#f3ecf7" stroke="#5F0080" stroke-width="1.6"/>
    <text x="178" y="50" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">임계 영역</text>
    <text x="178" y="63" text-anchor="middle" fill="#5f5a68" font-size="7">count++</text>
    <!-- lock icon -->
    <rect x="163" y="72" width="30" height="22" rx="3" fill="#fff" stroke="#5F0080" stroke-width="1.4"/>
    <path d="M170,72 v-6 a8,8 0 0 1 16,0 v6" fill="none" stroke="#5F0080" stroke-width="1.4"/>
    <!-- Thread A inside -->
    <circle cx="150" cy="105" r="9" fill="#e3f2ea" stroke="#1f7a4d" stroke-width="1.5"/>
    <text x="150" y="108" text-anchor="middle" fill="#1f7a4d" font-size="8">A</text>
    <text x="150" y="132" text-anchor="middle" fill="#1f7a4d" font-size="6.8" font-weight="700">락 획득 → 실행</text>
    <!-- Thread B waiting -->
    <circle cx="40" cy="77" r="11" fill="#f7e6dc" stroke="#b23a00" stroke-width="1.5"/>
    <text x="40" y="80" text-anchor="middle" fill="#b23a00" font-size="8">B</text>
    <text x="40" y="98" text-anchor="middle" fill="#b23a00" font-size="6.8" font-weight="700">대기(blocked)</text>
    <line x1="52" y1="77" x2="114" y2="77" stroke="#b23a00" stroke-width="1.3" stroke-dasharray="3 3" marker-end="url(#axb)"/>
    <text x="165" y="158" text-anchor="middle" fill="#5f5a68" font-size="7.2">A가 unlock 하면 그때 B가 들어간다. 순서가 강제돼 값이 안 샌다.</text>
    <defs><marker id="axb" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker></defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">락은 임계 영역을 "한 명만 통과하는 문"으로 만든다. 나머지는 문 앞에서 기다린다(blocking).</figcaption>
</figure>

## CAS(Compare-And-Swap) — 락 없이, 실패하면 다시

락은 확실하지만 비용이 있다. 대기하는 스레드는 **멈춰서(blocking)** 기다리고, 운영체제가 스레드를 재웠다 깨우는 **문맥 전환(context switch)** 비용도 든다. 경쟁이 심하지 않은데도 항상 문을 잠그는 건 과하다.

그래서 나온 다른 접근이 **CAS(Compare-And-Swap, 비교 후 교체)**다. **무엇**이냐면, CPU가 제공하는 **원자적(더 이상 안 쪼개지는) 단일 명령**으로, 이렇게 동작한다.

> "이 메모리 값이 **아직 내가 아는 그 값(기대값)이면**, 새 값으로 바꿔라. **이미 바뀌었으면** 바꾸지 말고 실패를 알려라."

이 비교와 교체가 **한 명령으로 원자적**이라, 그 사이에 끼어듦이 없다. 만약 실패하면(다른 스레드가 먼저 바꿨으면), **최신 값을 다시 읽어 재시도**한다. 이 "성공할 때까지 반복"을 흔히 **스핀(spin)**이라 한다.

자바에서는 `java.util.concurrent.atomic`의 **`AtomicInteger`** 같은 클래스가 내부적으로 CAS를 쓴다.

```java
// 이 코드가 하는 일: 락 없이, CAS로 안전하게 1씩 올린다
private AtomicInteger count = new AtomicInteger(0);

public void increment() {
    count.incrementAndGet();   // 내부에서 CAS 재시도 루프로 원자 증가
}
```

`incrementAndGet()` 안은 대략 "현재 값 읽기 → +1 계산 → CAS로 교체 시도 → 실패하면 처음부터"를 도는 루프다. 락을 안 걸고 실패 시 다시 하기 때문에, 이런 방식을 **락 프리(lock-free)** 또는 **낙관적(optimistic) 동시성**이라 부른다 — "대부분 충돌 안 날 거야"라고 낙관하고 일단 시도하는 것이다.

<!-- diagram: cas-loop -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 185" role="img" aria-label="CAS 재시도 루프: 현재 값을 읽고 계산한 뒤 기대값과 같으면 교체 성공, 다르면 다시 읽어 재시도">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="165" y="16" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="9.5">CAS 재시도 루프</text>
    <rect x="26" y="30" width="90" height="24" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="71" y="45" text-anchor="middle" fill="#5f5a68" font-size="7.5">현재 값 읽기 (10)</text>
    <rect x="26" y="70" width="90" height="24" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="71" y="85" text-anchor="middle" fill="#5f5a68" font-size="7.5">+1 계산 (11)</text>
    <!-- decision -->
    <path d="M170,58 l30,20 l-30,20 l-30,-20 z" fill="#fff" stroke="#5F0080" stroke-width="1.5"/>
    <text x="170" y="75" text-anchor="middle" fill="#5F0080" font-size="6.8">값이 아직</text>
    <text x="170" y="84" text-anchor="middle" fill="#5F0080" font-size="6.8">10인가?</text>
    <line x1="116" y1="82" x2="140" y2="78" stroke="#7d5a9e" stroke-width="1.3" marker-end="url(#ac)"/>
    <!-- success -->
    <rect x="234" y="64" width="84" height="28" rx="5" fill="#e3f2ea" stroke="#1f7a4d" stroke-width="1.5"/>
    <text x="276" y="76" text-anchor="middle" fill="#1f7a4d" font-size="7.5" font-weight="700">교체 성공</text>
    <text x="276" y="86" text-anchor="middle" fill="#1f7a4d" font-size="6.8">count=11</text>
    <line x1="200" y1="78" x2="232" y2="78" stroke="#1f7a4d" stroke-width="1.4" marker-end="url(#ac2)"/>
    <text x="216" y="72" text-anchor="middle" fill="#1f7a4d" font-size="6.5">예</text>
    <!-- fail loop back -->
    <line x1="170" y1="98" x2="170" y2="140" stroke="#b23a00" stroke-width="1.3"/>
    <text x="185" y="120" fill="#b23a00" font-size="6.5">아니오(딴 스레드가 먼저 바꿈)</text>
    <line x1="170" y1="140" x2="71" y2="140" stroke="#b23a00" stroke-width="1.3"/>
    <line x1="71" y1="140" x2="71" y2="56" stroke="#b23a00" stroke-width="1.3" marker-end="url(#af)"/>
    <text x="120" y="153" text-anchor="middle" fill="#b23a00" font-size="6.8" font-weight="700">최신 값 다시 읽어 재시도(스핀)</text>
    <defs>
      <marker id="ac" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#7d5a9e"/></marker>
      <marker id="ac2" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#1f7a4d"/></marker>
      <marker id="af" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker>
    </defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">CAS는 "기대값과 같을 때만 교체"한다. 그사이 값이 바뀌었으면 실패로 돌려보내 재시도하게 한다.</figcaption>
</figure>

## 락 vs CAS — 언제 무엇을?

둘은 우열이 아니라 **상황에 맞는 선택**이다.

- **락(비관적, pessimistic)**: "충돌이 잦을 것"이라 보고 미리 잠근다. 경쟁이 심하거나 임계 영역이 길 때 유리하다. 대기 스레드는 멈추므로(CPU를 안 태움) 오래 기다려도 낭비가 적다. 단, **데드락(deadlock — 서로 상대의 락을 기다리며 둘 다 못 나아가는 교착)** 위험과 문맥 전환 비용이 있다.
- **CAS(낙관적, optimistic)**: "충돌이 드물 것"이라 보고 일단 시도한다. 경쟁이 약하고 연산이 짧을 때(카운터·플래그 등) 락보다 빠르다. 단, **경쟁이 심하면 재시도(스핀)가 폭증**해 오히려 CPU를 낭비한다.

CAS의 유명한 함정 하나만 짚자. 값이 **A→B→A로 되돌아온** 경우, CAS는 "여전히 A네, 안 바뀌었군" 하고 속는다. 이걸 **ABA 문제**라 한다. 자바는 값에 **버전 번호(스탬프)**를 함께 붙이는 `AtomicStampedReference`로 이를 막는다 — 값이 같아도 스탬프가 다르면 바뀐 걸로 본다.

실무에선 대개 **직접 락/CAS를 짜기보다** 이미 이 원리로 만들어진 도구를 쓴다. `ConcurrentHashMap`, `AtomicLong`, `java.util.concurrent`의 큐·세마포어 등이 내부에서 락 또는 CAS를 적절히 섞어 쓴다.

## 상세 예시 — 세 가지 카운터의 결과 비교

**이 예시가 하는 일**: 같은 "2개 스레드가 각 100만 번 증가" 시나리오를 세 방식으로 돌렸을 때의 (지어낸) 결과다.

```text
방식                    최종 count(기대 2,000,000)   상대 소요
① int count++          1,370,412  ❌ (값 샘)        1.0x (하지만 틀림)
② synchronized         2,000,000  ✅               3.8x
③ AtomicInteger(CAS)   2,000,000  ✅               1.6x
```

읽는 법:

- **①**은 앞서 본 경쟁 상태로 63만 번이 증발했다. 빠르지만 **틀린 답**이라 의미가 없다.
- **②** `synchronized`는 정답을 주지만, 매 증가마다 락을 잡고 푸는 비용 탓에 가장 느리다. 임계 영역이 `count++` 한 줄로 아주 짧아, 이 경우엔 락 오버헤드가 상대적으로 크게 드러난다.
- **③** `AtomicInteger`(CAS)도 정답이면서 ②보다 빠르다. 연산이 짧고 락 대기가 없어, 이런 단순 카운터에는 CAS 계열이 잘 맞는다.

주의: 이 순위는 **"짧은 연산 + 중간 경쟁"**이라는 조건에서의 이야기다. 임계 영역이 길거나 경쟁이 극심하면 CAS의 재시도가 늘어 `synchronized`가 더 나을 수도 있다. **정답은 "측정해서 고른다"**이다.

## 한 줄 교훈

**여러 스레드가 같은 데이터를 만지면 경쟁 상태와 가시성 문제가 생기며, 이를 락(한 번에 하나만 들여보내는 비관적 방식)이나 CAS(일단 시도하고 실패 시 재시도하는 낙관적 방식)로 막는다 — 무엇을 쓸지는 경쟁 강도와 임계 영역 길이를 재서 정한다.**
