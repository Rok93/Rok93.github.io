---
layout: post
title: 'I/O 멀티플렉싱(select/epoll) — 스레드 하나로 연결 만 개를 지켜보는 법'
date: 2026-10-26 09:00:00 +0900
image: /assets/img/io-multiplexing/hero.jpg
generated: true
tags: [os, study]
sanitized: true
---

Nginx나 Redis는 연결 수만 개를 받으면서도 연결마다 스레드를 하나씩 만들지 않는다. Redis는 명령 처리를 사실상 스레드 하나로 한다. 그런데도 느리지 않다. 비결은 "데이터가 올 때까지 한 연결 앞에서 기다리는" 대신, **여러 연결을 한꺼번에 지켜보다가 준비된 것만 골라 처리**하는 데 있다. 이 방식을 **I/O 멀티플렉싱(I/O multiplexing — 여러 I/O 통로를 하나의 감시 지점으로 묶는 것)**이라 부른다. 이 글은 왜 이게 필요한지, 리눅스가 제공하는 세 도구 `select`·`poll`·`epoll`이 어떻게 다른지, 그리고 `epoll`의 두 가지 알림 방식(레벨 트리거·엣지 트리거)이 실제로 어떤 차이를 내는지 차례로 본다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·연결 수·실행 출력은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다. (단, <code>select</code>의 1024 제한, <code>pollfd</code> 구조체 크기, <code>epoll</code> API의 동작은 리눅스 실제 스펙 그대로입니다.)</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/io-multiplexing/hero.jpg" alt="보라색 톤의 식당. 웨이터 한 명이 가운데 서서 수많은 테이블 위의 호출벨을 한눈에 내려다보고 있다" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">웨이터(스레드)는 테이블(연결)마다 한 명씩 붙지 않는다. 호출벨(준비 알림)이 울린 테이블로만 간다.</figcaption>
</figure>

## 먼저 용어 두 개 — 파일 디스크립터와 블로킹

**파일 디스크립터(file descriptor, fd)**는 프로세스가 연 파일·소켓에 OS가 붙여 주는 **정수 번호**다. 0·1·2는 표준 입력·출력·에러이고, 소켓을 열면 3, 4, 5… 처럼 비어 있는 가장 작은 번호가 붙는다. 리눅스에선 네트워크 연결도 "fd 하나"로 다룬다.

**블로킹(blocking) I/O**는 `read()`를 불렀는데 읽을 데이터가 아직 없으면, 데이터가 올 때까지 **그 자리에서 스레드를 재우는** 방식이다. 리눅스 소켓의 기본값이 이것이다.

## 문제 — 연결마다 스레드를 붙이면 생기는 일

**무엇인가.** 블로킹 I/O로 서버를 짜는 가장 단순한 방법은 **연결 하나당 스레드 하나**(thread-per-connection)다. 각 스레드가 자기 연결의 `read()`에서 잠들어 있다가, 데이터가 오면 깨어나 처리한다.

**왜 문제인가.** 연결이 1만 개면 스레드도 1만 개다.

- **메모리**: 스레드마다 스택이 필요하다. 리눅스 기본 스레드 스택은 보통 8MB를 *예약*한다(실제로 쓰는 만큼만 RAM에 올라가긴 한다). 그래도 1만 개면 주소 공간과 커널 자료구조 부담이 크다.
- **문맥 교환(context switch — CPU가 실행 중인 스레드를 바꿔 끼우는 일)**: 깨어날 스레드가 많을수록 CPU는 일보다 "갈아 끼우기"에 시간을 더 쓴다.
- 대부분의 연결은 **대부분의 시간 동안 아무 일도 안 한다**. 채팅·푸시·keep-alive 연결이 그렇다. 잠자는 스레드 1만 개를 유지하는 비용을 내고, 실제 일은 그중 몇십 개만 한다.

이 "연결 1만 개를 어떻게 감당하나" 문제를 1999년 댄 케겔(Dan Kegel)이 **C10K 문제**라고 이름 붙였다.

**대안: 논블로킹 + 감시.** 소켓을 **논블로킹(non-blocking — 데이터가 없으면 기다리지 않고 즉시 `EAGAIN` 에러로 돌아오는)** 모드로 바꾸면 스레드가 잠들지 않는다. 하지만 1만 개 소켓을 무한 루프로 돌며 `read()`를 찔러 보면(**바쁜 대기, busy polling**) CPU가 100%로 헛돈다. 그래서 **"이 fd들 중 읽을 게 생긴 게 있으면 깨워 줘"**라고 커널에 한 번에 부탁하는 시스템 콜이 필요하다. 그게 I/O 멀티플렉싱이다.

<!-- diagram: thread-per-connection-vs-event-loop -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 190" role="img" aria-label="왼쪽은 연결 다섯 개에 스레드 다섯 개가 하나씩 붙어 대부분 잠들어 있는 그림. 오른쪽은 연결 다섯 개를 이벤트 루프 스레드 하나가 epoll로 감시하고 준비된 연결 두 개만 처리하는 그림.">
<text x="80" y="16" font-size="9" fill="#b23a00" font-weight="700" text-anchor="middle">연결당 스레드</text>
<text x="255" y="16" font-size="9" fill="#1f7a4d" font-weight="700" text-anchor="middle">I/O 멀티플렉싱</text>
<line x1="170" y1="8" x2="170" y2="182" stroke="#e4e0ec"/>
<g font-size="7" fill="#1a1720">
<rect x="12" y="28" width="40" height="18" rx="4" fill="#fff" stroke="#b9a9cc"/><text x="32" y="40" text-anchor="middle">fd 4</text>
<rect x="12" y="56" width="40" height="18" rx="4" fill="#fff" stroke="#b9a9cc"/><text x="32" y="68" text-anchor="middle">fd 5</text>
<rect x="12" y="84" width="40" height="18" rx="4" fill="#fff" stroke="#5F0080" stroke-width="1.5"/><text x="32" y="96" text-anchor="middle">fd 6</text>
<rect x="12" y="112" width="40" height="18" rx="4" fill="#fff" stroke="#b9a9cc"/><text x="32" y="124" text-anchor="middle">fd 7</text>
<rect x="12" y="140" width="40" height="18" rx="4" fill="#fff" stroke="#b9a9cc"/><text x="32" y="152" text-anchor="middle">fd 8</text>
</g>
<g font-size="7">
<line x1="52" y1="37" x2="92" y2="37" stroke="#b9a9cc"/><rect x="92" y="28" width="62" height="18" rx="4" fill="#e9e1f1"/><text x="123" y="40" fill="#5f5a68" text-anchor="middle">스레드 💤</text>
<line x1="52" y1="65" x2="92" y2="65" stroke="#b9a9cc"/><rect x="92" y="56" width="62" height="18" rx="4" fill="#e9e1f1"/><text x="123" y="68" fill="#5f5a68" text-anchor="middle">스레드 💤</text>
<line x1="52" y1="93" x2="92" y2="93" stroke="#5F0080"/><rect x="92" y="84" width="62" height="18" rx="4" fill="#5F0080"/><text x="123" y="96" fill="#fff" text-anchor="middle">스레드 작업 중</text>
<line x1="52" y1="121" x2="92" y2="121" stroke="#b9a9cc"/><rect x="92" y="112" width="62" height="18" rx="4" fill="#e9e1f1"/><text x="123" y="124" fill="#5f5a68" text-anchor="middle">스레드 💤</text>
<line x1="52" y1="149" x2="92" y2="149" stroke="#b9a9cc"/><rect x="92" y="140" width="62" height="18" rx="4" fill="#e9e1f1"/><text x="123" y="152" fill="#5f5a68" text-anchor="middle">스레드 💤</text>
</g>
<text x="83" y="178" font-size="7" fill="#b23a00" text-anchor="middle">연결 1만 = 스레드 1만, 대부분 잠만 잔다</text>
<g font-size="7" fill="#1a1720">
<rect x="184" y="28" width="40" height="18" rx="4" fill="#fff" stroke="#b9a9cc"/><text x="204" y="40" text-anchor="middle">fd 4</text>
<rect x="184" y="56" width="40" height="18" rx="4" fill="#fff" stroke="#b9a9cc"/><text x="204" y="68" text-anchor="middle">fd 5</text>
<rect x="184" y="84" width="40" height="18" rx="4" fill="#fff" stroke="#5F0080" stroke-width="1.5"/><text x="204" y="96" text-anchor="middle">fd 6 ●</text>
<rect x="184" y="112" width="40" height="18" rx="4" fill="#fff" stroke="#b9a9cc"/><text x="204" y="124" text-anchor="middle">fd 7</text>
<rect x="184" y="140" width="40" height="18" rx="4" fill="#fff" stroke="#5F0080" stroke-width="1.5"/><text x="204" y="152" text-anchor="middle">fd 8 ●</text>
</g>
<g stroke="#b9a9cc" fill="none"><path d="M224 37 L250 93"/><path d="M224 65 L250 93"/><path d="M224 121 L250 93"/></g>
<g stroke="#5F0080" stroke-width="1.5" fill="none"><path d="M224 93 L250 93"/><path d="M224 149 L250 93"/></g>
<rect x="250" y="74" width="80" height="38" rx="6" fill="#1f7a4d"/>
<text x="290" y="89" font-size="8" fill="#fff" font-weight="700" text-anchor="middle">이벤트 루프</text>
<text x="290" y="101" font-size="7" fill="#fff" text-anchor="middle">스레드 1개 + epoll</text>
<text x="257" y="178" font-size="7" fill="#1f7a4d" text-anchor="middle">준비된(●) fd 6·8만 골라 처리</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">왼쪽은 연결마다 스레드가 붙어 대부분 잠들어 있다. 오른쪽은 스레드 하나가 전체를 감시하다가 데이터가 도착한 연결(●)만 처리한다.</figcaption>
</figure>

## select — 가장 오래된 방법

**무엇인가.** `select()`는 1983년 4.2BSD에서 나온 시스템 콜이다. 감시할 fd들을 **비트맵(fd_set — fd 번호 자리에 1을 켜 둔 비트 배열)**으로 넘기면, 그중 하나라도 준비될 때까지 잠들었다가 깨어난다.

이 코드가 하는 일: fd 4와 7을 읽기 감시 대상으로 등록하고, 하나라도 읽을 수 있을 때까지 기다린 뒤 어느 쪽이 준비됐는지 확인한다(C).

```c
fd_set rfds;
FD_ZERO(&rfds);              // 비트맵 전부 0
FD_SET(4, &rfds);            // 4번 비트 켜기
FD_SET(7, &rfds);            // 7번 비트 켜기
int n = select(8, &rfds, NULL, NULL, NULL);  // 첫 인자 = 가장 큰 fd + 1
if (FD_ISSET(4, &rfds)) { /* fd 4 읽기 가능 */ }
if (FD_ISSET(7, &rfds)) { /* fd 7 읽기 가능 */ }
```

**한계는 세 가지다.**

1. **fd 번호 1024 제한.** `fd_set`의 크기는 컴파일 시점 상수 `FD_SETSIZE`(glibc 기준 **1024**)로 고정이다. 개수가 아니라 **번호**가 기준이라, fd 번호가 1024 이상이면 아예 넣을 수 없다.
2. **매번 통째로 복사·전수 검사.** 호출할 때마다 비트맵을 커널로 복사하고, 커널은 0번부터 `nfds-1`번까지 **모든 fd를 하나씩** 확인한다.
3. **결과가 입력을 덮어쓴다.** 돌아올 때 비트맵에는 "준비된 fd"만 남는다. 그래서 다음 호출 전에 감시 목록을 **다시 만들어야** 하고, 어느 게 준비됐는지 알려면 사용자 코드도 **전부 다시 훑어야** 한다.

결국 연결이 N개면 매 호출마다 O(N)의 일이 생긴다. 준비된 게 1개뿐이어도 그렇다.

## poll — 1024 제한만 풀었다

**무엇인가.** `poll()`은 비트맵 대신 **구조체 배열**을 받는다.

```c
struct pollfd {
    int   fd;       // 감시할 fd
    short events;   // 관심 이벤트 (POLLIN = 읽기 가능 등)
    short revents;  // 커널이 채워 주는 결과
};
```

**무엇이 나아졌나.** 번호 상한이 없고, 입력(`events`)과 결과(`revents`)가 분리돼 있어 배열을 매번 다시 만들 필요가 없다.

**무엇이 그대로인가.** 매 호출마다 **배열 전체를 커널로 복사**하고, 커널이 **전부 검사**하고, 사용자도 `revents`를 **전부 훑는다**. `pollfd`는 8바이트(4+2+2)이므로 연결 1만 개면 호출 한 번에 80,000바이트를 커널로 넘긴다. 비용 구조는 여전히 O(N)이다.

## epoll — 등록은 한 번, 결과는 준비된 것만

**무엇인가.** `epoll`은 리눅스 2.5.44(2002년)에 들어온 API다. 핵심 아이디어는 **감시 목록을 커널 안에 두고 계속 유지**하는 것이다. 세 개의 시스템 콜로 나뉜다.

- `epoll_create1()` — 커널 안에 epoll 인스턴스(감시 목록 + 준비 목록)를 만들고 그걸 가리키는 fd를 돌려준다.
- `epoll_ctl(epfd, ADD/MOD/DEL, fd, 이벤트)` — 감시 목록에 fd를 **한 번만** 추가·수정·삭제한다. 커널은 이 목록을 레드-블랙 트리(정렬된 균형 이진 트리)로 관리한다.
- `epoll_wait(epfd, 결과배열, 최대개수, 타임아웃)` — **준비된 fd만** 결과 배열에 담아 돌려준다.

**왜 빠른가.** 소켓에 데이터가 도착하면 커널의 네트워크 처리 경로가 그 소켓에 걸린 **콜백**을 호출해, 해당 fd를 **준비 목록(ready list)**에 넣어 둔다. `epoll_wait`는 그 준비 목록만 꺼내 오면 된다. 감시 중인 연결이 1만 개든 10만 개든, 호출 비용은 **준비된 개수**에 비례한다. 감시 목록을 매번 넘기지도 않는다.

<!-- diagram: select-vs-epoll -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 200" role="img" aria-label="select와 poll은 호출마다 감시 목록 전체를 커널에 복사하고 커널과 사용자 모두 전체를 훑는다. epoll은 감시 목록을 커널에 한 번 등록해 두고, 데이터가 오면 콜백이 준비 목록에 넣어, epoll_wait는 준비 목록만 가져온다.">
<text x="170" y="14" font-size="9" fill="#b23a00" font-weight="700" text-anchor="middle">select / poll — 매 호출마다</text>
<rect x="10" y="22" width="90" height="56" rx="6" fill="#fff" stroke="#b9a9cc"/>
<text x="55" y="36" font-size="7.5" fill="#1a1720" font-weight="700" text-anchor="middle">사용자 공간</text>
<text x="55" y="50" font-size="7" fill="#5f5a68" text-anchor="middle">fd 10,000개 목록</text>
<text x="55" y="66" font-size="7" fill="#b23a00" text-anchor="middle">결과 10,000개 훑기</text>
<rect x="240" y="22" width="90" height="56" rx="6" fill="#fff" stroke="#b9a9cc"/>
<text x="285" y="36" font-size="7.5" fill="#1a1720" font-weight="700" text-anchor="middle">커널</text>
<text x="285" y="50" font-size="7" fill="#b23a00" text-anchor="middle">10,000개 전부 검사</text>
<text x="285" y="66" font-size="7" fill="#5f5a68" text-anchor="middle">준비 10개</text>
<path d="M100 40 L236 40" stroke="#b23a00" stroke-width="1.5" fill="none" marker-end="url(#mo)"/>
<text x="170" y="36" font-size="7" fill="#b23a00" text-anchor="middle">① 목록 전체 복사 (매번)</text>
<path d="M240 62 L104 62" stroke="#b23a00" stroke-width="1.5" fill="none" marker-end="url(#mo)"/>
<text x="170" y="58" font-size="7" fill="#b23a00" text-anchor="middle">② 목록 전체 되돌려 줌</text>
<line x1="10" y1="92" x2="330" y2="92" stroke="#e4e0ec"/>
<text x="170" y="108" font-size="9" fill="#1f7a4d" font-weight="700" text-anchor="middle">epoll — 등록 한 번, 결과는 준비분만</text>
<rect x="10" y="116" width="90" height="74" rx="6" fill="#fff" stroke="#b9a9cc"/>
<text x="55" y="130" font-size="7.5" fill="#1a1720" font-weight="700" text-anchor="middle">사용자 공간</text>
<text x="55" y="152" font-size="7" fill="#5f5a68" text-anchor="middle">epoll_ctl (연결 시 1번)</text>
<text x="55" y="176" font-size="7" fill="#1f7a4d" text-anchor="middle">결과 10개만 처리</text>
<rect x="200" y="116" width="130" height="74" rx="6" fill="#fff" stroke="#b9a9cc"/>
<text x="265" y="130" font-size="7.5" fill="#1a1720" font-weight="700" text-anchor="middle">커널 (epoll 인스턴스)</text>
<rect x="208" y="138" width="56" height="44" rx="4" fill="#e9e1f1"/>
<text x="236" y="154" font-size="7" fill="#5F0080" text-anchor="middle">감시 목록</text><text x="236" y="166" font-size="6.5" fill="#5f5a68" text-anchor="middle">10,000개</text><text x="236" y="176" font-size="6.5" fill="#5f5a68" text-anchor="middle">(유지됨)</text>
<rect x="270" y="138" width="52" height="44" rx="4" fill="#1f7a4d"/>
<text x="296" y="154" font-size="7" fill="#fff" text-anchor="middle">준비 목록</text><text x="296" y="168" font-size="6.5" fill="#fff" text-anchor="middle">10개</text>
<path d="M264 160 L268 160" stroke="#1f7a4d" stroke-width="1.5"/>
<text x="296" y="198" font-size="6.5" fill="#5f5a68" text-anchor="middle">패킷 도착 → 콜백이 추가</text>
<path d="M100 148 L196 148" stroke="#5F0080" stroke-width="1.5" fill="none" marker-end="url(#mp)"/>
<path d="M200 174 L104 174" stroke="#1f7a4d" stroke-width="1.5" fill="none" marker-end="url(#mg)"/>
<text x="150" y="186" font-size="7" fill="#1f7a4d" text-anchor="middle">epoll_wait: 준비분만</text>
<defs>
<marker id="mo" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#b23a00"/></marker>
<marker id="mp" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#5F0080"/></marker>
<marker id="mg" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#1f7a4d"/></marker>
</defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">select/poll은 매 호출마다 목록 전체를 주고받고 전체를 훑는다. epoll은 감시 목록을 커널에 남겨 두고, 도착 시점에 준비 목록에 쌓아 둔 것만 가져온다.</figcaption>
</figure>

세 방식을 한 표로 정리하면 이렇다.

| | select | poll | epoll |
|---|---|---|---|
| 감시 목록 전달 | 호출마다 비트맵 복사 | 호출마다 배열 복사 | `epoll_ctl`로 한 번 등록 |
| fd 상한 | 번호 1024 미만 | 없음(프로세스 fd 한도까지) | 없음(프로세스 fd 한도까지) |
| 호출당 비용 | O(감시 fd 수) | O(감시 fd 수) | O(준비된 fd 수) |
| 이식성 | 거의 모든 OS | POSIX | 리눅스 전용 |

참고로 macOS·FreeBSD에는 비슷한 역할의 **kqueue**가 있다. Java NIO의 `Selector`는 리눅스에선 epoll, macOS에선 kqueue를 쓰고, Netty·Nginx·Redis·Node.js(libuv)의 **이벤트 루프(event loop — "준비된 이벤트를 기다렸다 처리하기"를 무한 반복하는 루프)**도 결국 이 시스템 콜 위에서 돈다.

**주의 — 멀티플렉싱이 "비동기 I/O"는 아니다.** epoll이 알려 주는 건 "지금 `read()`하면 **기다리지 않고** 바로 돌아온다"는 **준비 상태(readiness)**뿐이다. 데이터를 커널에서 사용자 메모리로 복사하는 `read()`는 여전히 사용자 스레드가 직접 부른다. 복사까지 커널이 끝내고 "다 됐다"고 알려 주는 방식(완료 기반, 예: 리눅스 `io_uring`, 윈도우 IOCP)은 결이 다른 모델이다.

## 레벨 트리거 vs 엣지 트리거

epoll은 "언제 알려 줄지"를 두 방식 중에서 고를 수 있다. 이름은 전기 신호에서 왔다.

- **레벨 트리거(level-triggered, LT — 기본값)**: 수신 버퍼에 **읽을 데이터가 남아 있는 동안** `epoll_wait`를 부를 때마다 계속 알려 준다. "상태"에 반응한다. select·poll도 이 방식이다.
- **엣지 트리거(edge-triggered, ET — `EPOLLET` 플래그)**: 데이터가 **새로 도착한 순간**에만 한 번 알려 준다. "변화"에 반응한다. 남은 데이터가 있어도, 새 데이터가 오기 전엔 다시 알려 주지 않는다.

**왜 차이가 중요한가.** ET는 같은 fd에 대한 반복 알림이 없어 호출 수가 줄어든다. 대신 규칙이 하나 생긴다. **알림을 받으면 `EAGAIN`이 날 때까지 끝까지 읽어야 한다.** 덜 읽고 다음 `epoll_wait`로 넘어가면, 남은 데이터는 상대가 뭔가 더 보내기 전까지 **영영 알림이 오지 않는다.** 그리고 "끝까지 읽기"를 하려면 소켓이 반드시 논블로킹이어야 한다. 블로킹 소켓이면 마지막 `read()`에서 스레드가 멈춰 버린다.

<!-- diagram: lt-vs-et -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 170" role="img" aria-label="수신 버퍼에 1000바이트가 한 번 도착한다. 레벨 트리거는 버퍼가 빌 때까지 epoll_wait마다 알림이 온다. 엣지 트리거는 도착 순간 한 번만 알림이 오고, 100바이트만 읽으면 나머지 900바이트는 알림 없이 남는다.">
<text x="12" y="20" font-size="7.5" fill="#5f5a68">수신 버퍼</text>
<path d="M70 50 L90 50 L90 22 L310 22" stroke="#5F0080" stroke-width="1.5" fill="none"/>
<text x="92" y="16" font-size="7" fill="#5F0080">1000B 도착</text>
<text x="315" y="25" font-size="7" fill="#5F0080">…</text>
<line x1="70" y1="54" x2="320" y2="54" stroke="#e4e0ec"/>
<text x="12" y="82" font-size="8" fill="#1f7a4d" font-weight="700">LT</text>
<text x="12" y="93" font-size="6.5" fill="#5f5a68">(레벨)</text>
<g fill="#1f7a4d">
<rect x="88" y="72" width="16" height="16" rx="3"/><rect x="128" y="72" width="16" height="16" rx="3"/><rect x="168" y="72" width="16" height="16" rx="3"/><rect x="208" y="72" width="16" height="16" rx="3"/><rect x="248" y="72" width="16" height="16" rx="3"/>
</g>
<text x="290" y="84" font-size="7" fill="#1f7a4d">… ×10</text>
<text x="190" y="102" font-size="6.5" fill="#5f5a68" text-anchor="middle">데이터가 남아 있는 동안 epoll_wait마다 알림</text>
<line x1="70" y1="112" x2="320" y2="112" stroke="#e4e0ec"/>
<text x="12" y="136" font-size="8" fill="#b23a00" font-weight="700">ET</text>
<text x="12" y="147" font-size="6.5" fill="#5f5a68">(엣지)</text>
<rect x="88" y="126" width="16" height="16" rx="3" fill="#b23a00"/>
<g fill="none" stroke="#b9a9cc" stroke-dasharray="2 2">
<rect x="128" y="126" width="16" height="16" rx="3"/><rect x="168" y="126" width="16" height="16" rx="3"/><rect x="208" y="126" width="16" height="16" rx="3"/><rect x="248" y="126" width="16" height="16" rx="3"/>
</g>
<text x="190" y="160" font-size="6.5" fill="#b23a00" text-anchor="middle">도착 순간 한 번만. 100B만 읽으면 900B는 알림 없이 방치</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">같은 1000바이트 도착에 대해, LT는 버퍼가 빌 때까지 계속 깨우고 ET는 도착 순간 한 번만 깨운다. ET에서는 알림 한 번에 끝까지 읽는 책임이 코드에 있다.</figcaption>
</figure>

## 상세 예시 ① — 비용을 숫자로 세어 보기

연결 10,000개를 붙잡고 있는 채팅 서버가 있다고 하자. 어느 순간이든 실제로 메시지를 보내는 연결은 평균 10개다. 대기 함수를 한 번 부를 때 생기는 일을 세어 보면 이렇다.

| | 커널로 넘기는 양 | 커널이 확인하는 fd | 사용자가 훑는 항목 |
|---|---|---|---|
| select | — (fd 번호 1024 이상은 등록 불가) | — | — |
| poll | 10,000 × 8B = **80,000B** | **10,000개** | **10,000개** |
| epoll | 0 (이미 등록됨) | 준비 목록 **10개** | **10개** |

- `select`는 이 규모에서 아예 출발선에 서지 못한다. 0·1·2를 빼면 fd 번호를 1021개까지밖에 못 쓴다.
- `poll`은 동작하지만, **준비된 10개를 찾으려고 10,000개를 매번 왕복**시킨다. 초당 수천 번 깨어나는 서버라면 이 낭비가 그대로 CPU 시간이 된다.
- `epoll`은 일한 만큼만 비용을 낸다. 감시 대상이 10배로 늘어도 준비된 수가 같다면 `epoll_wait` 비용은 거의 그대로다.

반대로 **감시 fd가 몇 개뿐이고 대부분이 늘 바쁜** 상황이라면 셋의 차이는 작다. epoll은 "많이 지켜보는데 조금만 바쁠 때" 진가가 난다.

## 상세 예시 ② — epoll 에코 서버를 직접 돌려 보기

이 코드가 하는 일: 리눅스에서 Python 표준 라이브러리의 `select.epoll`로, 스레드 하나가 여러 클라이언트의 메시지를 받아 그대로 돌려주는(에코) 서버를 만든다.

```python
# echo_epoll.py — 리눅스 전용
import socket, select

srv = socket.socket()
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(("0.0.0.0", 9000))
srv.listen(1024)
srv.setblocking(False)                      # 논블로킹

ep = select.epoll()                         # epoll_create1
ep.register(srv.fileno(), select.EPOLLIN)   # epoll_ctl(ADD): 새 연결 도착 감시
conns = {}

while True:
    for fd, events in ep.poll():            # epoll_wait: 준비된 fd만 돌아온다
        if fd == srv.fileno():              # 리슨 소켓이 준비 = 새 연결
            c, _ = srv.accept()
            c.setblocking(False)
            ep.register(c.fileno(), select.EPOLLIN)
            conns[c.fileno()] = c
        elif events & select.EPOLLIN:       # 클라이언트 소켓에 데이터 도착
            c = conns[fd]
            data = c.recv(4096)
            if data:
                c.send(data)                # 단순화: 실제로는 일부만 보내질 수 있다
            else:                           # 빈 바이트 = 상대가 연결을 닫음
                ep.unregister(fd)
                conns.pop(fd).close()
```

`while True` 한 바퀴가 이벤트 루프 한 번이다. 스레드는 `ep.poll()`에서만 잠들고, 깨어나면 준비된 fd만 처리하고 다시 잠든다.

이 명령이 하는 일: 서버를 띄운 뒤 `strace`로 시스템 콜을 관찰하면서, 다른 터미널에서 클라이언트 두 개가 메시지를 보낸다.

```bash
$ strace -e trace=epoll_ctl,epoll_wait,recvfrom,sendto python3 echo_epoll.py
epoll_ctl(4, EPOLL_CTL_ADD, 3, {events=EPOLLIN, data={u32=3, ...}}) = 0
epoll_wait(4, [{events=EPOLLIN, data={u32=3, ...}}], 1023, -1) = 1     # 새 연결
epoll_ctl(4, EPOLL_CTL_ADD, 5, {events=EPOLLIN, data={u32=5, ...}}) = 0
epoll_wait(4, [{events=EPOLLIN, data={u32=3, ...}}], 1023, -1) = 1     # 새 연결
epoll_ctl(4, EPOLL_CTL_ADD, 6, {events=EPOLLIN, data={u32=6, ...}}) = 0
epoll_wait(4, [{events=EPOLLIN, data={u32=6, ...}}], 1023, -1) = 1     # fd 6만 준비
recvfrom(6, "hello\n", 4096, 0, NULL, NULL) = 6
sendto(6, "hello\n", 6, 0, NULL, 0) = 6
epoll_wait(4, ...
```

읽을 포인트는 두 가지다.

- `epoll_ctl(ADD)`는 **연결마다 딱 한 번**만 나온다. 이후엔 `epoll_wait`만 반복된다. select/poll이었다면 매 바퀴 감시 목록 전체가 인자로 찍혔을 것이다.
- `epoll_wait`의 반환값 `= 1`은 "준비된 fd 1개"다. fd 5가 조용한 동안 fd 6만 돌아왔다. 세 번째 인자 `1023`은 Python이 잡은 결과 배열 크기(한 번에 돌려받을 최대 개수)이지, 감시 개수 제한이 아니다.

## 상세 예시 ③ — ET에서 "덜 읽기"가 만드는 멈춤

이번엔 위 서버를 두 군데만 바꿔 일부러 버그를 만든다. ① 등록할 때 `select.EPOLLIN | select.EPOLLET`로 엣지 트리거를 켜고, ② `recv(4096)`을 `recv(100)`으로 줄여 알림 한 번에 100바이트만 읽게 한다. 그리고 클라이언트가 1000바이트를 한 번에 보낸다.

```text
[LT, recv(100)]
epoll_wait → fd 6 준비 → recv 100B   (남은 900B)
epoll_wait → fd 6 준비 → recv 100B   (남은 800B)
...                                   (총 10번 깨어남)
epoll_wait → fd 6 준비 → recv 100B   (남은 0B)
epoll_wait → (다음 데이터까지 대기)

[ET, recv(100)]
epoll_wait → fd 6 준비 → recv 100B   (남은 900B)
epoll_wait → (대기…)                  ← 900B가 버퍼에 있는데도 안 깨어난다
```

- **LT**는 버퍼가 빌 때까지 매번 깨워 주므로, 조금씩 읽어도 결국 다 처리한다. 대신 같은 fd로 10번 깨어난다.
- **ET**는 "새로 도착한 순간"에만 알렸으므로, 남은 900바이트에 대해선 **다시 알리지 않는다.** 클라이언트는 응답을 기다리고, 서버는 알림을 기다린다. 둘 다 멈춘다.

이 코드가 하는 일: ET에서 올바르게 읽는 패턴. 알림 한 번에 `EAGAIN`(Python에선 `BlockingIOError`)이 날 때까지 반복해서 읽는다.

```python
elif events & select.EPOLLIN:
    c = conns[fd]
    while True:
        try:
            data = c.recv(100)
        except BlockingIOError:     # EAGAIN: 이번 알림분은 다 읽었다
            break
        if not data:                # 상대가 닫음
            ep.unregister(fd); conns.pop(fd).close()
            break
        c.send(data)
```

고치고 나면 ET도 1000바이트를 다 처리하고, `epoll_wait`는 **한 번만** 깨어난다. 이게 ET를 쓰는 이유이자 대가다. 깨어나는 횟수는 줄지만, "끝까지 읽기"를 빠뜨리면 조용히 멈춘다. 그래서 직접 짤 땐 기본값인 LT로 시작하고, ET는 Netty·Nginx처럼 이 규칙을 프레임워크가 지켜 주는 곳에서 쓰는 게 안전하다.

## 한 줄 교훈

연결이 많고 대부분 조용하다면, 연결마다 기다리지 말고 "준비된 것만 알려 달라"고 커널에 맡겨라. 그리고 엣지 트리거를 켰다면 알림 한 번에 끝까지 읽어라.
