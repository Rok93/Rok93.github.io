---
layout: post
title: '분산 합의 (2/4) — Raft 안전성: 커밋의 함정과 선거 제한'
date: 2026-07-24 10:00:00 +0900
tags: [distributed-systems, consensus, raft, study]
series: 분산 합의
series_part: 2
series_total: 4
---

[1편](/2026/07/24/raft-1-basics.html)에서 리더가 로그를 복제하는 데까지 왔다. 그런데 이렇게 쌓은 로그가 **절대 뒤집히지 않는다**는 보장은 어디서 올까? 이번 편이 Raft에서 가장 헷갈리는 지점이다.

## 커밋의 함정 — 과반에 있다고 안심하면 안 된다

<!-- raft-diagram:5 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 190" role="img">
            <g font-size="9.5" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" text-anchor="middle">
              <text x="150" y="16" fill="#b23a00" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="10.5" font-weight="700">이전 term 엔트리를 과반만 보고 믿으면?</text>
              <text x="70" y="42" fill="#5f5a68" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif">과반 복제됨</text>
              <g fill="#fbe9df" stroke="#b23a00"><rect x="44" y="50" width="52" height="22" rx="4"/></g>
              <text x="70" y="65" fill="#1a1720">t2:X</text>
              <text x="70" y="92" fill="#5f5a68" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="8.5">"커밋됐네" (착각)</text>
              <path d="M150 60 L188 60" stroke="#5f5a68" stroke-width="1.5" marker-end="url(#f)"/>
              <text x="235" y="42" fill="#5f5a68" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif">새 리더가 덮음</text>
              <g fill="#efe6f6" stroke="#5F0080"><rect x="204" y="50" width="52" height="22" rx="4"/></g>
              <text x="230" y="65" fill="#1a1720">t3:Y</text>
              <text x="230" y="92" fill="#b23a00" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="8.5">X 사라짐!</text>
              <line x1="30" y1="120" x2="270" y2="120" stroke="#e4e0ec"/>
              <text x="150" y="142" fill="#1f7a4d" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="10" font-weight="700">그래서: 현재 term 엔트리 커밋 시에만 확정</text>
              <text x="150" y="162" fill="#5f5a68" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="9">→ 이전 엔트리는 그때 앞자리로 함께 확정</text>
              <defs><marker id="f" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#5f5a68"/></marker></defs>
            </g>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">커밋의 함정 — 과반 복제 ≠ 무조건 커밋</figcaption>
</figure>

<!-- raft-diagram:6 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 340" role="img">
            <g font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" font-size="9">
              <!-- 상태 A -->
              <text x="8" y="16" fill="#b23a00" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="10.5" font-weight="700">상태 A — term2가 과반, 그래도 위험</text>
              <g text-anchor="middle" font-size="8.5">
                <text x="60" y="30" fill="#5f5a68">i1</text><text x="92" y="30" fill="#5f5a68">i2</text><text x="124" y="30" fill="#5f5a68">i3</text>
              </g>
              <g font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="8.5" fill="#1a1720" text-anchor="end">
                <text x="40" y="46">S1</text><text x="40" y="68">S2</text><text x="40" y="90">S3</text><text x="40" y="112">S4</text><text x="40" y="134">S5</text>
              </g>
              <!-- i1 all term1 gray -->
              <g fill="#7d8590" opacity="0.35" stroke="#7d8590">
                <rect x="46" y="36" width="28" height="16" rx="2"/><rect x="46" y="58" width="28" height="16" rx="2"/><rect x="46" y="80" width="28" height="16" rx="2"/><rect x="46" y="102" width="28" height="16" rx="2"/><rect x="46" y="124" width="28" height="16" rx="2"/>
              </g>
              <!-- i2: S1,S2,S3 term2 accent; S5 term3 warn -->
              <g stroke="#5F0080" fill="#5F0080" fill-opacity="0.28"><rect x="78" y="36" width="28" height="16" rx="2"/><rect x="78" y="58" width="28" height="16" rx="2"/><rect x="78" y="80" width="28" height="16" rx="2"/></g>
              <g stroke="#b23a00" fill="#b23a00" fill-opacity="0.28"><rect x="78" y="124" width="28" height="16" rx="2"/></g>
              <g text-anchor="middle" font-size="8" fill="#1a1720"><text x="92" y="47">t2</text><text x="92" y="69">t2</text><text x="92" y="91">t2</text><text x="92" y="135">t3</text></g>
              <text x="150" y="60" fill="#5F0080" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="8.5">t2@i2 = 3/5 과반</text>
              <text x="150" y="128" fill="#b23a00" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="8.5">S5는 t3 보유</text>
              <text x="12" y="156" fill="#b23a00" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="8.5">→ S5가 나중에 이기면 i2를 t3로 덮음 → t2 소멸 ✗</text>
              <line x1="12" y1="168" x2="288" y2="168" stroke="#e4e0ec"/>
              <!-- 상태 B -->
              <text x="8" y="188" fill="#1f7a4d" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="10.5" font-weight="700">상태 B — 현재 term(4) 엔트리를 과반 커밋</text>
              <g font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="8.5" fill="#1a1720" text-anchor="end">
                <text x="40" y="214">S1</text><text x="40" y="236">S2</text><text x="40" y="258">S3</text><text x="40" y="280">S4</text><text x="40" y="302">S5</text>
              </g>
              <g fill="#7d8590" opacity="0.35" stroke="#7d8590"><rect x="46" y="204" width="28" height="16" rx="2"/><rect x="46" y="226" width="28" height="16" rx="2"/><rect x="46" y="248" width="28" height="16" rx="2"/><rect x="46" y="270" width="28" height="16" rx="2"/><rect x="46" y="292" width="28" height="16" rx="2"/></g>
              <g stroke="#5F0080" fill="#5F0080" fill-opacity="0.28"><rect x="78" y="204" width="28" height="16" rx="2"/><rect x="78" y="226" width="28" height="16" rx="2"/><rect x="78" y="248" width="28" height="16" rx="2"/></g>
              <g stroke="#b23a00" fill="#b23a00" fill-opacity="0.28"><rect x="78" y="292" width="28" height="16" rx="2"/></g>
              <!-- i3 t4 green on S1,S2,S3 -->
              <g stroke="#1f7a4d" fill="#1f7a4d" fill-opacity="0.30"><rect x="110" y="204" width="28" height="16" rx="2"/><rect x="110" y="226" width="28" height="16" rx="2"/><rect x="110" y="248" width="28" height="16" rx="2"/></g>
              <g text-anchor="middle" font-size="8" fill="#1a1720"><text x="92" y="215">t2</text><text x="92" y="237">t2</text><text x="92" y="259">t2</text><text x="124" y="215">t4</text><text x="124" y="237">t4</text><text x="124" y="259">t4</text><text x="92" y="303">t3</text></g>
              <text x="150" y="230" fill="#1f7a4d" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="8">t4@i3 과반 커밋</text>
              <text x="12" y="324" fill="#1f7a4d" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="8.5">→ 미래 리더는 t4 필수 → S5 선출 불가 → i2도 안전 ✓</text>
            </g>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">Figure 8 — 과반인데도 사라질 수 있다</figcaption>
</figure>

엔트리(명령 한 줄)가 **과반 서버에 복제**되면 "커밋"(확정)되고, 커밋된 명령은 절대 안 바뀐다. 여기까진 직관대로다. **그런데 함정이 있다.**

로그로 그려보자. 서버 5대(S1~S5), 순번 2번 자리를 보자.

1. **과반인데도 위험**: 순번 2에 term 2짜리 엔트리가 S1·S2·S3, 즉 **5대 중 3대(과반)** 에 있다. 직관은 "커밋됐다"고 말한다.
2. **왜 위험한가**: 그런데 S5는 순번 2에 **더 높은 term(3)** 엔트리를 갖고 있다. Raft의 선거 규칙은 "로그가 더 최신인 후보"를 뽑는데, term이 높은 S5가 더 최신으로 인정돼 나중에 리더가 될 수 있다. 그러면 S5가 순번 2를 자기 것(term 3)으로 **덮어써** — 과반에 있던 term 2 엔트리가 **사라진다.**
3. **언제 안전해지나**: 리더가 **자기 현재 term(4)** 의 새 엔트리를 순번 3에 만들어 과반에 커밋하면, 이제 미래의 어떤 리더도 그 term 4 엔트리를 **반드시** 가져야 당선된다(없으면 덜 최신). term 4가 없는 S5는 더는 리더가 못 되고, 그 아래 순번 2의 term 2도 **덩달아 안전**해진다.

그래서 규칙은 이렇다. **리더는 "자기 현재 term" 엔트리가 과반이 됐을 때만 커밋을 확정한다.** 이전 term 엔트리는 그 위의 현재-term 엔트리가 커밋될 때 딸려서 함께 확정된다.

## 안전성의 뿌리 — 최신 로그를 가진 자만 리더가 된다

<!-- raft-diagram:7 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 180" role="img">
            <g font-size="9.5" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" text-anchor="middle">
              <text x="70" y="30" fill="#1a1720" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="10.5" font-weight="700">후보 A (로그 짧음)</text>
              <g fill="#efe6f6" stroke="#5F0080"><rect x="30" y="42" width="34" height="20" rx="3"/><rect x="68" y="42" width="34" height="20" rx="3"/></g>
              <text x="150" y="55" fill="#b23a00" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="16">✗</text>
              <text x="150" y="72" fill="#b23a00" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="9">투표 거부</text>
              <text x="230" y="30" fill="#1a1720" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="10.5" font-weight="700">후보 B (최신)</text>
              <g fill="#efe6f6" stroke="#5F0080"><rect x="188" y="42" width="30" height="20" rx="3"/><rect x="220" y="42" width="30" height="20" rx="3"/><rect x="252" y="42" width="30" height="20" rx="3"/></g>
              <text x="150" y="118" fill="#1f7a4d" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="15">✓</text>
              <text x="150" y="136" fill="#1f7a4d" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="10" font-weight="700">B가 리더 — 커밋된 명령은 반드시 보존</text>
              <text x="150" y="158" fill="#5f5a68" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" font-size="9">투표자는 자기보다 덜 최신인 후보를 거부한다</text>
            </g>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">안전성 — 최신 로그를 가진 자만 리더가 된다</figcaption>
</figure>

가장 무서운 사고는 "이미 커밋된 명령이 사라지는 것"이다. 이걸 막는 핵심 장치가 **선거 제한**이다.

투표자는 후보의 마지막 로그를 보고, **자기 것보다 덜 최신이면 표를 안 준다.** (최신 판정: 마지막 엔트리의 term이 크면 최신, 같으면 순번이 큰 쪽.)

그 결과 **커밋된 엔트리를 모두 가진 서버만 리더가 될 수 있다.** 그래서 한 번 커밋된 명령은 이후 어떤 리더에게도 반드시 남아 있다 — 유실도 역전도 원천 차단된다.

---

**2편 요약**: 커밋 규칙의 핵심은 "**현재 term 엔트리가 과반이면 커밋**", 이전 term 엔트리를 과반만으로 믿지 않는 것. 그리고 "최신 로그 가진 자만 리더"라는 선거 제한 하나가 커밋된 명령의 영속성을 떠받친다.

**다음 편**: 이론은 됐고, 이걸 실제로 굴리는 데 필요한 조각들(노드 추가·제거, 로그 압축, 읽기)과 "그래서 이게 어디에 쓰이나".
