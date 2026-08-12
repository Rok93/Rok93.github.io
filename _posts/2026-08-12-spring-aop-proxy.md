---
layout: post
title: 'AOP와 프록시 동작 원리 — @Transactional은 왜 같은 클래스 안에서 안 먹힐까?'
date: 2026-08-12 09:00:00 +0900
image: /assets/img/spring-aop-proxy/hero.jpg
generated: true
tags: [spring, study]
sanitized: true
---

메서드 하나에 `@Transactional`을 붙였는데, 로그를 찍어 보니 트랜잭션이 열리질 않는다. 분명 애너테이션은 그대로인데. 알고 보니 그 메서드를 **같은 클래스 안의 다른 메서드가 직접 호출**하고 있었다. 애너테이션은 바뀐 게 없는데, 부르는 위치 하나로 동작이 사라진 것이다. 이 황당한 현상의 뿌리에는 **AOP(관점 지향 프로그래밍, Aspect-Oriented Programming)**와 그걸 구현하는 **프록시(proxy, 대리인)**가 있다. 이 글은 Spring이 `@Transactional` 같은 공통 기능을 어떻게 몰래 끼워 넣는지, 그리고 그 방식 때문에 왜 "자기 호출(self-invocation)"이 함정이 되는지를 따라간다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 코드·시나리오·로그는 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다.</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/spring-aop-proxy/hero.jpg" alt="원본 상자를 그대로 감싼 반투명 대리인 상자가 원본 앞에 서서 검문과 기록을 대신 처리하는 모습" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">프록시는 원본과 똑같이 생긴 "대리인"이다. 바깥에서 오는 호출을 먼저 받아 부가 작업(트랜잭션·로그 등)을 처리한 뒤 원본에게 넘긴다.</figcaption>
</figure>

## AOP가 뭔가 — "흩어지는 공통 기능을 한 군데로"

먼저 **AOP(관점 지향 프로그래밍)**. 트랜잭션 처리, 로그 남기기, 권한 검사 같은 기능은 특정 업무 로직이 아니라 **여러 메서드에 공통으로 얹히는 부가 기능**이다. 이런 걸 **횡단 관심사(cross-cutting concern, 여러 모듈을 가로질러 반복되는 관심사)**라고 부른다.

이걸 그냥 짜면, 모든 서비스 메서드마다 "트랜잭션 시작 → 본 작업 → 커밋/롤백" 코드를 손으로 복붙하게 된다. 백 개 메서드면 백 번 반복이고, 규칙이 바뀌면 백 군데를 고쳐야 한다. AOP는 이 **반복되는 부가 기능을 한 곳(관점, aspect)에 모아 두고, "이런 메서드들에 적용해"라고 선언만 하면 자동으로 끼워 넣어** 준다.

AOP 용어 세 개만 짚자.

- **어드바이스(advice)**: 실제로 끼워 넣을 부가 코드. 예: "메서드 실행 전 트랜잭션을 열고, 끝나면 커밋한다."
- **포인트컷(pointcut)**: 어디에 끼워 넣을지 고르는 조건. 예: "`@Transactional`이 붙은 모든 메서드."
- **위빙(weaving)**: 어드바이스를 실제 대상 메서드에 엮어 넣는 과정.

<!-- diagram: cross-cutting -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 180" role="img" aria-label="여러 서비스 메서드에 트랜잭션·로그·권한 검사라는 공통 기능이 세로로 가로질러 반복되는 모습. AOP는 이 공통 기능을 하나의 관점으로 모은다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <text x="165" y="16" text-anchor="middle" fill="#5F0080" font-weight="700" font-size="9.5">횡단 관심사 — 여러 메서드를 가로지른다</text>
    <rect x="20" y="30" width="70" height="120" rx="6" fill="#fff" stroke="#7d5a9e" stroke-width="1.4"/>
    <rect x="130" y="30" width="70" height="120" rx="6" fill="#fff" stroke="#7d5a9e" stroke-width="1.4"/>
    <rect x="240" y="30" width="70" height="120" rx="6" fill="#fff" stroke="#7d5a9e" stroke-width="1.4"/>
    <text x="55" y="44" text-anchor="middle" fill="#5f5a68" font-size="7.5">주문 저장</text>
    <text x="165" y="44" text-anchor="middle" fill="#5f5a68" font-size="7.5">결제 처리</text>
    <text x="275" y="44" text-anchor="middle" fill="#5f5a68" font-size="7.5">배송 등록</text>
    <rect x="12" y="58" width="306" height="18" rx="4" fill="#f3ecf7" stroke="#5F0080" stroke-width="1.4"/>
    <text x="165" y="70" text-anchor="middle" fill="#5F0080" font-size="7.5" font-weight="700">트랜잭션</text>
    <rect x="12" y="86" width="306" height="18" rx="4" fill="#f3ecf7" stroke="#5F0080" stroke-width="1.4"/>
    <text x="165" y="98" text-anchor="middle" fill="#5F0080" font-size="7.5" font-weight="700">로그</text>
    <rect x="12" y="114" width="306" height="18" rx="4" fill="#f3ecf7" stroke="#5F0080" stroke-width="1.4"/>
    <text x="165" y="126" text-anchor="middle" fill="#5F0080" font-size="7.5" font-weight="700">권한 검사</text>
    <text x="165" y="147" text-anchor="middle" fill="#5f5a68" font-size="7">가로 줄 = 공통 기능. 메서드마다 복붙하는 대신 관점 하나로 모은다.</text>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">세로 상자는 각각의 업무 로직, 가로 줄은 모든 메서드에 공통으로 얹히는 부가 기능이다. AOP는 이 가로 줄을 한 군데로 모은다.</figcaption>
</figure>

## Spring은 이걸 "프록시"로 구현한다

그럼 Spring은 어드바이스를 대상 메서드에 어떻게 실제로 끼워 넣을까? 답은 **프록시(proxy)**다. 프록시는 **원본 객체와 똑같은 겉모습(같은 인터페이스/타입)을 가진 대리인 객체**다. 바깥에서 오는 호출을 원본 대신 먼저 받아서, 부가 작업을 처리한 뒤 진짜 원본에게 넘긴다.

핵심은 이거다. `@Transactional`이 붙은 `OrderService`를 다른 곳에서 주입(inject)받으면, 사실 **주입되는 건 진짜 `OrderService`가 아니라 그것을 감싼 프록시**다. 우리는 프록시인 줄도 모르고 평범하게 메서드를 부르지만, 그때마다 프록시가 중간에서 "트랜잭션 열기 → 원본 호출 → 커밋"을 대신 해 준다.

이 코드가 하는 일: 컨트롤러가 주입받은 서비스의 타입을 찍어 보면, 원본이 아니라 프록시 클래스 이름이 나온다.

```java
@RestController
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
        System.out.println(orderService.getClass());
        // 출력 예: class com.shop.OrderService$$SpringCGLIB$$abc123
        //         → 원본이 아니라 CGLIB이 만든 프록시다
    }
}
```

<!-- diagram: proxy-call-flow -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 150" role="img" aria-label="호출자가 프록시를 거쳐 원본을 부르는 흐름. 프록시가 원본 호출 전후로 트랜잭션 시작과 커밋을 감싼다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <rect x="14" y="55" width="60" height="40" rx="6" fill="#fff" stroke="#7d5a9e" stroke-width="1.6"/>
    <text x="44" y="79" text-anchor="middle" fill="#5f5a68" font-size="8">호출자</text>

    <rect x="110" y="30" width="110" height="90" rx="8" fill="#f3ecf7" stroke="#5F0080" stroke-width="2"/>
    <text x="165" y="45" text-anchor="middle" fill="#5F0080" font-size="8" font-weight="700">프록시(대리인)</text>
    <text x="165" y="60" text-anchor="middle" fill="#1f7a4d" font-size="7">① 트랜잭션 시작</text>
    <rect x="122" y="66" width="86" height="24" rx="4" fill="#fff" stroke="#7d5a9e" stroke-width="1.4"/>
    <text x="165" y="81" text-anchor="middle" fill="#5f5a68" font-size="7">② 원본 호출</text>
    <text x="165" y="104" text-anchor="middle" fill="#1f7a4d" font-size="7">③ 커밋 / 롤백</text>

    <rect x="256" y="55" width="60" height="40" rx="6" fill="#eaf5ef" stroke="#1f7a4d" stroke-width="1.8"/>
    <text x="286" y="72" text-anchor="middle" fill="#1f7a4d" font-size="8">원본</text>
    <text x="286" y="84" text-anchor="middle" fill="#5f5a68" font-size="7">업무 로직</text>

    <line x1="74" y1="75" x2="108" y2="75" stroke="#5F0080" stroke-width="1.6" marker-end="url(#ar)"/>
    <line x1="208" y1="78" x2="254" y2="78" stroke="#7d5a9e" stroke-width="1.4" marker-end="url(#ar)"/>
    <defs>
      <marker id="ar" markerWidth="7" markerHeight="7" refX="5" refY="2.5" orient="auto">
        <path d="M0,0 L5,2.5 L0,5 Z" fill="#5F0080"/>
      </marker>
    </defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">호출자는 원본을 부른다고 믿지만, 실제로는 프록시가 먼저 받아 트랜잭션을 감싸고 원본에게 넘긴다.</figcaption>
</figure>

### 프록시 만드는 두 방식 — JDK 동적 프록시 vs CGLIB

Spring이 프록시를 만드는 방법은 두 가지다.

- **JDK 동적 프록시(dynamic proxy)**: 대상이 **인터페이스를 구현**하고 있으면, 그 인터페이스를 똑같이 구현한 프록시를 런타임에 만든다. 자바 표준 기능이다.
- **CGLIB**: 인터페이스가 없으면, 대상 클래스를 **상속(extends)**한 자식 클래스를 만들어 프록시로 쓴다. 메서드를 오버라이드해서 부가 기능을 끼운다.

Spring Boot는 기본적으로 **CGLIB**을 쓴다(인터페이스가 있어도). 그래서 `final` 클래스나 `final` 메서드는 상속·오버라이드가 안 돼 프록시가 만들어지지 않는다 — 이것도 흔한 함정이다.

## 그래서 왜 "자기 호출"은 안 먹히나

이제 도입부의 수수께끼로 돌아가자. 부가 기능은 **프록시를 거쳐 들어올 때만** 작동한다. 그런데 **같은 클래스 안의 메서드끼리 직접 부르면 프록시를 거치지 않는다.**

이유는 단순하다. 클래스 내부에서 다른 메서드를 부를 때 쓰는 `this`는 **프록시가 아니라 원본 객체 자신**이다. 프록시는 바깥에서 객체로 들어오는 첫 관문일 뿐, 원본 내부의 `this.otherMethod()` 호출까지 가로채지는 못한다.

이 코드가 하는 일: 바깥에서 `placeOrder`를 부르면, 내부에서 `this.saveAudit`을 직접 호출한다. 이때 `saveAudit`의 `@Transactional`은 무시된다.

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        saveAudit("주문 저장됨");   // ← this.saveAudit() — 프록시를 안 거친다!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAudit(String msg) {   // 이 애너테이션은 여기서 무시됨
        auditRepository.save(new Audit(msg));
    }
}
```

`saveAudit`에 `REQUIRES_NEW`(항상 새 트랜잭션)를 걸었으니 "감사 로그는 독립 트랜잭션으로 남겠지" 기대하지만, 실제로는 새 트랜잭션이 열리지 않고 그냥 `placeOrder`의 트랜잭션에 묻어 간다. 애너테이션은 멀쩡한데 **호출 경로가 프록시를 우회**해서 벌어지는 일이다.

<!-- diagram: self-invocation -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 330 175" role="img" aria-label="바깥 호출은 프록시를 거쳐 placeOrder에 도달하지만, 원본 내부에서 saveAudit을 부르는 화살표는 프록시를 우회해 애너테이션이 무시된다">
  <g font-size="9" font-family="ui-monospace,SFMono-Regular,Menlo,Consolas,monospace">
    <rect x="14" y="70" width="50" height="36" rx="6" fill="#fff" stroke="#7d5a9e" stroke-width="1.6"/>
    <text x="39" y="92" text-anchor="middle" fill="#5f5a68" font-size="8">바깥</text>

    <rect x="90" y="55" width="56" height="66" rx="8" fill="#f3ecf7" stroke="#5F0080" stroke-width="2"/>
    <text x="118" y="72" text-anchor="middle" fill="#5F0080" font-size="7.5" font-weight="700">프록시</text>
    <text x="118" y="90" text-anchor="middle" fill="#1f7a4d" font-size="6.5">TX 적용</text>

    <rect x="170" y="30" width="150" height="120" rx="8" fill="#fff" stroke="#7d5a9e" stroke-width="1.6"/>
    <text x="245" y="46" text-anchor="middle" fill="#5f5a68" font-size="7.5" font-weight="700">원본 OrderService</text>
    <rect x="182" y="54" width="126" height="30" rx="5" fill="#eaf5ef" stroke="#1f7a4d" stroke-width="1.4"/>
    <text x="245" y="66" text-anchor="middle" fill="#1f7a4d" font-size="7">placeOrder</text>
    <text x="245" y="77" text-anchor="middle" fill="#5f5a68" font-size="6.5">@Transactional ✔</text>
    <rect x="182" y="104" width="126" height="30" rx="5" fill="#fdece3" stroke="#b23a00" stroke-width="1.4"/>
    <text x="245" y="116" text-anchor="middle" fill="#b23a00" font-size="7">saveAudit</text>
    <text x="245" y="127" text-anchor="middle" fill="#b23a00" font-size="6.5">@Transactional ✘ 무시</text>

    <line x1="64" y1="88" x2="88" y2="88" stroke="#5F0080" stroke-width="1.6" marker-end="url(#ar2)"/>
    <line x1="146" y1="80" x2="180" y2="72" stroke="#1f7a4d" stroke-width="1.6" marker-end="url(#ar2)"/>
    <path d="M245,84 C245,94 245,96 245,102" stroke="#b23a00" stroke-width="1.6" stroke-dasharray="4 3" fill="none" marker-end="url(#ar3)"/>
    <text x="292" y="96" text-anchor="middle" fill="#b23a00" font-size="6.5">this. 직접호출</text>
    <defs>
      <marker id="ar2" markerWidth="7" markerHeight="7" refX="5" refY="2.5" orient="auto"><path d="M0,0 L5,2.5 L0,5 Z" fill="#1f7a4d"/></marker>
      <marker id="ar3" markerWidth="7" markerHeight="7" refX="5" refY="2.5" orient="auto"><path d="M0,0 L5,2.5 L0,5 Z" fill="#b23a00"/></marker>
    </defs>
  </g>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">바깥 → 프록시 → placeOrder 경로에서는 트랜잭션이 걸리지만, 원본 안에서 this.saveAudit()로 직접 부르면 프록시를 건너뛰어 애너테이션이 무시된다.</figcaption>
</figure>

## 상세 예시 — 로그로 확인하는 자기 호출 함정

말로만 하면 안 믿기니, 트랜잭션 로그를 켜서 직접 확인해 보자. `application.yml`에 아래를 넣으면 트랜잭션이 열릴 때마다 로그가 찍힌다.

```yaml
logging:
  level:
    org.springframework.transaction.interceptor: TRACE
```

**경우 A — 자기 호출(함정)**: `placeOrder`가 내부에서 `this.saveAudit()`을 부른다. 로그에는 트랜잭션이 **딱 하나**만 찍힌다.

```text
TRACE  Getting transaction for [OrderService.placeOrder]
   (saveAudit 진입 로그 없음 — 프록시를 안 거쳐서 인터셉터가 못 봄)
TRACE  Completing transaction for [OrderService.placeOrder]
```

`REQUIRES_NEW`로 별도 트랜잭션이 열렸다면 `Getting transaction for [OrderService.saveAudit]`가 한 줄 더 있어야 하는데, 없다. 기대와 다르게 동작한다는 증거다.

**경우 B — 프록시를 거치게 고침**: 감사 저장을 다른 빈(bean)으로 분리해서, 프록시를 거쳐 호출되도록 바꾼다.

이 코드가 하는 일: 감사 로그 저장을 별도 `AuditService`로 빼내, `OrderService`가 그 빈을 주입받아 부른다. 이제 호출이 `AuditService`의 프록시를 지난다.

```java
@Service
public class OrderService {
    private final AuditService auditService;   // 다른 빈 = 다른 프록시

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        auditService.saveAudit("주문 저장됨");   // 프록시 경유 → 애너테이션 살아남
    }
}
```

이제 로그에는 트랜잭션이 **둘** 찍힌다.

```text
TRACE  Getting transaction for [OrderService.placeOrder]
TRACE  Suspending current transaction, creating new transaction for [AuditService.saveAudit]
TRACE  Completing transaction for [AuditService.saveAudit]
TRACE  Completing transaction for [OrderService.placeOrder]
```

`Suspending ... creating new transaction` 한 줄이 `REQUIRES_NEW`가 제대로 먹었다는 신호다. 바깥 트랜잭션을 잠시 멈추고 새 트랜잭션을 열었다는 뜻이다. **차이를 만든 건 애너테이션이 아니라, 프록시를 거치느냐 마느냐 하나뿐이다.**

## 자기 호출을 피하는 방법

정석은 **다른 빈으로 분리**(경우 B)다. 책임도 자연스럽게 나뉘어 가장 깔끔하다. 그 외 대안도 있다.

- **자기 자신을 프록시로 주입**: 자기 자신을 다시 주입받아(`@Autowired private OrderService self;`) `self.saveAudit()`으로 부른다. 프록시를 거치지만, 자기 참조라 어색하고 순환 참조 위험이 있어 최후의 수단이다.
- **AopContext 사용**: `((OrderService) AopContext.currentProxy()).saveAudit()`. 현재 프록시를 꺼내 부른다. 설정(`exposeProxy = true`)이 필요하고 코드가 지저분해진다.

대부분은 첫 번째(빈 분리)로 충분하다. 나머지는 "이런 우회도 있다" 정도로 알아 두면 된다.

## 한 줄 교훈

`@Transactional`·`@Cacheable` 같은 애너테이션은 **프록시를 거쳐 호출될 때만** 작동한다. 같은 클래스 안에서 `this`로 직접 부르면 프록시를 우회해 조용히 무시되니, 부가 기능이 필요한 메서드는 **다른 빈으로 분리**해 호출 경로가 프록시를 지나게 하라.
