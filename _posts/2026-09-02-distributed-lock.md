---
layout: post
title: '분산 락 — "여러 서버가 자원 하나를 두고 다툴 때"'
date: 2026-09-02 09:00:00 +0900
image: /assets/img/distributed-lock/hero.jpg
generated: true
tags: [distributed, study]
sanitized: true
---

한정판 운동화가 재고 1켤레 남았다. 같은 순간 손님 둘이 "구매" 버튼을 누른다. 요청은 서로 다른 서버로 흩어져 들어간다. 두 서버 모두 "재고 1개 있음"을 읽고, 둘 다 "판매 완료" 처리를 한다 — **재고 1개가 2개 팔렸다.** 서버가 한 대뿐이라면 프로그래밍 언어가 주는 락(lock — 한 번에 한 스레드만 코드 구간에 들어가게 막는 빗장)으로 간단히 막는다. 하지만 서버가 여러 대면, **각 서버의 락은 서로의 존재를 모른다.** **분산 락(distributed lock)**은 이 문제를 푼다 — 여러 대의 서버가 **공유 저장소(Redis·DB 등) 바깥에 하나의 빗장**을 두고, "지금 이 자원은 내가 쓰는 중"을 서로에게 알린다. 이 글은 왜 프로세스 안의 락으로는 부족한지, 분산 락을 Redis와 DB로 각각 어떻게 만드는지, 그리고 초보자가 반드시 밟는 함정(만료·소유권)이 무엇인지를 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·시나리오·코드는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/distributed-lock/hero.jpg" alt="여러 서버가 하나의 열쇠를 두고 경쟁하지만 오직 한 대만 열쇠를 쥐고 공유 금고에 접근하는 분산 락 은유" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">여러 서버가 자원 하나를 두고 다툴 때, 열쇠는 오직 한 대만 쥔다 — 이것이 분산 락이다.</figcaption>
</figure>

## 왜 프로세스 안의 락으로는 부족한가

**무엇**이 문제인지부터. 자바에는 `synchronized`나 `ReentrantLock` 같은 락이 있다. 이들은 **같은 JVM(자바 프로그램이 도는 하나의 실행 공간) 안의 스레드들**끼리만 순서를 정리한다. 한 대의 서버 안에서는 완벽하다.

문제는 요즘 서비스가 **서버 여러 대에 같은 애플리케이션을 복제해 띄운다**는 데 있다(수평 확장 — 트래픽을 여러 대가 나눠 받게 늘리는 것). 서버 A의 JVM과 서버 B의 JVM은 **메모리를 공유하지 않는다.** A가 건 `synchronized` 빗장은 A 안에서만 유효하고, B는 그런 빗장이 걸렸는지조차 모른다. 그래서 A와 B가 **동시에** 같은 재고를 읽고 같은 갱신을 한다.

정리하면, 프로세스 안의 락이 통하는 전제는 "**모두가 같은 메모리를 본다**"인데, 분산 환경에서는 이 전제가 깨진다. 그래서 락을 **모든 서버가 함께 바라보는 바깥 저장소**로 옮겨야 한다.

<!-- diagram: local-lock-fails -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 220" role="img" aria-label="같은 JVM 안에서는 로컬 락이 스레드를 정리하지만 서버가 두 대면 각자의 락이 서로를 몰라 동시에 자원에 접근해 충돌하는 것을 대비한 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="14" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">로컬 락은 한 서버 안에서만 통한다</text>
    <!-- server A -->
    <rect x="18" y="28" width="120" height="92" rx="8" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="78" y="44" text-anchor="middle" fill="#5F0080" font-size="9" font-weight="700">서버 A (JVM)</text>
    <text x="78" y="60" text-anchor="middle" fill="#1f7a4d" font-size="7.5">synchronized ✓</text>
    <rect x="34" y="68" width="88" height="18" rx="4" fill="#fff" stroke="#1f7a4d"/>
    <text x="78" y="80" text-anchor="middle" fill="#1a1720" font-size="7">스레드 순서 정리됨</text>
    <text x="78" y="104" text-anchor="middle" fill="#5f5a68" font-size="7">A의 빗장</text>
    <!-- server B -->
    <rect x="192" y="28" width="120" height="92" rx="8" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="252" y="44" text-anchor="middle" fill="#5F0080" font-size="9" font-weight="700">서버 B (JVM)</text>
    <text x="252" y="60" text-anchor="middle" fill="#1f7a4d" font-size="7.5">synchronized ✓</text>
    <rect x="208" y="68" width="88" height="18" rx="4" fill="#fff" stroke="#1f7a4d"/>
    <text x="252" y="80" text-anchor="middle" fill="#1a1720" font-size="7">스레드 순서 정리됨</text>
    <text x="252" y="104" text-anchor="middle" fill="#5f5a68" font-size="7">B의 빗장 (A를 모름)</text>
    <!-- shared resource -->
    <rect x="115" y="150" width="100" height="30" rx="6" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="165" y="163" text-anchor="middle" fill="#b23a00" font-size="8" font-weight="700">재고 = 1</text>
    <text x="165" y="174" text-anchor="middle" fill="#5f5a68" font-size="7">공유 DB</text>
    <line x1="78" y1="120" x2="140" y2="150" stroke="#b23a00" stroke-width="1.3"/>
    <line x1="252" y1="120" x2="190" y2="150" stroke="#b23a00" stroke-width="1.3"/>
    <text x="165" y="200" text-anchor="middle" fill="#b23a00" font-size="8" font-weight="700">둘 다 "1개 있음" 읽음 → 2개 판매</text>
    <text x="165" y="213" text-anchor="middle" fill="#5f5a68" font-size="7.5">서로의 빗장을 못 보기 때문</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">각 서버의 락은 자기 JVM 안에서만 유효하다. 서버가 여럿이면 바깥 빗장이 필요하다.</figcaption>
</figure>

## 분산 락이란 무엇인가

**분산 락은 여러 서버가 함께 접근하는 외부 저장소에 "이 자원을 지금 누가 쓰는지"를 표시하는 빗장**이다. 모든 서버가 자원에 손대기 전에 **같은 저장소에 "내가 잡았다"는 표식을 먼저 남기려 시도**하고, 성공한 딱 하나만 작업을 진행한다. 나머지는 기다리거나 포기한다.

빗장을 놓을 저장소로 흔히 두 가지를 쓴다.

- **Redis** — 메모리에 데이터를 두는 빠른 저장소. 락 획득/해제가 매우 빠르다. 대부분의 "빠른 상호배제"에 쓴다.
- **관계형 DB** — 이미 쓰고 있는 데이터베이스의 락 기능(비관적 락·네임드 락)을 그대로 활용한다. 별도 인프라가 필요 없다.

핵심 성질은 **상호배제(mutual exclusion — 한 번에 하나만 통과)**다. 여러 서버가 동시에 빗장을 놓으려 해도, 저장소가 **원자적으로(중간에 끼어들 틈 없이 한 번에)** "먼저 온 하나"만 성공시킨다. 이 "원자성"이 분산 락의 심장이다 — 이게 없으면 앞의 로컬 락과 똑같이 무너진다.

<!-- diagram: distributed-lock-basic -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 205" role="img" aria-label="세 서버가 공유 락 저장소에 동시에 락 획득을 시도하지만 하나만 성공하고 나머지는 대기하는 분산 락 흐름 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="14" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">셋이 시도, 하나만 획득</text>
    <!-- three servers -->
    <rect x="14" y="30" width="70" height="24" rx="5" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="49" y="45" text-anchor="middle" fill="#1f7a4d" font-size="8" font-weight="700">서버 A</text>
    <rect x="14" y="88" width="70" height="24" rx="5" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="49" y="103" text-anchor="middle" fill="#b23a00" font-size="8" font-weight="700">서버 B</text>
    <rect x="14" y="146" width="70" height="24" rx="5" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="49" y="161" text-anchor="middle" fill="#b23a00" font-size="8" font-weight="700">서버 C</text>
    <!-- lock store -->
    <rect x="205" y="76" width="108" height="52" rx="8" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="259" y="96" text-anchor="middle" fill="#5F0080" font-size="8.5" font-weight="700">공유 락 저장소</text>
    <text x="259" y="110" text-anchor="middle" fill="#5f5a68" font-size="7">Redis / DB</text>
    <text x="259" y="121" text-anchor="middle" fill="#1a1720" font-size="7">lock:shoe = A 🔒</text>
    <!-- arrows -->
    <line x1="84" y1="42" x2="205" y2="92" stroke="#1f7a4d" stroke-width="1.4"/>
    <text x="150" y="58" text-anchor="middle" fill="#1f7a4d" font-size="7" font-weight="700">획득 ✓</text>
    <line x1="84" y1="100" x2="205" y2="104" stroke="#b23a00" stroke-width="1.2" stroke-dasharray="3 3"/>
    <text x="150" y="98" text-anchor="middle" fill="#b23a00" font-size="7">실패 → 대기</text>
    <line x1="84" y1="158" x2="205" y2="116" stroke="#b23a00" stroke-width="1.2" stroke-dasharray="3 3"/>
    <text x="150" y="150" text-anchor="middle" fill="#b23a00" font-size="7">실패 → 대기</text>
    <text x="165" y="192" text-anchor="middle" fill="#5F0080" font-size="8">저장소가 "먼저 온 하나"만 원자적으로 통과시킨다</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">모든 서버가 같은 저장소에 빗장을 시도하고, 성공한 하나만 자원을 만진다.</figcaption>
</figure>

## Redis로 만드는 분산 락 — `SET key value NX EX`

가장 널리 쓰는 방식이다. Redis의 `SET` 명령에 옵션 두 개를 붙인다.

- **NX** = "**N**ot e**X**ists", 즉 **키가 없을 때만** 저장한다. 이미 누가 락을 잡아 키가 있으면 이 명령은 **실패**한다 — 이 한 방으로 상호배제가 된다.
- **EX** = 만료 시간(초). 락을 잡은 서버가 **죽어서 해제를 못 해도**, 시간이 지나면 락이 자동으로 풀린다(뒤에서 이게 왜 위험의 씨앗도 되는지 본다).

**이 코드가 하는 일:** 재고 자원에 대한 락을 Redis에 잡아 보고, 성공한 경우에만 재고 차감을 실행한 뒤 락을 푼다.

```java
String lockKey = "lock:shoe:1001";
String token   = UUID.randomUUID().toString();  // 내 락임을 증명할 고유값

// NX: 키가 없을 때만 성공, EX 10: 10초 뒤 자동 만료
boolean acquired = redis.set(lockKey, token, "NX", "EX", 10);

if (!acquired) {
    // 다른 서버가 이미 잡음 → 대기하거나 "잠시 후 다시" 응답
    throw new AlreadyLockedException();
}
try {
    decreaseStock(1001);   // 락을 쥔 딱 한 서버만 여기 들어온다
} finally {
    releaseLock(lockKey, token);  // 반드시 풀어준다 (아래에서 안전한 해제 설명)
}
```

`NX` 덕분에 **동시에 여러 서버가 `set`을 쳐도 성공하는 건 하나**뿐이다. Redis가 명령을 **한 줄씩 원자적으로** 처리하기 때문이다. 성공한 서버만 `decreaseStock`에 들어가고, 나머지는 예외를 받아 물러난다.

## 놓치기 쉬운 함정 — 만료(TTL)와 소유권

여기서 초보자가 거의 반드시 밟는 두 지뢰가 있다. 둘 다 **`EX`(자동 만료)** 때문에 생긴다.

**지뢰 1: 남의 락을 푼다.** 해제를 `redis.del(lockKey)`로 짜면, "키만 지운다"가 된다. 그런데 이런 순서를 상상해 보자.

1. 서버 A가 락을 잡는다(만료 10초).
2. A의 작업이 예상보다 오래 걸려 **10초를 넘긴다** → 락이 **자동 만료**돼 풀린다.
3. 서버 B가 그 틈에 **새로 락을 잡는다.**
4. 뒤늦게 A의 작업이 끝나 `del(lockKey)`을 실행 → **B가 잡은 락을 A가 지워버린다.**

이제 락이 없는데 B는 자기가 락을 쥐고 있다고 믿는다 → 다시 상호배제가 깨진다. 그래서 **해제할 때 "이 락이 정말 내 것인지" 먼저 확인**해야 한다. 앞 코드에서 `token`(내가 넣은 고유값)을 쓴 이유다. "값이 내 토큰과 같을 때만 지운다"를 **원자적으로** 해야 하므로, 보통 Redis의 Lua 스크립트(여러 명령을 한 덩어리로 실행)로 처리한다.

**이 코드가 하는 일:** 락에 저장된 값이 내 토큰과 같을 때만 그 키를 지운다. "확인"과 "삭제" 사이에 다른 요청이 끼어들 틈이 없다.

```lua
-- KEYS[1] = lockKey, ARGV[1] = 내 token
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])   -- 내 락일 때만 해제
else
    return 0                             -- 남의 락이면 건드리지 않음
end
```

**지뢰 2: 만료가 작업보다 짧으면 상호배제 자체가 깨진다.** 위 2~3단계처럼 A가 아직 일하는 중인데 락이 만료돼 B가 들어오면, **A와 B가 동시에 자원을 만진다.** 토큰 검사는 "남의 락을 지우는 것"은 막지만, **"두 서버가 겹쳐서 작업하는 것"** 자체는 막지 못한다. 근본 대책은 두 갈래다.

- **만료 시간을 작업 최대 시간보다 넉넉히** 잡거나, 작업이 길어지면 주기적으로 만료를 연장한다(watchdog — 살아 있는 동안 락 수명을 갱신하는 장치).
- **펜싱 토큰(fencing token):** 락을 줄 때마다 **1씩 증가하는 번호**를 함께 발급한다. 자원(DB 등)은 **더 큰 번호의 요청만 받아들이고**, 뒤늦게 도착한 옛 번호(A)의 요청은 거부한다. 이러면 A가 겹쳐 들어와도 자원 단에서 막힌다.

<!-- diagram: ttl-danger -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 210" role="img" aria-label="서버 A의 작업이 만료 시간을 넘겨 락이 풀리고 서버 B가 락을 잡은 뒤 A가 뒤늦게 남의 락을 지우는 위험 상황을 시간순으로 보여주는 도식">
  <g font-family="ui-sans-serif,system-ui,sans-serif">
    <text x="165" y="14" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="10">만료가 작업보다 짧으면?</text>
    <!-- timeline -->
    <line x1="30" y1="150" x2="300" y2="150" stroke="#5f5a68" stroke-width="1"/>
    <text x="300" y="164" text-anchor="end" fill="#5f5a68" font-size="7">시간 →</text>
    <!-- A lock span -->
    <rect x="40" y="52" width="120" height="16" rx="4" fill="#eaf5ef" stroke="#1f7a4d"/>
    <text x="100" y="64" text-anchor="middle" fill="#1f7a4d" font-size="7">A 락 (만료 10초)</text>
    <line x1="160" y1="60" x2="160" y2="150" stroke="#b23a00" stroke-width="1" stroke-dasharray="2 2"/>
    <text x="160" y="80" text-anchor="middle" fill="#b23a00" font-size="7">만료 자동 해제</text>
    <!-- A work span (longer) -->
    <rect x="40" y="92" width="180" height="16" rx="4" fill="#f7e6dc" stroke="#b23a00"/>
    <text x="130" y="104" text-anchor="middle" fill="#b23a00" font-size="7">A 실제 작업 (더 김)</text>
    <!-- B lock span -->
    <rect x="165" y="122" width="120" height="16" rx="4" fill="#f3ecf7" stroke="#7d5a9e"/>
    <text x="225" y="134" text-anchor="middle" fill="#5F0080" font-size="7">B 락 획득</text>
    <!-- danger zone -->
    <rect x="165" y="88" width="55" height="24" rx="3" fill="none" stroke="#b23a00" stroke-width="1.2" stroke-dasharray="3 2"/>
    <text x="192" y="200" text-anchor="middle" fill="#b23a00" font-size="8" font-weight="700">A·B 작업 겹침 = 상호배제 붕괴</text>
    <text x="192" y="180" text-anchor="middle" fill="#5f5a68" font-size="7">막는 법: 넉넉한 만료 · watchdog · 펜싱 토큰</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">자동 만료는 "죽은 서버가 락을 영영 쥐는 것"을 막지만, 만료가 너무 짧으면 오히려 겹침을 만든다.</figcaption>
</figure>

## DB로 만드는 분산 락 — 별도 인프라 없이

Redis가 없거나 이미 쓰는 DB로 충분한 경우, DB 자체의 락 기능을 빌린다. 두 가지가 흔하다.

- **비관적 락(pessimistic lock):** 조회할 때 `SELECT ... FOR UPDATE`를 쓰면, 그 행(row)에 **DB가 잠금을 걸어** 다른 트랜잭션은 그 행을 만질 수 없고 **대기**한다. 재고 행 하나를 `FOR UPDATE`로 잡으면, 그 행에 대해선 한 번에 한 트랜잭션만 통과한다.
- **네임드 락(named lock):** MySQL의 `GET_LOCK('이름', 시간)`처럼, 특정 **문자열 이름**에 대한 락을 DB에게 요청한다. 행이 아니라 "이름"에 거는 락이라, 자원을 자유롭게 정의할 수 있다.

**이 코드가 하는 일:** 재고 행을 `FOR UPDATE`로 잠근 채 읽어, 같은 행을 노리는 다른 트랜잭션이 끝날 때까지 기다리게 한 뒤 차감한다.

```sql
BEGIN;
-- 이 행을 잠근다. 다른 트랜잭션의 같은 행 FOR UPDATE는 여기서 대기
SELECT stock FROM product WHERE id = 1001 FOR UPDATE;

-- 잠근 동안 안전하게 확인 후 차감
UPDATE product SET stock = stock - 1 WHERE id = 1001 AND stock > 0;
COMMIT;   -- 커밋과 함께 락 해제
```

DB 락은 **트랜잭션이 끝나면(커밋/롤백) 자동으로 풀린다**는 큰 장점이 있다 — Redis처럼 "만료 vs 소유권" 지뢰를 직접 다룰 필요가 적다. 대신 **락을 DB에 거는 만큼 DB 부하가 커지고**, 대기가 길어지면 커넥션(DB 연결)이 묶여 병목이 될 수 있다. 트래픽이 아주 높으면 Redis가 유리하고, 정합성이 최우선이고 트래픽이 감당 가능하면 DB 락이 단순하고 안전하다.

## Redlock — Redis 여러 대로 안전성을 높이려는 시도

Redis 락에는 근본 약점이 하나 더 있다. **Redis 한 대가 죽으면** 그 락 정보가 사라질 수 있다는 점이다(복제 지연 중 장애 등). 그래서 Redis 창시자는 **Redlock**이라는 방식을 제안했다 — Redis를 **여러 대(보통 5대)** 두고, **과반수(3대 이상)**에서 락을 잡는 데 성공해야 "획득"으로 친다. 한두 대가 죽어도 과반수만 살아 있으면 락이 유지된다.

다만 Redlock은 **시계 오차·GC 멈춤** 같은 상황에서 완전하지 않다는 반론(분산 시스템 전문가 Martin Kleppmann의 비판)도 유명하다. 그래서 "돈이 걸린 강한 정합성"이 필요하면 **펜싱 토큰을 자원 단에서 함께 검증**하라는 게 실무의 결론이다. 락은 "대부분의 충돌을 줄이는 장치"이고, **최후의 정합성은 자원(DB의 유니크 제약·조건부 UPDATE 등)이 지킨다**고 보는 편이 안전하다.

## 한 줄 교훈

분산 락은 "여러 서버 중 하나만 통과"를 **바깥 저장소의 원자적 연산**으로 구현하는 기술이다. 하지만 진짜 어려움은 락을 **잡는 것**이 아니라 **안전하게 놓는 것** — "내 락만 푼다(소유권)"와 "작업 중 락이 만료되지 않게 한다(만료)"이다. 그래서 실무의 격언은 이렇다: **락은 충돌을 줄이는 1차 방어선일 뿐, 마지막 정합성은 자원 자신(유니크 제약·조건부 갱신·펜싱 토큰)이 지키게 하라.**
