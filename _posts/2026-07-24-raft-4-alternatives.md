---
layout: post
title: '분산 합의 (4/4) — 다른 해법과 비교: Kafka ISR, Paxos, MongoDB, Redis'
date: 2026-07-24 12:00:00 +0900
tags: [distributed-systems, consensus, raft, kafka, study]
series: 분산 합의
series_part: 4
series_total: 4
---

1~3편은 Raft 자체였다. 마지막 편은 **같은 문제(리더 선출 + 유실 없는 복제)를 다르게 푸는 방식**과 나란히 놓고 본다. ([1편](/2026/07/24/raft-1-basics.html)부터 보는 것을 권한다.)

## Kafka ISR — 같은 문제, 다른 해법

<!-- raft-diagram:9 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 200" role="img">
            <g text-anchor="middle" font-size="10">
              <rect x="30" y="40" width="150" height="70" rx="10" fill="none" stroke="#1f7a4d" stroke-dasharray="4 3"/>
              <text x="105" y="34" fill="#1f7a4d" font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">ISR (따라잡은 전원)</text>
              <circle cx="70" cy="75" r="22" fill="none" stroke="#5F0080" stroke-width="2.5"/>
              <text x="70" y="72" fill="#5F0080" font-weight="700" font-size="9">Leader</text>
              <text x="70" y="85" fill="#5f5a68" font-size="8">쓰기</text>
              <circle cx="140" cy="75" r="20" fill="none" stroke="#7d8590" stroke-width="2"/>
              <text x="140" y="72" fill="#1a1720" font-size="8.5">F1</text>
              <text x="140" y="84" fill="#5f5a68" font-size="7.5">동기</text>
              <circle cx="230" cy="75" r="20" fill="none" stroke="#b23a00" stroke-width="2" stroke-dasharray="3 2"/>
              <text x="230" y="72" fill="#b23a00" font-size="8.5">F2</text>
              <text x="230" y="84" fill="#b23a00" font-size="7.5">뒤처짐</text>
              <path d="M92 75 L120 75" stroke="#5f5a68" stroke-width="1.5" marker-end="url(#k)"/>
              <path d="M92 68 Q160 30 210 62" stroke="#b23a00" stroke-width="1.2" fill="none" stroke-dasharray="3 2" marker-end="url(#kw)"/>
              <text x="150" y="140" fill="#1a1720" font-size="9" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif">acks=all → ISR 전원 복제돼야 커밋(성공 응답)</text>
              <text x="150" y="158" fill="#5f5a68" font-size="8.5" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif">소비자는 High Watermark(ISR 전원 보유 지점)까지만 읽음</text>
              <text x="150" y="180" fill="#1f7a4d" font-size="8.5" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif">새 리더는 ISR 안에서만 선출 → 커밋분 유실 없음</text>
              <defs><marker id="k" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#5f5a68"/></marker><marker id="kw" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#b23a00"/></marker></defs>
            </g>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">Kafka ISR — 같은 문제, 다른 해법</figcaption>
</figure>

Kafka는 파티션마다 **리더 replica + 팔로워**가 있고, 쓰기는 리더가 받아 팔로워가 복제한다.

- **ISR(In-Sync Replicas)** = 리더를 따라잡은 replica 집합(동적으로 늘고 줆).
- `acks=all`이면 리더는 **ISR 전원**이 받은 뒤에야 커밋한다. 소비자는 **High Watermark**(ISR 전원이 가진 지점)까지만 읽는다.
- 유실을 막는 방식이 Raft와 다르다. Raft는 **과반 + 선거 제한**, Kafka는 **새 리더를 ISR 안에서만** 뽑는다(그래서 커밋된 데이터는 새 리더에 반드시 있다). ISR 밖에서 뽑도록 켜면(`unclean.leader.election`) 유실이 생겨 — 기본은 꺼짐.
- 그리고 "누가 리더냐 / ISR 멤버십"을 정하는 **조정 계층은 별도 합의**다(과거 ZooKeeper=ZAB, 현재 KRaft=Raft).

## Paxos vs Raft — 왜 Raft가 나왔나

<!-- raft-diagram:10 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 210" role="img">
            <g font-size="9" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif">
              <text x="8" y="14" fill="#5f5a68" font-weight="700" font-size="10">Paxos — 2단계, 대칭적</text>
              <rect x="14" y="24" width="66" height="26" rx="6" fill="none" stroke="#7d8590"/><text x="47" y="41" text-anchor="middle" fill="#1a1720" font-size="8.5">Proposer</text>
              <rect x="210" y="24" width="76" height="26" rx="6" fill="none" stroke="#7d8590"/><text x="248" y="41" text-anchor="middle" fill="#1a1720" font-size="8.5">Acceptors</text>
              <g font-size="7.5" fill="#5f5a68" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
                <text x="145" y="30" text-anchor="middle">① Prepare(n) →</text>
                <text x="145" y="42" text-anchor="middle">← Promise(+기존값)</text>
                <text x="145" y="56" text-anchor="middle">② Accept(n,v) →</text>
                <text x="145" y="68" text-anchor="middle">← Accepted (과반=확정)</text>
              </g>
              <line x1="30" y1="86" x2="270" y2="86" stroke="#e4e0ec"/>
              <text x="8" y="106" fill="#5F0080" font-weight="700" font-size="10">Raft — 강한 리더, 1방향</text>
              <rect x="14" y="116" width="66" height="26" rx="6" fill="none" stroke="#5F0080" stroke-width="2"/><text x="47" y="133" text-anchor="middle" fill="#5F0080" font-size="8.5" font-weight="700">Leader</text>
              <rect x="210" y="116" width="76" height="26" rx="6" fill="none" stroke="#7d8590"/><text x="248" y="133" text-anchor="middle" fill="#1a1720" font-size="8.5">Followers</text>
              <g font-size="7.5" fill="#5f5a68" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
                <text x="145" y="124" text-anchor="middle">AppendEntries →</text>
                <text x="145" y="136" text-anchor="middle">← ack (과반=커밋)</text>
              </g>
              <text x="150" y="164" text-anchor="middle" fill="#1a1720" font-size="8.5">로그로 쓰려면 Paxos는 슬롯마다 반복(Multi-Paxos)</text>
              <text x="150" y="180" text-anchor="middle" fill="#5f5a68" font-size="8">+ 성능 위해 리더를 두면 → 사실상 Raft와 닮아감</text>
              <text x="150" y="200" text-anchor="middle" fill="#1f7a4d" font-size="8.5" font-weight="700">안전성은 동급 · 차이는 '명료함'</text>
            </g>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">Paxos vs Raft — 왜 Raft가 나왔나</figcaption>
</figure>

**Paxos**(Lamport, 1998)는 합의 이론의 원조다. 한 값에 대한 합의를 **2단계**로 한다 — 제안번호를 선점하는 Prepare/Promise, 과반이 받으면 확정하는 Accept/Accepted. 로그로 쓰려면 이걸 자리마다 반복하는 **Multi-Paxos**가 되고, 성능을 위해 리더를 두면 Raft와 닮아간다.

그런데 Paxos는 **이해하고 구현하기 어렵기로 악명**이 높았다. 실전 세부(리더·로그·멤버십)가 명세에 덜 정해져 구현마다 제각각이었다. Raft는 **이해가능성**을 1순위로 다시 설계했다 — 강한 리더(로그는 리더→팔로워 한 방향), 로그에 구멍 불허, 선출·복제·안전성을 빠짐없이 규정.

중요한 점: **안전성 보장은 둘이 동급**이다(둘 다 과반 기반). Raft가 더 안전한 게 아니라, 유연성을 조금 포기하고 **더 명료하게** 만든 것이다. (실제로 Google Chubby·Spanner는 Paxos, etcd·Consul은 Raft를 쓴다.)

## MongoDB · Redis — 합의를 "어디에" 쓰나

<!-- raft-diagram:11 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 190" role="img">
            <g font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif">
              <text x="150" y="16" text-anchor="middle" fill="#1a1720" font-size="10" font-weight="700">쓰기 경로에 합의를 얼마나 쓰나</text>
              <line x1="24" y1="40" x2="276" y2="40" stroke="#e4e0ec" stroke-width="2"/>
              <g font-size="8.5" text-anchor="middle">
                <circle cx="40" cy="40" r="4" fill="#5F0080"/><text x="40" y="30" fill="#1a1720" font-weight="700">etcd</text><text x="40" y="58" fill="#5f5a68" font-size="7.5">데이터도 과반 합의</text>
                <circle cx="115" cy="40" r="4" fill="#5F0080"/><text x="115" y="30" fill="#1a1720" font-weight="700">Kafka</text><text x="115" y="58" fill="#5f5a68" font-size="7.5">조정=합의·데이터=ISR</text>
                <circle cx="195" cy="40" r="4" fill="#5F0080"/><text x="195" y="30" fill="#1a1720" font-weight="700">MongoDB</text><text x="195" y="58" fill="#5f5a68" font-size="7.5">선거=Raft·데이터=async</text>
                <circle cx="262" cy="40" r="4" fill="#b23a00"/><text x="262" y="30" fill="#b23a00" font-weight="700">Redis</text><text x="262" y="58" fill="#5f5a68" font-size="7.5">페일오버 조정만</text>
              </g>
              <text x="24" y="76" fill="#5f5a68" font-size="7.5">강함(전 경로)</text>
              <text x="276" y="76" text-anchor="end" fill="#5f5a68" font-size="7.5">약함(조정만/없음)</text>
              <line x1="30" y1="92" x2="270" y2="92" stroke="#e4e0ec"/>
              <text x="150" y="112" text-anchor="middle" fill="#1a1720" font-size="9">MongoDB: 선거는 Raft 기반, 데이터는 oplog를</text>
              <text x="150" y="125" text-anchor="middle" fill="#1a1720" font-size="9">비동기로 따라감 · 내구성은 <tspan font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace" font-size="8">w:majority</tspan> 옵션</text>
              <text x="150" y="148" text-anchor="middle" fill="#1a1720" font-size="9">Redis: 비동기 복제 + Sentinel/Cluster가</text>
              <text x="150" y="161" text-anchor="middle" fill="#1a1720" font-size="9">쿼럼으로 페일오버만 조정 · 쓰기 합의는 없음</text>
              <text x="150" y="182" text-anchor="middle" fill="#b23a00" font-size="8">→ Redis 기본은 페일오버 시 ack된 쓰기 유실 가능</text>
            </g>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">MongoDB · Redis — 합의를 어디에 쓰나</figcaption>
</figure>

- **MongoDB**: replica set은 primary 1 + secondary들. 쓰기는 primary가 받아 **oplog**(연산 로그)에 적고, secondary가 **비동기로 따라 적용**한다. **새 primary 선거는 Raft 기반.** 내구성은 write concern으로 조절 — `w:majority`면 과반 복제 후 확정돼 장애에도 살아남고, `w:1`이면 primary만 받고 끝이라 롤백(유실)될 수 있다.
- **Redis**: 기본이 **비동기 복제**(빠르지만 유실 위험). **Sentinel**(또는 Cluster)이 별도로 장애를 감지해 **쿼럼 합의로 페일오버만** 조정한다. 즉 **쓰기 자체엔 합의가 없다** — 기본 설정은 장애 시 응답까지 받은 쓰기도 사라질 수 있다. 강한 일관성은 옵션/실험적(RedisRaft).

정리하면, 같은 문제라도 시스템은 **"합의를 어디까지 쓸지"** 를 다르게 고른다.

| 시스템 | 데이터 복제 | 조정·선거 | 쓰기 내구성 |
|---|---|---|---|
| etcd·Consul | 합의(과반) | Raft | 과반 커밋 |
| Kafka | ISR(전원) | 합의(KRaft/ZAB) | acks=all + min.insync |
| MongoDB | async oplog | Raft 기반 선거 | w:majority(옵션) |
| Redis | async | 쿼럼 페일오버 | 기본 없음(유실 가능) |

핵심 축은 하나다. **합의를 "쓰기 커밋"에 요구하나 / "조정·페일오버"에만 쓰나 / 안 쓰나.** 시스템은 일관성과 속도의 트레이드오프에서 이 위치를 고른다.

---

**시리즈 마무리**: 합의는 하나의 정답이 아니다. Raft(과반)·Kafka ISR(전원)·Paxos·리더리스는 같은 문제의 다른 답이며, "조정"과 "데이터 복제"를 어디까지 합의로 묶느냐가 시스템의 성격을 가른다.

**한 가지 정정**: "클러스터는 다 Raft"라고 뭉뚱그리면 틀린다. Redis는 기본적으로 합의를 안 쓰고, Kafka의 데이터 복제(ISR)도 합의가 아니다. 이건 나도 처음에 헷갈렸다가 하나씩 확인하며 바로잡은 지점이다.
