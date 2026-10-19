---
layout: post
title: '프로세스·스레드·CPU 스케줄링 — CPU 하나가 수백 개의 일을 동시에 하는 척하는 법'
date: 2026-10-19 09:00:00 +0900
image: /assets/img/process-thread-scheduling/hero.jpg
generated: true
tags: [os, study]
sanitized: true
---

노트북의 CPU 코어는 많아야 십여 개인데, 작업 관리자를 열면 실행 중인 프로그램은 수백 개다. 브라우저 탭, 메신저, IDE, 음악 플레이어가 전부 "동시에" 돌아가는 것처럼 보인다. 사실 코어 하나는 한순간에 **한 가지 일만** 한다. 그런데도 동시에 도는 것처럼 보이는 건, 운영체제(OS — 하드웨어를 관리하고 프로그램들에 나눠 주는 기본 소프트웨어)가 아주 짧은 시간 단위로 일을 **번갈아** 시키기 때문이다. 이 글은 그 "일"의 단위인 **프로세스(process)**와 **스레드(thread)**가 무엇인지, OS가 이들을 어떻게 갈아 끼우는지(**컨텍스트 스위치**), 그리고 "다음엔 누구를 돌릴지" 정하는 **CPU 스케줄링** 알고리즘을 차례로 본다. 마지막엔 작업 3개를 FCFS·SJF·라운드 로빈으로 돌려 대기 시간을 직접 계산해 비교한다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·작업·명령 출력은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다. (단, 리눅스 nice 가중치 1024·335는 커널 소스의 표 값을 그대로 썼습니다.)</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/process-thread-scheduling/hero.jpg" alt="보라색 배경의 주방에서 요리사 한 명이 모래시계가 얹힌 냄비 네 개 사이를 바쁘게 오가고 있다" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">요리사(CPU 코어)는 한 명인데 냄비(작업)는 여럿이다. 모래시계가 다 되면 다음 냄비로 옮겨 간다 — 이게 시분할 스케줄링이다.</figcaption>
</figure>

## 프로세스 — 실행 중인 프로그램 한 벌

**무엇인가.** 디스크에 저장된 실행 파일은 그냥 "프로그램"이다. 이걸 실행해서 메모리에 올라가 돌아가기 시작한 것을 **프로세스**라 부른다. 같은 프로그램(예: 크롬)을 두 번 실행하면 프로세스가 두 개 생긴다.

프로세스는 자기만의 **주소 공간(address space — 이 프로세스가 볼 수 있는 메모리 범위)**을 가진다. 그 안은 대략 네 구역으로 나뉜다.

- **코드(text)**: 실행할 기계어 명령
- **데이터(data)**: 전역 변수·static 변수
- **힙(heap)**: 실행 중에 동적으로 할당하는 메모리 (Java의 `new`로 만든 객체 등)
- **스택(stack)**: 함수 호출 정보·지역 변수

OS는 프로세스마다 **PCB(Process Control Block — 프로세스 관리 장부)**를 둔다. 여기엔 프로세스 ID(PID), 현재 상태, 마지막으로 실행하던 위치(**PC, Program Counter — 다음에 실행할 명령의 주소**), CPU 레지스터 값, 메모리 정보, 열어 둔 파일 목록 등이 적힌다.

**왜 중요한가.** 프로세스끼리는 주소 공간이 **분리**돼 있다. A 프로세스가 잘못된 메모리를 건드려 죽어도 B 프로세스는 멀쩡하다. 대신 프로세스끼리 데이터를 주고받으려면 파이프·소켓·공유 메모리 같은 **IPC(Inter-Process Communication — 프로세스 간 통신)** 장치를 따로 써야 한다.

## 스레드 — 프로세스 안의 실행 흐름

**무엇인가.** **스레드**는 프로세스 안에서 실제로 명령을 하나씩 실행해 나가는 **흐름**이다. 프로세스 하나에 스레드가 여러 개 있을 수 있다.

핵심은 **무엇을 공유하고 무엇을 따로 갖느냐**다.

- **공유**: 코드·데이터·힙, 열린 파일 — 즉 주소 공간 전체
- **따로**: 스택, 레지스터 값, PC — 즉 "지금 어디까지 실행했나"

<!-- diagram: process-vs-thread -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 200" role="img" aria-label="왼쪽은 스레드가 하나인 프로세스로 코드, 데이터, 힙, 스택이 한 벌씩 있다. 오른쪽은 스레드가 셋인 프로세스로 코드, 데이터, 힙, 열린 파일은 하나를 함께 쓰고, 스택과 레지스터·PC는 스레드마다 따로 있다.">
<text x="80" y="16" font-size="9" fill="#1a1720" font-weight="700" text-anchor="middle">단일 스레드 프로세스</text>
<rect x="15" y="24" width="130" height="164" rx="8" fill="#fff" stroke="#b9a9cc"/>
<rect x="27" y="34" width="106" height="22" rx="4" fill="#e9e1f1"/><text x="80" y="49" font-size="8" fill="#1a1720" text-anchor="middle">코드</text>
<rect x="27" y="60" width="106" height="22" rx="4" fill="#e9e1f1"/><text x="80" y="75" font-size="8" fill="#1a1720" text-anchor="middle">데이터</text>
<rect x="27" y="86" width="106" height="22" rx="4" fill="#e9e1f1"/><text x="80" y="101" font-size="8" fill="#1a1720" text-anchor="middle">힙</text>
<rect x="27" y="118" width="106" height="30" rx="4" fill="#5F0080"/><text x="80" y="131" font-size="8" fill="#fff" text-anchor="middle">스택</text><text x="80" y="142" font-size="7" fill="#fff" text-anchor="middle">레지스터 · PC</text>
<text x="80" y="170" font-size="7.5" fill="#5f5a68" text-anchor="middle">실행 흐름 1개</text>
<text x="252" y="16" font-size="9" fill="#1a1720" font-weight="700" text-anchor="middle">멀티 스레드 프로세스</text>
<rect x="165" y="24" width="174" height="164" rx="8" fill="#fff" stroke="#b9a9cc"/>
<rect x="177" y="34" width="150" height="74" rx="4" fill="#e9e1f1" stroke="#7d5a9e" stroke-dasharray="3 2"/>
<text x="252" y="50" font-size="8" fill="#1a1720" text-anchor="middle">코드 · 데이터 · 힙 · 열린 파일</text>
<text x="252" y="66" font-size="8" fill="#1f7a4d" font-weight="700" text-anchor="middle">← 모든 스레드가 공유 →</text>
<text x="252" y="84" font-size="7" fill="#5f5a68" text-anchor="middle">(같은 객체를 동시에 건드릴 수 있다</text>
<text x="252" y="96" font-size="7" fill="#b23a00" text-anchor="middle">= 동기화가 필요하다)</text>
<rect x="177" y="118" width="46" height="30" rx="4" fill="#5F0080"/><text x="200" y="131" font-size="7.5" fill="#fff" text-anchor="middle">스택</text><text x="200" y="142" font-size="6.5" fill="#fff" text-anchor="middle">레지·PC</text>
<rect x="229" y="118" width="46" height="30" rx="4" fill="#5F0080"/><text x="252" y="131" font-size="7.5" fill="#fff" text-anchor="middle">스택</text><text x="252" y="142" font-size="6.5" fill="#fff" text-anchor="middle">레지·PC</text>
<rect x="281" y="118" width="46" height="30" rx="4" fill="#5F0080"/><text x="304" y="131" font-size="7.5" fill="#fff" text-anchor="middle">스택</text><text x="304" y="142" font-size="6.5" fill="#fff" text-anchor="middle">레지·PC</text>
<text x="200" y="160" font-size="7.5" fill="#1a1720" text-anchor="middle">T1</text>
<text x="252" y="160" font-size="7.5" fill="#1a1720" text-anchor="middle">T2</text>
<text x="304" y="160" font-size="7.5" fill="#1a1720" text-anchor="middle">T3</text>
<text x="252" y="178" font-size="7.5" fill="#5f5a68" text-anchor="middle">실행 흐름 3개 — 각자 따로</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">스레드는 "주소 공간은 같이 쓰고, 실행 위치만 따로 갖는" 실행 흐름이다.</figcaption>
</figure>

**왜 중요한가.** 공유 덕분에 장단점이 정확히 갈린다.

- **장점**: 스레드끼리는 같은 힙을 보므로 데이터를 그냥 주고받는다(IPC 불필요). 새로 만들 때도 주소 공간을 복사할 필요가 없어 프로세스보다 가볍다.
- **단점**: 여러 스레드가 같은 변수를 동시에 고치면 값이 꼬인다(**경쟁 조건, race condition**). 그래서 락 같은 동기화가 필요하다. 그리고 한 스레드가 잘못된 메모리 접근으로 프로세스를 죽이면 **같은 프로세스의 스레드가 전부** 함께 죽는다.

예를 들어 웹 서버(Tomcat 등)는 프로세스 하나 안에 요청 처리용 스레드를 수십~수백 개 띄운다. 요청마다 프로세스를 새로 만들면 무겁고, 스레드는 커넥션 풀·캐시 같은 힙 객체를 그대로 같이 쓸 수 있어서다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">참고: 리눅스 커널 안에서는 프로세스와 스레드를 둘 다 <b>task</b>(<code>task_struct</code>)라는 같은 구조로 다룬다. 차이는 <code>clone()</code>으로 만들 때 주소 공간·파일 테이블 등을 <i>공유하도록</i> 했느냐뿐이다. 스케줄러가 실제로 CPU에 올리는 단위도 이 task(=스레드)다. Java의 일반 스레드(platform thread)는 OS 스레드와 1:1로 대응한다.</p>

## 프로세스(스레드)의 상태

CPU에 올라갈 수 있는 실행 흐름은 늘 다섯 상태 중 하나에 있다.

- **New(생성)**: 막 만들어지는 중
- **Ready(준비)**: 실행할 준비는 끝났고 CPU 차례를 기다리는 중
- **Running(실행)**: 지금 CPU에서 명령을 실행 중 — 코어 하나당 한 개뿐
- **Waiting / Blocked(대기)**: 디스크 읽기·네트워크 응답·락 같은 **바깥 일**을 기다리는 중. CPU를 줘도 할 일이 없다
- **Terminated(종료)**: 끝남

<!-- diagram: state-transition -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 190" role="img" aria-label="상태 전이도. New에서 승인되면 Ready로 간다. Ready에서 디스패치되면 Running이 된다. Running은 타임 슬라이스를 다 쓰거나 선점되면 Ready로 돌아가고, I/O나 락을 요청하면 Waiting으로 가며, 끝나면 Terminated가 된다. Waiting은 I/O가 끝나면 Ready로 간다.">
<defs>
<marker id="pts-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5F0080"/></marker>
<marker id="pts-o" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#b23a00"/></marker>
<marker id="pts-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#1f7a4d"/></marker>
</defs>
<rect x="8" y="40" width="54" height="26" rx="13" fill="#fff" stroke="#b9a9cc"/><text x="35" y="57" font-size="8.5" fill="#1a1720" text-anchor="middle">New</text>
<rect x="96" y="40" width="62" height="26" rx="13" fill="#e9e1f1" stroke="#7d5a9e"/><text x="127" y="57" font-size="8.5" fill="#1a1720" font-weight="700" text-anchor="middle">Ready</text>
<rect x="200" y="40" width="66" height="26" rx="13" fill="#5F0080"/><text x="233" y="57" font-size="8.5" fill="#fff" font-weight="700" text-anchor="middle">Running</text>
<rect x="282" y="104" width="54" height="26" rx="13" fill="#fff" stroke="#b9a9cc"/><text x="309" y="121" font-size="8" fill="#1a1720" text-anchor="middle">Terminated</text>
<rect x="148" y="150" width="66" height="26" rx="13" fill="#fff" stroke="#b23a00"/><text x="181" y="167" font-size="8.5" fill="#b23a00" font-weight="700" text-anchor="middle">Waiting</text>
<line x1="62" y1="53" x2="94" y2="53" stroke="#5F0080" stroke-width="1.2" marker-end="url(#pts-a)"/>
<text x="78" y="47" font-size="6.5" fill="#5f5a68" text-anchor="middle">승인</text>
<path d="M158,47 Q179,30 198,47" fill="none" stroke="#5F0080" stroke-width="1.2" marker-end="url(#pts-a)"/>
<text x="179" y="30" font-size="7" fill="#5F0080" text-anchor="middle">디스패치</text>
<path d="M200,60 Q179,80 160,60" fill="none" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#pts-a)"/>
<text x="179" y="90" font-size="6.5" fill="#7d5a9e" text-anchor="middle">시간 다 씀 / 선점</text>
<path d="M240,66 Q240,140 216,158" fill="none" stroke="#b23a00" stroke-width="1.2" marker-end="url(#pts-o)"/>
<text x="268" y="148" font-size="6.5" fill="#b23a00" text-anchor="middle">I/O · 락 요청</text>
<path d="M148,163 Q110,150 120,68" fill="none" stroke="#1f7a4d" stroke-width="1.2" marker-end="url(#pts-g)"/>
<text x="90" y="128" font-size="6.5" fill="#1f7a4d" text-anchor="middle">I/O 완료</text>
<path d="M262,66 Q290,80 302,102" fill="none" stroke="#5F0080" stroke-width="1.2" marker-end="url(#pts-a)"/>
<text x="300" y="80" font-size="6.5" fill="#5f5a68" text-anchor="middle">종료</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">스케줄러가 고르는 대상은 Ready 줄에 선 것들뿐이다. Waiting 상태는 줄에서 빠져 있다가 I/O가 끝나야 다시 선다.</figcaption>
</figure>

여기서 중요한 구분이 하나 나온다. Running에서 내려오는 길이 두 갈래다.

- **스스로 내려옴(자발적, voluntary)**: I/O나 락을 기다리느라 CPU를 반납 → Waiting
- **강제로 내려옴(비자발적, involuntary)**: 아직 할 일이 있는데 OS가 "네 차례 끝" 하고 뺏음 → Ready. 이렇게 뺏는 걸 **선점(preemption)**이라 한다

이 둘의 비율을 보면 프로그램 성격이 보인다. 뒤의 예시에서 직접 확인한다.

## 컨텍스트 스위치 — 갈아 끼우는 비용

**무엇인가.** CPU에서 돌던 실행 흐름 A를 내리고 B를 올리는 일을 **컨텍스트 스위치(context switch)**라 한다. 순서는 이렇다.

1. A의 레지스터·PC를 A의 장부(PCB, 스레드라면 TCB)에 저장
2. B의 장부에서 레지스터·PC를 꺼내 CPU에 복원
3. B가 **다른 프로세스**라면 주소 공간도 바꾼다(페이지 테이블 교체)

**왜 중요한가.** 스위치하는 동안 CPU는 "쓸모 있는 일"을 하지 않는다. 직접 비용은 보통 마이크로초(μs) 단위로 작지만, **간접 비용**이 더 크다. 새로 올라온 B가 쓰는 데이터는 CPU 캐시(코어 옆의 아주 빠른 메모리)에 아직 없어서, 한동안 느린 메인 메모리에서 읽어 와야 한다(캐시가 "식은" 상태).

같은 프로세스의 스레드끼리 바꿀 때는 3번(주소 공간 교체)이 필요 없다. 그래서 **스레드 전환이 프로세스 전환보다 싸다**. 주소 공간을 바꾸면 주소 변환 캐시(**TLB, Translation Lookaside Buffer**)를 비우거나 구분해야 하는 비용이 붙기 때문이다.

결론: 스위치는 공짜가 아니다. 그래서 스케줄러는 "얼마나 자주 바꿀지"를 늘 저울질한다.

## CPU 스케줄링 — 다음엔 누구를 올릴까

Ready 줄에 여러 개가 서 있을 때 **다음에 누구를 Running으로 올릴지** 정하는 것이 **CPU 스케줄러**의 일이다. 좋은 스케줄링이 노리는 지표는 이렇다.

- **대기 시간(waiting time)**: Ready 줄에서 기다린 시간의 합
- **반환 시간(turnaround time)**: 도착부터 완료까지 걸린 전체 시간
- **응답 시간(response time)**: 도착부터 **처음** CPU를 받기까지 — 사람이 "반응이 빠르다"고 느끼는 지표
- **처리량(throughput)**: 단위 시간당 끝낸 작업 수

이 지표들은 서로 부딪힌다. 응답 시간을 줄이려고 자주 바꾸면 스위치 비용이 늘어 처리량이 떨어진다.

스케줄러는 크게 두 부류다.

- **비선점(non-preemptive)**: 한 번 CPU를 받으면 스스로 내려놓을 때(끝나거나 I/O 대기)까지 계속 쓴다
- **선점(preemptive)**: OS가 타이머 인터럽트 등으로 중간에 뺏을 수 있다. 오늘날 범용 OS(리눅스·윈도우·macOS)는 선점형이다

### FCFS (First-Come, First-Served — 먼저 온 순서대로)

줄 선 순서대로 끝까지 실행한다. 비선점.

- **장점**: 가장 단순하고 공정해 보인다.
- **단점**: 긴 작업이 앞에 서면 뒤의 짧은 작업들이 전부 기다린다. 마트 계산대에서 카트 가득 실은 손님 뒤에 우유 하나 든 손님들이 줄줄이 서 있는 모습이다. 이를 **호위 효과(convoy effect)**라 부른다.

### SJF (Shortest Job First — 짧은 작업 먼저)

실행 시간(**CPU burst — 한 번 CPU를 잡고 연속으로 쓰는 시간**)이 가장 짧은 작업부터 실행한다.

- **장점**: 모든 작업이 동시에 와 있다면, 평균 대기 시간이 **가능한 최소**가 된다(증명된 성질).
- **단점 1**: 실행 시간을 **미리 알 수 없다**. 실제 OS는 과거 burst 길이로 추정만 할 수 있다.
- **단점 2**: 짧은 작업이 계속 들어오면 긴 작업은 영영 못 돈다. 이를 **기아(starvation)**라 한다.

선점형 버전은 **SRTF(Shortest Remaining Time First)**로, 남은 시간이 더 짧은 작업이 도착하면 지금 것을 뺏는다.

### 우선순위(Priority) 스케줄링

작업마다 우선순위를 매기고 높은 것부터 돌린다. SJF는 "짧을수록 높은 우선순위"인 특수한 경우다. 마찬가지로 기아가 생기므로, 오래 기다린 작업의 우선순위를 서서히 올려 주는 **에이징(aging)**으로 막는다.

### RR (Round Robin — 돌아가며 조금씩)

각 작업에 **타임 퀀텀(time quantum — 한 번에 쓸 수 있는 시간 조각, 타임 슬라이스라고도 함)**만큼만 CPU를 주고, 다 쓰면 줄 맨 뒤로 보낸다. 선점형. 대표 그림의 요리사가 모래시계가 끝날 때마다 다음 냄비로 옮겨 가는 방식이다.

- **장점**: 아무도 오래 굶지 않고, 응답 시간이 짧다. 대화형 시스템에 맞다.
- **퀀텀 크기가 핵심이다**: 너무 크면 사실상 FCFS가 된다. 너무 작으면 일보다 스위치에 시간을 더 쓴다.

### MLFQ (Multi-Level Feedback Queue — 다단계 피드백 큐)

우선순위가 다른 줄 여러 개를 두고, **행동을 보고 줄을 옮긴다**. 퀀텀을 다 쓰고도 계속 CPU를 원하면(계산 위주) 낮은 줄로 내리고, 금방 I/O로 내려놓는 작업(대화형)은 높은 줄에 둔다. 실행 시간을 미리 몰라도 "짧은/대화형 작업 우대"라는 SJF의 효과를 흉내 낼 수 있어, 여러 OS 스케줄러의 기본 아이디어가 됐다.

## 상세 예시 1 — 같은 작업 3개, 알고리즘만 바꿔 보기

작업 3개가 **0초에 동시에** P1, P2, P3 순서로 도착했다고 하자. 각자의 CPU burst는 다음과 같다(단위: ms, 스위치 비용은 0으로 단순화).

| 작업 | 도착 | CPU burst |
|---|---|---|
| P1 | 0 | 6 |
| P2 | 0 | 3 |
| P3 | 0 | 1 |

세 알고리즘으로 돌리면 실행 순서(간트 차트 — 시간축 위에 누가 언제 CPU를 썼는지 그린 막대 그림)는 이렇다. RR의 퀀텀은 2ms다.

<!-- diagram: gantt-fcfs-sjf-rr -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 170" role="img" aria-label="간트 차트 세 줄. FCFS는 P1이 0부터 6까지, P2가 6부터 9까지, P3가 9부터 10까지 실행한다. SJF는 P3가 0부터 1, P2가 1부터 4, P1이 4부터 10까지 실행한다. 퀀텀 2의 라운드 로빈은 P1 0~2, P2 2~4, P3 4~5, P1 5~7, P2 7~8, P1 8~10 순서로 실행한다.">
<text x="8" y="40" font-size="8.5" fill="#1a1720" font-weight="700">FCFS</text>
<rect x="70" y="26" width="150" height="22" fill="#5F0080"/><text x="145" y="41" font-size="8" fill="#fff" text-anchor="middle">P1</text>
<rect x="220" y="26" width="75" height="22" fill="#7d5a9e"/><text x="257" y="41" font-size="8" fill="#fff" text-anchor="middle">P2</text>
<rect x="295" y="26" width="25" height="22" fill="#b9a9cc"/><text x="307" y="41" font-size="8" fill="#1a1720" text-anchor="middle">P3</text>
<text x="8" y="80" font-size="8.5" fill="#1a1720" font-weight="700">SJF</text>
<rect x="70" y="66" width="25" height="22" fill="#b9a9cc"/><text x="82" y="81" font-size="8" fill="#1a1720" text-anchor="middle">P3</text>
<rect x="95" y="66" width="75" height="22" fill="#7d5a9e"/><text x="132" y="81" font-size="8" fill="#fff" text-anchor="middle">P2</text>
<rect x="170" y="66" width="150" height="22" fill="#5F0080"/><text x="245" y="81" font-size="8" fill="#fff" text-anchor="middle">P1</text>
<text x="8" y="116" font-size="8.5" fill="#1a1720" font-weight="700">RR</text>
<text x="8" y="126" font-size="6.5" fill="#5f5a68">퀀텀 2</text>
<rect x="70" y="106" width="50" height="22" fill="#5F0080"/><text x="95" y="121" font-size="8" fill="#fff" text-anchor="middle">P1</text>
<rect x="120" y="106" width="50" height="22" fill="#7d5a9e"/><text x="145" y="121" font-size="8" fill="#fff" text-anchor="middle">P2</text>
<rect x="170" y="106" width="25" height="22" fill="#b9a9cc"/><text x="182" y="121" font-size="8" fill="#1a1720" text-anchor="middle">P3</text>
<rect x="195" y="106" width="50" height="22" fill="#5F0080"/><text x="220" y="121" font-size="8" fill="#fff" text-anchor="middle">P1</text>
<rect x="245" y="106" width="25" height="22" fill="#7d5a9e"/><text x="257" y="121" font-size="8" fill="#fff" text-anchor="middle">P2</text>
<rect x="270" y="106" width="50" height="22" fill="#5F0080"/><text x="295" y="121" font-size="8" fill="#fff" text-anchor="middle">P1</text>
<g stroke="#ffffff" stroke-width="1"><line x1="220" y1="26" x2="220" y2="48"/><line x1="295" y1="26" x2="295" y2="48"/><line x1="95" y1="66" x2="95" y2="88"/><line x1="170" y1="66" x2="170" y2="88"/><line x1="120" y1="106" x2="120" y2="128"/><line x1="170" y1="106" x2="170" y2="128"/><line x1="195" y1="106" x2="195" y2="128"/><line x1="245" y1="106" x2="245" y2="128"/><line x1="270" y1="106" x2="270" y2="128"/></g>
<line x1="70" y1="140" x2="320" y2="140" stroke="#5f5a68" stroke-width="0.8"/>
<g font-size="7" fill="#5f5a68" text-anchor="middle">
<text x="70" y="151">0</text><text x="95" y="151">1</text><text x="120" y="151">2</text><text x="145" y="151">3</text><text x="170" y="151">4</text><text x="195" y="151">5</text><text x="220" y="151">6</text><text x="245" y="151">7</text><text x="270" y="151">8</text><text x="295" y="151">9</text><text x="320" y="151">10</text>
</g>
<text x="195" y="164" font-size="7" fill="#5f5a68" text-anchor="middle">시간 (ms)</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">셋 다 10ms에 모든 일이 끝난다. 달라지는 건 "누가 얼마나 기다렸나"다.</figcaption>
</figure>

RR 순서를 한 줄씩 따라가면 이렇다. 줄은 처음에 `[P1, P2, P3]`다.

1. 0~2: P1 실행(남은 4) → 줄 뒤로: `[P2, P3, P1]`
2. 2~4: P2 실행(남은 1) → `[P3, P1, P2]`
3. 4~5: P3 실행, 1ms라 **끝** → `[P1, P2]`
4. 5~7: P1 실행(남은 2) → `[P2, P1]`
5. 7~8: P2 실행, **끝** → `[P1]`
6. 8~10: P1 실행, **끝**

이제 지표를 계산한다. 모두 0초에 도착했으므로 **반환 시간 = 완료 시각**, **대기 시간 = 반환 시간 − burst**, **응답 시간 = 처음 CPU를 받은 시각**이다.

| 알고리즘 | 대기 (P1 / P2 / P3) | 평균 대기 | 평균 반환 | 평균 응답 | 스위치 횟수 |
|---|---|---|---|---|---|
| FCFS | 0 / 6 / 9 | **5.00** | 8.33 | 5.00 | 2 |
| SJF | 4 / 1 / 0 | **1.67** | 5.00 | 1.67 | 2 |
| RR (q=2) | 4 / 5 / 4 | **4.33** | 7.67 | 2.00 | 5 |

검산 한 줄씩:
- FCFS 반환 6·9·10 → 합 25 / 3 = 8.33. 대기 0+6+9 = 15 / 3 = 5.00
- SJF 반환 10·4·1 → 합 15 / 3 = 5.00. 대기 4+1+0 = 5 / 3 = 1.67
- RR 반환 10·8·5 → 합 23 / 3 = 7.67. 대기 4+5+4 = 13 / 3 = 4.33. 응답 0·2·4 → 6 / 3 = 2.00

표에서 읽을 것은 세 가지다.

1. **FCFS는 순서 운이 나쁘면 크게 손해 본다.** 6ms짜리 P1이 맨 앞에 서는 바람에 1ms짜리 P3가 9ms를 기다렸다. 호위 효과 그대로다.
2. **SJF가 평균 대기는 최소지만, 미래를 알아야 한다.** 실제로는 burst 길이를 모르니 이 숫자는 "이론상 하한"이다.
3. **RR은 대기 시간은 SJF만 못하지만 응답 시간이 짧다(2.00).** P3는 4ms 만에 처음 CPU를 받았다(FCFS에선 9ms). 대신 스위치가 5번으로 2.5배 늘었다. 실제로는 스위치마다 비용이 붙으므로, 퀀텀을 1로 줄이면 응답은 더 빨라져도 스위치 비용이 더 커진다.

## 상세 예시 2 — 리눅스는 어떻게 나눠 주나 (CFS와 nice)

실제 리눅스는 위 알고리즘 하나를 그대로 쓰지 않는다. 오랫동안(2.6.23~6.5) 기본 스케줄러는 **CFS(Completely Fair Scheduler — 완전 공정 스케줄러)**였다. 아이디어는 "모든 task가 CPU를 **가중치에 비례해 공평하게** 나눠 쓴 것처럼 만들자"다.

- 각 task는 **vruntime(가상 실행 시간)**을 가진다. CPU를 쓴 만큼 늘어나는데, 가중치가 높을수록 **천천히** 는다.
- 스케줄러는 항상 **vruntime이 가장 작은**(=가장 덜 쓴) task를 고른다. 이를 빨리 찾으려고 레드-블랙 트리(자동으로 균형을 맞추는 정렬 트리)에 task를 정렬해 둔다.
- 가중치는 **nice 값**(−20~19, 낮을수록 우선)으로 정한다. nice 0의 가중치는 **1024**, nice 5는 **335**다. nice가 1 오를 때마다 가중치가 약 1.25배씩 줄도록 표가 짜여 있다.

이 코드가 하는 일: CPU를 계속 쓰는 프로세스 두 개를 **같은 코어 하나**에 묶고, 하나는 nice 0, 하나는 nice 5로 띄운다.

```bash
# 코어 0에만 묶어서(taskset -c 0) 무한 루프 두 개 실행
taskset -c 0 sh -c 'while :; do :; done' &          # nice 0
taskset -c 0 nice -n 5 sh -c 'while :; do :; done' & # nice 5
top -p "$(pgrep -d, -f 'while :')"
```

가중치 비율대로라면 CPU 몫은 이렇게 나온다.

- nice 0: 1024 / (1024 + 335) = 1024 / 1359 ≈ **75.3%**
- nice 5: 335 / 1359 ≈ **24.7%**

(합 100.0%.) 그래서 `top`의 `%CPU`는 대략 75 대 25 근처로 보인다(예시 값, 환경에 따라 조금씩 다르다). "nice 5면 조금 양보한다" 정도가 아니라 **세 배 가까이** 차이 나는 셈이다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">리눅스 6.6(2023년 말)부터는 기본 스케줄러가 CFS에서 <b>EEVDF</b>(Earliest Eligible Virtual Deadline First)로 바뀌었다. "가중치에 비례한 공평한 몫"이라는 목표와 nice 가중치 표는 그대로이고, 다음 task를 고르는 규칙에 "가상 마감 시각"을 더해 지연에 민감한 task를 더 잘 챙기도록 다듬은 것이다.</p>

### 내 프로그램은 어느 쪽으로 내려오나 — 컨텍스트 스위치 세어 보기

앞에서 본 "자발적 vs 비자발적" 스위치는 리눅스에서 직접 셀 수 있다.

이 코드가 하는 일: 프로세스의 누적 컨텍스트 스위치 횟수를 `/proc`에서 읽는다.

```bash
grep ctxt /proc/<PID>/status
```

```text
# (예시) 계속 계산만 하는 프로세스 — 위의 무한 루프
voluntary_ctxt_switches:        3
nonvoluntary_ctxt_switches:     48211

# (예시) 대부분 네트워크 응답을 기다리는 웹 서버 워커 스레드
voluntary_ctxt_switches:        91530
nonvoluntary_ctxt_switches:     127
```

- **nonvoluntary가 압도적** → CPU를 더 쓰고 싶은데 계속 뺏기는 중. CPU가 모자라거나 계산 위주(CPU-bound) 작업이다.
- **voluntary가 압도적** → 대부분 I/O·락을 기다리며 스스로 내려온다. I/O 위주(I/O-bound) 작업이다. 이런 작업이 느리다면 CPU보다 **기다리는 대상**(DB, 외부 API, 락 경합)을 먼저 의심한다.

시스템 전체의 초당 스위치 수는 `vmstat 1`의 `cs` 열로 본다. 스레드 수를 코어 수보다 훨씬 많이 늘렸는데 처리량은 그대로고 `cs`만 치솟는다면, 일보다 갈아 끼우는 데 시간을 쓰고 있다는 신호다.

## 정리

| 개념 | 한 줄 요약 |
|---|---|
| 프로세스 | 실행 중인 프로그램 한 벌. 주소 공간이 **분리**돼 안전하지만 통신이 번거롭다 |
| 스레드 | 프로세스 안의 실행 흐름. 주소 공간을 **공유**해 가볍지만 동기화가 필요하다 |
| 컨텍스트 스위치 | 실행 흐름 교체. 레지스터 저장·복원 + (프로세스 간이면) 주소 공간 교체 + 캐시 식음 |
| FCFS | 단순. 긴 작업 뒤에 짧은 작업이 갇힌다(호위 효과) |
| SJF / SRTF | 평균 대기 최소. 실행 시간을 미리 알아야 하고 기아 위험 |
| RR | 응답 시간 짧음. 퀀텀이 크면 FCFS, 작으면 스위치 비용 폭증 |
| CFS / EEVDF | 리눅스 기본. nice 가중치에 비례해 CPU 몫을 나눈다 |

## 한 줄 교훈

CPU 코어 하나는 한순간에 한 가지 일만 한다. "동시에 도는 것처럼" 보이는 건 스케줄러가 **누구를, 얼마 동안** 돌릴지 정해 계속 갈아 끼우기 때문이고, 그 결정은 늘 **응답 시간 대 스위치 비용**의 저울질이다 — 그래서 스레드를 무작정 늘리는 게 아니라, 내 작업이 CPU를 기다리는지(nonvoluntary) I/O를 기다리는지(voluntary)부터 확인해야 한다.
