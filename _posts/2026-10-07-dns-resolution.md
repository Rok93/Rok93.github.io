---
layout: post
title: 'DNS 동작 원리와 캐싱 — 주소창에 이름을 치면 누가 어디까지 물어보나'
date: 2026-10-07 09:00:00 +0900
image: /assets/img/dns-resolution/hero.jpg
generated: true
tags: [network, study]
sanitized: true
---

브라우저 주소창에 `shop.example.com`을 치면, 컴퓨터는 이 이름만으로는 아무 데도 갈 수 없다. 인터넷에서 패킷은 **IP 주소(Internet Protocol address — 네트워크상 컴퓨터의 숫자 주소, 예: 203.0.113.10)**로만 전달되기 때문이다. 사람이 외우기 쉬운 이름을 기계가 쓰는 숫자로 바꿔주는 전화번호부가 **DNS(Domain Name System — 도메인 이름을 IP 주소로 바꿔주는 분산 시스템)**다. 이 글은 이름 하나가 IP로 바뀌기까지 "누가 누구에게 물어보는지"를 순서대로 따라가고, 그 과정이 매번 반복되지 않도록 어디서 어떻게 **캐싱(caching — 한 번 알아낸 답을 잠시 저장해 두고 재사용)**되는지, 그리고 그 캐시 때문에 서버 IP를 바꿨는데 한동안 옛 주소로 트래픽이 가는 일이 왜 생기는지까지 숫자로 본다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 도메인·IP·수치·명령 출력은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다. (도메인은 문서용으로 예약된 example.com, IP는 문서용 대역 203.0.113.0/24를 씁니다.)</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/dns-resolution/hero.jpg" alt="메모장을 든 우체부가 세 갈래 계단길 앞에 서 있고, 길 끝마다 크기가 다른 집이 있다" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">우체부(리졸버)는 메모장(캐시)에 적힌 주소면 바로 가고, 없으면 큰 집(루트)부터 차례로 물어 내려간다.</figcaption>
</figure>

## DNS는 무엇인가 — 계층으로 나뉜 전화번호부

DNS는 전 세계 이름을 한 서버가 다 들고 있는 게 아니다. 이름을 점(`.`)으로 나눈 **계층(hierarchy)**마다 담당 서버가 따로 있다. `shop.example.com.`을 오른쪽부터 읽으면 이렇다.

- **루트(root, `.`)** — 맨 꼭대기. "`.com`은 누가 담당하는지"만 안다. 전 세계에 13개 이름(a~m.root-servers.net)이 있고, 실제 서버는 **애니캐스트(anycast — 같은 IP를 여러 지역에 두고 가장 가까운 곳이 응답하는 방식)**로 수백 대가 흩어져 있다.
- **TLD(Top-Level Domain — 최상위 도메인, `.com`·`.kr` 등)** 서버 — "`example.com`은 누가 담당하는지"를 안다.
- **권한 서버(authoritative name server — 그 도메인의 실제 레코드를 가진 서버)** — `shop.example.com`의 IP를 **정답으로** 들고 있다. 도메인 소유자가 관리한다.

이 세 층은 "정답을 가진 쪽"이다. 그리고 그 정답을 대신 찾아다 주는 쪽이 따로 있다.

- **스텁 리졸버(stub resolver — OS 안에 들어 있는 간단한 질문자)** — 프로그램이 `getaddrinfo()` 같은 함수를 부르면 동작한다. 스스로 찾아다니지 못하고 "알아서 찾아 달라"고 다음 단계에 넘긴다.
- **재귀 리졸버(recursive resolver — 끝까지 대신 찾아주는 서버)** — 통신사가 제공하거나 `8.8.8.8`(Google), `1.1.1.1`(Cloudflare) 같은 공개 서버다. 루트 → TLD → 권한 서버를 차례로 돌며 답을 모은 뒤 돌려주고, **그 답을 캐시에 저장**한다.

### 레코드 종류 — 답의 모양

권한 서버가 들고 있는 한 줄 한 줄을 **리소스 레코드(resource record, RR)**라 한다. 자주 보는 것만 추리면 이렇다.

| 타입 | 뜻 | 예 |
|---|---|---|
| **A** | 이름 → IPv4 주소 | `shop.example.com. 300 IN A 203.0.113.10` |
| **AAAA** | 이름 → IPv6 주소 | `shop.example.com. 300 IN AAAA 2001:db8::10` |
| **CNAME** | 이름 → 다른 이름(별칭) | `www.example.com. 300 IN CNAME shop.example.com.` |
| **NS** | 이 도메인의 권한 서버는 누구인가 | `example.com. 86400 IN NS ns1.example.com.` |
| **MX** | 메일은 어디로 | `example.com. 3600 IN MX 10 mail.example.com.` |
| **TXT** | 자유 텍스트(소유 확인, SPF 등) | `example.com. 3600 IN TXT "v=spf1 ..."` |
| **SOA** | 존(zone)의 관리 정보 | 일련번호, 부정 캐시 TTL 등 |

가운데 숫자(`300`, `86400`)가 이 글의 핵심인 **TTL(Time To Live — 이 답을 몇 초 동안 믿고 재사용해도 되는지)**이다. 권한 서버가 레코드마다 정한다.

## 이름이 IP로 바뀌는 순서 — 재귀와 반복

브라우저가 `shop.example.com`을 처음 묻는 순간부터, 아무 데도 캐시가 없다고 가정하고 따라가 보자.

<!-- diagram: dns-resolution-flow -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 230" role="img" aria-label="클라이언트가 재귀 리졸버에 한 번 묻고, 재귀 리졸버가 루트, TLD, 권한 서버를 차례로 돌며 참조를 받아 최종 IP를 얻어 돌려주는 흐름">
<defs>
<marker id="dr-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5f5a68"/></marker>
<marker id="dr-p" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5F0080"/></marker>
<marker id="dr-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#1f7a4d"/></marker>
</defs>
<rect x="8" y="90" width="62" height="40" rx="8" fill="#fff" stroke="#b9a9cc" stroke-width="1.4"/>
<text x="39" y="107" font-size="8" fill="#1a1720" text-anchor="middle">브라우저</text>
<text x="39" y="119" font-size="7" fill="#5f5a68" text-anchor="middle">+ 스텁 리졸버</text>
<rect x="110" y="84" width="76" height="52" rx="8" fill="#5F0080"/>
<text x="148" y="105" font-size="8" fill="#fff" text-anchor="middle" font-weight="700">재귀 리졸버</text>
<text x="148" y="118" font-size="7" fill="#e8dff0" text-anchor="middle">(캐시 보유)</text>
<rect x="250" y="12" width="78" height="30" rx="8" fill="#fff" stroke="#7d5a9e" stroke-width="1.4"/>
<text x="289" y="31" font-size="8" fill="#1a1720" text-anchor="middle">루트 서버 ( . )</text>
<rect x="250" y="95" width="78" height="30" rx="8" fill="#fff" stroke="#7d5a9e" stroke-width="1.4"/>
<text x="289" y="114" font-size="8" fill="#1a1720" text-anchor="middle">TLD 서버 (.com)</text>
<rect x="250" y="178" width="78" height="40" rx="8" fill="#fff" stroke="#1f7a4d" stroke-width="1.4"/>
<text x="289" y="194" font-size="8" fill="#1a1720" text-anchor="middle">권한 서버</text>
<text x="289" y="206" font-size="7" fill="#5f5a68" text-anchor="middle">example.com</text>
<line x1="70" y1="102" x2="110" y2="102" stroke="#5F0080" stroke-width="1.4" marker-end="url(#dr-p)"/>
<text x="90" y="97" font-size="7" fill="#5F0080" text-anchor="middle">① 재귀 질의</text>
<line x1="110" y1="120" x2="70" y2="120" stroke="#1f7a4d" stroke-width="1.4" marker-end="url(#dr-g)"/>
<text x="90" y="133" font-size="7" fill="#1f7a4d" text-anchor="middle">⑧ 203.0.113.10</text>
<line x1="186" y1="92" x2="250" y2="30" stroke="#5f5a68" stroke-width="1.2" marker-end="url(#dr-a)"/>
<text x="205" y="52" font-size="7" fill="#5f5a68">② shop.example.com?</text>
<line x1="250" y1="38" x2="190" y2="96" stroke="#5f5a68" stroke-width="1.2" stroke-dasharray="3 2" marker-end="url(#dr-a)"/>
<text x="196" y="72" font-size="7" fill="#5f5a68">③ .com은 저쪽</text>
<line x1="186" y1="107" x2="250" y2="107" stroke="#5f5a68" stroke-width="1.2" marker-end="url(#dr-a)"/>
<text x="218" y="103" font-size="7" fill="#5f5a68" text-anchor="middle">④ 같은 질문</text>
<line x1="250" y1="117" x2="186" y2="117" stroke="#5f5a68" stroke-width="1.2" stroke-dasharray="3 2" marker-end="url(#dr-a)"/>
<text x="218" y="129" font-size="7" fill="#5f5a68" text-anchor="middle">⑤ example.com은 ns1</text>
<line x1="186" y1="128" x2="250" y2="190" stroke="#5f5a68" stroke-width="1.2" marker-end="url(#dr-a)"/>
<text x="196" y="160" font-size="7" fill="#5f5a68">⑥ 같은 질문</text>
<line x1="250" y1="200" x2="190" y2="134" stroke="#1f7a4d" stroke-width="1.4" marker-end="url(#dr-g)"/>
<text x="206" y="184" font-size="7" fill="#1f7a4d">⑦ A 203.0.113.10 (TTL 300)</text>
<text x="170" y="226" font-size="7" fill="#5f5a68" text-anchor="middle">실선 = 질문, 점선 = "나는 모르니 저쪽에 물어봐"(참조)</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">클라이언트는 한 번만 묻고(재귀), 재귀 리졸버가 세 층을 차례로 돌며(반복) 답을 모아 온다.</figcaption>
</figure>

1. **브라우저 → 스텁 리졸버.** 브라우저는 자기 캐시를 보고, 없으면 OS에 "`shop.example.com` IP 알려줘"라고 한다. OS는 먼저 `/etc/hosts` 파일(손으로 적어둔 이름표)을 보고, 그다음 OS 자체 캐시를 본다.
2. **스텁 → 재귀 리졸버.** 둘 다 없으면 설정된 재귀 리졸버(예: `1.1.1.1`)에 질의를 보낸다. 이때 질의 헤더의 **RD 비트(Recursion Desired — "끝까지 찾아 달라" 표시)**를 켠다. 이게 **재귀 질의(recursive query)**다. 클라이언트는 이 한 번으로 끝이다.
3. **재귀 리졸버 → 루트.** 리졸버는 캐시에 없으면 루트 서버에 묻는다. 루트는 `shop.example.com`을 모른다. 대신 "`.com`은 이 서버들이 담당한다"는 **NS 레코드**를 돌려준다. 이렇게 답 대신 다음 서버를 알려주는 응답을 **참조(referral)**라 한다.
4. **재귀 리졸버 → TLD(.com).** 같은 질문을 `.com` 서버에 한다. 역시 모르지만 "`example.com`은 `ns1.example.com`이 담당한다"는 NS 레코드를 준다. 이때 `ns1.example.com`의 IP도 같이 준다. 안 그러면 "`ns1.example.com`의 IP를 알려면 `example.com` 서버에 물어야 하는데, 그 서버가 `ns1`이다"라는 **순환**에 빠지기 때문이다. 이 끼워 주는 IP를 **글루 레코드(glue record)**라 한다.
5. **재귀 리졸버 → 권한 서버.** 드디어 `ns1.example.com`에 묻는다. 권한 서버는 **AA 비트(Authoritative Answer — "내가 정답 주인"이라는 표시)**를 켜고 `A 203.0.113.10`, TTL 300을 돌려준다.
6. **재귀 리졸버 → 클라이언트.** 리졸버는 이 답을 캐시에 넣고 클라이언트에 돌려준다.

3~5번처럼 "하나 묻고, 참조 받고, 다음에 또 묻고"를 리졸버가 직접 반복하는 걸 **반복 질의(iterative query)**라 한다. 요약하면 **클라이언트↔리졸버는 재귀, 리졸버↔루트/TLD/권한 서버는 반복**이다.

### 전송 — UDP 53, 그리고 TCP로 넘어가는 순간

DNS는 기본적으로 **UDP 53번 포트**를 쓴다. 질문 한 번, 답 한 번이면 끝나는 짧은 대화라 TCP처럼 연결을 맺는 비용이 아깝기 때문이다. 원래 규격(RFC 1035)의 UDP 응답 상한은 512바이트였고, 지금은 **EDNS(0)(Extension Mechanisms for DNS — 더 큰 UDP 응답을 허용하는 확장)**로 보통 1232~4096바이트까지 쓴다. 답이 그래도 안 들어가면 서버가 **TC 비트(Truncated — "잘렸다" 표시)**를 켜서 보내고, 클라이언트는 같은 질문을 **TCP 53**으로 다시 한다. 존 전체를 복사하는 **존 전송(zone transfer, AXFR)**도 TCP를 쓴다.

최근에는 질의 내용을 중간에서 못 엿보게 **DoT(DNS over TLS, 853번 포트)**·**DoH(DNS over HTTPS, 443번 포트)**로 암호화하기도 한다. 동작 순서는 같고 포장만 바뀐다.

## 캐싱 — 왜, 어디서, 얼마나

위 과정을 접속할 때마다 반복하면 루트 서버가 남아나지 않는다. 그래서 답은 **지나온 곳마다 저장**된다.

<!-- diagram: dns-cache-layers -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 170" role="img" aria-label="브라우저 캐시, OS 캐시, 재귀 리졸버 캐시, 권한 서버 순으로 네 층이 나란히 있고, 질문은 왼쪽에서 오른쪽으로 캐시에 없을 때만 다음 층으로 넘어간다. 각 층에 남은 TTL이 표시된다.">
<defs>
<marker id="dc-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5f5a68"/></marker>
</defs>
<rect x="6" y="30" width="70" height="60" rx="8" fill="#fff" stroke="#b9a9cc" stroke-width="1.4"/>
<text x="41" y="48" font-size="8" fill="#1a1720" text-anchor="middle" font-weight="700">브라우저</text>
<text x="41" y="62" font-size="7" fill="#5f5a68" text-anchor="middle">자체 캐시</text>
<text x="41" y="78" font-size="7" fill="#7d5a9e" text-anchor="middle">수십 초~수 분</text>
<rect x="92" y="30" width="70" height="60" rx="8" fill="#fff" stroke="#b9a9cc" stroke-width="1.4"/>
<text x="127" y="48" font-size="8" fill="#1a1720" text-anchor="middle" font-weight="700">OS</text>
<text x="127" y="62" font-size="7" fill="#5f5a68" text-anchor="middle">/etc/hosts · 캐시</text>
<text x="127" y="78" font-size="7" fill="#7d5a9e" text-anchor="middle">TTL까지</text>
<rect x="178" y="30" width="70" height="60" rx="8" fill="#5F0080"/>
<text x="213" y="48" font-size="8" fill="#fff" text-anchor="middle" font-weight="700">재귀 리졸버</text>
<text x="213" y="62" font-size="7" fill="#e8dff0" text-anchor="middle">공유 캐시</text>
<text x="213" y="78" font-size="7" fill="#e8dff0" text-anchor="middle">TTL까지</text>
<rect x="264" y="30" width="70" height="60" rx="8" fill="#fff" stroke="#1f7a4d" stroke-width="1.4"/>
<text x="299" y="48" font-size="8" fill="#1a1720" text-anchor="middle" font-weight="700">권한 서버</text>
<text x="299" y="62" font-size="7" fill="#5f5a68" text-anchor="middle">정답 원본</text>
<text x="299" y="78" font-size="7" fill="#1f7a4d" text-anchor="middle">TTL 300 설정</text>
<line x1="76" y1="60" x2="92" y2="60" stroke="#5f5a68" stroke-width="1.2" marker-end="url(#dc-a)"/>
<line x1="162" y1="60" x2="178" y2="60" stroke="#5f5a68" stroke-width="1.2" marker-end="url(#dc-a)"/>
<line x1="248" y1="60" x2="264" y2="60" stroke="#5f5a68" stroke-width="1.2" marker-end="url(#dc-a)"/>
<text x="170" y="22" font-size="7" fill="#5f5a68" text-anchor="middle">캐시에 없을 때만 → 오른쪽으로</text>
<rect x="6" y="104" width="328" height="56" rx="8" fill="#fff" stroke="#e4e0ec"/>
<text x="14" y="120" font-size="8" fill="#1a1720" font-weight="700">같은 답을 10:00:00에 받아 캐시했다면</text>
<text x="14" y="134" font-size="7" fill="#5f5a68">10:00:00 응답 TTL 300  →  10:02:00 응답 TTL 180  →  10:04:59 응답 TTL 1</text>
<text x="14" y="148" font-size="7" fill="#b23a00">10:05:00 만료 → 권한 서버에 다시 물어 TTL 300으로 새로 채움</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">캐시는 받은 TTL을 그대로 두지 않고 1초마다 줄여서 돌려준다. 그래야 어느 층에서 받든 "정답이 만료되는 시각"이 같다.</figcaption>
</figure>

### TTL은 "남은 시간"으로 전달된다

권한 서버가 TTL 300으로 답하면, 재귀 리졸버는 "10:00:00에 받았고 300초짜리"로 저장한다. 2분 뒤 다른 사용자가 같은 걸 물으면 리졸버는 **TTL 180**으로 답한다. 그 사용자의 OS는 180초만 믿는다. 이렇게 하면 캐시가 여러 겹 쌓여도 모두가 **10:05:00에 함께 만료**된다. 만약 각 층이 받은 TTL을 300으로 되돌려 저장하면, 층을 거칠 때마다 수명이 5분씩 늘어나 권한 서버의 의도와 멀어진다.

### 부정 캐시 — "없다"도 기억한다

없는 이름을 물으면 권한 서버는 **NXDOMAIN(Non-Existent Domain — 그런 이름 없음)**으로 답한다. 이 "없음"도 캐시된다(**부정 캐싱, negative caching**, RFC 2308). 수명은 그 존의 **SOA 레코드**에 적힌 값(SOA의 MINIMUM 필드와 SOA 자체 TTL 중 작은 쪽)이다. 오타 난 이름을 매번 권한 서버까지 물어보지 않기 위해서다. 뒤집어 말하면, **새 서브도메인을 만들기 직전에 누가 그 이름을 물어봤다면**, 레코드를 추가해도 부정 캐시가 풀릴 때까지 "없음"이 돌아온다.

### 애플리케이션 안의 한 층 더 — JVM 캐시

캐시는 OS 위에도 있다. **JVM(Java Virtual Machine)**은 `InetAddress`로 한 번 찾은 이름을 자체 캐시에 둔다. 수명은 `java.security` 파일의 `networkaddress.cache.ttl`로 정하며, 현재 JDK의 기본값은 **30초**다(성공 응답 기준). 실패 응답은 `networkaddress.cache.negative.ttl`, 기본 **10초**다. `-1`로 두면 **영원히** 캐시한다. 오래된 가이드에서 "자바는 DNS를 영원히 캐시한다"고 하는 건 보안 관리자(SecurityManager)를 켠 옛 환경 얘기라, 지금 기본값과는 다르다. 다만 **HTTP 클라이언트나 커넥션 풀이 열어둔 연결**은 DNS와 무관하게 옛 IP에 붙어 있으므로, "DNS는 갱신됐는데 트래픽은 그대로"인 상황은 여기서도 생긴다.

## 상세 예시 — 서버 IP를 바꿨는데 왜 한 시간 동안 옛 주소로 가나

가상의 서비스 `shop.example.com`이 `203.0.113.10`에서 `203.0.113.20`으로 서버를 옮긴다고 하자. 레코드는 TTL **3600**(1시간)으로 운영해 왔다.

### 1) 바꾸기 전 — 지금 상태 확인

이 명령이 하는 일: 재귀 리졸버(1.1.1.1)에 `shop.example.com`의 A 레코드를 묻고, 남은 TTL을 본다.

```bash
$ dig @1.1.1.1 shop.example.com A +noall +answer
shop.example.com.   3600   IN   A   203.0.113.10

# 2분 뒤 같은 질문
$ dig @1.1.1.1 shop.example.com A +noall +answer
shop.example.com.   3480   IN   A   203.0.113.10
```

두 번째 응답의 TTL이 `3480`으로 줄었다. 리졸버가 캐시에서 꺼내 주면서 지난 120초를 뺀 것이다. 권한 서버에 직접 물으면 항상 원래 값이 나온다.

이 명령이 하는 일: 권한 서버 `ns1.example.com`에 직접 묻는다(캐시를 거치지 않음).

```bash
$ dig @ns1.example.com shop.example.com A +noall +answer
shop.example.com.   3600   IN   A   203.0.113.10
```

### 2) 그냥 바꾸면 생기는 일

10:00에 권한 서버의 레코드를 `203.0.113.20`으로 바꿨다. 그런데 09:59에 어떤 재귀 리졸버가 옛 답을 캐시했다면, 그 리졸버를 쓰는 모든 사용자는 **10:59까지** 옛 IP를 받는다. 리졸버는 권한 서버가 바뀐 걸 알 방법이 없다. 캐시가 만료돼야 비로소 다시 묻는다.

<!-- diagram: ttl-switch-timeline -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 180" role="img" aria-label="위 타임라인은 TTL 3600 그대로 IP를 바꿔 최대 1시간 동안 옛 IP가 섞여 나가는 모습, 아래 타임라인은 하루 전에 TTL을 60으로 줄여 두어 전환 후 1분 안에 모두 새 IP로 넘어가는 모습">
<text x="8" y="16" font-size="9" fill="#b23a00" font-weight="700">A. TTL 3600 그대로 바꿈</text>
<line x1="20" y1="48" x2="320" y2="48" stroke="#b9a9cc" stroke-width="2"/>
<rect x="20" y="30" width="110" height="14" rx="3" fill="#b9a9cc"/>
<text x="75" y="40" font-size="7" fill="#fff" text-anchor="middle">옛 IP .10 (TTL 3600)</text>
<rect x="130" y="30" width="110" height="14" rx="3" fill="#b23a00"/>
<text x="185" y="40" font-size="7" fill="#fff" text-anchor="middle">옛·새 IP 섞임 (최대 60분)</text>
<rect x="240" y="30" width="80" height="14" rx="3" fill="#1f7a4d"/>
<text x="280" y="40" font-size="7" fill="#fff" text-anchor="middle">새 IP .20</text>
<line x1="130" y1="26" x2="130" y2="54" stroke="#1a1720" stroke-width="1"/>
<text x="130" y="64" font-size="7" fill="#1a1720" text-anchor="middle">10:00 레코드 변경</text>
<text x="240" y="64" font-size="7" fill="#1a1720" text-anchor="middle">11:00</text>
<text x="8" y="94" font-size="9" fill="#1f7a4d" font-weight="700">B. 하루 전 TTL을 60으로 낮춰 둠</text>
<line x1="20" y1="126" x2="320" y2="126" stroke="#b9a9cc" stroke-width="2"/>
<rect x="20" y="108" width="60" height="14" rx="3" fill="#b9a9cc"/>
<text x="50" y="118" font-size="7" fill="#fff" text-anchor="middle">.10 (3600)</text>
<rect x="80" y="108" width="100" height="14" rx="3" fill="#7d5a9e"/>
<text x="130" y="118" font-size="7" fill="#fff" text-anchor="middle">.10 (TTL 60) ≥ 1시간 유지</text>
<rect x="180" y="108" width="12" height="14" rx="3" fill="#b23a00"/>
<rect x="192" y="108" width="128" height="14" rx="3" fill="#1f7a4d"/>
<text x="256" y="118" font-size="7" fill="#fff" text-anchor="middle">새 IP .20 → 이후 TTL 3600 복귀</text>
<line x1="80" y1="104" x2="80" y2="132" stroke="#1a1720" stroke-width="1"/>
<text x="80" y="142" font-size="7" fill="#1a1720" text-anchor="middle">전날 TTL 60으로</text>
<line x1="180" y1="104" x2="180" y2="132" stroke="#1a1720" stroke-width="1"/>
<text x="180" y="142" font-size="7" fill="#1a1720" text-anchor="middle">10:00 변경</text>
<text x="186" y="156" font-size="7" fill="#b23a00">섞이는 구간 ≤ 1분</text>
<text x="170" y="174" font-size="7" fill="#5f5a68" text-anchor="middle">회색 = 옛 IP만, 주황 = 옛·새 혼재, 초록 = 새 IP만</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">TTL은 "바꾼 뒤 전파되는 시간"이 아니라 "바꾸기 전에 뿌려둔 답의 유효기간"이다. 그래서 줄이는 것도 미리 해야 한다.</figcaption>
</figure>

### 3) 올바른 순서 — TTL부터 줄인다

핵심은 **TTL 변경 자체도 캐시된다**는 점이다. TTL을 60으로 낮춘 새 레코드가 전 세계 리졸버에 퍼지려면, 그 전에 뿌려둔 TTL 3600짜리 답이 다 만료돼야 한다. 그러니 순서는 이렇다.

이 절차가 하는 일: 옛 TTL이 다 만료될 시간을 벌어 둔 뒤 IP를 바꾸고, 안정되면 TTL을 되돌린다.

```text
D-1 09:00   shop.example.com  A 203.0.113.10  TTL 3600 → TTL 60 으로 변경
            (이후 최소 3600초, 넉넉히 하루 기다림 — 옛 3600짜리 캐시 소진)
D   10:00   shop.example.com  A 203.0.113.20  TTL 60   ← IP 변경
            (리졸버 캐시는 길어야 60초 뒤 새 IP로 전환)
D   10:05   dig @1.1.1.1 / @8.8.8.8 로 새 IP 확인
D+1         안정 확인 후 TTL 60 → 3600 복귀 (권한 서버 질의 부하 원상복구)
```

변경 직후 확인은 재귀 리졸버 여러 곳에 해 본다. 리졸버마다 캐시한 시각이 달라서 전환 시점이 제각각이기 때문이다.

```bash
$ dig @1.1.1.1 shop.example.com A +noall +answer
shop.example.com.   41   IN   A   203.0.113.20     # 이미 새 IP
$ dig @8.8.8.8 shop.example.com A +noall +answer
shop.example.com.   13   IN   A   203.0.113.10     # 13초 뒤 갱신될 옛 IP
```

### 4) 그래도 안 바뀐다면 — 아래쪽 캐시를 의심한다

리졸버는 새 IP를 주는데 특정 서버의 애플리케이션만 옛 IP로 붙는다면, 순서대로 본다.

1. **커넥션 풀** — 이미 열린 TCP 연결은 DNS를 다시 보지 않는다. 풀을 비우거나 연결 수명(max lifetime)을 둬야 한다.
2. **JVM 캐시** — `networkaddress.cache.ttl`이 `-1`로 박혀 있으면 프로세스를 재시작하기 전까지 옛 IP다.
3. **OS 캐시** — macOS는 `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder`, Windows는 `ipconfig /flushdns`, systemd-resolved는 `resolvectl flush-caches`로 비운다.
4. **`/etc/hosts`** — 테스트하려고 적어둔 줄이 남아 있는 경우가 의외로 흔하다.

### 5) 참고 — 전체 경로를 눈으로 보기

이 명령이 하는 일: 재귀 리졸버의 캐시를 쓰지 않고, 루트부터 권한 서버까지 반복 질의를 내 컴퓨터에서 직접 따라간다.

```bash
$ dig +trace shop.example.com A

.                   518400  IN  NS  a.root-servers.net.     # ① 루트 서버 목록
...
com.                172800  IN  NS  a.gtld-servers.net.     # ② 루트의 참조: .com은 저쪽
...
example.com.        172800  IN  NS  ns1.example.com.        # ③ .com의 참조: example.com은 ns1
ns1.example.com.    172800  IN  A   203.0.113.53            #    글루 레코드
...
shop.example.com.   60      IN  A   203.0.113.20            # ④ 권한 서버의 정답 (AA)
```

위 도식의 ②~⑦이 그대로 출력된다. NS 레코드의 TTL이 `172800`(2일)로 긴 이유도 보인다. 위층은 거의 안 바뀌니 오래 캐시해도 되고, 그래야 루트·TLD 부하가 낮아진다.

## 한 줄 교훈

DNS는 "클라이언트는 한 번 묻고, 리졸버가 루트→TLD→권한 서버를 대신 도는" 구조이고, 그 답은 지나온 모든 층에 **TTL이 줄어드는 형태로** 캐시된다. 그래서 레코드 변경은 바꾸는 순간이 아니라 **바꾸기 전에 뿌려둔 TTL이 끝나는 순간** 반영된다. IP를 바꿀 계획이면 TTL을 먼저, 그리고 미리 줄여라.
