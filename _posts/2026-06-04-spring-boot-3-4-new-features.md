---
layout: post
title: "Spring Boot 3.4 신기능 완벽 정리"
description: "Spring Boot 3.4는 구조화된 로깅, Graceful Shutdown 기본 활성화, Virtual Threads 확장, 컨테이너 지원 강화, Actuator…"
date: 2026-06-04 17:02:15 +0900
categories: [Spring Boot, Java]
tags: [Spring Boot, Spring Framework, Virtual Threads, Observability, Docker Compose, Actuator]
image:
  path: https://source.unsplash.com/featured/?spring,java,programming
  alt: Spring Boot 3.4 신기능
---

## 개요

Spring Boot 3.4는 구조화된 로깅, Graceful Shutdown 기본 활성화, Virtual Threads 확장, 컨테이너 지원 강화, Actuator 접근 제어 재설계 등 운영 환경 편의성과 개발자 경험을 대폭 개선한 버전이다. 2025년 출시 예정인 Spring Boot 4.0의 전 세대로서, 클라우드 네이티브 애플리케이션 개발에 필요한 핵심 기능들이 집중적으로 강화되었다. 이 포스트에서는 Spring Boot 3.4의 주요 신기능과 Breaking Changes, 그리고 마이그레이션 시 주의사항을 정리한다.

---

## 1. 구조화된 로깅 (Structured Logging)

기계 판독 가능한 로그 형식을 기본 지원한다. 지원 포맷은 다음과 같다.

| 포맷 | 용도 |
|------|------|
| **Elastic Common Schema (ECS)** | Elastic 스택과의 통합에 최적화 |
| **Logstash** | Logstash 파이프라인과 직접 연동 |
| **Graylog Extended Log Format (GELF)** | Graylog 로그 관리 시스템 지원 |

별도 파싱 없이 로그 수집·분석 파이프라인을 바로 구성할 수 있어, ELK 스택이나 Graylog 기반 운영 환경에서 특히 유용하다.

```yaml
# application.yml
logging:
  structured:
    format:
      console: ecs       # ECS 포맷으로 콘솔 출력
      file: logstash     # Logstash 포맷으로 파일 저장
```

### 출력 예시 (ECS 포맷)

```json
{
  "@timestamp": "2026-06-04T08:02:15.000Z",
  "log.level": "INFO",
  "message": "Started Application in 2.345 seconds",
  "service.name": "my-service",
  "service.version": "1.0.0"
}
```

---

## 2. Graceful Shutdown 기본 활성화

임베드된 웹 서버(Jetty, Reactor Netty, Tomcat, Undertow)의 graceful shutdown이 이제 기본으로 활성화된다. 이전처럼 `server.shutdown=graceful`을 명시하지 않아도 동작한다.

이전 동작으로 되돌리려면 아래와 같이 설정한다.

```yaml
server:
  shutdown: immediate
```

운영 환경에서 shutdown 타임아웃도 함께 검토하는 것이 좋다.

```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

---

## 3. Virtual Threads 확장

Project Loom(JDK 21 이상) 기반 Virtual Threads 지원이 더욱 확장되었다. `OtlpMeterRegistry`와 Undertow 웹 서버가 추가로 Virtual Threads를 활용하도록 지원된다. I/O 바운드 애플리케이션에서 OS 스레드 대비 메모리 오버헤드를 크게 줄이면서 높은 동시성을 확보할 수 있다.

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

### Virtual Thread vs Platform Thread 비교

```kotlin
// Virtual Threads 활성화 후 - 기존 명령형 코드 그대로 사용 가능
@RestController
class OrderController(private val orderService: OrderService) {

    @GetMapping("/orders/{id}")
    fun getOrder(@PathVariable id: Long): OrderResponse {
        // I/O 대기 시 Virtual Thread가 carrier thread를 반납 → 높은 처리량
        return orderService.findById(id)
    }
}
```

별도의 코드 변경 없이 `spring.threads.virtual.enabled=true` 설정만으로 혜택을 받을 수 있다.

---

## 4. Docker Compose 다중 파일 및 신규 서비스 지원

여러 Docker Compose 설정 파일을 병합하여 사용하는 기능이 추가되었다. 환경별로 다른 compose 파일을 관리하기가 훨씬 편리해졌다.

```yaml
spring:
  docker:
    compose:
      file:
        - docker-compose.yml
        - docker-compose.local.yml
```

새롭게 지원되는 서비스 목록은 다음과 같다.

| 카테고리 | 신규 지원 서비스 |
|----------|----------------|
| 캐시 | Redis Stack |
| 메시징 | Kafka |
| 모니터링 | Grafana LGTM |
| 분산 캐시 | Hazelcast |

Bitnami 컨테이너 이미지 지원도 추가되어 로컬 개발 환경 구성이 더욱 단순해졌다.

---

## 5. Testcontainers 지원 확대

테스트 환경에서 실제 외부 서비스를 손쉽게 연동할 수 있도록 지원 범위가 넓어졌다.

```kotlin
@SpringBootTest
class KafkaIntegrationTest {

    @Container
    @ServiceConnection  // 자동으로 Spring 설정에 연결
    val kafka = KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:latest"))

    @Autowired
    lateinit var kafkaTemplate: KafkaTemplate<String, String>

    @Test
    @DisplayName("Kafka 메시지 발행 및 수신 통합 테스트")
    fun `kafka message publish and consume`() {
        kafkaTemplate.send("test-topic", "hello")
        // 검증 로직
    }
}
```

새로 추가된 지원 대상: **Kafka**, **Redis Stack**, **Grafana LGTM**, **Hazelcast**, **OTLP 로깅**

---

## 6. Actuator 접근 제어 재설계

기존의 단순 enabled/disabled 방식에서 세 단계 접근 레벨 모델로 전환되었다.

| 접근 레벨 | 설명 |
|-----------|------|
| `none` | 엔드포인트 비활성화 |
| `read-only` | 읽기 전용 접근 허용 |
| `unrestricted` | 모든 작업 허용 |

```yaml
management:
  endpoints:
    access:
      default: read-only   # 전체 기본값
  endpoint:
    shutdown:
      access: none         # shutdown은 완전 비활성화
    health:
      access: unrestricted # health는 전체 허용
```

기존 `management.endpoint.<id>.enabled` 속성은 deprecated 처리되므로 마이그레이션 시 신규 속성으로 교체해야 한다.

---

## 7. ClientHttpRequestFactoryBuilder

HTTP 클라이언트 팩토리를 타입 안전하게 구성할 수 있는 새로운 빌더 인터페이스가 제공된다.

```kotlin
@Configuration
class HttpClientConfig {

    @Bean
    fun restClient(builder: RestClient.Builder): RestClient {
        val factory = ClientHttpRequestFactoryBuilder
            .httpComponents()
            .withDefaultRequestConfig { config ->
                config.setConnectionRequestTimeout(Timeout.ofSeconds(5))
            }
            .build()

        return builder
            .requestFactory(factory)
            .baseUrl("https://api.example.com")
            .build()
    }
}
```

HTTP 클라이언트 선택 우선순위도 변경되었다: `Apache HTTP Components → Jetty → Reactor Netty → JDK → Simple JDK`

---

## 8. SSL 번들 개선

SSL 인증서 정보를 `/actuator/info`에서 직접 확인할 수 있으며, 인증서 만료 임박 시 헬스 체크 상태가 `OUT_OF_SERVICE`로 전환된다.

```yaml
spring:
  ssl:
    bundle:
      jks:
        mybundle:
          keystore:
            location: classpath:application.jks
            password: secret
```

SNI(Server Name Indication) 지원도 추가되어 단일 서버에서 여러 SSL 인증서를 호스트명 기반으로 선택할 수 있다. 멀티 테넌트 애플리케이션 구성에 유용하다.

---

## 9. SBOM 엔드포인트

새로운 `/actuator/sbom` 엔드포인트를 통해 애플리케이션의 소프트웨어 구성 요소 목록(Software Bill of Materials)을 제공한다. 보안 취약점 추적 및 공급망 보안 관리에 활용할 수 있다.

```bash
curl http://localhost:8080/actuator/sbom
```

---

## 10. 관찰성(Observability) 개선

Micrometer, OpenTelemetry, Prometheus와의 통합이 강화되었다.

- `spring.application.group` 속성으로 애플리케이션 그룹화 지원
- OTLP gRPC 전송 지원 추가
- OpenTelemetry 로깅 내보내기 자동 구성

```yaml
spring:
  application:
    name: order-service
    group: payment-domain   # 관련 서비스 그룹화

management:
  otlp:
    tracing:
      endpoint: http://otel-collector:4317
    logging:
      export:
        enabled: true
```

---

## Breaking Changes 및 마이그레이션 팁

### 1. Graceful Shutdown 기본 활성화

3.3에서 업그레이드 시 shutdown 타임아웃 설정을 반드시 검토한다. 기존처럼 즉시 종료가 필요한 경우 `server.shutdown=immediate`를 명시한다.

### 2. Bean Validation 변경

`@ConfigurationProperties` 중첩 속성 검증은 이제 `@Valid`가 있는 경우에만 수행된다.

```kotlin
// 3.3 이하에서는 암묵적으로 중첩 검증 수행
@ConfigurationProperties(prefix = "app")
data class AppProperties(
    @field:Valid  // 3.4부터는 @Valid 명시 필수
    val database: DatabaseProperties
)
```

### 3. Actuator 속성 변경

```yaml
# Before (deprecated)
management:
  endpoint:
    health:
      enabled: true

# After
management:
  endpoint:
    health:
      access: unrestricted
```

### 4. Gradle 최소 버전 변경

Gradle 7.5 ~ 8.3은 더 이상 지원되지 않는다. Gradle 7.6.4 이상 또는 8.4 이상으로 업그레이드가 필요하다.

### 5. OCI 이미지 빌드 기본 빌더 변경

기본 Paketo 빌더가 `paketobuildpacks/builder-jammy-java-tiny`로 변경된다. 이전 빌더를 계속 사용하려면 명시적으로 지정해야 한다.

---

## 주요 의존성 업그레이드

| 라이브러리 | 버전 |
|-----------|------|
| Spring Framework | 6.2 |
| Spring Security | 6.4 |
| Spring Data | 2024.1 |
| Micrometer | 1.14 |
| OpenTelemetry | 1.43 |
| Hibernate | 6.6 |
| Flyway | 10.20 |
| Liquibase | 4.29 |

---

## 정리

- **구조화된 로깅**: ECS, Logstash, GELF 포맷을 설정 한 줄로 활성화, 로그 파이프라인 통합이 쉬워짐
- **Graceful Shutdown 기본 활성화**: 3.3에서 업그레이드 시 shutdown 타임아웃 설정을 반드시 검토해야 함
- **Virtual Threads 확장**: `spring.threads.virtual.enabled=true` 하나로 코드 변경 없이 높은 동시성 확보
- **Actuator 접근 제어 재설계**: `none / read-only / unrestricted` 세 단계로 세밀한 제어 가능, 기존 `enabled` 속성은 deprecated
- **SBOM 엔드포인트 추가**: `/actuator/sbom`으로 공급망 보안 관리 지원
- **Testcontainers 및 Docker Compose 지원 확대**: Kafka, Redis Stack, Grafana LGTM, Hazelcast 등 추가

---

## 참고 자료

- [Spring Boot 3.4 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.4-Release-Notes)
- [Spring Framework 6.2 and Spring Boot 3.4 Improve Containers, Actuators - InfoQ](https://www.infoq.com/news/2024/11/spring-6-2-spring-boot-3-4/)
- [What's New in Spring Boot 3.4 - Medium](https://medium.com/@vikasgoel53/embracing-the-future-whats-new-in-spring-boot-3-4-and-the-3-x-era-c04a7c3c4b1e)
- [Spring Boot 3.5 - InfoQ](https://www.infoq.com/news/2025/05/spring-boot-3-5/)
