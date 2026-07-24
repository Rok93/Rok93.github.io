---
layout: post
title: '분산 합의 (3/4) — Raft 실전: 운영 조각과 어디에 쓰나'
date: 2026-07-24 11:00:00 +0900
tags: [distributed-systems, consensus, raft, study]
series: 분산 합의
series_part: 3
series_total: 4
---

[1편](/2026/07/24/raft-1-basics.html)·[2편](/2026/07/24/raft-2-safety.html)에서 Raft의 핵심 세 덩어리(선출·복제·안전성)를 봤다. 이번엔 실제로 굴리는 데 필요한 조각과, "이게 대체 어디에 쓰이나".

## 실전에 필요한 세 조각

<!-- raft-diagram:8 -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:440px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 300 180" role="img">
            <g font-size="10" font-family="-apple-system,BlinkMacSystemFont,'Apple SD Gothic Neo','Noto Sans KR',sans-serif" text-anchor="middle">
              <rect x="24" y="24" width="252" height="38" rx="8" fill="#efe6f6" stroke="#e4e0ec"/>
              <text x="150" y="40" fill="#1a1720" font-weight="700" font-size="10.5">멤버십 변경</text>
              <text x="150" y="54" fill="#5f5a68" font-size="9">한 번에 한 대씩(또는 2단계)로 리더 둘 방지</text>
              <rect x="24" y="70" width="252" height="38" rx="8" fill="#efe6f6" stroke="#e4e0ec"/>
              <text x="150" y="86" fill="#1a1720" font-weight="700" font-size="10.5">스냅샷(로그 압축)</text>
              <text x="150" y="100" fill="#5f5a68" font-size="9">상태를 떠서 앞 로그 버림 · 뒤처진 팔로워엔 통째 전송</text>
              <rect x="24" y="116" width="252" height="38" rx="8" fill="#efe6f6" stroke="#e4e0ec"/>
              <text x="150" y="132" fill="#1a1720" font-weight="700" font-size="10.5">클라이언트 · 읽기</text>
              <text x="150" y="146" fill="#5f5a68" font-size="9">요청에 일련번호(중복 방지) · 읽기 전 리더 재확인</text>
            </g>
          </svg>
<figcaption style="margin-top:10px;font-size:13px;color:#5f5a68;text-align:center;">실전에 필요한 나머지 조각</figcaption>
</figure>

- **멤버십 변경**: 노드를 추가·제거할 때 설정이 서버마다 순간적으로 달라지면 리더가 둘 생길 수 있다(양쪽이 각자 과반이라 착각). 그래서 **한 번에 한 대씩** 바꾸거나, 2단계(joint consensus)를 거친다.
- **스냅샷(로그 압축)**: 로그를 영원히 쌓을 순 없으니, 상태를 통째로 떠서(스냅샷) 앞부분 로그를 버린다. 너무 뒤처진 팔로워에겐 스냅샷을 통째로 보낸다.
- **클라이언트와 읽기**: 재시도 때문에 같은 명령이 두 번 적용되지 않게 요청마다 **일련번호**를 붙인다. 읽기 전용 요청도 "내가 아직 진짜 리더인가"를 확인해야 **낡은 값(stale read)** 을 안 준다.

## 그래서 어디에 쓰나 — 그리고 안 쓰는 곳

"클러스터·레플리카가 있으면 다 이런 합의를 쓴다"는 생각은 **절반만 맞다.** 풀어야 할 **문제**(리더 뽑기 + 일관된 복제)는 거의 모든 분산 저장소에 공통이지만, **해법**은 갈린다.

| 분류 | 시스템 | 비고 |
|---|---|---|
| Raft(또는 변형) | etcd, Consul, CockroachDB·TiKV, Kafka(KRaft 모드), MongoDB(리더 선거), RabbitMQ 쿼럼 큐, Vault·Nomad | Kafka·Mongo는 "Raft 기반/변형" |
| 다른 합의 | ZooKeeper(ZAB), Google Spanner·Chubby(Paxos) | |
| 합의 안 씀(쓰기 경로) | Redis(비동기 복제+Sentinel), Cassandra(리더 없는 쿼럼) | Redis는 기본적으로 Raft 아님 |

여기서 중요한 구분 하나. 많은 시스템이 **"누가 리더냐" 같은 조정**엔 합의를 쓰면서도, **데이터 복제 자체**는 더 가벼운 방식을 쓴다 — Kafka는 ISR, MongoDB는 oplog, Redis는 비동기 복제. (다음 편에서 이걸 대비한다.)

---

**3편 요약**: Raft를 실무에 굴리려면 멤버십 변경·스냅샷·읽기 안전이 필요하다. 그리고 "클러스터=Raft"가 아니라, 시스템마다 **조정 계층**과 **데이터 복제 계층**의 해법이 다르다.

**다음 편(마지막)**: Kafka·Paxos·MongoDB·Redis는 같은 문제를 어떻게 다르게 푸는지 — "합의를 어디에 쓰나" 스펙트럼으로 정리.
