---
layout: post
title: 'TLS 핸드셰이크 — 처음 만난 서버와 "우리만 아는 비밀 열쇠"를 만드는 법'
date: 2026-09-23 09:00:00 +0900
image: /assets/img/tls-handshake/hero.jpg
generated: true
tags: [network, study]
sanitized: true
---

주소창에 `https://`가 붙은 사이트에 들어가면, 브라우저와 서버는 **처음 만난 사이인데도** 몇십 밀리초 만에 "둘만 아는 비밀 열쇠"를 만들어낸다. 그 사이를 오가는 데이터는 카페 와이파이·통신사·중간 라우터 누구나 엿볼 수 있는 **공개된 길**을 지나는데도 말이다. 이 마법 같은 과정이 **TLS 핸드셰이크(TLS handshake — 암호화 통신을 시작하기 전, 양쪽이 "누구인지 확인하고 암호 열쇠를 합의하는" 인사 절차)**다. TLS(Transport Layer Security)는 HTTPS의 'S'를 담당하는 보안 계층이다. 이 글은 핸드셰이크가 **무엇을** 해내야 하는지, **왜** 그 순서로 하는지, 그리고 TLS 1.2 → 1.3에서 **어떻게 왕복 한 번을 줄였는지**를 도식과 실제 명령어 예시로 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·출력·설정은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/tls-handshake/hero.jpg" alt="공개된 광장에서 두 사람이 서로 신분증을 확인하며 자물쇠를 주고받는 그림" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">모두가 지켜보는 광장에서, 서로 신분을 확인하고 둘만 여는 자물쇠를 맞추는 것 — 그게 TLS 핸드셰이크다.</figcaption>
</figure>

## 핸드셰이크가 해내야 하는 세 가지

**무엇**부터. 핸드셰이크는 본 데이터를 보내기 전에 딱 세 가지 목표를 이룬다.

1. **서버 인증(authentication)** — "지금 대화하는 상대가 진짜 `bank.example.com`인가?"를 확인한다. 가짜 서버가 중간에 끼어드는 **중간자 공격(MITM, Man-In-The-Middle — 둘 사이에 몰래 끼어 대화를 엿보거나 바꾸는 공격)**을 막기 위해서다.
2. **키 합의(key agreement)** — 둘만 아는 **세션 키(session key — 이번 연결에서만 쓰는 암호 열쇠)**를 만든다. 핵심은 이 열쇠를 **네트워크로 직접 보내지 않고도** 양쪽이 똑같이 계산해낸다는 점이다.
3. **암호 방식 협상** — 어떤 알고리즘으로 암호화할지 고른다. 이 조합을 **암호 스위트(cipher suite)**라 부른다. 예: `TLS_AES_128_GCM_SHA256`.

**왜** 세 개가 다 필요한가? 키만 합의하고 인증을 빼면, 가짜 서버와도 "안전한 비밀 열쇠"를 만들어버린다. 문은 잠갔는데 **도둑과 같은 방에 들어가서 잠근 꼴**이다. 반대로 인증만 하고 키 합의를 빼면 상대는 확실한데 대화가 다 들린다. 둘이 함께여야 의미가 있다.

## 열쇠를 보내지 않고 합의하기 — (EC)DHE 키 교환

"열쇠를 보내지 않고 똑같은 열쇠를 갖는다"가 가장 신기한 대목이다. 여기에 쓰이는 방법이 **디피-헬만 키 교환(Diffie-Hellman key exchange)**이고, TLS에서는 주로 타원곡선 버전인 **ECDHE(Elliptic Curve Diffie-Hellman Ephemeral)**를 쓴다. 이름 끝의 **Ephemeral(임시)**은 "연결마다 새로 만들고 쓰고 버리는 키"라는 뜻이다.

흔히 쓰는 비유는 **물감 섞기**다(수학적으로 정확한 설명은 아니지만 구조는 같다).

- 둘 다 아는 **공용 색(노랑)**이 있다. 공개돼도 괜찮다.
- 클라이언트는 **자기만 아는 비밀 색(빨강)**을, 서버는 **자기만 아는 비밀 색(파랑)**을 고른다.
- 각자 공용 색에 비밀 색을 섞어 **혼합색**을 만들어 **공개적으로** 교환한다. (노랑+빨강 = 주황, 노랑+파랑 = 초록)
- 받은 혼합색에 **자기 비밀 색을 한 번 더** 섞는다. 클라이언트: 초록+빨강, 서버: 주황+파랑 → **둘 다 노랑+빨강+파랑**, 같은 색이 된다.

엿보는 사람은 노랑·주황·초록을 다 봤지만, **섞인 색에서 원래 비밀 색을 뽑아내는 건 사실상 불가능**하다. 실제 ECDHE에서는 "섞기"가 타원곡선 위의 곱셈이고, "되돌리기"는 현실적인 시간 안에 풀 수 없는 수학 문제(이산 로그 문제)다.

<!-- diagram: ecdhe-key-exchange -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 200" role="img" aria-label="클라이언트와 서버가 각자의 비밀값은 숨긴 채 공개값만 교환하고, 받은 공개값에 자기 비밀값을 더해 같은 공유 비밀을 계산하는 ECDHE 키 교환 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="16" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">ECDHE — 비밀은 숨기고, 공개값만 교환</text>
    <text x="60" y="36" text-anchor="middle" fill="#1a1720" font-weight="700" font-size="9">클라이언트</text>
    <text x="270" y="36" text-anchor="middle" fill="#1a1720" font-weight="700" font-size="9">서버</text>
    <rect x="15" y="44" width="90" height="22" rx="4" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="60" y="58" text-anchor="middle" fill="#1a1720" font-size="8">비밀값 a (안 보냄)</text>
    <rect x="225" y="44" width="90" height="22" rx="4" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="270" y="58" text-anchor="middle" fill="#1a1720" font-size="8">비밀값 b (안 보냄)</text>
    <rect x="15" y="76" width="90" height="22" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="60" y="90" text-anchor="middle" fill="#1a1720" font-size="8">공개값 A = a·G</text>
    <rect x="225" y="76" width="90" height="22" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="270" y="90" text-anchor="middle" fill="#1a1720" font-size="8">공개값 B = b·G</text>
    <line x1="108" y1="84" x2="222" y2="108" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#ah1)"/>
    <line x1="222" y1="84" x2="108" y2="108" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#ah1)"/>
    <text x="165" y="80" text-anchor="middle" fill="#5f5a68" font-size="7">공개된 길로 교환 (엿봐도 OK)</text>
    <rect x="15" y="116" width="90" height="22" rx="4" fill="#e3f1ea" stroke="#1f7a4d"/>
    <text x="60" y="130" text-anchor="middle" fill="#1a1720" font-size="8">a·B = a·b·G</text>
    <rect x="225" y="116" width="90" height="22" rx="4" fill="#e3f1ea" stroke="#1f7a4d"/>
    <text x="270" y="130" text-anchor="middle" fill="#1a1720" font-size="8">b·A = a·b·G</text>
    <text x="165" y="158" text-anchor="middle" fill="#1f7a4d" font-size="8.5" font-weight="700">양쪽이 같은 공유 비밀 → 세션 키 유도</text>
    <text x="165" y="176" text-anchor="middle" fill="#5f5a68" font-size="7.5">도청자는 A·B·G만 봄 → a·b·G 계산 불가 (이산 로그 문제)</text>
    <text x="165" y="190" text-anchor="middle" fill="#5f5a68" font-size="7">G: 모두가 아는 타원곡선의 기준점</text>
    <defs><marker id="ah1" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#7d5a9e"/></marker></defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">열쇠 자체는 한 번도 네트워크를 지나가지 않는다. 각자 계산해서 같은 값에 도달할 뿐이다.</figcaption>
</figure>

여기서 중요한 성질이 하나 따라온다. **전방 비밀성(forward secrecy — 나중에 서버의 개인키가 털려도, 과거에 녹화해둔 대화는 풀 수 없는 성질)**이다. 세션 키가 연결마다 새로 만든 임시 비밀값(a, b)에서 나오고 그 값은 쓰고 버리기 때문에, 서버의 장기 개인키를 훔쳐도 과거 세션 키를 되살릴 수 없다. 옛날 TLS에서 쓰던 **RSA 키 전송 방식**(클라이언트가 세션 키 재료를 서버 공개키로 잠가 보내는 방식)은 이 성질이 없어서, 개인키 하나가 털리면 녹화된 과거 트래픽이 전부 풀렸다. TLS 1.3이 RSA 키 전송을 **아예 없앤** 이유다.

## 상대가 진짜인지 확인하기 — 인증서와 신뢰 사슬

키 교환만으로는 "상대가 누구인지"를 모른다. 이걸 해결하는 게 **인증서(certificate — "이 공개키의 주인은 bank.example.com이다"라고 제3자가 서명해준 전자 신분증)**다.

서버는 핸드셰이크 중에 인증서를 보내고, 브라우저는 다음을 확인한다.

- **서명 사슬(chain of trust)**: 서버 인증서는 **중간 인증기관(Intermediate CA)**이 서명했고, 중간 CA 인증서는 **루트 인증기관(Root CA)**이 서명했다. 루트 CA 인증서는 OS·브라우저에 **미리 내장된 신뢰 저장소(trust store)**에 들어 있다. 사슬을 따라 올라가 내장된 루트에 닿으면 통과.
- **도메인 일치**: 인증서의 **SAN(Subject Alternative Name — 이 인증서가 유효한 도메인 목록)**에 접속한 도메인이 있는가.
- **유효기간**: 만료되지 않았는가.
- **개인키 소유 증명**: 인증서는 공개 문서라 누구나 복사해서 내밀 수 있다. 그래서 서버는 **CertificateVerify** 메시지로 "지금까지의 핸드셰이크 내용"에 **자기 개인키로 서명**해 보낸다. 브라우저는 인증서 속 공개키로 이 서명을 검증한다. 개인키가 없는 가짜 서버는 이 서명을 만들 수 없다.

<!-- diagram: chain-of-trust -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 190" role="img" aria-label="서버 인증서는 중간 CA가 서명하고 중간 CA는 루트 CA가 서명하며, 루트 CA는 브라우저 신뢰 저장소에 내장되어 있어 사슬을 따라 검증하는 신뢰 사슬 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="16" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">신뢰 사슬 — 내장된 루트까지 거슬러 올라간다</text>
    <rect x="30" y="30" width="150" height="30" rx="5" fill="#e3f1ea" stroke="#1f7a4d"/>
    <text x="105" y="43" text-anchor="middle" fill="#1a1720" font-size="8.5" font-weight="700">Root CA</text>
    <text x="105" y="54" text-anchor="middle" fill="#5f5a68" font-size="7">자기 자신이 서명 (self-signed)</text>
    <rect x="30" y="85" width="150" height="30" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="105" y="98" text-anchor="middle" fill="#1a1720" font-size="8.5" font-weight="700">Intermediate CA</text>
    <text x="105" y="109" text-anchor="middle" fill="#5f5a68" font-size="7">Root CA가 서명</text>
    <rect x="30" y="140" width="150" height="30" rx="5" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="105" y="153" text-anchor="middle" fill="#1a1720" font-size="8.5" font-weight="700">bank.example.com</text>
    <text x="105" y="164" text-anchor="middle" fill="#5f5a68" font-size="7">Intermediate CA가 서명</text>
    <line x1="105" y1="84" x2="105" y2="62" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#ah2)"/>
    <line x1="105" y1="139" x2="105" y2="117" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#ah2)"/>
    <rect x="205" y="30" width="110" height="30" rx="5" fill="#ffffff" stroke="#1f7a4d" stroke-dasharray="3 2"/>
    <text x="260" y="43" text-anchor="middle" fill="#1f7a4d" font-size="8">브라우저/OS</text>
    <text x="260" y="54" text-anchor="middle" fill="#1f7a4d" font-size="8">신뢰 저장소에 내장</text>
    <line x1="203" y1="45" x2="183" y2="45" stroke="#1f7a4d" stroke-width="1"/>
    <text x="260" y="96" text-anchor="middle" fill="#5f5a68" font-size="7.5">서버가 보내는 것:</text>
    <text x="260" y="108" text-anchor="middle" fill="#5f5a68" font-size="7.5">자기 인증서 + 중간 CA</text>
    <text x="260" y="150" text-anchor="middle" fill="#5f5a68" font-size="7.5">추가 확인: 도메인(SAN)</text>
    <text x="260" y="162" text-anchor="middle" fill="#5f5a68" font-size="7.5">· 유효기간 · 개인키 서명</text>
  </g>
  <defs><marker id="ah2" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#7d5a9e"/></marker></defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">화살표는 "누가 서명했나"를 거꾸로 따라가는 방향. 끝이 내장된 루트에 닿아야 믿는다.</figcaption>
</figure>

실무에서 자주 보는 오류가 여기서 나온다. 서버가 **중간 CA 인증서를 빼먹고** 자기 인증서만 보내면, 일부 클라이언트(특히 서버 간 통신에 쓰는 HTTP 클라이언트)는 사슬을 잇지 못해 `unable to get local issuer certificate` 같은 오류를 낸다. 브라우저는 중간 인증서를 캐시하거나 따로 받아오기도 해서 "브라우저에선 되는데 서버 코드에선 안 된다"는 헷갈리는 상황이 생긴다.

## 순서대로 보기 — TLS 1.2는 2왕복, TLS 1.3은 1왕복

이제 세 가지 목표를 **어떤 메시지 순서로** 해내는지 보자. **RTT(Round-Trip Time — 요청을 보내고 응답이 돌아오기까지 걸리는 왕복 시간)** 단위로 세면 차이가 선명하다.

**TLS 1.2 (ECDHE 기준)** — 2 RTT

1. 클라이언트 → **ClientHello**: "내가 지원하는 TLS 버전·암호 스위트 목록은 이거야" + 무작위값.
2. 서버 → **ServerHello**(스위트 선택) + **Certificate**(인증서) + **ServerKeyExchange**(서버의 ECDHE 공개값, 서명 포함) + **ServerHelloDone**.
3. 클라이언트 → **ClientKeyExchange**(클라이언트의 ECDHE 공개값) + **ChangeCipherSpec**("이제부터 암호화한다") + **Finished**.
4. 서버 → **ChangeCipherSpec** + **Finished**. → 여기서야 데이터 전송 시작.

**왜** 2왕복이나 걸리나? 클라이언트가 **"서버가 어떤 방식을 고를지 몰라서"** 첫 메시지에 키 교환 재료를 못 싣기 때문이다. 먼저 물어보고(1왕복), 답을 듣고 나서 재료를 보낸다(2왕복).

**TLS 1.3** — 1 RTT

TLS 1.3(2018년, RFC 8446)은 발상을 바꿨다. **"어차피 거의 다 ECDHE(X25519 같은 곡선)를 쓰니까, 첫 메시지에 공개값을 미리 실어 보내자."**

1. 클라이언트 → **ClientHello + key_share**(내 ECDHE 공개값을 **추측해서 미리** 첨부).
2. 서버 → **ServerHello + key_share**(서버 공개값). 이 순간 양쪽은 이미 공유 비밀을 계산할 수 있다. 그래서 이어지는 **EncryptedExtensions·Certificate·CertificateVerify·Finished**는 **이미 암호화된 채로** 보낸다.
3. 클라이언트 → **Finished** + 곧바로 **애플리케이션 데이터**.

클라이언트의 추측이 틀리면(서버가 그 곡선을 지원 안 하면) 서버가 **HelloRetryRequest**로 "이 곡선으로 다시 보내줘"라고 답한다. 이 경우엔 1왕복이 더 든다. 하지만 대부분은 첫 추측이 맞는다.

<!-- diagram: tls12-vs-tls13 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 230" role="img" aria-label="TLS 1.2는 ClientHello, ServerHello와 인증서, ClientKeyExchange와 Finished, 서버 Finished까지 2왕복 후 데이터 전송, TLS 1.3은 ClientHello에 key_share를 실어 1왕복 후 데이터 전송하는 비교 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="85" y="16" text-anchor="middle" fill="#b23a00" font-weight="700" font-size="10">TLS 1.2 — 2 RTT</text>
    <text x="255" y="16" text-anchor="middle" fill="#1f7a4d" font-weight="700" font-size="10">TLS 1.3 — 1 RTT</text>
    <line x1="170" y1="24" x2="170" y2="222" stroke="#e4e0ec" stroke-width="1"/>
    <text x="20" y="32" fill="#5f5a68" font-size="7.5">클라</text>
    <text x="135" y="32" fill="#5f5a68" font-size="7.5">서버</text>
    <line x1="30" y1="36" x2="30" y2="215" stroke="#b9a9cc" stroke-width="1.5"/>
    <line x1="145" y1="36" x2="145" y2="215" stroke="#b9a9cc" stroke-width="1.5"/>
    <line x1="30" y1="46" x2="145" y2="66" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#ah3)"/>
    <text x="88" y="50" text-anchor="middle" fill="#1a1720" font-size="7">ClientHello</text>
    <line x1="145" y1="76" x2="30" y2="96" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#ah3)"/>
    <text x="88" y="80" text-anchor="middle" fill="#1a1720" font-size="7">ServerHello·Cert·KeyEx·Done</text>
    <line x1="30" y1="112" x2="145" y2="132" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#ah3)"/>
    <text x="88" y="116" text-anchor="middle" fill="#1a1720" font-size="7">ClientKeyEx·CCS·Finished</text>
    <line x1="145" y1="142" x2="30" y2="162" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#ah3)"/>
    <text x="88" y="146" text-anchor="middle" fill="#1a1720" font-size="7">CCS·Finished</text>
    <line x1="30" y1="180" x2="145" y2="196" stroke="#1f7a4d" stroke-width="1.6" marker-end="url(#ah4)"/>
    <text x="88" y="184" text-anchor="middle" fill="#1f7a4d" font-size="7.5" font-weight="700">데이터 시작</text>
    <text x="190" y="32" fill="#5f5a68" font-size="7.5">클라</text>
    <text x="305" y="32" fill="#5f5a68" font-size="7.5">서버</text>
    <line x1="200" y1="36" x2="200" y2="215" stroke="#b9a9cc" stroke-width="1.5"/>
    <line x1="315" y1="36" x2="315" y2="215" stroke="#b9a9cc" stroke-width="1.5"/>
    <line x1="200" y1="46" x2="315" y2="66" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#ah3)"/>
    <text x="258" y="50" text-anchor="middle" fill="#1a1720" font-size="7">ClientHello + key_share</text>
    <line x1="315" y1="76" x2="200" y2="96" stroke="#7d5a9e" stroke-width="1.2" marker-end="url(#ah3)"/>
    <text x="258" y="80" text-anchor="middle" fill="#1a1720" font-size="7">ServerHello + key_share</text>
    <text x="258" y="104" text-anchor="middle" fill="#5F0080" font-size="6.8">🔒 {Cert·CertVerify·Finished}</text>
    <line x1="200" y1="114" x2="315" y2="132" stroke="#1f7a4d" stroke-width="1.6" marker-end="url(#ah4)"/>
    <text x="258" y="118" text-anchor="middle" fill="#1f7a4d" font-size="7.5" font-weight="700">Finished + 데이터 시작</text>
    <text x="258" y="160" text-anchor="middle" fill="#1f7a4d" font-size="7.5">첫 메시지에 키 재료를 미리 실어</text>
    <text x="258" y="172" text-anchor="middle" fill="#1f7a4d" font-size="7.5">왕복 1번 절약</text>
    <text x="258" y="190" text-anchor="middle" fill="#5f5a68" font-size="7">🔒 = 이미 암호화된 메시지</text>
  </g>
  <defs>
    <marker id="ah3" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#7d5a9e"/></marker>
    <marker id="ah4" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="5" markerHeight="5" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#1f7a4d"/></marker>
  </defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">TLS 1.3은 "먼저 묻고 나서 재료 보내기"를 "재료를 미리 실어 보내기"로 바꿔 왕복 한 번을 없앴다. 인증서도 암호화되어 오간다.</figcaption>
</figure>

TLS 1.3이 바꾼 건 속도만이 아니다.

- **인증서까지 암호화**: TLS 1.2에선 인증서가 평문으로 오가 "누구와 통신하는지"가 노출됐다. 1.3에선 ServerHello 이후 전부 암호화된다. (단, 접속할 도메인을 알리는 **SNI(Server Name Indication)**는 ClientHello에 여전히 평문으로 실린다. 이를 가리는 ECH라는 확장이 별도로 추진 중이다.)
- **약한 선택지 제거**: RSA 키 전송, 정적 DH, CBC 모드, RC4, SHA-1 기반 서명 등을 뺐다. 남은 암호 스위트는 **AEAD(인증 암호화 — 암호화와 위변조 검사를 한 번에 하는 방식)** 계열인 AES-GCM, ChaCha20-Poly1305 등 5개뿐이다. 고를 게 적으니 잘못 고를 일도 적다.

## 두 번째 방문은 더 빠르게 — 세션 재개와 0-RTT

한 번 핸드셰이크를 마친 서버는 **세션 티켓(session ticket — "다음에 오면 이걸 보여줘"라며 주는 재입장권)**을 준다. 다음 접속에서 클라이언트가 이 티켓(TLS 1.3에선 **PSK, Pre-Shared Key — 미리 공유된 키**라 부름)을 내밀면 인증서 검증 같은 무거운 과정을 건너뛴다.

TLS 1.3은 한 발 더 나가 **0-RTT(early data)**를 허용한다. ClientHello와 **함께 첫 요청 데이터를 바로** 보내는 것이다. 왕복을 기다릴 필요가 아예 없다.

**왜** 조심해야 하나? 0-RTT 데이터는 **재전송 공격(replay attack — 공격자가 가로챈 요청을 그대로 다시 보내는 공격)**에 약하다. 공격자가 "송금 1만 원" 요청을 녹화해 다시 보내면 서버가 두 번 처리할 수 있다. 그래서 0-RTT는 **GET처럼 여러 번 실행돼도 결과가 같은(멱등한) 요청**에만 쓰는 게 원칙이다. 결제·주문 같은 요청을 0-RTT로 받으면 안 된다.

## 상세 예시 — 명령어로 핸드셰이크를 직접 들여다보기

말로만 보면 추상적이니 직접 확인해보자. `curl`의 상세 출력으로 핸드셰이크 메시지가 오가는 순서를 볼 수 있다.

**이 명령이 하는 일**: TLS 1.3으로 접속하면서, 헤더·본문은 버리고 핸드셰이크 로그만 걸러 본다.

```bash
curl -sv --tlsv1.3 https://example.com -o /dev/null 2>&1 | grep -E 'TLS|SSL connection'
```

예시 출력(curl이 OpenSSL을 쓸 때의 형식. 빌드·버전에 따라 문구는 조금 다를 수 있다):

```text
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
```

앞 절의 TLS 1.3 흐름이 그대로 보인다. OUT(Client hello) → IN(Server hello부터 Finished까지 한 묶음) → OUT(Finished). **왕복 한 번**이다. 중간의 `Change cipher spec`은 TLS 1.3에선 의미 없는 메시지인데, 옛 TLS만 아는 중간 장비(방화벽 등)가 연결을 끊지 않도록 **호환용으로 끼워 보내는** 것이다(RFC 8446의 middlebox compatibility mode).

이번엔 **시간**을 재보자. `curl`의 `-w` 옵션으로 "TCP 연결 완료 시각(`time_connect`)"과 "TLS 핸드셰이크 완료 시각(`time_appconnect`)"을 뽑으면, 그 차이가 곧 **핸드셰이크에 걸린 시간**이다.

**이 명령이 하는 일**: 같은 서버에 TLS 1.2 고정 / TLS 1.3으로 각각 접속해 단계별 소요 시간을 출력한다.

```bash
# TLS 1.2로 상한 고정
curl -so /dev/null --tlsv1.2 --tls-max 1.2 \
  -w 'TLS1.2  connect=%{time_connect}s  tls_done=%{time_appconnect}s\n' https://example.com
# TLS 1.3
curl -so /dev/null --tlsv1.3 \
  -w 'TLS1.3  connect=%{time_connect}s  tls_done=%{time_appconnect}s\n' https://example.com
```

서버까지 RTT가 약 50ms인 환경을 가정한 예시 결과:

```text
TLS1.2  connect=0.051s  tls_done=0.154s
TLS1.3  connect=0.050s  tls_done=0.103s
```

**아래 표가 보여주는 것**: TCP 연결 이후 TLS에 쓴 시간이 RTT 몇 번 분량인지.

| 버전 | TCP 완료 | TLS 완료 | TLS에 쓴 시간 | RTT(50ms) 환산 |
|---|---|---|---|---|
| TLS 1.2 | 0.051s | 0.154s | 0.103s | 약 2 RTT |
| TLS 1.3 | 0.050s | 0.103s | 0.053s | 약 1 RTT |

TLS에 쓴 시간이 **103ms → 53ms, 약 절반**이다. 남는 몇 ms는 암호 연산·서버 처리 시간이다. RTT가 50ms인 가까운 서버에서도 50ms를 아끼는데, 해외 서버처럼 RTT가 200ms라면 **연결 하나당 200ms**가 줄어든다. 모바일에서 새 연결을 자주 맺는 앱이라면 체감 차이가 크다.

마지막으로 인증서 사슬을 직접 보고 싶다면:

**이 명령이 하는 일**: 서버가 보내준 인증서 사슬과 검증 결과를 출력한다.

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null
```

출력의 `Certificate chain` 부분에서 `0 s:`(서버 인증서) → `1 s:`(중간 CA)가 나오고, 끝부분의 `Verify return code: 0 (ok)`이면 사슬 검증 통과다. 앞서 말한 "중간 인증서 누락" 문제는 여기서 사슬이 `0`번 하나만 보이고 검증 코드가 `20 (unable to get local issuer certificate)`로 나오는 식으로 드러난다.

## 실무에서 알아둘 점

- **TLS는 TCP 위에 얹힌다.** HTTPS 첫 요청까지의 대기는 "TCP 핸드셰이크 1 RTT + TLS 핸드셰이크 1~2 RTT"다. 그래서 **연결 재사용(Keep-Alive, 커넥션 풀)**이 중요하다. 핸드셰이크 비용을 연결 하나에 한 번만 내기 때문이다. (HTTP/3의 QUIC는 이 둘을 하나로 합쳐 왕복을 더 줄였다.)
- **인증서 만료는 장애의 단골 원인이다.** 만료일 모니터링과 자동 갱신(예: ACME 프로토콜)을 걸어두자.
- **서버에서 TLS 1.0·1.1은 끄자.** 두 버전은 RFC 8996(2021)으로 공식 폐기(deprecated)됐다. 현재는 TLS 1.2 이상, 가능하면 1.3을 권장한다.
- **0-RTT는 기본으로 켜지 말고, 멱등한 요청에만** 허용하자.

## 한 줄 교훈

TLS 핸드셰이크는 **"상대가 진짜인지(인증서) 확인하면서, 열쇠를 보내지 않고 같은 열쇠를 계산해내는(ECDHE)"** 절차다. TLS 1.3은 "어차피 쓸 키 재료를 첫 인사에 미리 실어 보내자"는 발상 하나로 왕복을 절반으로 줄였다. HTTPS가 느리다고 느껴질 땐, 핸드셰이크를 **몇 번이나, 몇 왕복으로** 하고 있는지부터 세어보자.
