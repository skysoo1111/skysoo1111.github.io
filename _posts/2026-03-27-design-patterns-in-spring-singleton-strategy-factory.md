---
layout: post
title: "디자인 패턴 정리 - Spring 환경에서의 Singleton, Strategy, Factory 외 주요 패턴 실전 가이드"
date: 2026-03-27 10:00:00 +0900
categories: [Java]
tags: [java, spring, design-pattern, singleton, strategy, factory, proxy, observer, template-method, gof]
---

## 디자인 패턴이란?

디자인 패턴(Design Pattern)은 소프트웨어 설계에서 반복적으로 등장하는 문제에 대한 검증된 해결책이다. 1994년 Erich Gamma 외 3인(Gang of Four, GoF)이 저서 *"Design Patterns: Elements of Reusable Object-Oriented Software"*에서 23가지 패턴을 정리했다.

GoF 패턴은 세 가지로 분류된다.

| 분류 | 설명 | 대표 패턴 |
|------|------|----------|
| **생성(Creational)** | 객체 생성 방식을 추상화 | Singleton, Factory Method, Abstract Factory, Builder, Prototype |
| **구조(Structural)** | 클래스/객체의 구성 방식 | Proxy, Decorator, Adapter, Facade, Composite |
| **행동(Behavioral)** | 객체 간 책임 분배와 알고리즘 | Strategy, Observer, Template Method, Command, Chain of Responsibility |

Spring Framework 자체가 GoF 패턴의 집약체다. Spring을 제대로 이해하려면 이 패턴들을 반드시 알아야 한다.

---

## 1. 싱글톤 패턴 (Singleton)

### 개념

하나의 클래스에 인스턴스가 단 하나만 존재하도록 보장하는 패턴이다.

**전통적인 싱글톤 구현 (권장하지 않음):**

```java
public class DatabaseConnection {
    private static DatabaseConnection instance;

    private DatabaseConnection() {}

    public static synchronized DatabaseConnection getInstance() {
        if (instance == null) {
            instance = new DatabaseConnection();
        }
        return instance;
    }
}
```

이 방식은 멀티스레드 환경에서 이중 검사(double-checked locking)나 `synchronized` 처리 없이는 안전하지 않다.

### Spring Bean과 싱글톤

Spring의 IoC 컨테이너는 기본적으로 모든 Bean을 **싱글톤 스코프**로 관리한다. 개발자가 직접 싱글톤 패턴을 구현할 필요가 없다.

```java
@Component
// @Scope("singleton") 은 기본값이므로 생략 가능
public class UserService {
    // Spring ApplicationContext 당 하나의 인스턴스만 생성됨
}
```

Spring이 지원하는 Bean 스코프:

| 스코프 | 설명 |
|--------|------|
| `singleton` | ApplicationContext 당 1개 (기본값) |
| `prototype` | 요청할 때마다 새 인스턴스 생성 |
| `request` | HTTP 요청 당 1개 |
| `session` | HTTP 세션 당 1개 |
| `application` | ServletContext 당 1개 |

### 싱글톤 빈의 스레드 안전성 주의점

Spring 싱글톤 Bean은 **스레드 안전을 보장하지 않는다**. 여러 요청이 동시에 같은 Bean 인스턴스를 사용하기 때문에 가변 상태(mutable state)가 있으면 경쟁 조건(race condition)이 발생한다.

```java
// 잘못된 예 - 공유 가변 상태
@Service
public class CounterService {
    private int count = 0; // 여러 스레드가 동시에 접근 -> 문제

    public int increment() {
        return ++count; // race condition!
    }
}

// 올바른 예 - 무상태(stateless) 유지
@Service
public class CalculatorService {
    public int add(int a, int b) {
        return a + b; // 로컬 변수만 사용 -> 스레드 안전
    }
}

// 불가피하게 상태가 필요한 경우
@Service
public class StatefulCounterService {
    private final AtomicInteger count = new AtomicInteger(0); // 스레드 안전

    public int increment() {
        return count.incrementAndGet();
    }
}
```

**모범 사례:** Spring Bean은 가능하면 **무상태(stateless)**로 설계한다.

---

## 2. 전략 패턴 (Strategy)

### 개념

알고리즘 군을 정의하고 각각을 캡슐화하여 교환 가능하게 만드는 패턴이다. 클라이언트와 알고리즘을 분리하여 런타임에 알고리즘을 교체할 수 있다.

### Spring DI + 인터페이스로 구현

Spring의 의존성 주입(DI)은 전략 패턴을 매우 자연스럽게 구현할 수 있게 해준다.

**결제 수단 선택 예시:**

```java
// 전략 인터페이스
public interface PaymentStrategy {
    void pay(int amount);
    String getType();
}

// 신용카드 전략
@Component("creditCard")
public class CreditCardPayment implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("신용카드로 " + amount + "원 결제");
    }

    @Override
    public String getType() {
        return "creditCard";
    }
}

// PayPal 전략
@Component("paypal")
public class PayPalPayment implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("PayPal로 " + amount + "원 결제");
    }

    @Override
    public String getType() {
        return "paypal";
    }
}

// 카카오페이 전략
@Component("kakaoPay")
public class KakaoPayPayment implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("카카오페이로 " + amount + "원 결제");
    }

    @Override
    public String getType() {
        return "kakaoPay";
    }
}
```

```java
// Context - 전략을 사용하는 서비스
@Service
public class OrderService {
    private final Map<String, PaymentStrategy> paymentStrategies;

    // Spring이 PaymentStrategy 구현체들을 Map으로 자동 주입
    // key: Bean 이름, value: Bean 인스턴스
    public OrderService(Map<String, PaymentStrategy> paymentStrategies) {
        this.paymentStrategies = paymentStrategies;
    }

    public void processPayment(String paymentMethod, int amount) {
        PaymentStrategy strategy = paymentStrategies.get(paymentMethod);
        if (strategy == null) {
            throw new IllegalArgumentException("지원하지 않는 결제 수단: " + paymentMethod);
        }
        strategy.pay(amount);
    }
}
```

새로운 결제 수단을 추가할 때 `OrderService`를 수정하지 않고 새 `@Component`만 추가하면 된다. **OCP(개방-폐쇄 원칙)**를 자연스럽게 따른다.

**Kotlin 버전:**

```kotlin
interface PaymentStrategy {
    fun pay(amount: Int)
}

@Component("toss")
class TossPayment : PaymentStrategy {
    override fun pay(amount: Int) = println("토스로 ${amount}원 결제")
}

@Service
class OrderService(
    private val paymentStrategies: Map<String, PaymentStrategy>
) {
    fun processPayment(method: String, amount: Int) {
        paymentStrategies[method]?.pay(amount)
            ?: throw IllegalArgumentException("지원하지 않는 결제 수단: $method")
    }
}
```

---

## 3. 팩토리 패턴 (Factory Method / Abstract Factory)

### 개념

- **Factory Method**: 객체 생성 인터페이스를 정의하고 서브클래스가 어떤 클래스를 인스턴스화할지 결정
- **Abstract Factory**: 관련 객체들의 집합을 생성하는 인터페이스 제공

### Spring의 BeanFactory와 FactoryBean

Spring의 `BeanFactory`는 팩토리 패턴의 대표적인 구현체다.

```java
// @Bean 메서드 = Factory Method 패턴
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
            .driverClassName("com.mysql.cj.jdbc.Driver")
            .url("jdbc:mysql://localhost:3306/mydb")
            .username("user")
            .password("password")
            .build();
    }
}
```

**FactoryBean 인터페이스로 복잡한 객체 생성:**

```java
public class RedisConnectionFactoryBean implements FactoryBean<RedisConnectionFactory> {
    private final String host;
    private final int port;

    public RedisConnectionFactoryBean(String host, int port) {
        this.host = host;
        this.port = port;
    }

    @Override
    public RedisConnectionFactory getObject() {
        LettuceConnectionFactory factory = new LettuceConnectionFactory(host, port);
        factory.afterPropertiesSet();
        return factory;
    }

    @Override
    public Class<?> getObjectType() {
        return RedisConnectionFactory.class;
    }
}
```

**조건부 Bean 생성 - Abstract Factory 응용:**

```java
public interface NotificationService {
    void send(String message, String recipient);
}

@Component
public class EmailNotificationService implements NotificationService {
    @Override
    public void send(String message, String recipient) {
        // 이메일 발송 로직
    }
}

@Component
public class SmsNotificationService implements NotificationService {
    @Override
    public void send(String message, String recipient) {
        // SMS 발송 로직
    }
}

@Configuration
public class NotificationConfig {

    @Bean
    @ConditionalOnProperty(name = "notification.channel", havingValue = "email")
    public NotificationService emailNotificationService() {
        return new EmailNotificationService();
    }

    @Bean
    @ConditionalOnProperty(name = "notification.channel", havingValue = "sms")
    public NotificationService smsNotificationService() {
        return new SmsNotificationService();
    }
}
```

---

## 4. 템플릿 메서드 패턴 (Template Method)

### 개념

알고리즘의 뼈대(skeleton)를 상위 클래스에서 정의하고, 세부 구현을 하위 클래스에 위임하는 패턴이다. 공통 흐름은 상위 클래스에서 처리하고, 변하는 부분만 오버라이드한다.

### Spring의 xxxTemplate 클래스들

Spring의 `JdbcTemplate`, `RestTemplate`, `TransactionTemplate`은 모두 템플릿 메서드 패턴을 활용한다.

**JdbcTemplate:**

```java
@Repository
public class UserRepository {
    private final JdbcTemplate jdbcTemplate;

    public UserRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<User> findAll() {
        // JdbcTemplate이 Connection 획득, PreparedStatement 생성,
        // 예외 처리, 리소스 반환 등 보일러플레이트를 처리
        // 개발자는 SQL과 RowMapper만 제공
        return jdbcTemplate.query(
            "SELECT id, name, email FROM users",
            (rs, rowNum) -> new User(
                rs.getLong("id"),
                rs.getString("name"),
                rs.getString("email")
            )
        );
    }

    public Optional<User> findById(Long id) {
        return jdbcTemplate.query(
            "SELECT id, name, email FROM users WHERE id = ?",
            (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name"), rs.getString("email")),
            id
        ).stream().findFirst();
    }
}
```

**직접 구현해보는 템플릿 메서드:**

```java
// 추상 템플릿 클래스
public abstract class DataExportTemplate {

    // 템플릿 메서드 - 알고리즘 골격
    public final void export(String destination) {
        List<Object> data = fetchData();       // 추상 메서드
        List<Object> processed = process(data); // 훅 메서드 (선택적 오버라이드)
        write(processed, destination);          // 추상 메서드
        notifyComplete();                       // 공통 구현
    }

    protected abstract List<Object> fetchData();
    protected abstract void write(List<Object> data, String destination);

    // 훅 메서드 - 기본 구현 제공, 필요 시 오버라이드
    protected List<Object> process(List<Object> data) {
        return data; // 기본은 그대로 반환
    }

    private void notifyComplete() {
        System.out.println("내보내기 완료");
    }
}

// 구체 구현
@Component
public class CsvDataExport extends DataExportTemplate {
    private final UserRepository userRepository;

    public CsvDataExport(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    protected List<Object> fetchData() {
        return new ArrayList<>(userRepository.findAll());
    }

    @Override
    protected void write(List<Object> data, String destination) {
        // CSV 파일 작성 로직
    }
}
```

---

## 5. 프록시 패턴 (Proxy)

### 개념

실제 객체에 대한 접근을 제어하기 위해 대리 객체(proxy)를 두는 패턴이다. 프록시는 실제 객체와 동일한 인터페이스를 구현하며, 요청을 실제 객체에 위임하기 전후에 추가 로직을 수행할 수 있다.

### Spring AOP와 @Transactional

Spring AOP는 프록시 패턴의 대표적인 활용 사례다.

```java
// @Transactional은 Spring이 프록시를 생성하여 트랜잭션 관리
@Service
public class OrderService {

    @Transactional
    public void placeOrder(OrderRequest request) {
        // Spring이 생성한 프록시가 실제 메서드 호출 전후에:
        // 1. 트랜잭션 시작 (begin)
        // 2. 메서드 실행
        // 3. 성공 시 커밋 (commit)
        // 4. 예외 시 롤백 (rollback)
        Order order = Order.from(request);
        orderRepository.save(order);
        inventoryService.decreaseStock(order.getItems());
        paymentService.charge(order.getTotalAmount());
    }
}
```

**중요 주의사항 - self-invocation 문제:**

```java
@Service
public class OrderService {

    @Transactional
    public void outerMethod() {
        innerMethod(); // 같은 클래스 내부 호출 -> 프록시 우회 -> @Transactional 무효!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void innerMethod() {
        // 이 트랜잭션 설정은 outerMethod()에서 직접 호출 시 적용되지 않음
    }
}
```

**커스텀 AOP 구현:**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ExecutionTime {}

@Aspect
@Component
public class ExecutionTimeAspect {

    private static final Logger log = LoggerFactory.getLogger(ExecutionTimeAspect.class);

    @Around("@annotation(ExecutionTime)")
    public Object measureExecutionTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        String methodName = pjp.getSignature().toShortString();

        try {
            Object result = pjp.proceed();
            log.info("{} 실행 시간: {}ms", methodName, System.currentTimeMillis() - start);
            return result;
        } catch (Exception e) {
            log.error("{} 실행 중 예외 발생: {}", methodName, e.getMessage());
            throw e;
        }
    }
}

// 사용
@Service
public class ReportService {

    @ExecutionTime
    public Report generateReport(Long reportId) {
        // 실행 시간 자동 측정
        return buildReport(reportId);
    }
}
```

---

## 6. 옵저버 패턴 (Observer)

### 개념

객체의 상태 변화를 다른 객체들에게 자동으로 알리는 패턴이다. 발행-구독(Publish-Subscribe) 구조로도 알려져 있다.

### Spring의 ApplicationEvent

```java
// 이벤트 클래스 정의
public class UserRegisteredEvent extends ApplicationEvent {
    private final User user;

    public UserRegisteredEvent(Object source, User user) {
        super(source);
        this.user = user;
    }

    public User getUser() {
        return user;
    }
}
```

```java
// 이벤트 발행자 (Subject/Publisher)
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public User register(UserRegisterRequest request) {
        User user = User.builder()
            .email(request.getEmail())
            .name(request.getName())
            .build();

        User savedUser = userRepository.save(user);

        // 이벤트 발행 - 리스너들은 알아서 처리
        eventPublisher.publishEvent(new UserRegisteredEvent(this, savedUser));

        return savedUser;
    }
}
```

```java
// 이벤트 리스너 (Observer/Subscriber) - 관심사 분리
@Component
@RequiredArgsConstructor
public class NotificationEventListener {
    private final EmailService emailService;
    private final SlackService slackService;

    // 동기 처리
    @EventListener
    public void sendWelcomeEmail(UserRegisteredEvent event) {
        emailService.sendWelcomeEmail(event.getUser().getEmail());
    }

    // 비동기 처리
    @EventListener
    @Async
    public void sendSlackNotification(UserRegisteredEvent event) {
        slackService.notify("새 회원 가입: " + event.getUser().getName());
    }

    // 트랜잭션 커밋 후 처리 (DB 저장이 확정된 후 실행)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void sendAnalyticsEvent(UserRegisteredEvent event) {
        analyticsService.track("user_registered", event.getUser().getId());
    }
}
```

**Spring 이벤트의 장점:**
- `UserService`는 이메일/슬랙/분석 서비스를 직접 의존하지 않음
- 새로운 처리 로직 추가 시 기존 코드 수정 불필요 (OCP)
- `@Async`로 비동기 처리 가능
- `@TransactionalEventListener`로 트랜잭션 경계와 연동 가능

---

## 7. 패턴 선택 가이드

어떤 패턴을 언제 사용해야 하는지 정리했다.

| 상황 | 적용 패턴 | Spring 적용 방법 |
|------|-----------|-----------------|
| 전역 상태 / 공유 자원 관리 | Singleton | `@Component` (기본 스코프) |
| 여러 알고리즘 중 런타임 선택 | Strategy | 인터페이스 + DI, `Map<String, Strategy>` |
| 복잡한 객체 생성 캡슐화 | Factory Method | `@Bean` 메서드, `FactoryBean` |
| 공통 흐름 + 세부 구현 분리 | Template Method | `JdbcTemplate`, 추상 클래스 확장 |
| 횡단 관심사 (로깅, 트랜잭션) | Proxy | `@Aspect`, `@Transactional` |
| 이벤트 기반 느슨한 결합 | Observer | `ApplicationEvent`, `@EventListener` |
| 환경별 다른 구현체 사용 | Abstract Factory | `@ConditionalOnProperty` |

### 패턴 조합 예시

실제 프로젝트에서는 여러 패턴을 함께 사용한다.

```java
// Strategy + Observer + Template Method 조합
@Service
public class PaymentFacade {
    private final Map<String, PaymentStrategy> strategies;   // Strategy
    private final ApplicationEventPublisher eventPublisher;  // Observer

    @Transactional                                           // Proxy (AOP)
    public PaymentResult pay(PaymentRequest request) {
        PaymentStrategy strategy = strategies.get(request.getMethod()); // Strategy 선택
        PaymentResult result = strategy.execute(request);               // Template Method

        if (result.isSuccess()) {
            eventPublisher.publishEvent(                                 // Observer 발행
                new PaymentCompletedEvent(this, result)
            );
        }

        return result;
    }
}
```

---

## 마치며

Spring Framework를 사용하면서 무심코 쓰던 `@Transactional`, `@EventListener`, `JdbcTemplate` 등이 모두 GoF 디자인 패턴의 구현임을 알 수 있다. 패턴을 코드로 외우려 하기보다는 **"어떤 문제를 해결하기 위한 패턴인가"**를 이해하는 것이 중요하다.

Spring이 제공하는 패턴 구현체들을 잘 활용하면 직접 패턴을 구현하는 수고 없이도 유지보수성 높은 코드를 작성할 수 있다.
