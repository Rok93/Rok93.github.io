---
layout: post
title: '처리율 제한(Rate Limiting) — 토큰 버킷부터 슬라이딩 윈도우까지'
date: 2026-10-28 09:00:00 +0900
image: /assets/img/rate-limiting/hero.jpg
generated: true
tags: [system-design, study]
sanitized: true
---

공개 API를 열어 두면 언젠가 이런 일이 생긴다. 누군가의 스크립트가 버그로 무한 루프를 돌며 초당 수천 번 같은 API를 부른다. 서버는 그 한 명의 요청을 처리하느라 바빠지고, 나머지 사용자 전부가 느려진다. 이걸 막는 장치가 **처리율 제한(rate limiting — 정해진 시간 동안 받아 줄 요청 수에 상한을 두는 것)**이다. 이 글은 처리율 제한이 왜 필요한지, 대표 알고리즘 다섯 가지(고정 윈도우·슬라이딩 윈도우 로그·슬라이딩 윈도우 카운터·토큰 버킷·리키 버킷)가 어떻게 다른지, 그리고 서버가 여러 대일 때 무엇을 조심해야 하는지 차례로 본다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·트래픽·설정은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다. (단, HTTP 429 상태 코드와 <code>Retry-After</code> 헤더, Redis Lua 스크립트의 원자적 실행은 실제 스펙 그대로이며, 시뮬레이션 출력은 글 속 코드를 직접 실행한 결과입니다.)</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/rate-limiting/hero.jpg" alt="보라색 톤의 개찰구 앞에 사람들이 줄 서 있고, 옆의 투명한 통에 동전 모양 토큰이 하나씩 떨어져 쌓이는 그림" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">토큰은 일정한 속도로 통에 쌓이고, 요청은 토큰을 하나씩 가져가야 개찰구를 통과한다. 이게 토큰 버킷이다.</figcaption>
</figure>

## 처리율 제한이란, 그리고 왜 필요한가

**무엇인가.** "사용자 한 명은 1분에 100번까지만 호출할 수 있다"처럼 **누가(키) · 얼마 동안(윈도우) · 몇 번까지(한도)**를 정해 두고, 한도를 넘는 요청은 처리하지 않고 거절하는 장치다. 여기서 "누가"를 구분하는 값을 **키(key)**라 한다. 사용자 ID, API 키, IP 주소 등이 키가 된다.

**왜 필요한가.**

- **자원 보호**: 한 사용자의 폭주가 서버·DB를 독점해 다른 사용자까지 느려지는 일을 막는다.
- **공정성**: 모두에게 같은 몫을 나눠 준다.
- **비용 통제**: 외부 유료 API(문자 발송, 결제 조회 등)를 부르는 서비스라면 호출 수가 곧 돈이다.
- **악용 방어**: 비밀번호 무차별 대입(brute force — 가능한 값을 전부 넣어 보는 공격)처럼 "많이 시도해야 성공하는" 공격의 속도를 떨어뜨린다.

**거절은 어떻게 알리나.** HTTP에는 이 용도의 상태 코드가 따로 있다. **`429 Too Many Requests`**(RFC 6585)다. 여기에 **`Retry-After`** 헤더(RFC 9110)로 "몇 초 뒤에 다시 오라"를 알려 주면, 잘 만든 클라이언트는 그만큼 기다렸다가 재시도한다. 남은 한도를 알려 주는 `X-RateLimit-Limit`·`X-RateLimit-Remaining`·`X-RateLimit-Reset` 같은 헤더도 널리 쓰인다. 다만 이 셋은 표준이 아니라 업계 관행이고, 표준 이름(`RateLimit` 계열)은 IETF에서 아직 초안 단계로 논의 중이다.

이 응답이 하는 일: 한도를 넘은 요청에 "지금은 안 되니 30초 뒤에 다시 오라"고 알린다.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
Content-Type: application/json

{"error": "rate_limited", "message": "1분에 100회까지 호출할 수 있습니다."}
```

## 고정 윈도우 카운터 — 가장 단순한 방법

**무엇인가.** 시간을 1분, 10초 같은 **고정 칸(window)**으로 자르고, 칸마다 카운터를 하나 둔다. 요청이 오면 지금 칸의 카운터를 1 올리고, 한도를 넘으면 거절한다. 다음 칸이 시작되면 카운터는 0부터 다시 센다.

**왜 쓰나.** 키마다 숫자 하나만 저장하면 된다. 메모리가 적고 구현이 쉽다. Redis의 `INCR`(1 증가) + `EXPIRE`(만료 시간 설정) 두 명령이면 끝난다.

**약점 — 경계 몰림.** 칸의 **경계 직전과 직후**에 요청을 몰아 보내면, 칸은 두 개로 나뉘어 각각 한도 안에 들지만 실제로는 짧은 시간에 한도의 **최대 2배**가 통과한다. "10초에 10건" 규칙인데 9.5초에 10건, 10.2초에 10건을 보내면 0.7초 만에 20건이 들어온다.

<!-- diagram: fixed-window-boundary -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 320 170" role="img" aria-label="시간축이 0초, 10초, 20초로 나뉜 고정 윈도우 두 칸. 첫 칸 끝인 9.5초에 요청 10건, 둘째 칸 시작인 10.2초에 요청 10건이 몰려 있고, 두 칸 모두 한도 10건 이내라 통과하지만 경계 부근 0.7초 동안 20건이 들어온다.">
<text x="160" y="14" font-size="9" fill="#1a1720" font-weight="700" text-anchor="middle">규칙: 10초 칸마다 최대 10건</text>
<rect x="20" y="30" width="140" height="70" rx="4" fill="#fff" stroke="#b9a9cc"/>
<rect x="160" y="30" width="140" height="70" rx="4" fill="#fff" stroke="#b9a9cc"/>
<text x="90" y="44" font-size="8" fill="#5f5a68" text-anchor="middle">칸 A (0~10초)</text>
<text x="230" y="44" font-size="8" fill="#5f5a68" text-anchor="middle">칸 B (10~20초)</text>
<g fill="#7d5a9e">
<rect x="147" y="55" width="8" height="40" rx="2"/>
<rect x="163" y="55" width="8" height="40" rx="2"/>
</g>
<text x="151" y="52" font-size="7" fill="#7d5a9e" text-anchor="end">10건</text>
<text x="167" y="52" font-size="7" fill="#7d5a9e">10건</text>
<line x1="20" y1="108" x2="300" y2="108" stroke="#1a1720"/>
<g font-size="7" fill="#5f5a68" text-anchor="middle">
<line x1="20" y1="105" x2="20" y2="111" stroke="#1a1720"/><text x="20" y="120">0초</text>
<line x1="160" y1="105" x2="160" y2="111" stroke="#1a1720"/><text x="160" y="120">10초</text>
<line x1="300" y1="105" x2="300" y2="111" stroke="#1a1720"/><text x="300" y="120">20초</text>
</g>
<text x="90" y="80" font-size="8" fill="#1f7a4d" text-anchor="middle">카운터 10 ✓</text>
<text x="230" y="80" font-size="8" fill="#1f7a4d" text-anchor="middle">카운터 10 ✓</text>
<path d="M143 128 L175 128" stroke="#b23a00" stroke-width="1.5"/>
<path d="M143 124 L143 132 M175 124 L175 132" stroke="#b23a00" stroke-width="1.5"/>
<text x="160" y="146" font-size="8" fill="#b23a00" font-weight="700" text-anchor="middle">0.7초 동안 20건 통과 (한도의 2배)</text>
<text x="160" y="160" font-size="7" fill="#5f5a68" text-anchor="middle">칸 단위로 보면 둘 다 규칙을 지켰다</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">고정 윈도우는 칸 안의 개수만 센다. 경계를 사이에 두고 몰린 요청은 서로 다른 칸으로 세어져 둘 다 통과한다.</figcaption>
</figure>

## 슬라이딩 윈도우 — 경계를 없애는 두 가지 방법

경계 몰림을 없애려면 "고정된 칸"이 아니라 **"지금 시각부터 거슬러 올라간 10초"**를 봐야 한다. 이렇게 창이 시간을 따라 미끄러지듯 움직인다고 해서 **슬라이딩 윈도우(sliding window)**라 부른다. 구현은 두 갈래다.

### 슬라이딩 윈도우 로그 — 정확하지만 무겁다

**무엇인가.** 요청이 올 때마다 **도착 시각을 전부 기록(log)**한다. 새 요청이 오면 "지금−10초"보다 오래된 기록을 지우고, 남은 기록 수가 한도 미만이면 통과시킨다.

**장단점.** 어느 순간을 잘라 봐도 한도를 넘지 않는다. 가장 정확하다. 대신 키마다 **요청 수만큼 타임스탬프를 저장**해야 한다. 한도가 1분에 1만 건이면 키 하나에 기록 1만 개다. Redis라면 정렬 집합(Sorted Set — 점수 순으로 정렬된 집합, 여기선 점수=시각)으로 구현하는 게 보통이다.

### 슬라이딩 윈도우 카운터 — 근사로 가볍게

**무엇인가.** 고정 윈도우의 카운터 두 개(**직전 칸**, **현재 칸**)만 들고, 직전 칸의 개수를 **지금 창과 겹치는 비율만큼만** 반영해 추정한다.

`추정 개수 = 직전 칸 개수 × (1 − 현재 칸 진행률) + 현재 칸 개수`

예를 들어 10.2초면 현재 칸(10~20초)은 2% 진행됐다. "지금부터 거슬러 10초"는 직전 칸의 98%와 겹친다. 직전 칸에 10건이 있었으니 `10 × 0.98 + 0 = 9.8`로 추정한다. 한도 10 미만이라 1건은 통과하고, 그다음은 `9.8 + 1 = 10.8`이라 거절된다.

**장단점.** 저장할 건 숫자 두 개뿐이다. 직전 칸 요청이 **고르게 퍼져 있었다고 가정**한 근사라 오차는 있다. 하지만 경계 몰림 같은 2배 폭주는 막는다.

## 토큰 버킷 — 순간 몰림을 일정량 허용한다

**무엇인가.** 양동이(bucket)에 **토큰**이 담겨 있다고 생각하자.

- 토큰은 **일정한 속도(rate, 예: 초당 1개)**로 채워진다.
- 양동이에는 **용량(capacity, 예: 10개)**이 있어, 꽉 차면 더 채워지지 않는다.
- 요청 하나가 들어오려면 토큰 1개를 꺼내야 한다. 토큰이 없으면 거절한다.

**왜 많이 쓰나.** 두 숫자의 뜻이 명확하다. **채움 속도는 장기 평균 처리율**, **용량은 한 번에 몰려도 받아 줄 수 있는 양(버스트, burst)**이다. 사람의 사용 패턴은 원래 들쭉날쭉하다. 한동안 조용하다가 화면을 열면 API 몇 개가 한꺼번에 나간다. 토큰 버킷은 쉬는 동안 쌓인 토큰으로 이런 짧은 몰림을 받아 주면서도, 길게 보면 평균 속도를 넘지 못하게 한다. AWS API Gateway 같은 많은 서비스가 이 방식으로 제한을 설명한다.

**구현 팁.** 진짜로 1초마다 토큰을 넣는 타이머를 돌릴 필요는 없다. 요청이 올 때 "마지막 계산 이후 흐른 시간 × 채움 속도"만큼 한꺼번에 더해 주면 된다(게으른 채움, lazy refill). 그래서 키마다 **토큰 수와 마지막 계산 시각**, 숫자 두 개만 저장한다.

## 리키 버킷 — 출구를 일정하게 만든다

**무엇인가.** 바닥에 구멍 난 양동이(leaky bucket)를 떠올리자. 요청은 위로 들어와 양동이(**큐**)에 쌓이고, 바닥 구멍으로 **일정한 속도로만** 빠져나가 처리된다. 양동이가 넘치면 새 요청은 버린다.

**토큰 버킷과의 차이.** 토큰 버킷은 토큰이 있으면 몰린 요청을 **그 즉시 한꺼번에** 통과시킨다. 리키 버킷(큐 방식)은 몰린 요청을 **줄 세워 일정 간격으로 흘려보낸다**. 그래서 뒤쪽 시스템이 받는 부하가 매끈하다. 대신 줄 선 요청은 그만큼 늦게 처리된다. Nginx의 `limit_req`가 공식 문서에서 이 "leaky bucket" 방식이라고 밝히고 있다(옵션 `burst`로 큐 크기, `nodelay`로 줄 세우지 않고 즉시 처리 여부를 정한다).

한 가지 주의할 점이 있다. "리키 버킷"이라는 이름은 문헌에 따라 두 가지로 쓰인다. 위처럼 **큐로 출구를 고르게 만드는 방식**을 가리키기도 하고, 단순히 **넘침 여부만 재는 측정기(meter)**를 가리키기도 한다. 후자는 수학적으로 토큰 버킷과 같은 동작이다. 글이나 라이브러리에서 이 이름을 보면 어느 뜻인지 확인하는 게 좋다.

<!-- diagram: token-vs-leaky-bucket -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 200" role="img" aria-label="왼쪽 토큰 버킷: 토큰이 초당 1개씩 들어와 용량 10개까지 쌓이고, 몰려온 요청은 토큰을 하나씩 가져가 즉시 한꺼번에 통과한다. 오른쪽 리키 버킷: 몰려온 요청이 큐에 쌓이고 바닥 구멍으로 일정 간격으로 하나씩 빠져나간다.">
<text x="82" y="14" font-size="9" fill="#5F0080" font-weight="700" text-anchor="middle">토큰 버킷</text>
<text x="257" y="14" font-size="9" fill="#5F0080" font-weight="700" text-anchor="middle">리키 버킷 (큐 방식)</text>
<line x1="170" y1="6" x2="170" y2="194" stroke="#e4e0ec"/>
<text x="82" y="30" font-size="7" fill="#5f5a68" text-anchor="middle">토큰 +1개/초 ↓</text>
<path d="M52 38 L58 100 L106 100 L112 38" fill="#fff" stroke="#7d5a9e" stroke-width="1.5"/>
<g fill="#5F0080">
<circle cx="66" cy="92" r="4"/><circle cx="76" cy="92" r="4"/><circle cx="86" cy="92" r="4"/><circle cx="96" cy="92" r="4"/>
<circle cx="71" cy="83" r="4"/><circle cx="81" cy="83" r="4"/><circle cx="91" cy="83" r="4"/>
</g>
<text x="82" y="62" font-size="7" fill="#5f5a68" text-anchor="middle">용량 10</text>
<g font-size="7" fill="#1a1720">
<rect x="10" y="118" width="54" height="16" rx="3" fill="#fff" stroke="#b9a9cc"/><text x="37" y="129" text-anchor="middle">요청 ×5 몰림</text>
</g>
<path d="M64 126 L100 126" stroke="#7d5a9e" marker-end="url(#a1)"/>
<text x="82" y="122" font-size="6" fill="#7d5a9e" text-anchor="middle">토큰 5개 꺼냄</text>
<g fill="#1f7a4d">
<rect x="104" y="118" width="10" height="16" rx="2"/><rect x="117" y="118" width="10" height="16" rx="2"/><rect x="130" y="118" width="10" height="16" rx="2"/><rect x="143" y="118" width="10" height="16" rx="2"/><rect x="156" y="118" width="10" height="16" rx="2"/>
</g>
<text x="128" y="148" font-size="7" fill="#1f7a4d" text-anchor="middle">즉시 한꺼번에 통과</text>
<text x="82" y="172" font-size="7" fill="#1a1720" text-anchor="middle">장점: 몰림을 바로 받아 줌</text>
<text x="82" y="184" font-size="7" fill="#b23a00" text-anchor="middle">단점: 뒤쪽에 순간 부하 전달</text>
<g font-size="7" fill="#1a1720">
<rect x="184" y="24" width="54" height="16" rx="3" fill="#fff" stroke="#b9a9cc"/><text x="211" y="35" text-anchor="middle">요청 ×5 몰림</text>
</g>
<path d="M238 32 L250 46" stroke="#7d5a9e" marker-end="url(#a1)"/>
<path d="M232 50 L240 112 L274 112 L282 50" fill="#fff" stroke="#7d5a9e" stroke-width="1.5"/>
<g fill="#7d5a9e">
<rect x="245" y="98" width="24" height="9" rx="2"/><rect x="245" y="87" width="24" height="9" rx="2"/><rect x="245" y="76" width="24" height="9" rx="2"/><rect x="245" y="65" width="24" height="9" rx="2"/>
</g>
<text x="300" y="82" font-size="7" fill="#5f5a68">큐</text>
<path d="M257 112 L257 126" stroke="#7d5a9e" stroke-width="1.5" marker-end="url(#a1)"/>
<text x="268" y="122" font-size="6" fill="#7d5a9e">일정 속도로 새어 나감</text>
<g fill="#1f7a4d">
<rect x="230" y="134" width="10" height="12" rx="2"/><rect x="252" y="134" width="10" height="12" rx="2"/><rect x="274" y="134" width="10" height="12" rx="2"/>
</g>
<text x="257" y="158" font-size="7" fill="#1f7a4d" text-anchor="middle">1건씩 같은 간격으로 처리</text>
<text x="257" y="172" font-size="7" fill="#1a1720" text-anchor="middle">장점: 뒤쪽 부하가 매끈함</text>
<text x="257" y="184" font-size="7" fill="#b23a00" text-anchor="middle">단점: 줄 선 요청은 늦어짐</text>
<defs><marker id="a1" viewBox="0 0 6 6" refX="5" refY="3" markerWidth="5" markerHeight="5" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#7d5a9e"/></marker></defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">같은 "5건 몰림"을 토큰 버킷은 쌓아 둔 토큰으로 즉시 통과시키고, 리키 버킷은 큐에 줄 세워 일정한 간격으로 흘려보낸다.</figcaption>
</figure>

## 상세 예시 — 같은 몰림, 세 알고리즘의 반응

앞에서 말로 한 걸 직접 돌려 보자. 규칙은 **"10초에 10건(평균 초당 1건)"**이다. 클라이언트가 **9.5초에 10건, 10.2초에 10건**을 몰아 보낸다. 토큰 버킷은 용량 10, 초당 1개 채움, 처음엔 꽉 찬 상태로 둔다.

이 코드가 하는 일: 고정 윈도우·슬라이딩 윈도우 카운터·토큰 버킷을 각각 구현하고, 같은 요청 20건을 흘려 몇 건이 통과하는지 센다.

```python
import math

LIMIT, WINDOW = 10, 10.0   # 10초에 10건 (평균 초당 1건)

class FixedWindow:
    def __init__(self): self.win, self.count = None, 0
    def allow(self, t):
        w = math.floor(t / WINDOW)
        if w != self.win: self.win, self.count = w, 0   # 새 칸이면 0부터
        if self.count < LIMIT: self.count += 1; return True
        return False

class SlidingWindowCounter:
    def __init__(self): self.counts = {}
    def allow(self, t):
        w = math.floor(t / WINDOW)
        elapsed = (t - w * WINDOW) / WINDOW            # 현재 칸 진행률 0~1
        est = self.counts.get(w - 1, 0) * (1 - elapsed) + self.counts.get(w, 0)
        if est < LIMIT: self.counts[w] = self.counts.get(w, 0) + 1; return True
        return False

class TokenBucket:
    def __init__(self, capacity=10, rate=1.0):
        self.cap, self.rate, self.tokens, self.last = capacity, rate, capacity, 0.0
    def allow(self, t):
        # 게으른 채움: 흐른 시간만큼 한꺼번에 더하되 용량은 넘지 않게
        self.tokens = min(self.cap, self.tokens + (t - self.last) * self.rate)
        self.last = t
        if self.tokens >= 1: self.tokens -= 1; return True
        return False

reqs = [9.5] * 10 + [10.2] * 10   # 9.5초에 10건, 10.2초에 10건
for name, rl in [("고정 윈도우", FixedWindow()),
                 ("슬라이딩 윈도우 카운터", SlidingWindowCounter()),
                 ("토큰 버킷", TokenBucket())]:
    res = [rl.allow(t) for t in reqs]
    print(f"{name:<14} 9.5초 통과 {sum(res[:10]):>2}건 | "
          f"10.2초 통과 {sum(res[10:]):>2}건 | 0.7초 동안 합계 {sum(res):>2}건")
```

실행 결과:

```text
고정 윈도우         9.5초 통과 10건 | 10.2초 통과 10건 | 0.7초 동안 합계 20건
슬라이딩 윈도우 카운터   9.5초 통과 10건 | 10.2초 통과  1건 | 0.7초 동안 합계 11건
토큰 버킷          9.5초 통과 10건 | 10.2초 통과  0건 | 0.7초 동안 합계 10건
```

**읽는 법.**

- **고정 윈도우**는 9.5초 몫(칸 A)과 10.2초 몫(칸 B)을 따로 센다. 0.7초 만에 **20건**, 한도의 2배가 통과했다. 앞 도식 그대로다.
- **슬라이딩 윈도우 카운터**는 10.2초에 직전 칸 10건을 98%만큼(9.8건) 반영한다. 1건만 더 받고 막았다. 합계 **11건**. 정확히 10은 아니지만 2배 폭주는 막았다. 이 1건의 차이가 "근사"의 대가다.
- **토큰 버킷**은 9.5초에 쌓여 있던 토큰 10개를 다 썼다. 10.2초까지 0.7초 동안 채워진 토큰은 0.7개라 1건도 못 받는다. 합계 **10건**. 대신 이후엔 1초마다 1건씩 다시 통과한다. 예를 들어 15.2초에 다시 몰아 보내면, 그 사이 5초 동안 채워진 토큰 5개만큼 5건이 통과한다.

정리하면, 같은 "10초에 10건"이라도 **알고리즘에 따라 몰림을 얼마나 허용하는지가 다르다**. 경계 몰림이 문제라면 고정 윈도우는 피하고, 짧은 몰림은 받아 주되 평균은 지키고 싶다면 토큰 버킷이 잘 맞는다.

| 알고리즘 | 키당 저장 | 정확도 | 몰림(버스트) | 대표 쓰임 |
|---|---|---|---|---|
| 고정 윈도우 | 숫자 1개 | 경계에서 최대 2배 | 경계에서 과다 허용 | 단순한 일일 호출 한도 |
| 슬라이딩 윈도우 로그 | 요청 수만큼 시각 | 정확 | 허용 안 함 | 한도가 작고 정확해야 할 때 |
| 슬라이딩 윈도우 카운터 | 숫자 2개 | 근사(작은 오차) | 거의 허용 안 함 | 대규모 API 게이트웨이 |
| 토큰 버킷 | 숫자 2개 | 정확(평균 기준) | 용량만큼 허용 | 일반 API 한도 |
| 리키 버킷(큐) | 큐 | 정확(출구 기준) | 큐에 줄 세움 | 뒤쪽 부하 평탄화 |

## 서버가 여러 대일 때 — 카운터를 어디에 두나

**문제.** 서버가 한 대면 카운터를 메모리에 두면 끝이다. 하지만 로드밸런서 뒤에 서버가 4대 있고 각자 메모리에 카운터를 두면, 요청이 4대로 흩어지는 동안 각 서버는 4분의 1만 본다. "1분에 100건" 규칙이 실제로는 최대 400건이 된다. 그래서 보통 **Redis 같은 공유 저장소에 카운터를 모은다**.

**새 문제 — 경쟁 조건(race condition — 두 작업이 동시에 같은 값을 읽고 써서 결과가 꼬이는 것).** 토큰 버킷을 Redis로 구현한다고 하자. 서버가 "토큰 수 읽기 → 계산 → 쓰기"를 따로 하면, 두 서버가 **같은 토큰 1개를 동시에 읽고** 둘 다 "1개 남았네, 통과!"라고 판단할 수 있다.

<!-- diagram: distributed-race -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 190" role="img" aria-label="위쪽: 서버 A와 서버 B가 Redis에서 토큰 1을 동시에 읽고 각자 0을 써서 요청 두 건이 모두 통과하는 경쟁 조건. 아래쪽: Lua 스크립트로 읽기-계산-쓰기를 한 번에 실행하면 서버 A만 통과하고 서버 B는 거절된다.">
<text x="8" y="14" font-size="8" fill="#b23a00" font-weight="700">❌ 따로 읽고 쓰기 (GET → 계산 → SET)</text>
<rect x="8" y="22" width="60" height="18" rx="4" fill="#fff" stroke="#b9a9cc"/><text x="38" y="34" font-size="7" fill="#1a1720" text-anchor="middle">서버 A</text>
<rect x="8" y="58" width="60" height="18" rx="4" fill="#fff" stroke="#b9a9cc"/><text x="38" y="70" font-size="7" fill="#1a1720" text-anchor="middle">서버 B</text>
<rect x="135" y="36" width="60" height="26" rx="4" fill="#e9e1f1" stroke="#7d5a9e"/><text x="165" y="47" font-size="7" fill="#1a1720" text-anchor="middle">Redis</text><text x="165" y="57" font-size="7" fill="#5f5a68" text-anchor="middle">tokens = 1</text>
<g stroke="#7d5a9e" fill="none"><path d="M68 29 L135 44"/><path d="M68 67 L135 55"/></g>
<text x="100" y="30" font-size="6" fill="#5f5a68" text-anchor="middle">읽음: 1</text>
<text x="100" y="74" font-size="6" fill="#5f5a68" text-anchor="middle">읽음: 1</text>
<text x="210" y="30" font-size="7" fill="#b23a00">A: 1 ≥ 1 → 통과, 0 저장</text>
<text x="210" y="72" font-size="7" fill="#b23a00">B: 1 ≥ 1 → 통과, 0 저장</text>
<text x="165" y="88" font-size="7" fill="#b23a00" font-weight="700" text-anchor="middle">토큰 1개로 2건 통과</text>
<line x1="8" y1="98" x2="322" y2="98" stroke="#e4e0ec"/>
<text x="8" y="114" font-size="8" fill="#1f7a4d" font-weight="700">✅ Lua 스크립트 (한 덩어리로 원자 실행)</text>
<rect x="8" y="122" width="60" height="18" rx="4" fill="#fff" stroke="#b9a9cc"/><text x="38" y="134" font-size="7" fill="#1a1720" text-anchor="middle">서버 A</text>
<rect x="8" y="158" width="60" height="18" rx="4" fill="#fff" stroke="#b9a9cc"/><text x="38" y="170" font-size="7" fill="#1a1720" text-anchor="middle">서버 B</text>
<rect x="135" y="136" width="60" height="26" rx="4" fill="#e9e1f1" stroke="#1f7a4d"/><text x="165" y="147" font-size="7" fill="#1a1720" text-anchor="middle">Redis</text><text x="165" y="157" font-size="7" fill="#5f5a68" text-anchor="middle">한 번에 하나씩</text>
<g stroke="#1f7a4d" fill="none"><path d="M68 131 L135 144"/><path d="M68 167 L135 155"/></g>
<text x="100" y="128" font-size="6" fill="#5f5a68" text-anchor="middle">① 먼저</text>
<text x="100" y="175" font-size="6" fill="#5f5a68" text-anchor="middle">② 다음</text>
<text x="210" y="134" font-size="7" fill="#1f7a4d">A: 1 → 0, 통과</text>
<text x="210" y="172" font-size="7" fill="#b23a00">B: 0 → 거절 (429)</text>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">읽기와 쓰기 사이에 다른 서버가 끼어들면 같은 토큰을 두 번 쓴다. Lua 스크립트로 묶으면 Redis가 스크립트 하나를 끝까지 실행한 뒤에야 다음 명령을 처리한다.</figcaption>
</figure>

**해결 — Lua 스크립트로 묶기.** Redis는 **Lua 스크립트 하나를 실행하는 동안 다른 명령을 끼워 넣지 않는다**(원자적 실행, atomic — 중간 상태를 남에게 보이지 않고 통째로 실행). 그래서 "읽기 → 계산 → 쓰기"를 스크립트 하나에 넣으면 경쟁 조건이 사라진다.

이 코드가 하는 일: Redis 안에서 토큰 버킷 한 번의 판단(채우기 → 꺼내기 → 저장)을 원자적으로 수행하고, 통과면 1·거절이면 0을 돌려준다.

```lua
-- KEYS[1]: 버킷 키(예: rl:user:42)
-- ARGV[1]: 용량, ARGV[2]: 초당 채움 속도, ARGV[3]: 현재 시각(초)
local cap  = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now  = tonumber(ARGV[3])

local b = redis.call('HMGET', KEYS[1], 'tokens', 'last')
local tokens = tonumber(b[1]) or cap      -- 처음 보는 키면 꽉 찬 상태
local last   = tonumber(b[2]) or now

tokens = math.min(cap, tokens + (now - last) * rate)   -- 게으른 채움
local allowed = 0
if tokens >= 1 then
  tokens = tokens - 1
  allowed = 1
end

redis.call('HSET', KEYS[1], 'tokens', tokens, 'last', now)
redis.call('EXPIRE', KEYS[1], math.ceil(cap / rate) * 2)  -- 안 쓰는 키는 정리
return allowed
```

몇 가지 덧붙인다.

- **시각은 누가 주나.** 위 예시는 서버가 현재 시각을 넘긴다. 서버마다 시계가 조금씩 어긋나면 채움 계산이 흔들린다. Redis의 `TIME` 명령으로 Redis 쪽 시각을 쓰는 방법도 있다.
- **고정 윈도우라면 더 간단하다.** `INCR`은 그 자체로 원자적이라 카운터만 올리면 된다. 다만 `INCR`과 `EXPIRE`를 따로 보내다 그 사이에 서버가 죽으면 만료 없는 키가 남아 영원히 막힐 수 있다. 이것도 스크립트로 묶거나, 처음 생성될 때만 만료를 거는 식으로 처리한다.
- **Redis가 죽으면?** 처리율 제한기가 모든 요청 앞에 서 있으니, 제한기 장애가 곧 서비스 장애가 될 수 있다. 저장소에 닿지 못할 때 **전부 통과(fail-open)**시킬지 **전부 거절(fail-closed)**할지 미리 정해 둬야 한다. 일반 API는 서비스를 살리는 fail-open을, 로그인 시도 제한처럼 보안 목적이면 fail-closed를 고르는 경우가 많다.

## 어디에 두고, 무엇으로 셀까

마지막으로 실무에서 함께 정해야 할 것들이다.

- **위치**: 애플리케이션 코드 안, 앞단의 API 게이트웨이·리버스 프록시(Nginx 등), 또는 둘 다. 게이트웨이에 두면 애플리케이션까지 오기 전에 걸러져 자원을 더 아낀다. 사용자 등급별 한도처럼 업무 규칙이 섞이면 애플리케이션 쪽이 편하다.
- **키**: 로그인 사용자는 사용자 ID, 비로그인은 IP가 흔하다. IP는 회사·학교처럼 여러 명이 IP 하나를 공유하는 환경(NAT)에서 무고한 사용자를 함께 막을 수 있다.
- **여러 겹**: "초당 10건 + 하루 1만 건"처럼 짧은 한도와 긴 한도를 함께 거는 경우가 많다. 짧은 쪽은 순간 폭주를, 긴 쪽은 총량을 막는다.
- **클라이언트 쪽 예의**: 429를 받은 클라이언트는 `Retry-After`를 지키고, 없으면 **지수 백오프(exponential backoff — 재시도 간격을 1초, 2초, 4초…로 늘리는 것)**에 약간의 무작위 지연(jitter)을 섞어 재시도해야 한다. 모두가 같은 순간에 재시도하면 또 한 번 몰림이 생긴다.

## 한 줄 교훈

처리율 제한은 "몇 건까지"보다 **"몰림을 어떻게 다룰지"**를 고르는 일이다. 경계 몰림을 피하려면 슬라이딩 윈도우를, 짧은 몰림을 받아 주며 평균을 지키려면 토큰 버킷을, 뒤쪽을 고르게 보호하려면 리키 버킷을 고르고, 서버가 여러 대라면 카운터를 한곳에 모아 **원자적으로** 갱신하자.
