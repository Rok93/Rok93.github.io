---
layout: post
title: '로드밸런싱(L4/L7)과 알고리즘 — 요청 하나가 여러 서버 중 어디로 가는지 누가 어떻게 정하나'
date: 2026-10-12 09:00:00 +0900
image: /assets/img/load-balancing/hero.jpg
generated: true
tags: [network, study]
sanitized: true
---

서버 한 대가 초당 1,000개 요청을 받다가 1,500개가 들어오면, 답은 보통 "서버를 두 대, 세 대로 늘리자"다. 그런데 서버를 늘리는 순간 새 문제가 생긴다. 사용자 요청 하나가 들어왔을 때 **그걸 어느 서버로 보낼지 누가 정하나?** 이 역할을 맡는 장비 또는 소프트웨어가 **로드밸런서(load balancer — 들어오는 요청을 여러 서버에 나눠 보내는 분배기)**이고, 나눠 보내는 행위를 **로드밸런싱(load balancing — 부하 분산)**이라 한다. 이 글은 로드밸런서가 요청의 "어디까지"를 보고 결정하는지(L4와 L7의 차이), 그리고 "어떤 규칙으로" 서버를 고르는지(알고리즘)를 숫자로 따라간다. 마지막엔 같은 요청 묶음을 두 알고리즘으로 돌렸을 때 결과가 어떻게 갈리는지도 본다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·트래픽·설정은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다. (IP는 문서용 대역 203.0.113.0/24와 사설 대역 10.0.0.0/8을 씁니다.)</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/load-balancing/hero.jpg" alt="교차로 한가운데 선 교통 경찰이 들어오는 차를 어느 갈래 길로 보낼지 정하고, 길 끝에는 작은 창구 건물이 있다" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">로드밸런서는 교차로의 교통 경찰이다. 들어온 차(요청)를 어느 길 끝 창구(서버)로 보낼지 정하는데, "번호판만 보고" 보낼지 "짐칸까지 열어 보고" 보낼지에 따라 L4와 L7로 갈린다.</figcaption>
</figure>

## 로드밸런서는 무엇인가 — 클라이언트와 서버 사이의 분배기

로드밸런서는 클라이언트와 서버 묶음 사이에 선다. 클라이언트는 로드밸런서의 주소(예: `203.0.113.10`) 하나만 알고, 그 뒤에 서버가 몇 대인지 모른다. 로드밸런서가 받은 요청을 뒤편 서버(**백엔드(backend)** 또는 **업스트림(upstream)**이라 부른다) 중 하나로 넘기고, 응답을 받아 클라이언트에게 돌려준다.

왜 중요한가. 세 가지 일을 한 번에 해결하기 때문이다.

- **확장(scale-out)** — 서버를 한 대 더 꽂으면 처리량이 늘어난다. 클라이언트 쪽은 바꿀 게 없다.
- **고가용성(high availability — 일부가 죽어도 서비스가 계속되는 성질)** — 서버 한 대가 죽으면 그쪽으로 요청을 안 보내면 된다. 이를 위해 로드밸런서는 **헬스 체크(health check — 서버가 살아 있는지 주기적으로 확인)**를 한다.
- **무중단 배포** — 서버를 하나씩 빼서 새 버전으로 올리고 다시 꽂을 수 있다.

여기까지는 로드밸런서가 "무엇"인지다. 이제 핵심 질문으로 간다. 로드밸런서는 요청의 **어디까지 열어 보고** 서버를 고르나?

## L4와 L7 — 번호판만 보나, 짐칸까지 여나

L4, L7의 L은 **OSI 7계층(OSI model — 네트워크 통신을 7개 층으로 나눈 모델)**의 계층 번호다. **L4는 전송 계층(transport layer — TCP/UDP가 사는 층)**, **L7은 응용 계층(application layer — HTTP가 사는 층)**이다. 로드밸런서가 어느 층의 정보까지 읽느냐가 이름이 된다.

<!-- diagram: l4-vs-l7 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 200" role="img" aria-label="하나의 패킷을 겹겹이 봉투로 그린 그림. 바깥 봉투에 IP 주소와 포트가, 안쪽 봉투에 HTTP 메서드와 경로와 헤더가 적혀 있다. L4 로드밸런서는 바깥 봉투까지만 읽고, L7 로드밸런서는 안쪽까지 연다.">
<defs>
<marker id="l47-p" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5F0080"/></marker>
<marker id="l47-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#1f7a4d"/></marker>
</defs>
<rect x="110" y="20" width="220" height="160" rx="10" fill="#fff" stroke="#b9a9cc" stroke-width="1.4"/>
<text x="124" y="38" font-size="8" fill="#5f5a68" font-weight="700">IP / TCP 헤더 (L3·L4)</text>
<text x="124" y="52" font-size="7.5" fill="#1a1720">src 198.51.100.7:51234</text>
<text x="124" y="64" font-size="7.5" fill="#1a1720">dst 203.0.113.10:443</text>
<rect x="124" y="76" width="192" height="92" rx="8" fill="#f4eff9" stroke="#7d5a9e" stroke-width="1.2"/>
<text x="136" y="92" font-size="8" fill="#5f5a68" font-weight="700">HTTP 요청 (L7)</text>
<text x="136" y="106" font-size="7.5" fill="#1a1720">GET /api/orders/42</text>
<text x="136" y="118" font-size="7.5" fill="#1a1720">Host: shop.example.com</text>
<text x="136" y="130" font-size="7.5" fill="#1a1720">Cookie: session=ab12</text>
<text x="136" y="142" font-size="7.5" fill="#1a1720">Accept: application/json</text>
<text x="136" y="158" font-size="7" fill="#5f5a68">(TLS면 복호화해야 보임)</text>
<rect x="6" y="28" width="86" height="40" rx="8" fill="#fff" stroke="#5F0080" stroke-width="1.4"/>
<text x="49" y="45" font-size="8" fill="#5F0080" text-anchor="middle" font-weight="700">L4 LB</text>
<text x="49" y="58" font-size="7" fill="#5f5a68" text-anchor="middle">IP·포트만 읽음</text>
<line x1="92" y1="48" x2="110" y2="48" stroke="#5F0080" stroke-width="1.4" marker-end="url(#l47-p)"/>
<rect x="6" y="100" width="86" height="40" rx="8" fill="#fff" stroke="#1f7a4d" stroke-width="1.4"/>
<text x="49" y="117" font-size="8" fill="#1f7a4d" text-anchor="middle" font-weight="700">L7 LB</text>
<text x="49" y="130" font-size="7" fill="#5f5a68" text-anchor="middle">경로·헤더까지 읽음</text>
<line x1="92" y1="120" x2="124" y2="120" stroke="#1f7a4d" stroke-width="1.4" marker-end="url(#l47-g)"/>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">같은 패킷이라도 L4는 바깥 봉투(IP·포트)까지만, L7은 안쪽 봉투(HTTP 경로·헤더·쿠키)까지 열어 본다. 볼 수 있는 정보가 다르니 내릴 수 있는 결정도 다르다.</figcaption>
</figure>

### L4 로드밸런서 — 연결 단위로 넘긴다

L4 로드밸런서가 보는 건 **출발지 IP·포트, 목적지 IP·포트, 프로토콜(TCP/UDP)** 이 다섯 가지뿐이다. HTTP 요청이 `GET`인지 `POST`인지, 경로가 `/api`인지 `/static`인지는 모른다. 암호화된 **TLS(Transport Layer Security — HTTPS의 암호화 층)** 내용은 더더욱 모른다.

그래서 결정 단위가 **연결(TCP connection)**이다. 클라이언트가 TCP 연결을 처음 맺는 순간(SYN 패킷) 서버 하나를 고르고, 그 연결로 오가는 이후 패킷은 전부 같은 서버로 보낸다. 중간에 요청 내용을 보고 다른 서버로 바꾸는 일은 없다.

패킷을 넘기는 방식은 크게 둘이다.

- **NAT(Network Address Translation — 패킷의 주소를 바꿔 쓰는 기법) 방식** — 목적지 IP를 로드밸런서 주소(`203.0.113.10`)에서 서버 주소(`10.0.1.11`)로 바꿔 보낸다. 응답은 다시 로드밸런서를 거쳐 돌아온다.
- **DSR(Direct Server Return — 응답은 서버가 직접 보내는 방식)** — 요청만 로드밸런서를 거치고, 응답은 서버가 클라이언트에게 바로 보낸다. 응답이 요청보다 훨씬 큰 서비스(동영상, 파일 다운로드)에서 로드밸런서 부담을 크게 줄인다.

장점은 **빠르고 싸다**는 것이다. 패킷 헤더 몇 바이트만 보고 넘기니 CPU를 거의 안 쓴다. 단점은 **똑똑한 결정을 못 한다**는 것이다. "이미지 요청은 저쪽 서버로"가 불가능하다.

### L7 로드밸런서 — 요청 단위로 넘긴다

L7 로드밸런서는 클라이언트의 TCP 연결을 **자기가 직접 받아서 끝낸다(terminate)**. HTTPS면 TLS도 여기서 풀어 평문 HTTP를 본다. 그다음 요청 내용을 읽고, **백엔드 서버와는 별도의 연결**을 맺어 요청을 전달한다. 클라이언트 입장에서 로드밸런서는 서버이고, 서버 입장에서 로드밸런서는 클라이언트다. 그래서 L7 로드밸런서를 **리버스 프록시(reverse proxy — 서버 앞에서 서버 대신 요청을 받는 중개자)**라고도 부른다.

요청 내용을 보니 결정 단위가 **HTTP 요청 하나**다. 같은 TCP 연결로 들어온 요청이라도 첫 번째는 A 서버, 두 번째는 B 서버로 보낼 수 있다. 그리고 보이는 게 많으니 할 수 있는 것도 많다.

- **경로 기반 라우팅** — `/api/*`는 API 서버 묶음으로, `/static/*`은 정적 파일 서버 묶음으로.
- **호스트 기반 라우팅** — `Host: shop.example.com`은 쇼핑 서버로, `Host: admin.example.com`은 관리자 서버로. IP 하나로 여러 서비스를 돌릴 수 있다.
- **쿠키 기반 고정 세션(sticky session — 같은 사용자를 계속 같은 서버로 보내기)** — 서버 메모리에 로그인 상태를 들고 있는 구조에서 필요하다.
- **재시도·압축·캐싱·속도 제한** — 요청을 이해하니 가능하다.

대가는 **비용**이다. 모든 연결을 끝내고 HTTP를 파싱하고 TLS를 복호화하니 CPU와 메모리를 L4보다 훨씬 많이 쓴다. 그래서 큰 서비스는 둘을 겹쳐 쓴다. 맨 앞에 L4가 넓게 받아 L7 로드밸런서 여러 대로 뿌리고, L7이 요청을 읽어 서비스별 서버로 보낸다.

| | L4 | L7 |
|---|---|---|
| 보는 정보 | IP·포트·프로토콜 | HTTP 경로·헤더·쿠키·본문 |
| 결정 단위 | TCP/UDP 연결 1개 | HTTP 요청 1개 |
| TLS | 못 봄(그냥 통과) | 복호화 후 봄 |
| 할 수 있는 것 | 연결 분배, DSR | 경로/호스트 라우팅, 고정 세션, 재시도, 캐싱 |
| 비용 | 매우 낮음 | 높음(파싱·복호화) |
| 예 | AWS NLB, LVS, HAProxy(tcp 모드) | AWS ALB, nginx, HAProxy(http 모드), Envoy |

## 알고리즘 — 서버가 셋일 때 이번 요청은 어디로?

L4든 L7이든 결국 "후보 서버 N대 중 하나"를 골라야 한다. 그 고르는 규칙이 알고리즘이다. 자주 쓰는 것을 단순한 순서로 본다.

### 라운드 로빈(Round Robin) — 순서대로 돌리기

A → B → C → A → B → C. 가장 단순하고, 상태를 기억할 게 "다음 차례" 하나뿐이다. 서버 성능이 같고 요청 무게도 비슷하면 충분하다. 문제는 **요청 무게가 다를 때**다. 어떤 요청은 5ms, 어떤 요청은 500ms가 걸리는데 순서만 지키면 느린 요청이 한 서버에 몰릴 수 있다. 아래 상세 예시에서 숫자로 본다.

### 가중 라운드 로빈(Weighted Round Robin) — 센 서버에 더 많이

서버 성능이 다를 때 쓴다. A에 가중치 3, B에 1을 주면 요청 4개 중 3개는 A로 간다. 순서는 구현마다 다른데, nginx는 A,A,A,B처럼 몰아 보내지 않고 A,B,A,A 식으로 **부드럽게 섞는다(smooth weighted round robin)**. 가중치는 "비율"이지 "절대 횟수"가 아니다.

### 최소 연결(Least Connections) — 지금 제일 한가한 서버로

각 서버가 **현재 처리 중인 연결(또는 요청) 수**를 세고, 가장 적은 서버로 보낸다. 느린 요청이 걸려 있는 서버는 연결 수가 줄지 않으니 자연히 새 요청을 덜 받는다. 라운드 로빈의 "무게 차이" 문제를 그대로 해결한다. 서버 성능이 다르면 **가중 최소 연결(weighted least connections)**로 "연결 수 ÷ 가중치"가 가장 작은 서버를 고른다.

### 최소 응답 시간(Least Response Time) — 제일 빨리 답하는 서버로

연결 수에 더해 **최근 응답 시간 평균**을 본다. 연결 수는 같은데 한 서버가 유난히 느리면(디스크 문제, GC 등) 그쪽을 피한다. 측정 비용이 들어 보통 L7에서 지원한다.

### 해시(Hash) — 같은 키는 항상 같은 서버로

출발지 IP, URL, 쿠키 값 같은 **키를 해시(hash — 임의 데이터를 고정 길이 숫자로 바꾸는 함수)해서 서버를 정한다.** `hash(키) mod N`이 기본형이다. 같은 키는 항상 같은 서버로 가니 **고정 세션**이나 **서버 로컬 캐시 적중률**에 좋다.

단순 `mod N`의 함정은 **서버 수 N이 바뀔 때**다. 3대에서 4대가 되면 `mod 3`과 `mod 4`의 결과가 거의 다 달라져, 키의 대부분이 다른 서버로 옮겨간다. 캐시가 전부 무효가 되는 셈이다. 이걸 막는 게 **일관 해싱(consistent hashing — 서버가 추가·제거돼도 키 대부분은 제자리에 남는 해시 방식)**이다.

<!-- diagram: consistent-hash-ring -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 200" role="img" aria-label="왼쪽은 해시 값을 0부터 2의 32제곱까지 원형 고리로 그린 것으로 서버 A, B, C가 고리 위 세 지점에 놓여 있고 키는 시계 방향으로 만나는 첫 서버에 배정된다. 오른쪽은 단순 mod 방식과 일관 해싱에서 서버 추가 시 이동하는 키 비율을 비교한다.">
<circle cx="90" cy="100" r="70" fill="none" stroke="#b9a9cc" stroke-width="2"/>
<text x="90" y="24" font-size="7" fill="#5f5a68" text-anchor="middle">0 / 2³²</text>
<circle cx="90" cy="30" r="7" fill="#5F0080"/>
<text x="90" y="33" font-size="7" fill="#fff" text-anchor="middle" font-weight="700">A</text>
<circle cx="150.6" cy="135" r="7" fill="#5F0080"/>
<text x="150.6" y="138" font-size="7" fill="#fff" text-anchor="middle" font-weight="700">B</text>
<circle cx="29.4" cy="135" r="7" fill="#5F0080"/>
<text x="29.4" y="141" font-size="7" fill="#fff" text-anchor="middle" font-weight="700">C</text>
<circle cx="139" cy="51" r="4" fill="#1f7a4d"/>
<text x="152" y="48" font-size="7" fill="#1f7a4d">key1 → B</text>
<circle cx="110" cy="166" r="4" fill="#1f7a4d"/>
<text x="112" y="182" font-size="7" fill="#1f7a4d">key2 → C</text>
<circle cx="24" cy="80" r="4" fill="#1f7a4d"/>
<text x="4" y="70" font-size="7" fill="#1f7a4d">key3 → A</text>
<path d="M 100 36 A 70 70 0 0 1 148 128" fill="none" stroke="#1f7a4d" stroke-width="1.2" stroke-dasharray="3 2"/>
<text x="90" y="105" font-size="7" fill="#5f5a68" text-anchor="middle">시계 방향으로</text>
<text x="90" y="116" font-size="7" fill="#5f5a68" text-anchor="middle">처음 만나는 서버</text>
<text x="255" y="40" font-size="8" fill="#1a1720" text-anchor="middle" font-weight="700">서버 3대 → 4대로 늘릴 때</text>
<text x="255" y="54" font-size="7.5" fill="#5f5a68" text-anchor="middle">옮겨가는 키 비율</text>
<rect x="190" y="70" width="60" height="20" rx="4" fill="#fff" stroke="#b9a9cc"/>
<text x="220" y="84" font-size="7.5" fill="#1a1720" text-anchor="middle">hash mod N</text>
<rect x="256" y="70" width="70" height="20" rx="4" fill="#b23a00"/>
<text x="291" y="84" font-size="7.5" fill="#fff" text-anchor="middle" font-weight="700">약 75%</text>
<rect x="190" y="100" width="60" height="20" rx="4" fill="#fff" stroke="#b9a9cc"/>
<text x="220" y="114" font-size="7.5" fill="#1a1720" text-anchor="middle">일관 해싱</text>
<rect x="256" y="100" width="70" height="20" rx="4" fill="#1f7a4d"/>
<text x="291" y="114" font-size="7.5" fill="#fff" text-anchor="middle" font-weight="700">약 25%</text>
<text x="255" y="140" font-size="7" fill="#5f5a68" text-anchor="middle">mod N: 결과가 거의 다 바뀜</text>
<text x="255" y="152" font-size="7" fill="#5f5a68" text-anchor="middle">일관 해싱: 새 서버 몫(1/N)만 이동</text>
<text x="255" y="172" font-size="7" fill="#5f5a68" text-anchor="middle">* 키가 고르게 퍼진 이상적 경우의 기대값</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">일관 해싱은 해시 공간을 고리로 보고 서버도 그 위에 올린다. 키는 시계 방향으로 처음 만나는 서버로 간다. 서버 D를 끼워 넣으면 D 바로 앞 구간의 키만 D로 옮겨가고 나머지는 그대로다.</figcaption>
</figure>

실제 구현은 서버 하나를 고리 위 한 점이 아니라 **수십~수백 개의 가상 노드(virtual node)**로 흩뿌린다. 한 점만 두면 서버 간 담당 구간 크기가 운에 따라 크게 달라지기 때문이다.

### 무작위 둘 중 덜 바쁜 쪽(Power of Two Choices, P2C) — 싸고 꽤 좋은 절충

최소 연결은 "N대 전부의 연결 수를 비교"해야 한다. 로드밸런서가 여러 대이고 각자 정보가 조금씩 다르면 **모두가 동시에 '제일 한가해 보이는' 한 서버로 몰려드는 떼몰림(herd)** 현상도 생긴다. P2C는 **무작위로 2대만 뽑아 그중 덜 바쁜 쪽**으로 보낸다. 비교 비용은 상수인데, 완전 무작위보다 부하 쏠림이 지수적으로 줄어든다는 게 알려져 있다. Envoy, HAProxy 등이 지원한다.

| 알고리즘 | 결정 근거 | 잘 맞는 경우 | 약점 |
|---|---|---|---|
| 라운드 로빈 | 순서 | 서버·요청이 균일 | 요청 무게 차이에 약함 |
| 가중 라운드 로빈 | 순서 + 가중치 | 서버 성능이 다름 | 위와 같음 |
| 최소 연결 | 현재 연결 수 | 요청 처리 시간이 들쭉날쭉 | 연결 수 집계 필요 |
| 최소 응답 시간 | 연결 수 + 응답 시간 | 서버 상태가 수시로 변함 | 측정 비용, 보통 L7 |
| 해시 / 일관 해싱 | 키의 해시 | 고정 세션, 캐시 적중 | 키 분포가 치우치면 쏠림 |
| P2C | 무작위 2개 중 덜 바쁜 쪽 | 서버 많음, LB 여러 대 | 최소 연결보다 약간 덜 정확 |

## 상세 예시 — 같은 요청 6개, 라운드 로빈과 최소 연결의 결과 차이

서버 A·B·C 3대가 있고, 요청 6개가 거의 동시에(0ms, 1ms, 2ms, 3ms, 4ms, 5ms) 들어온다고 하자. 처리 시간은 요청마다 다르다.

| 요청 | 도착 | 처리 시간 |
|---|---|---|
| r1 | 0ms | **300ms** (무거운 보고서 조회) |
| r2 | 1ms | 10ms |
| r3 | 2ms | 10ms |
| r4 | 3ms | **300ms** (무거운 보고서 조회) |
| r5 | 4ms | 10ms |
| r6 | 5ms | 10ms |

**라운드 로빈**은 순서만 본다. r1→A, r2→B, r3→C, r4→A, r5→B, r6→C.

- A는 r1(300ms)을 처리하는 중에 r4(300ms)를 또 받는다. 서버가 요청을 동시에 처리할 수 있어도 CPU를 나눠 쓰니 둘 다 느려지고, 처리 스레드가 1개라면 r4는 r1이 끝날 때까지 약 297ms를 기다린다.
- B와 C는 10ms짜리 둘씩 받고 금방 논다.

**최소 연결**은 보낼 때마다 "지금 처리 중인 연결 수"를 본다.

| 시각 | 요청 | A / B / C 처리 중 연결 수 | 선택 |
|---|---|---|---|
| 0ms | r1 | 0 / 0 / 0 | A (동률이면 순서대로) |
| 1ms | r2 | 1 / 0 / 0 | B |
| 2ms | r3 | 1 / 1 / 0 | C |
| 3ms | r4 | 1 / 1 / 1 | A (동률) |
| 4ms | r5 | 2 / 1 / 1 | B |
| 5ms | r6 | 2 / 2 / 1 | C |

어라, 결과가 똑같다. **요청이 거의 동시에 들어와서 연결 수가 아직 차이 나지 않았기 때문이다.** 최소 연결의 진가는 시간이 조금 흐른 뒤에 나온다. 시각을 조금 늘려 r4~r6이 15ms 이후에 들어온다고 바꾸면:

| 시각 | 요청 | A / B / C 처리 중 연결 수 | 라운드 로빈 | 최소 연결 |
|---|---|---|---|---|
| 0ms | r1 (300ms) | 0 / 0 / 0 | A | A |
| 1ms | r2 (10ms) | 1 / 0 / 0 | B | B |
| 2ms | r3 (10ms) | 1 / 1 / 0 | C | C |
| 15ms | r4 (300ms) | 1 / 0 / 0 (r2·r3 끝남) | **A** | **B** |
| 16ms | r5 (10ms) | RR: 2/0/0 · LC: 1/1/0 | B | C |
| 17ms | r6 (10ms) | RR: 2/1/0 · LC: 1/1/1 | C | A (동률) |

- 라운드 로빈은 15ms 시점에 A가 아직 r1을 붙들고 있는데도 **차례라서** r4(300ms)를 또 A에 준다. A에만 무거운 요청 둘이 겹친다.
- 최소 연결은 그 시점에 B·C가 0이라 r4를 **B**에 준다. 무거운 요청이 A·B로 갈라진다.

이 코드가 하는 일: nginx에서 서버 3대를 하나의 업스트림으로 묶고, 알고리즘을 라운드 로빈(기본)에서 최소 연결로 바꾼다.

```nginx
upstream app_pool {
    least_conn;                     # 이 한 줄을 지우면 기본값인 라운드 로빈
    server 10.0.1.11:8080;
    server 10.0.1.12:8080;
    server 10.0.1.13:8080 weight=2; # C가 2배 성능이면 가중치 (least_conn에서는 연결수÷가중치로 비교)
    server 10.0.1.14:8080 backup;   # 위 셋이 모두 죽었을 때만 쓰는 예비
}

server {
    listen 443 ssl;
    server_name shop.example.com;

    location /api/ {                # L7: 경로를 보고 다른 풀로 보낼 수 있다
        proxy_pass http://app_pool;
    }
    location /static/ {
        proxy_pass http://static_pool;
    }
}
```

`least_conn;` 한 줄이 위 표의 두 열을 가른다. nginx 오픈소스판은 라운드 로빈·`least_conn`·`hash`(일관 해싱은 `hash $key consistent`)·`ip_hash`·`random two`(P2C)를 지원하고, 응답 시간 기반(`least_time`)은 유료판에만 있다.

## 헬스 체크 — 죽은 서버로 보내지 않기

알고리즘이 아무리 좋아도 죽은 서버를 후보에 넣으면 소용없다. 헬스 체크는 두 종류다.

- **능동(active)** — 로드밸런서가 주기적으로(예: 5초마다) 각 서버에 `GET /health`를 보내고, 연속 N번(예: 3번) 실패하면 후보에서 뺀다. 다시 연속 M번 성공하면 넣는다. 사용자 요청과 무관하게 돌아 **빨리 알아채지만** 체크 트래픽이 든다.
- **수동(passive)** — 실제 사용자 요청의 결과를 본다. 어떤 서버가 연속으로 연결 실패나 5xx를 내면 잠시 뺀다(nginx의 `max_fails`·`fail_timeout`). 체크 트래픽은 없지만 **사용자 요청 몇 개는 실패를 겪은 뒤에야** 알아챈다.

둘을 같이 쓰는 게 보통이다. 그리고 `/health`가 "프로세스가 떠 있다"만 확인하는지, "DB 연결까지 된다"를 확인하는지에 따라 로드밸런서가 걸러내는 장애 범위가 달라진다. 너무 깊게 확인하면 DB가 잠깐 느릴 때 모든 서버가 동시에 빠지는 사고가 난다.

<!-- diagram: health-check-flow -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 150" role="img" aria-label="로드밸런서가 서버 A, B, C에 주기적으로 헬스 체크를 보낸다. A와 C는 200 응답으로 정상이고 B는 3번 연속 실패해 후보에서 빠진다. 사용자 요청은 A와 C로만 간다.">
<defs>
<marker id="hc-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#1f7a4d"/></marker>
<marker id="hc-r" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#b23a00"/></marker>
<marker id="hc-p" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#5F0080"/></marker>
</defs>
<rect x="8" y="55" width="56" height="40" rx="8" fill="#fff" stroke="#b9a9cc" stroke-width="1.4"/>
<text x="36" y="79" font-size="8" fill="#1a1720" text-anchor="middle">사용자</text>
<rect x="100" y="48" width="76" height="54" rx="8" fill="#5F0080"/>
<text x="138" y="70" font-size="8" fill="#fff" text-anchor="middle" font-weight="700">로드밸런서</text>
<text x="138" y="84" font-size="7" fill="#e8dff0" text-anchor="middle">후보: A, C</text>
<line x1="64" y1="75" x2="100" y2="75" stroke="#5F0080" stroke-width="1.4" marker-end="url(#hc-p)"/>
<rect x="256" y="10" width="72" height="30" rx="8" fill="#fff" stroke="#1f7a4d" stroke-width="1.4"/>
<text x="292" y="29" font-size="8" fill="#1a1720" text-anchor="middle">A  ✓ 200</text>
<rect x="256" y="60" width="72" height="30" rx="8" fill="#fff" stroke="#b23a00" stroke-width="1.4" stroke-dasharray="3 2"/>
<text x="292" y="79" font-size="8" fill="#b23a00" text-anchor="middle">B  ✗✗✗ 제외</text>
<rect x="256" y="110" width="72" height="30" rx="8" fill="#fff" stroke="#1f7a4d" stroke-width="1.4"/>
<text x="292" y="129" font-size="8" fill="#1a1720" text-anchor="middle">C  ✓ 200</text>
<line x1="176" y1="60" x2="256" y2="27" stroke="#1f7a4d" stroke-width="1.4" marker-end="url(#hc-g)"/>
<line x1="176" y1="75" x2="256" y2="75" stroke="#b23a00" stroke-width="1.2" stroke-dasharray="3 2" marker-end="url(#hc-r)"/>
<line x1="176" y1="90" x2="256" y2="124" stroke="#1f7a4d" stroke-width="1.4" marker-end="url(#hc-g)"/>
<text x="216" y="38" font-size="7" fill="#1f7a4d" text-anchor="middle">요청 + 체크</text>
<text x="216" y="70" font-size="7" fill="#b23a00" text-anchor="middle">체크만 (5초마다)</text>
<text x="216" y="118" font-size="7" fill="#1f7a4d" text-anchor="middle">요청 + 체크</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">B가 연속 3회 체크에 실패하면 후보에서 빠지고, 사용자 요청은 A·C로만 간다. 체크는 B에도 계속 보내서 살아나면 다시 넣는다.</figcaption>
</figure>

## 고정 세션 — 편하지만 분산을 망친다

로그인 상태를 서버 메모리(세션)에 들고 있으면, 사용자가 다음 요청에서 다른 서버로 가는 순간 "로그인이 풀린다". 해결은 둘 중 하나다.

1. **고정 세션** — 쿠키(L7)나 출발지 IP 해시(L4/L7)로 같은 사용자를 같은 서버에 묶는다. 쉽지만, 그 서버가 죽으면 세션이 날아가고, 특정 사용자 묶음이 몰린 서버에 부하가 쏠린다. 사내망처럼 모두가 같은 공인 IP로 나오면 IP 해시는 전원을 한 서버로 보낸다.
2. **세션을 서버 밖으로** — Redis 같은 공용 저장소에 세션을 두거나, **JWT(JSON Web Token — 서명된 토큰에 상태를 담아 클라이언트가 들고 다니는 방식)**로 서버를 무상태(stateless)로 만든다. 그러면 아무 서버로 가도 된다.

로드밸런싱을 제대로 쓰려면 2번이 정답에 가깝다. 고정 세션은 "당장 못 바꾸는 레거시"를 위한 임시 처방으로 보는 게 안전하다.

## 한 줄 교훈

로드밸런서는 **어디까지 보느냐(L4는 IP·포트, L7은 HTTP 내용)**로 할 수 있는 일이 정해지고, **어떤 규칙으로 고르느냐(라운드 로빈·최소 연결·해시·P2C)**로 분산 품질이 정해진다. 요청 무게가 들쭉날쭉하면 라운드 로빈 대신 최소 연결을, 서버가 자주 늘고 줄면 단순 해시 대신 일관 해싱을, 그리고 어떤 알고리즘이든 헬스 체크 없이는 죽은 서버로 요청을 보낸다는 걸 기억하면 된다.
