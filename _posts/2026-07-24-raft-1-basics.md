---
layout: post
title: '분산 합의 (1/4) — Raft 기초: 합의란, 리더 선출, 로그 복제'
date: 2026-07-24 09:00:00 +0900
tags: [distributed-systems, consensus, raft, study]
series: 분산 합의
series_part: 1
series_total: 4
---

쿠버네티스의 두뇌 etcd, 서비스 디스커버리 Consul… 이런 시스템은 **서버 한 대가 죽어도 멈추지 않는다.** 비결은 공통이다. 여러 서버가 "**같은 명령을 같은 순서로**" 실행하도록 맞추는 **합의(consensus)** 장치를 쓴다. 그 대표 주자가 **Raft**다.

이 시리즈는 Raft를 4편으로 뜯어본다. 1편은 기초 — 합의가 뭔지, 리더를 어떻게 뽑고, 로그를 어떻게 복제하는지.

<!-- raft-diagram:0 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 200" role="img">
            <g font-size="12" text-anchor="middle">
              <rect x="18" y="70" width="80" height="46" rx="9" fill="none" stroke="#5F0080" stroke-width="2"/>
              <text x="58" y="90" fill="#1a1720" font-weight="700">① 선출</text>
              <text x="58" y="106" fill="#5f5a68" font-size="10">리더 1명</text>
              <rect x="110" y="70" width="80" height="46" rx="9" fill="none" stroke="#5F0080" stroke-width="2"/>
              <text x="150" y="90" fill="#1a1720" font-weight="700">② 복제</text>
              <text x="150" y="106" fill="#5f5a68" font-size="10">로그 전파</text>
              <rect x="202" y="70" width="82" height="46" rx="9" fill="none" stroke="#5F0080" stroke-width="2"/>
              <text x="243" y="90" fill="#1a1720" font-weight="700">③ 안전성</text>
              <text x="243" y="106" fill="#5f5a68" font-size="10">안 뒤집힘</text>
              <path d="M98 93 L110 93" stroke="#5f5a68" stroke-width="2" marker-end="url(#a)"/>
              <path d="M190 93 L202 93" stroke="#5f5a68" stroke-width="2" marker-end="url(#a)"/>
              <text x="150" y="30" fill="#1a1720" font-weight="700" font-size="13">복제된 로그에 모두 동의</text>
              <text x="150" y="160" fill="#5f5a68" font-size="10.5">+ 실전 조각: 멤버십 변경 · 스냅샷 · 읽기</text>
            </g>
            <defs><marker id="a" markerWidth="7" markerHeight="7" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#5f5a68"/></marker></defs>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">전체 지도 — Raft를 세 덩어리로 나눠 정복</figcaption>
</figure>

## 합의 = 복제된 로그

<!-- raft-diagram:1 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 180" role="img">
            <g font-size="11" text-anchor="middle">
              <text x="150" y="20" fill="#5f5a68" font-size="10">각 서버의 로그(명령 순서)가 똑같다</text>
              <g>
                <text x="34" y="48" fill="#1a1720" font-size="10">S1</text>
                <text x="34" y="88" fill="#1a1720" font-size="10">S2</text>
                <text x="34" y="128" fill="#1a1720" font-size="10">S3</text>
              </g>
              <g font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" font-size="10">
                <g fill="#efe6f6" stroke="#5F0080">
                  <rect x="60" y="34" width="52" height="22" rx="4"/><rect x="118" y="34" width="52" height="22" rx="4"/><rect x="176" y="34" width="66" height="22" rx="4"/>
                  <rect x="60" y="74" width="52" height="22" rx="4"/><rect x="118" y="74" width="52" height="22" rx="4"/><rect x="176" y="74" width="66" height="22" rx="4"/>
                  <rect x="60" y="114" width="52" height="22" rx="4"/><rect x="118" y="114" width="52" height="22" rx="4"/><rect x="176" y="114" width="66" height="22" rx="4"/>
                </g>
                <g fill="#1a1720">
                  <text x="86" y="49">x=3</text><text x="144" y="49">x=5</text><text x="209" y="49">del y</text>
                  <text x="86" y="89">x=3</text><text x="144" y="89">x=5</text><text x="209" y="89">del y</text>
                  <text x="86" y="129">x=3</text><text x="144" y="129">x=5</text><text x="209" y="129">del y</text>
                </g>
              </g>
              <text x="150" y="164" fill="#5f5a68" font-size="10">→ 한 대가 죽어도 나머지가 같은 상태라 서비스 지속</text>
            </g>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">복제 상태 머신 — 같은 명령을 같은 순서로</figcaption>
</figure>

합의란 여러 서버가 "어떤 명령을 어떤 순서로 실행할지"에 동의하게 만드는 것이다. 이 명령의 나열을 **복제된 로그(replicated log)** 라고 부른다(예: `x=3` → `x=5` → `y 삭제`). 모든 서버가 같은 로그를 같은 순서로 재생하면 상태가 똑같아진다. 그래서 몇 대가 죽어도 남은 서버로 서비스가 이어진다.

## 역할 셋과 term(번호표)

<!-- raft-diagram:2 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 190" role="img">
            <g font-size="11" text-anchor="middle">
              <circle cx="60" cy="45" r="26" fill="none" stroke="#5F0080" stroke-width="2.5"/>
              <text x="60" y="42" fill="#5F0080" font-weight="700">Leader</text>
              <text x="60" y="56" fill="#5f5a68" font-size="9">쓰기 담당</text>
              <circle cx="150" cy="45" r="24" fill="none" stroke="#7d8590" stroke-width="2"/>
              <text x="150" y="42" fill="#7d8590" font-weight="700" font-size="10">Follower</text>
              <text x="150" y="56" fill="#5f5a68" font-size="9">따름</text>
              <circle cx="240" cy="45" r="24" fill="none" stroke="#c8860a" stroke-width="2"/>
              <text x="240" y="42" fill="#c8860a" font-weight="700" font-size="10">Candidate</text>
              <text x="240" y="56" fill="#5f5a68" font-size="9">선거중</text>
              <text x="150" y="105" fill="#5f5a68" font-size="10">term = 계속 커지는 번호표 (임기)</text>
              <g font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" font-size="10">
                <rect x="40" y="118" width="60" height="24" rx="4" fill="#efe6f6" stroke="#e4e0ec"/><text x="70" y="134" fill="#1a1720">term 1</text>
                <rect x="110" y="118" width="60" height="24" rx="4" fill="#efe6f6" stroke="#e4e0ec"/><text x="140" y="134" fill="#1a1720">term 2</text>
                <rect x="180" y="118" width="80" height="24" rx="4" fill="#efe6f6" stroke="#e4e0ec"/><text x="220" y="134" fill="#1a1720">term 3 …</text>
              </g>
              <text x="150" y="165" fill="#5f5a68" font-size="9.5">한 term에 리더는 최대 1명</text>
            </g>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">역할 셋과 term(번호표)</figcaption>
</figure>

서버는 늘 셋 중 하나다.

- **리더(Leader)** — 모든 쓰기를 받는 서버
- **팔로워(Follower)** — 리더를 따르는 서버
- **후보(Candidate)** — 리더가 없어 선거에 나선 상태

여기에 **term(임기)** 이라는 장치가 있다. 계속 1씩 커지는 **번호표**다. 물리적 시계 대신 이 번호로 "누구 정보가 더 최신인지"를 가린다. 낡은 번호표를 단 메시지는 무시한다. 규칙 하나: **한 term에 리더는 최대 한 명.**

## 리더 선출 — 과반의 표를 먼저 모은 자

<!-- raft-diagram:3 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 190" role="img">
            <g font-size="10" text-anchor="middle">
              <circle cx="150" cy="95" r="30" fill="none" stroke="#c8860a" stroke-width="2.5"/>
              <text x="150" y="92" fill="#c8860a" font-weight="700">후보</text>
              <text x="150" y="106" fill="#5f5a68" font-size="8.5">"나 뽑아줘"</text>
              <g fill="#7d8590">
                <circle cx="150" cy="28" r="16" fill="none" stroke="#7d8590"/><text x="150" y="31" fill="#1a1720">투표</text>
                <circle cx="60" cy="70" r="16" fill="none" stroke="#7d8590"/><text x="60" y="73" fill="#1a1720">투표</text>
                <circle cx="240" cy="70" r="16" fill="none" stroke="#7d8590"/><text x="240" y="73" fill="#1a1720">투표</text>
                <circle cx="90" cy="160" r="16" fill="none" stroke="#7d8590"/><text x="90" y="163" fill="#5f5a68">…</text>
              </g>
              <g stroke="#c8860a" stroke-width="1.5" marker-end="url(#v)">
                <path d="M150 65 L150 46"/><path d="M126 82 L74 74"/><path d="M174 82 L226 74"/>
              </g>
              <defs><marker id="v" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#c8860a"/></marker></defs>
              <text x="150" y="184" fill="#5f5a68" font-size="9">과반(5대 중 3표) 얻으면 리더 당선</text>
            </g>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">리더 선출 — 과반의 표를 먼저 모은 자</figcaption>
</figure>

리더는 주기적으로 **하트비트**(살아있다는 신호)를 보낸다. 팔로워가 일정 시간(**랜덤**, 보통 150~300ms) 동안 이 신호를 못 받으면 "리더가 죽었나?" 하고 후보로 변신한다 — 번호표를 올리고, 자기에게 투표하고, 나머지에게 표를 요청한다. **과반**(전체의 절반+1)을 얻으면 리더가 된다.

타임아웃을 **랜덤**으로 두는 이유가 있다. 다 같이 후보가 되면 표가 갈려 아무도 과반을 못 얻고 선거만 반복된다. 시간을 흩뿌려 한 명이 먼저 튀어나오게 하는 것이다.

## 로그 복제 — 리더 로그에 강제로 맞춘다

<!-- raft-diagram:4 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 190" role="img">
            <g font-size="10" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
              <text x="8" y="32" fill="#5F0080" font-weight="700" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif">리더</text>
              <g fill="#efe6f6" stroke="#5F0080"><rect x="60" y="20" width="40" height="20" rx="3"/><rect x="104" y="20" width="40" height="20" rx="3"/><rect x="148" y="20" width="40" height="20" rx="3"/><rect x="192" y="20" width="40" height="20" rx="3"/></g>
              <g fill="#1a1720" text-anchor="middle"><text x="80" y="34">A</text><text x="124" y="34">B</text><text x="168" y="34">C</text><text x="212" y="34">D</text></g>
              <text x="8" y="92" fill="#7d8590" font-weight="700" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif">팔로워</text>
              <g fill="#efe6f6" stroke="#5F0080"><rect x="60" y="80" width="40" height="20" rx="3"/><rect x="104" y="80" width="40" height="20" rx="3"/></g>
              <g fill="none" stroke="#b23a00"><rect x="148" y="80" width="40" height="20" rx="3" stroke-dasharray="3 2"/></g>
              <g fill="#1a1720" text-anchor="middle"><text x="80" y="94">A</text><text x="124" y="94">B</text><text x="168" y="94" fill="#b23a00" font-size="9">?</text></g>
              <text x="150" y="128" fill="#5f5a68" font-size="9.5" text-anchor="middle" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif">"C 앞(=B)이 나랑 같아?" → 같으면 C 붙이고, 다르면 거부</text>
              <text x="150" y="150" fill="#5f5a68" font-size="9.5" text-anchor="middle" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif">거부되면 리더가 한 칸씩 되짚어 팔로워를 맞춘다</text>
              <path d="M124 44 L124 76" stroke="#5f5a68" stroke-width="1.5" marker-end="url(#d2)"/>
              <path d="M168 44 L168 76" stroke="#b23a00" stroke-width="1.5" marker-end="url(#d3)"/>
              <defs><marker id="d2" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#5f5a68"/></marker><marker id="d3" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker></defs>
            </g>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">로그 복제 — 리더 로그에 강제로 맞춘다</figcaption>
</figure>

쓰기 명령은 **리더만** 받는다. 리더는 자기 로그에 붙인 뒤 팔로워에게 복제 신호(**AppendEntries**)를 보낸다. 이 신호엔 "**바로 앞 엔트리 정보**"가 붙어서, 팔로워는 "그 지점까지 내 로그가 리더와 같은가"를 검사한다. 다르면 **거부**한다. 거부당하면 리더는 한 칸씩 뒤로 되짚어 다시 보내며 **팔로워 로그를 자기와 똑같이 맞춘다.**

그 결과 이런 성질이 생긴다 — 어떤 위치의 (번호표, 순번)이 같으면 그 앞은 전부 동일하다.

---

**1편 요약**: 합의의 목표는 "로그(명령 순서)를 똑같이 맞추는 것". Raft는 리더 하나가 자기 로그를 진실로 삼아 복제하고, term이라는 번호표로 최신을 가린다.

**다음 편**: 이렇게 쌓은 로그가 절대 뒤집히지 않게 하는 "안전성" — 그리고 직관이 틀리는 커밋의 함정.
