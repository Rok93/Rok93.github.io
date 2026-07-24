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

Kafka는 파티션마다 **리더 replica + 팔로워**가 있고, 쓰기는 리더가 받아 팔로워가 복제한다.

- **ISR(In-Sync Replicas)** = 리더를 따라잡은 replica 집합(동적으로 늘고 줆).
- `acks=all`이면 리더는 **ISR 전원**이 받은 뒤에야 커밋한다. 소비자는 **High Watermark**(ISR 전원이 가진 지점)까지만 읽는다.
- 유실을 막는 방식이 Raft와 다르다. Raft는 **과반 + 선거 제한**, Kafka는 **새 리더를 ISR 안에서만** 뽑는다(그래서 커밋된 데이터는 새 리더에 반드시 있다). ISR 밖에서 뽑도록 켜면(`unclean.leader.election`) 유실이 생겨 — 기본은 꺼짐.
- 그리고 "누가 리더냐 / ISR 멤버십"을 정하는 **조정 계층은 별도 합의**다(과거 ZooKeeper=ZAB, 현재 KRaft=Raft).

## Paxos vs Raft — 왜 Raft가 나왔나

**Paxos**(Lamport, 1998)는 합의 이론의 원조다. 한 값에 대한 합의를 **2단계**로 한다 — 제안번호를 선점하는 Prepare/Promise, 과반이 받으면 확정하는 Accept/Accepted. 로그로 쓰려면 이걸 자리마다 반복하는 **Multi-Paxos**가 되고, 성능을 위해 리더를 두면 Raft와 닮아간다.

그런데 Paxos는 **이해하고 구현하기 어렵기로 악명**이 높았다. 실전 세부(리더·로그·멤버십)가 명세에 덜 정해져 구현마다 제각각이었다. Raft는 **이해가능성**을 1순위로 다시 설계했다 — 강한 리더(로그는 리더→팔로워 한 방향), 로그에 구멍 불허, 선출·복제·안전성을 빠짐없이 규정.

중요한 점: **안전성 보장은 둘이 동급**이다(둘 다 과반 기반). Raft가 더 안전한 게 아니라, 유연성을 조금 포기하고 **더 명료하게** 만든 것이다. (실제로 Google Chubby·Spanner는 Paxos, etcd·Consul은 Raft를 쓴다.)

## MongoDB · Redis — 합의를 "어디에" 쓰나

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
