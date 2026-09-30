---
layout: post
title: 'TCP vs UDP, 그리고 흐름 제어·혼잡 제어 — "얼마나 빨리 보내도 되나"를 누가 정하나'
date: 2026-09-30 09:00:00 +0900
image: /assets/img/tcp-udp-flow-congestion/hero.jpg
generated: true
tags: [network, study]
sanitized: true
---

인터넷으로 데이터를 보낼 때 가장 아래쪽에서 늘 같은 질문이 반복된다. **"보낸 게 잘 도착했나?"** 그리고 **"얼마나 빨리 보내도 되나?"** 앞의 질문을 책임지고 챙기는 쪽이 **TCP(Transmission Control Protocol — 순서와 도착을 보장하는 전송 규약)**이고, 아예 신경 쓰지 않는 대신 가볍고 빠른 쪽이 **UDP(User Datagram Protocol — 보내고 잊는 전송 규약)**다. 뒤의 질문은 TCP 안에서 두 장치가 나눠 맡는다. 받는 쪽 사정을 보는 **흐름 제어(flow control)**, 그리고 길(네트워크) 사정을 보는 **혼잡 제어(congestion control)**다. 이 글은 두 프로토콜의 차이를 먼저 보고, TCP가 "보낼 속도"를 어떻게 스스로 조절하는지 숫자로 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·트래픽·출력은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/tcp-udp-flow-congestion/hero.jpg" alt="고속도로에 줄지어 달리는 트럭 행렬과, 그 옆을 흩어져 날아가는 종이비행기들" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">순서대로 줄지어 달리며 속도를 맞추는 트럭(TCP)과, 각자 알아서 날아가는 종이비행기(UDP).</figcaption>
</figure>

## TCP와 UDP — 무엇이 다른가

둘 다 **전송 계층(transport layer — IP가 "어느 컴퓨터로"를 담당한다면, "그 컴퓨터의 어느 프로그램으로, 어떤 방식으로"를 담당하는 층)** 프로토콜이다. 차이는 "얼마나 챙겨주느냐"다.

**TCP**는 이런 것들을 보장한다.

- **연결(connection)** — 데이터를 보내기 전에 **3-way 핸드셰이크(SYN → SYN-ACK → ACK, 세 번 주고받는 인사)**로 "지금부터 대화하자"를 합의한다.
- **순서 보장** — 모든 바이트에 **시퀀스 번호(sequence number — 몇 번째 바이트인지 붙이는 번호)**를 매긴다. 도착 순서가 뒤섞여도 받는 쪽이 번호대로 다시 맞춘다.
- **도착 보장** — 받는 쪽은 **ACK(acknowledgment — "여기까지 잘 받았다"는 확인 응답)**를 돌려준다. ACK가 안 오면 보낸 쪽이 **재전송(retransmission)**한다.
- **속도 조절** — 흐름 제어와 혼잡 제어(이 글의 본론).

**UDP**는 이 중 아무것도 하지 않는다. 헤더도 8바이트로 단순하다(TCP는 최소 20바이트). 핸드셰이크 없이 바로 보내고, 잃어버려도 다시 보내지 않고, 순서도 맞춰주지 않는다. 대신 보낸 **데이터그램(datagram — 한 번에 보내는 독립된 꾸러미)** 단위의 경계는 지켜준다. TCP는 경계 없이 이어진 **바이트 스트림(byte stream)**이라, "100바이트 두 번 보냄"이 "200바이트 한 번 받음"으로 올 수 있다.

<!-- diagram: tcp-vs-udp -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 210" role="img" aria-label="왼쪽 TCP는 3-way 핸드셰이크 후 데이터를 보내고 ACK를 받으며, 잃어버린 데이터를 재전송한다. 오른쪽 UDP는 준비 없이 바로 보내고, 잃어버린 데이터그램은 그대로 사라진다.">
<defs>
<marker id="tu-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5f5a68"/></marker>
<marker id="tu-p" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5F0080"/></marker>
<marker id="tu-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#1f7a4d"/></marker>
</defs>
<text x="85" y="14" font-size="12" font-weight="700" fill="#5F0080" text-anchor="middle">TCP</text>
<text x="255" y="14" font-size="12" font-weight="700" fill="#7d5a9e" text-anchor="middle">UDP</text>
<line x1="170" y1="6" x2="170" y2="204" stroke="#e4e0ec" stroke-width="1"/>
<text x="30" y="32" font-size="8" fill="#1a1720" text-anchor="middle">클라이언트</text>
<text x="140" y="32" font-size="8" fill="#1a1720" text-anchor="middle">서버</text>
<line x1="30" y1="38" x2="30" y2="196" stroke="#b9a9cc" stroke-width="2"/>
<line x1="140" y1="38" x2="140" y2="196" stroke="#b9a9cc" stroke-width="2"/>
<line x1="30" y1="48" x2="140" y2="58" stroke="#5f5a68" stroke-width="1.2" marker-end="url(#tu-a)"/>
<text x="85" y="49" font-size="7" fill="#5f5a68" text-anchor="middle">SYN</text>
<line x1="140" y1="64" x2="30" y2="74" stroke="#5f5a68" stroke-width="1.2" marker-end="url(#tu-a)"/>
<text x="85" y="65" font-size="7" fill="#5f5a68" text-anchor="middle">SYN-ACK</text>
<line x1="30" y1="80" x2="140" y2="90" stroke="#5f5a68" stroke-width="1.2" marker-end="url(#tu-a)"/>
<text x="85" y="81" font-size="7" fill="#5f5a68" text-anchor="middle">ACK (연결 성립)</text>
<line x1="30" y1="102" x2="140" y2="112" stroke="#5F0080" stroke-width="1.4" marker-end="url(#tu-p)"/>
<text x="85" y="103" font-size="7" fill="#5F0080" text-anchor="middle">데이터 seq=1</text>
<line x1="140" y1="118" x2="30" y2="128" stroke="#5f5a68" stroke-width="1.2" marker-end="url(#tu-a)"/>
<text x="85" y="119" font-size="7" fill="#5f5a68" text-anchor="middle">ACK</text>
<line x1="30" y1="140" x2="82" y2="145" stroke="#5F0080" stroke-width="1.4"/>
<text x="87" y="149" font-size="11" font-weight="700" fill="#b23a00" text-anchor="middle">✕</text>
<text x="85" y="139" font-size="7" fill="#b23a00" text-anchor="middle">seq=2 유실</text>
<line x1="30" y1="166" x2="140" y2="176" stroke="#1f7a4d" stroke-width="1.4" marker-end="url(#tu-g)"/>
<text x="85" y="167" font-size="7" fill="#1f7a4d" text-anchor="middle">ACK 안 옴 → 재전송</text>
<text x="200" y="32" font-size="8" fill="#1a1720" text-anchor="middle">클라이언트</text>
<text x="310" y="32" font-size="8" fill="#1a1720" text-anchor="middle">서버</text>
<line x1="200" y1="38" x2="200" y2="196" stroke="#b9a9cc" stroke-width="2"/>
<line x1="310" y1="38" x2="310" y2="196" stroke="#b9a9cc" stroke-width="2"/>
<text x="255" y="50" font-size="7" fill="#5f5a68" text-anchor="middle">핸드셰이크 없이 바로</text>
<line x1="200" y1="60" x2="310" y2="70" stroke="#7d5a9e" stroke-width="1.4" marker-end="url(#tu-a)"/>
<text x="255" y="61" font-size="7" fill="#7d5a9e" text-anchor="middle">데이터그램 1</text>
<line x1="200" y1="90" x2="252" y2="95" stroke="#7d5a9e" stroke-width="1.4"/>
<text x="257" y="99" font-size="11" font-weight="700" fill="#b23a00" text-anchor="middle">✕</text>
<text x="255" y="89" font-size="7" fill="#b23a00" text-anchor="middle">데이터그램 2 유실</text>
<line x1="200" y1="120" x2="310" y2="130" stroke="#7d5a9e" stroke-width="1.4" marker-end="url(#tu-a)"/>
<text x="255" y="121" font-size="7" fill="#7d5a9e" text-anchor="middle">데이터그램 3</text>
<text x="255" y="160" font-size="7.5" fill="#b23a00" text-anchor="middle">2번은 그냥 사라짐</text>
<text x="255" y="172" font-size="7.5" fill="#b23a00" text-anchor="middle">(재전송·순서 보장 없음)</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">TCP는 인사부터 하고 매번 확인받으며, 잃어버리면 다시 보낸다. UDP는 인사도 확인도 없이 던지고 끝이다.</figcaption>
</figure>

**왜** 굳이 UDP를 쓸까? "챙겨주는 것"에는 비용이 있기 때문이다. 핸드셰이크만큼 첫 데이터가 늦고, 앞 패킷 하나가 유실되면 뒤에 도착한 패킷도 앞 패킷이 재전송될 때까지 애플리케이션에 넘기지 못한다. 이걸 **HOL 블로킹(Head-of-Line blocking — 줄 맨 앞이 막히면 뒤가 전부 기다리는 현상)**이라 부른다.

| 용도 | 고르는 쪽 | 이유 |
|---|---|---|
| 웹 페이지, API, DB 연결, 파일 전송 | TCP | 한 바이트라도 빠지면 안 된다 |
| DNS 조회 | UDP(기본) | 질문 1개·답 1개라 연결 비용이 아깝다. 응답이 크면 TCP로 전환 |
| 화상통화, 게임 위치 동기화 | UDP | 0.5초 전 화면을 재전송받아봐야 쓸모없다. 최신 것만 중요 |
| HTTP/3 (QUIC) | UDP 위에 직접 구현 | 신뢰성·혼잡 제어를 UDP 위 사용자 공간에서 다시 만들어 HOL 블로킹을 줄인다 |

마지막 줄이 중요하다. UDP는 "신뢰성이 필요 없는 곳"에만 쓰는 게 아니라, **"신뢰성을 내 방식대로 만들고 싶은 곳"**의 바탕으로도 쓰인다. 이전 글 [HTTP/1.1·2·3 진화](/2026/09/21/http-1-2-3.html)에서 본 QUIC가 그 예다.

## 흐름 제어 — 받는 쪽이 감당할 만큼만

**무엇**: 받는 쪽이 처리할 수 있는 속도에 맞춰 보내는 쪽의 속도를 제한하는 장치다.

**왜**: 받는 쪽 운영체제는 도착한 데이터를 **수신 버퍼(receive buffer)**에 쌓아두고, 애플리케이션이 읽어 가면 비운다. 애플리케이션이 느리게 읽는데 보내는 쪽이 계속 쏟아부으면 버퍼가 넘치고, 넘친 데이터는 버려진다. 버려진 걸 다시 보내는 건 순수한 낭비다.

**어떻게**: 받는 쪽은 ACK를 보낼 때마다 TCP 헤더의 **윈도(window) 필드**에 "내 버퍼에 지금 이만큼 빈자리가 있다"를 적어 보낸다. 이 값이 **rwnd(receive window, 수신 윈도)**다. 보내는 쪽은 **ACK를 아직 못 받은 데이터(in-flight — 날아가는 중인 데이터)의 양이 rwnd를 넘지 않게** 보낸다.

<!-- diagram: sliding-window -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 160" role="img" aria-label="보낼 바이트를 칸으로 나열한 슬라이딩 윈도 그림. 왼쪽부터 ACK 받은 칸, 전송했지만 ACK 대기 중인 칸, 지금 보내도 되는 칸, 아직 보내면 안 되는 칸 순서다. 가운데 두 구간을 감싸는 창의 크기가 min(cwnd, rwnd)이고, ACK가 오면 창이 오른쪽으로 미끄러진다.">
<defs>
<marker id="sw-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5F0080"/></marker>
</defs>
<text x="178" y="20" font-size="9" font-weight="700" fill="#5F0080" text-anchor="middle">보낼 수 있는 창 = min(cwnd, rwnd) = 7칸</text>
<path d="M88,40 L88,30 L268,30 L268,40" fill="none" stroke="#5F0080" stroke-width="1.6"/>
<rect x="10" y="46" width="24" height="28" rx="3" fill="#1f7a4d"/>
<rect x="36" y="46" width="24" height="28" rx="3" fill="#1f7a4d"/>
<rect x="62" y="46" width="24" height="28" rx="3" fill="#1f7a4d"/>
<rect x="88" y="46" width="24" height="28" rx="3" fill="#5F0080"/>
<rect x="114" y="46" width="24" height="28" rx="3" fill="#5F0080"/>
<rect x="140" y="46" width="24" height="28" rx="3" fill="#5F0080"/>
<rect x="166" y="46" width="24" height="28" rx="3" fill="#5F0080"/>
<rect x="192" y="46" width="24" height="28" rx="3" fill="#b9a9cc"/>
<rect x="218" y="46" width="24" height="28" rx="3" fill="#b9a9cc"/>
<rect x="244" y="46" width="24" height="28" rx="3" fill="#b9a9cc"/>
<rect x="270" y="46" width="24" height="28" rx="3" fill="#e4e0ec"/>
<rect x="296" y="46" width="24" height="28" rx="3" fill="#e4e0ec"/>
<text x="22" y="64" font-size="9" fill="#fff" text-anchor="middle">1</text>
<text x="48" y="64" font-size="9" fill="#fff" text-anchor="middle">2</text>
<text x="74" y="64" font-size="9" fill="#fff" text-anchor="middle">3</text>
<text x="100" y="64" font-size="9" fill="#fff" text-anchor="middle">4</text>
<text x="126" y="64" font-size="9" fill="#fff" text-anchor="middle">5</text>
<text x="152" y="64" font-size="9" fill="#fff" text-anchor="middle">6</text>
<text x="178" y="64" font-size="9" fill="#fff" text-anchor="middle">7</text>
<text x="204" y="64" font-size="9" fill="#1a1720" text-anchor="middle">8</text>
<text x="230" y="64" font-size="9" fill="#1a1720" text-anchor="middle">9</text>
<text x="256" y="64" font-size="9" fill="#1a1720" text-anchor="middle">10</text>
<text x="282" y="64" font-size="9" fill="#5f5a68" text-anchor="middle">11</text>
<text x="308" y="64" font-size="9" fill="#5f5a68" text-anchor="middle">12</text>
<text x="48" y="90" font-size="8" fill="#1f7a4d" text-anchor="middle">ACK 받음</text>
<text x="139" y="90" font-size="8" fill="#5F0080" text-anchor="middle">보냄·ACK 대기</text>
<text x="230" y="90" font-size="8" fill="#7d5a9e" text-anchor="middle">지금 보내도 됨</text>
<text x="295" y="90" font-size="8" fill="#5f5a68" text-anchor="middle">아직 안 됨</text>
<line x1="120" y1="118" x2="230" y2="118" stroke="#5F0080" stroke-width="1.6" marker-end="url(#sw-a)"/>
<text x="175" y="112" font-size="8" fill="#5F0080" text-anchor="middle">4번 ACK 도착 → 창이 오른쪽으로 한 칸</text>
<text x="165" y="142" font-size="8" fill="#5f5a68" text-anchor="middle">rwnd는 받는 쪽이 매 ACK에 적어 보내는 "수신 버퍼 빈자리"</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">슬라이딩 윈도(sliding window). ACK가 올 때마다 창의 왼쪽 끝이 밀리고, 그만큼 새로 보낼 칸이 생긴다.</figcaption>
</figure>

이 창이 ACK에 맞춰 옆으로 미끄러진다고 해서 **슬라이딩 윈도(sliding window)**라 부른다. 두 가지 경계 상황을 알아두자.

- **rwnd = 0 (zero window)** — 받는 쪽 버퍼가 꽉 찼다는 뜻이다. 보내는 쪽은 전송을 멈추고, 가끔 작은 **윈도 프로브(window probe — "이제 자리 났어?"라고 묻는 탐침 패킷)**를 보내 창이 다시 열렸는지 확인한다. 받는 쪽 애플리케이션이 멈춰 있으면 여기서 영원히 기다린다.
- **윈도 필드는 16비트다** — 그대로면 최대 65,535바이트(약 64KB)밖에 표현 못 한다. 요즘 회선에선 너무 작아서, 핸드셰이크 때 **윈도 스케일(window scale) 옵션**으로 "이 값에 2의 N제곱을 곱해 읽어라"를 합의한다(RFC 7323, 최대 약 1GB).

## 혼잡 제어 — 길이 감당할 만큼만

흐름 제어만으로는 부족하다. 받는 쪽 버퍼가 넉넉해도, **중간 길(라우터·회선)**이 막혀 있을 수 있기 때문이다. 모든 송신자가 "받는 쪽은 괜찮대!" 하며 최대로 쏟아내면 라우터 큐가 넘쳐 패킷이 버려지고, 모두가 재전송하느라 길이 더 막힌다. 1980년대 인터넷이 실제로 이렇게 **혼잡 붕괴(congestion collapse)**를 겪었고, 그 뒤 TCP에 혼잡 제어가 들어갔다.

**무엇**: 보내는 쪽이 스스로 **cwnd(congestion window, 혼잡 윈도)**라는 값을 들고, 네트워크 상태를 추측해 늘리고 줄이는 장치다. 받는 쪽이 알려주는 rwnd와 달리, cwnd는 **보내는 쪽 혼자** 계산한다. 길이 막혔는지 알려주는 사람이 없으니, **"패킷이 사라졌다 = 길이 막혔다"**로 추측한다.

결국 실제로 한 번에 보낼 수 있는 양은 이렇다.

> **보낼 수 있는 양 = min(cwnd, rwnd)** — 받는 쪽 사정과 길 사정 중 더 빡빡한 쪽에 맞춘다.

전통적인 알고리즘(TCP Reno 계열, RFC 5681)은 네 단계로 cwnd를 움직인다. 단위는 **MSS(Maximum Segment Size — 패킷 하나에 실을 수 있는 최대 데이터 크기, 이더넷에서 보통 1,460바이트)**다.

1. **슬로 스타트(slow start)** — 이름과 달리 빠르게 는다. ACK 하나 받을 때마다 cwnd를 1 MSS씩 늘리므로, 한 왕복(RTT, Round-Trip Time)마다 **두 배**가 된다. 1 → 2 → 4 → 8…
2. **혼잡 회피(congestion avoidance)** — cwnd가 **ssthresh(slow start threshold — "여기서부턴 조심하자" 문턱값)**에 닿으면 증가를 늦춘다. 한 RTT에 **1 MSS씩**만 는다.
3. **빠른 재전송·빠른 회복(fast retransmit / fast recovery)** — 같은 번호의 ACK가 3번 더 오면(**3 duplicate ACK** — "N번 다음이 안 왔어"를 세 번 반복) 그 패킷 하나만 유실됐다고 보고, 타임아웃을 기다리지 않고 바로 재전송한다. 뒤 패킷들은 도착하고 있다는 뜻이니 "살짝 막힘"으로 판단해 ssthresh와 cwnd를 **대략 절반**으로 줄이고 혼잡 회피를 이어간다.
4. **타임아웃(RTO, Retransmission TimeOut)** — ACK가 아예 안 오면 "심하게 막힘"으로 판단한다. ssthresh를 절반으로, cwnd는 **1 MSS**로 떨어뜨리고 슬로 스타트부터 다시 한다.

"조금씩 늘리고(Additive Increase), 문제가 생기면 확 줄인다(Multiplicative Decrease)"는 이 원칙을 **AIMD**라 부른다. 여러 연결이 같은 길을 나눠 쓸 때 서로 공평한 몫으로 수렴하게 해주는 성질이 있다.

<!-- diagram: cwnd-sawtooth -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 210" role="img" aria-label="시간에 따른 cwnd 변화 그래프. 1에서 16까지 두 배씩 늘어나는 슬로 스타트, 16부터 20까지 1씩 느는 혼잡 회피, 3 duplicate ACK로 10으로 반감, 다시 1씩 늘다가 14에서 타임아웃으로 1로 추락한 뒤 슬로 스타트를 재시작한다.">
<line x1="40" y1="180" x2="326" y2="180" stroke="#5f5a68" stroke-width="1"/>
<line x1="40" y1="180" x2="40" y2="26" stroke="#5f5a68" stroke-width="1"/>
<text x="183" y="200" font-size="8" fill="#5f5a68" text-anchor="middle">시간 (RTT 단위, 0 ~ 20)</text>
<text x="14" y="104" font-size="8" fill="#5f5a68" text-anchor="middle" transform="rotate(-90 14 104)">cwnd (MSS)</text>
<text x="36" y="183" font-size="7" fill="#5f5a68" text-anchor="end">0</text>
<text x="36" y="120" font-size="7" fill="#5f5a68" text-anchor="end">10</text>
<text x="36" y="83" font-size="7" fill="#5f5a68" text-anchor="end">16</text>
<text x="36" y="58" font-size="7" fill="#5f5a68" text-anchor="end">20</text>
<line x1="40" y1="80" x2="152" y2="80" stroke="#b9a9cc" stroke-width="1" stroke-dasharray="3 3"/>
<line x1="152" y1="117.5" x2="222" y2="117.5" stroke="#b9a9cc" stroke-width="1" stroke-dasharray="3 3"/>
<line x1="222" y1="136.25" x2="320" y2="136.25" stroke="#b9a9cc" stroke-width="1" stroke-dasharray="3 3"/>
<text x="44" y="76" font-size="7" fill="#7d5a9e">ssthresh=16</text>
<text x="226" y="146" font-size="7" fill="#7d5a9e">ssthresh=7</text>
<polyline fill="none" stroke="#5F0080" stroke-width="2" stroke-linejoin="round" points="40,173.75 54,167.5 68,155 82,130 96,80 110,73.75 124,67.5 138,61.25 152,55 166,117.5 180,111.25 194,105 208,98.75 222,92.5 236,173.75 250,167.5 264,155 278,136.25 292,130 306,123.75 320,117.5"/>
<text x="84" y="150" font-size="7.5" fill="#1f7a4d" text-anchor="start">슬로 스타트(×2)</text>
<text x="124" y="48" font-size="7.5" fill="#1f7a4d" text-anchor="middle">혼잡 회피(+1)</text>
<circle cx="152" cy="55" r="3" fill="#b23a00"/>
<text x="158" y="40" font-size="7.5" fill="#b23a00" text-anchor="start">3 dup ACK → 절반(10)</text>
<circle cx="222" cy="92.5" r="3" fill="#b23a00"/>
<text x="226" y="82" font-size="7.5" fill="#b23a00" text-anchor="start">타임아웃 → 1</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">cwnd의 톱니 모양. 가볍게 막히면(3 dup ACK) 절반, 심하게 막히면(타임아웃) 1부터 다시.</figcaption>
</figure>

## 예시 1 — cwnd를 한 칸씩 따라가 보기

위 그래프를 표로 풀어보자. 이해를 돕기 위해 교과서처럼 cwnd = 1, ssthresh = 16에서 시작한다(실제 리눅스는 초기 cwnd가 10 MSS다, RFC 6928).

| RTT | cwnd | 무슨 일이 | 단계 |
|---|---|---|---|
| 0 → 4 | 1 → 2 → 4 → 8 → 16 | 매 RTT 두 배 | 슬로 스타트 |
| 4 → 8 | 16 → 17 → … → 20 | ssthresh(16) 도달, 매 RTT +1 | 혼잡 회피 |
| 8 | 20 | **3 duplicate ACK** → 유실 1개 재전송 | 빠른 재전송 |
| 9 → 13 | 10 → 11 → … → 14 | ssthresh = 20/2 = 10, cwnd = 10부터 +1 | 빠른 회복 → 혼잡 회피 |
| 13 | 14 | **타임아웃** (ACK 전혀 안 옴) | RTO |
| 14 → 17 | 1 → 2 → 4 → 7 | ssthresh = 14/2 = 7, cwnd = 1부터 다시. 8이 아니라 7에서 멈춤 | 슬로 스타트 |
| 17 → 20 | 7 → 8 → 9 → 10 | 매 RTT +1 | 혼잡 회피 |

눈여겨볼 점은 두 가지다.

- **3 dup ACK와 타임아웃의 대가가 다르다.** 전자는 20 → 10(절반), 후자는 14 → 1이다. 뒤따르는 패킷의 ACK가 오고 있다는 건 "길은 뚫려 있다"는 증거라서 덜 줄인다.
- **슬로 스타트는 ssthresh를 넘지 않는다.** RTT 17에서 4의 두 배인 8이 아니라 7에서 혼잡 회피로 넘어간다.

(정밀하게는 빠른 회복 중 cwnd를 잠시 ssthresh + 3으로 올렸다가 회복이 끝나면 ssthresh로 내리는 과정이 있다. 표에서는 흐름을 보이려 생략했다.)

## 예시 2 — 창 크기가 속도의 상한이다

TCP는 한 RTT 동안 최대 **창 크기만큼**만 보낼 수 있다. 그래서 최대 속도는 대략 이렇게 계산된다.

> **처리량 ≈ min(cwnd, rwnd) ÷ RTT**

서울에서 미국 서버까지 **RTT 100ms**, 회선은 **100Mbps**라고 하자.

- 윈도 스케일 없이 rwnd가 최대 65,535바이트라면: 65,535B ÷ 0.1s = 655,350 B/s ≈ **5.24 Mbps**. 100Mbps 회선을 5%만 쓴다.
- 회선을 꽉 채우려면 창이 **BDP(Bandwidth-Delay Product, 대역폭 × 지연 — "길 위에 동시에 떠 있을 수 있는 데이터 양")** 이상이어야 한다: 100Mbps × 0.1s = 10,000,000비트 = **1,250,000바이트(약 1.25MB)**.

리눅스에서는 `ss` 명령으로 실제 연결의 cwnd·RTT를 볼 수 있다. 이 명령이 하는 일은 "지금 열린 TCP 연결마다 내부 상태(-i)를 보여주기"다.

```bash
ss -ti dst 203.0.113.10
```

연결 직후(가상 예시):

```text
ESTAB 0 0 10.0.0.5:51234 203.0.113.10:443
	 cubic wscale:7,7 rto:304 rtt:100.2/0.5 mss:1448 cwnd:10 delivery_rate 1.16Mbps
```

몇 초 뒤, 큰 파일을 받는 중(가상 예시):

```text
ESTAB 0 0 10.0.0.5:51234 203.0.113.10:443
	 cubic wscale:7,7 rto:304 rtt:100.4/0.8 mss:1448 cwnd:870 ssthresh:620 delivery_rate 100Mbps
```

숫자를 검산해보자.

- **before**: cwnd 10 × mss 1,448B = 14,480B. 이걸 RTT 0.1초마다 보내면 144,800 B/s ≈ **1.16 Mbps**. 막 연결돼 아직 슬로 스타트 초입이라 느리다.
- **after**: cwnd 870 × 1,448B = 1,259,760B ≈ BDP(1.25MB). 0.1초마다 보내면 약 12.6 MB/s ≈ **100 Mbps**. 창이 BDP만큼 자라서야 회선을 꽉 채운다.
- `wscale:7,7`은 양쪽이 윈도 스케일 2⁷ = 128배를 합의했다는 뜻이다. 이게 없으면 rwnd가 64KB에 묶여 after 상태에 도달하지 못한다.
- `cubic`은 혼잡 제어 알고리즘 이름이다. 아래에서 설명한다.

현재 서버가 쓰는 알고리즘은 이렇게 확인한다. 이 명령이 하는 일은 "커널의 기본 혼잡 제어 알고리즘 설정값 읽기"다.

```bash
sysctl net.ipv4.tcp_congestion_control
# net.ipv4.tcp_congestion_control = cubic
```

## Reno 이후 — CUBIC과 BBR

위에서 본 Reno 방식은 회선이 빠르고 멀수록(BDP가 클수록) 불리하다. 한 번 절반으로 줄면 RTT마다 1 MSS씩만 늘어서, cwnd 870짜리 창을 회복하는 데 수백 RTT가 걸린다.

- **CUBIC** — 리눅스 기본값(2.6.19부터). 창을 3차 함수 곡선으로 키운다. 손실 직전 크기 근처에선 천천히, 멀어지면 빠르게 늘려 큰 창을 빨리 회복한다. 손실 시 절반이 아니라 **0.7배**로 줄인다.
- **BBR** (Google) — "손실 = 혼잡"이라는 가정 자체를 버린다. 실제 **병목 대역폭과 최소 RTT를 측정**해 그만큼만 보낸다. 무선처럼 혼잡이 아닌 이유로 패킷이 잘 사라지는 환경에서 강하다.

어느 쪽이든 목표는 같다. **"길이 감당할 만큼만, 그러나 최대한 꽉 채워서."**

## 실무에서 알아둘 점

- **멀리 있는 서버와 대용량 전송이 느리면 창 크기부터 의심하자.** 처리량 ≈ 창 ÷ RTT다. `ss -ti`로 cwnd·rtt·wscale을 보면 병목이 받는 쪽(rwnd)인지 길(cwnd)인지 가늠할 수 있다.
- **새 연결은 늘 느리게 시작한다.** 슬로 스타트 때문이다. 짧은 요청을 자주 보내면 매번 창을 처음부터 키우게 되니, **커넥션 풀·Keep-Alive로 연결을 재사용**하는 게 핸드셰이크 절약 이상의 효과를 낸다.
- **zero window가 반복되면 네트워크가 아니라 받는 쪽 애플리케이션이 느린 것이다.** 버퍼를 못 비우고 있다는 신호다.
- **UDP를 고르면 신뢰성·혼잡 제어는 내 몫이다.** 제어 없이 UDP를 최대 속도로 쏘면 다른 트래픽까지 밀어낸다. 그래서 QUIC 같은 프로토콜은 UDP 위에 혼잡 제어를 다시 구현한다.

## 한 줄 교훈

TCP는 **"받는 쪽 사정(rwnd)과 길 사정(cwnd) 중 더 빡빡한 쪽"**에 맞춰 속도를 스스로 조절하는 프로토콜이고, UDP는 그 조절을 전부 **애플리케이션에 맡기는** 프로토콜이다. 전송이 느리다면 "회선이 느리다"보다 먼저 **"창이 몇이고, RTT가 얼마인가"**를 세어보자.
