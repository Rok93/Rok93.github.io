---
layout: post
title: 'WebSocket·SSE 실시간 통신 — 서버가 먼저 말을 걸려면 무엇이 바뀌어야 하나'
date: 2026-10-14 09:00:00 +0900
image: /assets/img/websocket-sse/hero.jpg
generated: true
tags: [network, study]
sanitized: true
---

HTTP는 원래 "클라이언트가 묻고, 서버가 답한다"는 규칙으로 움직인다. 서버는 질문을 받기 전에는 아무 말도 못 한다. 그런데 채팅 메시지, 주문 상태 알림("배송이 시작됐어요"), 주식 시세처럼 **서버 쪽에서 먼저 생긴 일을 바로 알려야** 하는 기능이 많다. 이렇게 일이 생기자마자 상대에게 전달하는 방식을 **실시간 통신(real-time communication)**이라 부른다. 이 글은 HTTP 위에서 실시간을 흉내 내던 방법(폴링·롱 폴링)부터, 이를 제대로 풀려고 나온 **SSE(Server-Sent Events — 서버가 클라이언트에게 이벤트를 계속 흘려보내는 표준)**와 **WebSocket(웹소켓 — 연결 하나로 양쪽이 아무 때나 메시지를 주고받는 프로토콜)**까지 차례로 본다. 마지막엔 "1분 동안 알림 3건"을 네 방식으로 보냈을 때 요청 수·바이트·지연이 어떻게 갈리는지 숫자로 비교한다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·트래픽·설정은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다. (단, WebSocket 핸드셰이크 키와 프레임 바이트는 RFC 6455 본문의 예시 값을 그대로 썼습니다.)</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/websocket-sse/hero.jpg" alt="보라색 건물 두 채 사이에 양쪽을 가리키는 화살표와 한쪽만 가리키는 화살표가 떠 있고, 아래로 두 건물을 잇는 관이 놓여 있다" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">두 건물을 잇는 관을 한 번 깔아 두면, 매번 문을 두드릴 필요 없이 소식이 흐른다. 한쪽으로만 흐르게 하면 SSE, 양쪽으로 흐르게 하면 WebSocket이다.</figcaption>
</figure>

## 문제 — HTTP는 서버가 먼저 말할 수 없다

HTTP 요청-응답은 전화 통화보다 **편지 왕복**에 가깝다. 클라이언트가 편지(요청)를 보내면 서버가 답장(응답)을 하고, 그걸로 그 대화는 끝난다. 서버에 새 소식이 생겨도 클라이언트가 다음 편지를 보내기 전엔 전할 길이 없다.

그래서 처음엔 "자주 묻기"로 버텼다.

### 폴링(polling) — 일정 간격으로 계속 묻기

클라이언트가 예를 들어 5초마다 `GET /notifications`를 보낸다. 새 소식이 있으면 받고, 없으면 빈 응답을 받는다.

- **장점**: 아무 서버, 아무 프록시에서나 된다. 그냥 평범한 HTTP다.
- **단점**: 대부분의 요청이 **헛걸음**이다. 소식이 없어도 매번 요청 헤더·응답 헤더(쿠키, 인증 토큰 등 수백 바이트)가 오간다. 그리고 소식이 생긴 순간부터 다음 폴링까지 **최대 한 주기(5초)만큼 늦게** 알게 된다. 주기를 줄이면 지연은 줄지만 헛걸음이 늘어난다.

### 롱 폴링(long polling) — 답이 생길 때까지 붙잡아 두기

요청은 똑같이 보내는데, 서버가 **소식이 생길 때까지 응답을 미룬다**. 소식이 생기면 그때 응답하고, 클라이언트는 응답을 받자마자 다음 요청을 바로 보낸다. 오래 소식이 없으면 서버가 일정 시간(예: 30초) 뒤 빈 응답으로 끊고, 클라이언트가 다시 건다.

- **장점**: 헛걸음이 크게 줄고, 소식이 생기면 거의 바로 전달된다. 여전히 평범한 HTTP라 호환성이 좋다.
- **단점**: 소식 하나마다 요청-응답이 한 번씩 새로 생긴다. 소식이 몰려오면 "응답 → 재요청" 사이 틈에 생긴 소식은 다음 요청에서야 받는다. 서버는 붙잡아 둔 요청마다 연결을 들고 있어야 한다.

두 방식 모두 "클라이언트가 묻는다"는 틀은 그대로다. 틀 자체를 바꾼 게 SSE와 WebSocket이다.

<!-- diagram: four-approaches-timeline -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 220" role="img" aria-label="같은 시간축 위에 네 방식을 나란히 그린 그림. 폴링은 짧은 요청이 일정 간격으로 많이 반복되고 대부분 비어 있다. 롱 폴링은 긴 요청이 소식이 생길 때 끝나고 바로 다시 시작된다. SSE는 연결 하나가 열린 채 서버에서 클라이언트로만 이벤트가 흐른다. WebSocket은 연결 하나가 열린 채 양방향으로 메시지가 흐른다.">
<defs>
<marker id="ws4-p" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5F0080"/></marker>
<marker id="ws4-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#1f7a4d"/></marker>
</defs>
<text x="200" y="14" font-size="7.5" fill="#5f5a68" text-anchor="middle">시간 →   (★ = 서버에 새 소식이 생긴 순간)</text>
<text x="100" y="28" font-size="8" fill="#b23a00" text-anchor="middle">★</text>
<text x="196" y="28" font-size="8" fill="#b23a00" text-anchor="middle">★</text>
<text x="280" y="28" font-size="8" fill="#b23a00" text-anchor="middle">★</text>
<line x1="100" y1="32" x2="100" y2="210" stroke="#b23a00" stroke-width="0.6" stroke-dasharray="2 2"/>
<line x1="196" y1="32" x2="196" y2="210" stroke="#b23a00" stroke-width="0.6" stroke-dasharray="2 2"/>
<line x1="280" y1="32" x2="280" y2="210" stroke="#b23a00" stroke-width="0.6" stroke-dasharray="2 2"/>
<text x="6" y="54" font-size="8" fill="#1a1720" font-weight="700">폴링</text>
<rect x="60" y="44" width="8" height="14" rx="2" fill="#e4e0ec"/>
<rect x="84" y="44" width="8" height="14" rx="2" fill="#e4e0ec"/>
<rect x="108" y="44" width="8" height="14" rx="2" fill="#5F0080"/>
<rect x="132" y="44" width="8" height="14" rx="2" fill="#e4e0ec"/>
<rect x="156" y="44" width="8" height="14" rx="2" fill="#e4e0ec"/>
<rect x="180" y="44" width="8" height="14" rx="2" fill="#e4e0ec"/>
<rect x="204" y="44" width="8" height="14" rx="2" fill="#5F0080"/>
<rect x="228" y="44" width="8" height="14" rx="2" fill="#e4e0ec"/>
<rect x="252" y="44" width="8" height="14" rx="2" fill="#e4e0ec"/>
<rect x="276" y="44" width="8" height="14" rx="2" fill="#e4e0ec"/>
<rect x="300" y="44" width="8" height="14" rx="2" fill="#5F0080"/>
<rect x="324" y="44" width="8" height="14" rx="2" fill="#e4e0ec"/>
<text x="196" y="70" font-size="6.5" fill="#5f5a68" text-anchor="middle">회색 = 헛걸음, 보라 = 소식 받음 (다음 차례까지 늦게 앎)</text>
<text x="6" y="98" font-size="8" fill="#1a1720" font-weight="700">롱 폴링</text>
<rect x="60" y="88" width="40" height="14" rx="2" fill="none" stroke="#5F0080" stroke-width="1.2"/>
<rect x="102" y="88" width="94" height="14" rx="2" fill="none" stroke="#5F0080" stroke-width="1.2"/>
<rect x="198" y="88" width="82" height="14" rx="2" fill="none" stroke="#5F0080" stroke-width="1.2"/>
<rect x="282" y="88" width="50" height="14" rx="2" fill="none" stroke="#5F0080" stroke-width="1.2" stroke-dasharray="3 2"/>
<text x="196" y="114" font-size="6.5" fill="#5f5a68" text-anchor="middle">요청을 붙잡아 두다가 소식이 생기면 응답 → 즉시 재요청</text>
<text x="6" y="142" font-size="8" fill="#1a1720" font-weight="700">SSE</text>
<rect x="60" y="134" width="272" height="10" rx="5" fill="#f4eff9" stroke="#7d5a9e" stroke-width="1"/>
<line x1="100" y1="128" x2="100" y2="150" stroke="#5F0080" stroke-width="1.4" marker-end="url(#ws4-p)"/>
<line x1="196" y1="128" x2="196" y2="150" stroke="#5F0080" stroke-width="1.4" marker-end="url(#ws4-p)"/>
<line x1="280" y1="128" x2="280" y2="150" stroke="#5F0080" stroke-width="1.4" marker-end="url(#ws4-p)"/>
<text x="196" y="162" font-size="6.5" fill="#5f5a68" text-anchor="middle">연결 하나를 열어 두고 서버 → 클라이언트로만 흘려보냄</text>
<text x="6" y="188" font-size="8" fill="#1a1720" font-weight="700">WebSocket</text>
<rect x="60" y="180" width="272" height="10" rx="5" fill="#eaf5ef" stroke="#1f7a4d" stroke-width="1"/>
<line x1="100" y1="174" x2="100" y2="196" stroke="#5F0080" stroke-width="1.4" marker-end="url(#ws4-p)"/>
<line x1="150" y1="196" x2="150" y2="174" stroke="#1f7a4d" stroke-width="1.4" marker-end="url(#ws4-g)"/>
<line x1="196" y1="174" x2="196" y2="196" stroke="#5F0080" stroke-width="1.4" marker-end="url(#ws4-p)"/>
<line x1="240" y1="196" x2="240" y2="174" stroke="#1f7a4d" stroke-width="1.4" marker-end="url(#ws4-g)"/>
<line x1="280" y1="174" x2="280" y2="196" stroke="#5F0080" stroke-width="1.4" marker-end="url(#ws4-p)"/>
<text x="196" y="208" font-size="6.5" fill="#5f5a68" text-anchor="middle">연결 하나로 양쪽이 아무 때나 (아래↓ 서버→클라, 위↑ 클라→서버)</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">같은 소식 3건을 네 방식으로 받는 모습. 위 둘은 "클라이언트가 묻는" 틀 안에서 버티고, 아래 둘은 연결을 열어 둔 채 서버가 먼저 보낸다.</figcaption>
</figure>

## SSE — HTTP 응답을 끝내지 않고 계속 흘려보내기

### 무엇인가

SSE는 HTML 표준(HTML Living Standard)에 들어 있는 기능이다. 아이디어는 단순하다. **HTTP 응답을 하나 시작한 뒤, 끝내지 않고 계속 이어서 쓴다.** 서버는 소식이 생길 때마다 그 응답 본문에 몇 줄을 덧붙이고, 브라우저는 그 줄을 이벤트로 받아 처리한다.

새 프로토콜이 아니라 **평범한 HTTP 응답**이라는 게 핵심이다. 다른 점은 두 가지뿐이다.

- 응답 헤더 `Content-Type: text/event-stream` — "이 본문은 이벤트 스트림이다"라는 표시.
- 본문을 닫지 않고 계속 쓴다.

### 메시지 형식

이 예시가 보여주는 것: 서버가 응답 본문으로 흘려보내는 SSE 스트림의 실제 모습이다. 이벤트 하나는 **빈 줄로 끝난다.**

```text
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

: 연결됨 (콜론으로 시작하는 줄은 주석. 무시된다)
retry: 3000

id: 101
event: order-status
data: {"orderId":42,"status":"SHIPPED"}

id: 102
event: order-status
data: {"orderId":42,"status":"DELIVERED"}

```

필드는 네 개다.

| 필드 | 뜻 |
|---|---|
| `data:` | 이벤트 내용. 여러 줄이면 `data:` 줄을 여러 개 쓰고, 받는 쪽에서 줄바꿈으로 이어 붙인다 |
| `event:` | 이벤트 이름. 없으면 기본 이름 `message` |
| `id:` | 이벤트 번호. 브라우저가 "마지막으로 받은 번호"로 기억한다 |
| `retry:` | 연결이 끊겼을 때 다시 연결하기 전 기다릴 시간(ms) |

### 왜 쓰나 — 공짜로 따라오는 자동 재연결

브라우저 쪽 API인 `EventSource`는 연결이 끊기면 **알아서 다시 연결한다.** 그리고 다시 연결할 때 요청 헤더에 `Last-Event-ID: 102`처럼 **마지막으로 받은 `id`를 실어 보낸다.** 서버가 이 값을 보고 "103번부터" 다시 보내면, 끊긴 동안 놓친 이벤트를 메울 수 있다. 이 재전송 로직은 서버가 구현해야 하지만, 재연결과 번호 기억은 브라우저가 해 준다.

이 코드가 하는 일: 브라우저에서 SSE 스트림을 구독하고, `order-status` 이벤트가 올 때마다 화면을 갱신한다.

```javascript
const es = new EventSource("/orders/42/events");   // GET 요청 한 번. 응답이 안 끝난다

es.addEventListener("order-status", (e) => {        // event: order-status 인 것만
  const { status } = JSON.parse(e.data);
  render(status);
  // e.lastEventId === "102" 같은 식으로 id도 볼 수 있다
});

es.onerror = () => {
  // 끊기면 여기로 온다. 대부분은 아무것도 안 해도 브라우저가 retry 뒤 재연결한다
};
```

### 한계

- **서버 → 클라이언트 한 방향뿐.** 클라이언트가 서버에 뭔가 보내려면 별도의 평범한 HTTP 요청(`fetch`)을 쓴다.
- **텍스트(UTF-8)만.** 이미지 같은 바이너리는 Base64 등으로 바꿔야 한다.
- **HTTP/1.1에선 연결 수 제한에 걸린다.** 브라우저는 HTTP/1.1에서 한 도메인당 동시 연결을 6개 정도로 제한한다. SSE 연결이 하나씩 그 자리를 차지하니, 탭을 여러 개 열면 다른 요청이 막힐 수 있다. **HTTP/2**는 연결 하나에 여러 요청을 나란히 싣는 **멀티플렉싱(multiplexing)**을 하므로 이 문제가 크게 줄어든다.
- **기본 `EventSource`는 요청 헤더를 직접 못 붙인다.** `Authorization` 헤더 대신 쿠키(`withCredentials`)나 URL 파라미터로 인증해야 한다. 헤더가 꼭 필요하면 `fetch` 스트림으로 직접 파싱하는 라이브러리를 쓴다.

## WebSocket — HTTP로 시작해서 다른 프로토콜로 갈아타기

### 무엇인가

WebSocket은 2011년 **RFC 6455**로 표준화된 별도 프로토콜이다. **TCP 연결 하나를 열어 두고, 클라이언트와 서버가 아무 때나, 서로 동시에 메시지를 보낼 수 있다.** 이런 성질을 **전이중(full-duplex — 양쪽이 동시에 말할 수 있는)**이라 부른다. 주소는 `ws://`(평문), `wss://`(TLS 암호화)로 시작한다.

재미있는 건 시작이다. WebSocket은 **HTTP 요청으로 문을 두드린 뒤, 같은 TCP 연결을 WebSocket으로 갈아탄다.** 이 갈아타기를 **업그레이드(Upgrade)**, 그 첫 인사를 **핸드셰이크(handshake)**라 한다. 덕분에 80/443 포트와 기존 웹 인프라를 그대로 쓸 수 있다.

### 핸드셰이크 — 한 번의 HTTP 왕복

이 예시가 보여주는 것: RFC 6455 본문에 실린 핸드셰이크 예시다. 클라이언트가 "WebSocket으로 바꾸자"고 요청하고, 서버가 `101`로 승낙한다.

```text
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

- `Upgrade: websocket` + `Connection: Upgrade` — "이 연결을 websocket으로 바꾸고 싶다."
- `Sec-WebSocket-Key` — 클라이언트가 만든 무작위 16바이트를 Base64로 적은 값. (위 값은 디코딩하면 `the sample nonce`다.)
- `101 Switching Protocols` — 서버의 승낙. 이 응답 직후부터 같은 TCP 연결 위로 HTTP가 아니라 WebSocket 프레임이 오간다.
- `Sec-WebSocket-Accept` — 서버가 키를 정말 이해했다는 증명이다. 계산법은 고정이다.

이 코드가 하는 일: `Sec-WebSocket-Key`로부터 `Sec-WebSocket-Accept`를 계산한다. 뒤에 붙이는 긴 문자열은 RFC에 정해진 고정값(GUID)이다.

```python
import base64, hashlib

key = "dGhlIHNhbXBsZSBub25jZQ=="
GUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
accept = base64.b64encode(hashlib.sha1((key + GUID).encode()).digest()).decode()
print(accept)   # s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

이 검증은 **보안 인증이 아니다.** "WebSocket을 모르는 서버(나 캐시 프록시)가 엉뚱하게 응답한 걸 연결 성공으로 착각하지 않게" 하는 장치다. 사용자 인증은 핸드셰이크 요청의 쿠키·토큰으로 따로 해야 한다.

<!-- diagram: websocket-handshake -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 320 210" role="img" aria-label="클라이언트와 서버 사이의 순서도. 먼저 TCP 연결을 맺고, 클라이언트가 Upgrade websocket 헤더를 담은 HTTP GET을 보내면 서버가 101 Switching Protocols로 답한다. 그 아래부터는 같은 연결 위로 WebSocket 프레임이 양방향으로 오가고, 마지막에 Close 프레임을 주고받아 끝낸다.">
<defs>
<marker id="wsh-p" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5F0080"/></marker>
<marker id="wsh-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#1f7a4d"/></marker>
<marker id="wsh-s" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5f5a68"/></marker>
</defs>
<rect x="20" y="8" width="70" height="20" rx="6" fill="#fff" stroke="#5F0080" stroke-width="1.3"/>
<text x="55" y="22" font-size="8" fill="#5F0080" text-anchor="middle" font-weight="700">클라이언트</text>
<rect x="230" y="8" width="70" height="20" rx="6" fill="#fff" stroke="#5F0080" stroke-width="1.3"/>
<text x="265" y="22" font-size="8" fill="#5F0080" text-anchor="middle" font-weight="700">서버</text>
<line x1="55" y1="28" x2="55" y2="205" stroke="#b9a9cc" stroke-width="1"/>
<line x1="265" y1="28" x2="265" y2="205" stroke="#b9a9cc" stroke-width="1"/>
<line x1="55" y1="42" x2="263" y2="48" stroke="#5f5a68" stroke-width="1" marker-end="url(#wsh-s)"/>
<line x1="265" y1="52" x2="57" y2="58" stroke="#5f5a68" stroke-width="1" marker-end="url(#wsh-s)"/>
<text x="160" y="40" font-size="7" fill="#5f5a68" text-anchor="middle">TCP 3-way 핸드셰이크 (wss면 TLS도)</text>
<rect x="8" y="64" width="304" height="46" rx="6" fill="#f4eff9" stroke="none"/>
<text x="16" y="74" font-size="6.5" fill="#7d5a9e" font-weight="700">HTTP 구간</text>
<line x1="55" y1="80" x2="263" y2="86" stroke="#5F0080" stroke-width="1.4" marker-end="url(#wsh-p)"/>
<text x="160" y="78" font-size="7.5" fill="#1a1720" text-anchor="middle">GET /chat · Upgrade: websocket</text>
<line x1="265" y1="94" x2="57" y2="100" stroke="#5F0080" stroke-width="1.4" marker-end="url(#wsh-p)"/>
<text x="160" y="107" font-size="7.5" fill="#1a1720" text-anchor="middle">101 Switching Protocols</text>
<rect x="8" y="116" width="304" height="62" rx="6" fill="#eaf5ef" stroke="none"/>
<text x="16" y="126" font-size="6.5" fill="#1f7a4d" font-weight="700">WebSocket 구간 (같은 TCP 연결)</text>
<line x1="265" y1="132" x2="57" y2="136" stroke="#1f7a4d" stroke-width="1.3" marker-end="url(#wsh-g)"/>
<text x="160" y="131" font-size="7" fill="#1a1720" text-anchor="middle">텍스트 프레임 "주문 42 배송 시작"</text>
<line x1="55" y1="146" x2="263" y2="150" stroke="#1f7a4d" stroke-width="1.3" marker-end="url(#wsh-g)"/>
<text x="160" y="145" font-size="7" fill="#1a1720" text-anchor="middle">텍스트 프레임 "확인" (마스킹됨)</text>
<line x1="265" y1="160" x2="57" y2="164" stroke="#1f7a4d" stroke-width="1" stroke-dasharray="3 2" marker-end="url(#wsh-g)"/>
<text x="160" y="172" font-size="7" fill="#5f5a68" text-anchor="middle">Ping ↔ Pong (살아 있나 확인)</text>
<line x1="55" y1="188" x2="263" y2="192" stroke="#b23a00" stroke-width="1.3" marker-end="url(#wsh-s)"/>
<line x1="265" y1="196" x2="57" y2="200" stroke="#b23a00" stroke-width="1.3" marker-end="url(#wsh-s)"/>
<text x="160" y="186" font-size="7" fill="#b23a00" text-anchor="middle">Close ↔ Close</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">WebSocket은 HTTP 요청 한 번으로 문을 연 뒤(보라 구간), 같은 TCP 연결을 WebSocket으로 갈아타 양방향 프레임을 주고받는다(초록 구간). 끝낼 때도 Close 프레임을 서로 주고받는다.</figcaption>
</figure>

### 프레임 — 메시지를 담는 작은 상자

업그레이드 뒤에는 데이터가 **프레임(frame)** 단위로 오간다. 프레임 앞부분(헤더)은 최소 2바이트다. HTTP 요청마다 붙던 수백 바이트 헤더와 비교하면 아주 가볍다.

- **FIN** (1비트) — 이 프레임이 메시지의 마지막 조각인지. 큰 메시지는 여러 프레임으로 쪼갤 수 있다.
- **opcode** (4비트) — 프레임 종류. `0x1` 텍스트, `0x2` 바이너리, `0x8` 닫기(Close), `0x9` Ping, `0xA` Pong, `0x0` 이전 메시지의 이어지는 조각.
- **MASK** (1비트) + **길이** (7비트) — 길이가 125바이트 이하면 여기에 바로 적고, 126이면 뒤 2바이트, 127이면 뒤 8바이트에 진짜 길이를 적는다.
- **마스킹 키(masking key)** (4바이트) — **클라이언트 → 서버 프레임은 반드시 마스킹한다.** 본문을 4바이트 키로 XOR(비트를 뒤섞는 연산)해서 보낸다. 서버 → 클라이언트 프레임은 마스킹하지 않는다.

마스킹은 암호화가 아니다(키가 프레임 안에 그대로 들어 있다). 목적은 중간에 낀 낡은 캐시 프록시가 WebSocket 데이터를 HTTP 요청으로 잘못 해석해 **엉뚱한 응답을 캐시에 저장하는 공격(cache poisoning — 캐시 오염)**을 막는 것이다. 공격자가 보낼 바이트를 예측할 수 없게 만드는 장치다.

<!-- diagram: websocket-frame-bytes -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 160" role="img" aria-label="Hello라는 텍스트 메시지를 담은 WebSocket 프레임 두 개를 바이트 단위로 그린 그림. 서버에서 클라이언트로 가는 프레임은 81 05 다음에 Hello 다섯 바이트로 총 7바이트다. 클라이언트에서 서버로 가는 프레임은 81 85 다음에 마스킹 키 37 fa 21 3d와 마스킹된 다섯 바이트로 총 11바이트다.">
<text x="6" y="16" font-size="8" fill="#1a1720" font-weight="700">서버 → 클라이언트 "Hello" (마스킹 없음, 7바이트)</text>
<rect x="6" y="24" width="34" height="22" rx="3" fill="#5F0080"/><text x="23" y="39" font-size="8" fill="#fff" text-anchor="middle" font-family="monospace">81</text>
<rect x="42" y="24" width="34" height="22" rx="3" fill="#7d5a9e"/><text x="59" y="39" font-size="8" fill="#fff" text-anchor="middle" font-family="monospace">05</text>
<rect x="78" y="24" width="34" height="22" rx="3" fill="#fff" stroke="#1f7a4d"/><text x="95" y="39" font-size="8" fill="#1a1720" text-anchor="middle" font-family="monospace">48 H</text>
<rect x="114" y="24" width="34" height="22" rx="3" fill="#fff" stroke="#1f7a4d"/><text x="131" y="39" font-size="8" fill="#1a1720" text-anchor="middle" font-family="monospace">65 e</text>
<rect x="150" y="24" width="34" height="22" rx="3" fill="#fff" stroke="#1f7a4d"/><text x="167" y="39" font-size="8" fill="#1a1720" text-anchor="middle" font-family="monospace">6c l</text>
<rect x="186" y="24" width="34" height="22" rx="3" fill="#fff" stroke="#1f7a4d"/><text x="203" y="39" font-size="8" fill="#1a1720" text-anchor="middle" font-family="monospace">6c l</text>
<rect x="222" y="24" width="34" height="22" rx="3" fill="#fff" stroke="#1f7a4d"/><text x="239" y="39" font-size="8" fill="#1a1720" text-anchor="middle" font-family="monospace">6f o</text>
<text x="23" y="58" font-size="6.5" fill="#5f5a68" text-anchor="middle">FIN=1</text>
<text x="23" y="66" font-size="6.5" fill="#5f5a68" text-anchor="middle">텍스트</text>
<text x="59" y="58" font-size="6.5" fill="#5f5a68" text-anchor="middle">MASK=0</text>
<text x="59" y="66" font-size="6.5" fill="#5f5a68" text-anchor="middle">길이 5</text>
<text x="167" y="62" font-size="6.5" fill="#1f7a4d" text-anchor="middle">본문 그대로</text>
<text x="6" y="92" font-size="8" fill="#1a1720" font-weight="700">클라이언트 → 서버 "Hello" (마스킹, 11바이트)</text>
<rect x="6" y="100" width="26" height="22" rx="3" fill="#5F0080"/><text x="19" y="115" font-size="7.5" fill="#fff" text-anchor="middle" font-family="monospace">81</text>
<rect x="34" y="100" width="26" height="22" rx="3" fill="#7d5a9e"/><text x="47" y="115" font-size="7.5" fill="#fff" text-anchor="middle" font-family="monospace">85</text>
<rect x="62" y="100" width="26" height="22" rx="3" fill="#fbeee6" stroke="#b23a00"/><text x="75" y="115" font-size="7.5" fill="#b23a00" text-anchor="middle" font-family="monospace">37</text>
<rect x="90" y="100" width="26" height="22" rx="3" fill="#fbeee6" stroke="#b23a00"/><text x="103" y="115" font-size="7.5" fill="#b23a00" text-anchor="middle" font-family="monospace">fa</text>
<rect x="118" y="100" width="26" height="22" rx="3" fill="#fbeee6" stroke="#b23a00"/><text x="131" y="115" font-size="7.5" fill="#b23a00" text-anchor="middle" font-family="monospace">21</text>
<rect x="146" y="100" width="26" height="22" rx="3" fill="#fbeee6" stroke="#b23a00"/><text x="159" y="115" font-size="7.5" fill="#b23a00" text-anchor="middle" font-family="monospace">3d</text>
<rect x="174" y="100" width="26" height="22" rx="3" fill="#fff" stroke="#1f7a4d"/><text x="187" y="115" font-size="7.5" fill="#1a1720" text-anchor="middle" font-family="monospace">7f</text>
<rect x="202" y="100" width="26" height="22" rx="3" fill="#fff" stroke="#1f7a4d"/><text x="215" y="115" font-size="7.5" fill="#1a1720" text-anchor="middle" font-family="monospace">9f</text>
<rect x="230" y="100" width="26" height="22" rx="3" fill="#fff" stroke="#1f7a4d"/><text x="243" y="115" font-size="7.5" fill="#1a1720" text-anchor="middle" font-family="monospace">4d</text>
<rect x="258" y="100" width="26" height="22" rx="3" fill="#fff" stroke="#1f7a4d"/><text x="271" y="115" font-size="7.5" fill="#1a1720" text-anchor="middle" font-family="monospace">51</text>
<rect x="286" y="100" width="26" height="22" rx="3" fill="#fff" stroke="#1f7a4d"/><text x="299" y="115" font-size="7.5" fill="#1a1720" text-anchor="middle" font-family="monospace">58</text>
<text x="47" y="134" font-size="6.5" fill="#5f5a68" text-anchor="middle">MASK=1</text>
<text x="47" y="142" font-size="6.5" fill="#5f5a68" text-anchor="middle">길이 5</text>
<text x="117" y="138" font-size="6.5" fill="#b23a00" text-anchor="middle">마스킹 키 4바이트</text>
<text x="243" y="138" font-size="6.5" fill="#1f7a4d" text-anchor="middle">"Hello" XOR 키 (반복 적용)</text>
<text x="243" y="150" font-size="6.5" fill="#5f5a68" text-anchor="middle">48⊕37=7f, 65⊕fa=9f …</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">RFC 6455에 실린 "Hello" 프레임 예시. 헤더는 2바이트(81 05)뿐이고, 클라이언트가 보낼 때만 마스킹 키 4바이트가 더 붙는다. 같은 내용을 HTTP 요청으로 보내면 헤더만 수백 바이트다.</figcaption>
</figure>

이 코드가 하는 일: 브라우저에서 WebSocket을 열고, 서버 메시지를 받으며, 클라이언트도 메시지를 보낸다. 마스킹·프레이밍은 브라우저가 알아서 한다.

```javascript
const ws = new WebSocket("wss://shop.example.com/ws/orders");

ws.onopen = () => ws.send(JSON.stringify({ type: "subscribe", orderId: 42 })); // 클라 → 서버
ws.onmessage = (e) => render(JSON.parse(e.data));                             // 서버 → 클라
ws.onclose = (e) => {
  // SSE와 달리 자동 재연결이 없다. 재연결·놓친 메시지 복구는 직접 구현해야 한다
  setTimeout(reconnect, backoff());
};
```

### 연결을 오래 살려 두는 비용

WebSocket과 SSE는 연결 하나를 **몇 분~몇 시간** 열어 둔다. 이게 평범한 HTTP와 가장 큰 운영 차이다.

- **중간 장비의 유휴 타임아웃(idle timeout)** — 로드밸런서·프록시는 한동안 데이터가 안 흐르는 연결을 끊는다. 예를 들어 nginx의 `proxy_read_timeout` 기본값은 60초다. 그래서 WebSocket은 **Ping/Pong 프레임**을, SSE는 **주석 줄(`: ping`)**을 주기적으로 흘려 연결이 살아 있음을 보여준다(heartbeat — 심장 박동).
- **프록시 설정** — nginx 뒤에 둘 때 WebSocket은 업그레이드 헤더를 넘겨줘야 하고, SSE는 응답을 모아 두는 **버퍼링(buffering)**을 꺼야 이벤트가 즉시 흘러간다.
- **서버 확장** — 서버가 여러 대면, 사용자 A는 서버 1에, 사용자 B는 서버 2에 연결돼 있다. 서버 1에서 생긴 "B에게 보낼 소식"은 서버 2로 넘겨야 한다. 보통 Redis Pub/Sub이나 Kafka 같은 **메시지 브로커**로 모든 서버에 소식을 뿌리고, 연결을 들고 있는 서버가 전달한다.
- **로드밸런싱 알고리즘** — 연결이 오래 사니 "요청 수"보다 **"지금 열린 연결 수"**가 부하를 잘 나타낸다. 지난 글의 **최소 연결(least connections)** 알고리즘이 잘 맞는 이유다.

이 코드가 하는 일: nginx에서 `/ws/`는 WebSocket으로 업그레이드해 넘기고, `/events/`는 SSE가 버퍼링 없이 흐르게 한다.

```nginx
location /ws/ {
    proxy_pass http://app_pool;
    proxy_http_version 1.1;                    # 업그레이드는 HTTP/1.1 기능
    proxy_set_header Upgrade $http_upgrade;    # 클라이언트의 Upgrade 헤더를 그대로 전달
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;                  # 기본 60초면 조용한 연결이 끊긴다 (heartbeat도 같이)
}

location /events/ {
    proxy_pass http://app_pool;
    proxy_http_version 1.1;
    proxy_set_header Connection "";            # 업스트림 연결을 keep-alive로
    proxy_buffering off;                       # 모아 두지 말고 이벤트를 바로 흘려보냄
    proxy_read_timeout 3600s;
}
```

## 상세 예시 — 1분 동안 알림 3건을 네 방식으로

조건을 정한다. 주문 상태 알림이 0초부터 60초 사이에 **7초, 31초, 52초**에 하나씩, 총 3건 생긴다. HTTP 요청-응답 한 번에 오가는 헤더를 약 700바이트로 잡는다(쿠키·인증 토큰 포함, 가정값). 알림 본문은 약 45바이트다.

**폴링 (5초 간격)** — 5, 10, 15, … 60초에 묻는다. 총 12번.

| 알림 발생 | 받는 시각 | 지연 |
|---|---|---|
| 7초 | 10초 | 3초 |
| 31초 | 35초 | 4초 |
| 52초 | 55초 | 3초 |

12번 중 9번이 헛걸음(75%). 평균 지연 (3+4+3)÷3 ≈ 3.3초.

**롱 폴링 (최대 30초 대기)** — 0초에 요청 → 7초에 응답, 즉시 재요청 → 31초에 응답, 재요청 → 52초에 응답, 재요청 → 60초 시점에도 대기 중. 총 4번, 지연은 거의 0(응답 왕복 시간뿐).

**SSE** — 0초에 요청 1번, 그 뒤엔 같은 응답 위로 이벤트 3개(`id`·`event`·`data` 줄 포함 약 80바이트씩)와 15초마다 `: ping\n\n`(8바이트) 4개.

**WebSocket** — 0초에 핸드셰이크 1번, 그 뒤엔 서버 → 클라 프레임 3개(헤더 2 + 본문 45 = 47바이트씩)와 Ping/Pong.

| 방식 | HTTP 요청 수 | 대략 바이트 (헤더 700B 가정) | 평균 지연 | 방향 |
|---|---|---|---|---|
| 폴링 5초 | 12 | 12×700 + 3×45 ≈ **8.5KB** | ≈ 3.3초 | 클 → 서 질문만 |
| 롱 폴링 | 4 | 4×700 + 3×45 ≈ **2.9KB** | ≈ 0초 | 클 → 서 질문만 |
| SSE | 1 | 700 + 3×80 + 4×8 ≈ **0.97KB** | ≈ 0초 | 서 → 클 |
| WebSocket | 1 (업그레이드) | 700 + 3×47 + 핑 몇 개 ≈ **0.9KB** | ≈ 0초 | 양방향 |

숫자에서 읽을 것은 두 가지다.

1. **폴링은 "지연"과 "헛걸음"을 맞바꾼다.** 주기를 1초로 줄이면 평균 지연은 약 0.5초로 줄지만 요청은 60번(약 42KB)으로 5배가 된다. 사용자가 1만 명이면 그만큼 곱해진다.
2. **SSE와 WebSocket은 비용이 "연결 수"로 옮겨 간다.** 요청·바이트는 거의 안 드는 대신, 사용자 1만 명이면 **열린 연결 1만 개**를 서버·로드밸런서가 들고 있어야 한다. 그래서 위 운영 항목(타임아웃·heartbeat·브로커)이 따라온다.

## 무엇을 고를까

| 상황 | 추천 | 이유 |
|---|---|---|
| 알림·피드·진행률처럼 **서버 → 클라만** 필요 | **SSE** | 평범한 HTTP라 인프라 친화적, 자동 재연결·`Last-Event-ID` 기본 제공 |
| 채팅·협업 편집·게임처럼 **양방향·고빈도** | **WebSocket** | 전이중, 프레임 헤더가 작음, 바이너리 지원 |
| 갱신이 드물고(분 단위) 구현을 최소화하고 싶다 | **폴링** | 가장 단순. 지연 허용 범위가 주기보다 크면 충분 |
| WebSocket·SSE를 막는 낡은 프록시 환경 | **롱 폴링** | 평범한 요청-응답이라 어디서나 통과 |

헷갈리면 이렇게 묻는다. **"클라이언트도 실시간으로 자주 말해야 하나?"** 아니면 SSE, 그렇다면 WebSocket이다. 클라이언트가 가끔 보내는 정도(예: "읽음 처리")는 SSE + 일반 `fetch` 조합으로 충분하다.

## 한 줄 교훈

HTTP는 "묻기 전엔 답하지 않는" 구조라, 폴링·롱 폴링은 **질문을 자주 하거나 길게 붙잡아** 실시간을 흉내 낸다. SSE는 **응답을 끝내지 않고 계속 써서** 서버 → 클라 한 방향을, WebSocket은 **HTTP로 문을 연 뒤 프로토콜을 갈아타서** 양방향을 연다. 대신 둘 다 비용이 "요청 수"에서 **"오래 열린 연결 수"**로 옮겨 가므로, heartbeat·프록시 설정·서버 간 메시지 전달까지 함께 설계해야 한다.
