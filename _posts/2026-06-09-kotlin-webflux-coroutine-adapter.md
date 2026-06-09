---
layout: post
title: "Kotlin WebFlux에서 요청 처리 흐름과 코루틴 어댑터 역할 완벽 정리"
date: 2026-06-09 15:04:06 +0900
categories: [Kotlin, WebFlux]
tags: [Kotlin, WebFlux, Coroutine, Reactor, Spring, coroutine-adapter]
image:
  path: https://source.unsplash.com/featured/?kotlin,reactive,programming
  alt: Kotlin WebFlux 코루틴 어댑터
---

## 개요

Spring WebFlux는 Reactor 기반의 논블로킹 웹 프레임워크다. 그러나 Reactor의 `Mono`/`Flux` 체인은 복잡한 비즈니스 로직을 표현할 때 가독성이 급격히 떨어진다. Kotlin 코루틴은 이 문제를 해결한다. `suspend` 함수로 비동기 코드를 명령형 스타일로 작성하면서도 내부적으로는 Reactor 파이프라인 위에서 완전한 논블로킹 처리가 이루어진다. 이 포스트에서는 HTTP 요청이 들어온 순간부터 응답이 반환되기까지 WebFlux + 코루틴 스택 전체 흐름과, 그 중심에 있는 코루틴 어댑터의 역할을 정리한다.

---

## 요청 처리 전체 흐름

HTTP 요청 하나가 들어왔을 때 내부에서 벌어지는 일을 순서대로 나열하면 다음과 같다.

```
[HTTP 요청 수신]
       ↓
[Reactor Netty (논블로킹 I/O)]
       ↓
[HttpHandler → WebHttpHandlerBuilder]
       ↓
[DispatcherHandler.handle()]
       ↓
[HandlerMapping → HandlerAdapter 선택]
       ↓
[suspend function 감지]
       ↓
[CoroutinesUtils.invokeSuspendingFunction()]
       ↓
[mono { } 빌더 → MonoCoroutine 생성]
       ↓
[코루틴 실행: 일시중단 / 재개]
       ↓
[Mono.success() 또는 Mono.error() 발행]
       ↓
[Reactor 파이프라인 → 직렬화 → HTTP 응답]
```

`HandlerAdapter`가 핸들러 메서드를 호출하기 직전, 해당 메서드가 `suspend` 함수인지 리플렉션으로 확인한다. `suspend` 함수라면 일반 `invoke` 대신 `CoroutinesUtils.invokeSuspendingFunction()`을 통해 실행한다. 이 유틸리티가 바로 Reactive 세계와 Coroutine 세계 사이의 공식 어댑터 지점이다.

### Reactive ↔ Coroutine 변환 테이블

| Reactive (Reactor) | Coroutine (Kotlin) | 방향 |
|---|---|---|
| `Mono<T>` | `suspend fun(): T` | Reactive → Coroutine |
| `Mono<Void>` | `suspend fun()` | Reactive → Coroutine |
| `Flux<T>` | `fun(): Flow<T>` | Reactive → Coroutine |
| `suspend fun(): T` | `Mono<T>` | Coroutine → Reactive |
| `fun(): Flow<T>` | `Flux<T>` | Coroutine → Reactive |
| `Mono<T>.awaitSingle()` | `T` | Mono → 값 추출 |
| `mono { ... }` | `Mono<T>` | 블록 → Mono 래핑 |
| `Flow<T>.asFlux()` | `Flux<T>` | Flow → Flux |

---

## 코루틴 어댑터 핵심 역할

### CoroutinesUtils.invokeSuspendingFunction()

Spring Framework 내부에 위치한 이 유틸리티는 `suspend` 함수를 `Mono`로 변환하는 핵심 진입점이다. 동작 방식을 단순화하면 다음과 같다.

1. Kotlin 리플렉션으로 대상 메서드가 `suspend` 함수인지 확인한다.
2. `kotlinx-coroutines-reactor`의 `mono { }` 빌더를 이용해 코루틴 블록을 `Mono`로 감싼다.
3. Reactor `Subscriber`가 구독을 시작하면 코루틴이 실행된다.
4. `suspend` 지점에서 코루틴이 일시 중단되고, I/O 완료 시 Reactor 이벤트 루프 스레드에서 재개된다.
5. 코루틴이 값을 반환하면 `Mono.success(value)`로, 예외를 던지면 `Mono.error(ex)`로 신호를 발행한다.

### mono { } 빌더와 핵심 API

`kotlinx-coroutines-reactor` 라이브러리가 제공하는 핵심 브릿지 API는 다음과 같다.

```kotlin
// suspend → Mono: 코루틴 블록을 Reactor Mono로 래핑
fun <T> mono(block: suspend CoroutineScope.() -> T?): Mono<T>

// Mono → suspend: Mono 값을 코루틴에서 꺼내기
suspend fun <T> Mono<T>.awaitSingle(): T
suspend fun <T> Mono<T>.awaitSingleOrNull(): T?
suspend fun <T> Mono<T>.awaitFirstOrNull(): T?
suspend fun <T> Mono<T>.awaitFirstOrDefault(default: T): T

// Flow ↔ Flux 변환
fun <T> Flow<T>.asFlux(): Flux<T>
fun <T> Flux<T>.asFlow(): Flow<T>
```

`mono { }` 빌더는 내부적으로 `MonoCoroutine`이라는 Coroutine + Publisher 하이브리드 객체를 생성한다. 이 객체는 Reactor의 `Mono` 인터페이스를 구현하는 동시에 코루틴의 실행 컨텍스트(Job, CoroutineContext)를 보유한다.

---

## 코드 예제

### @RestController

`@RestController` 방식에서는 Spring이 자동으로 코루틴 어댑터를 적용한다. 개발자는 `suspend`와 `Flow` 반환 타입만 신경 쓰면 된다.

```kotlin
@RestController
@RequestMapping("/users")
class UserController(private val service: UserService) {

    // suspend fun → Mono<User> 자동 변환
    @GetMapping("/{id}")
    suspend fun getUser(@PathVariable id: Long): User =
        service.findById(id)

    // Flow<User> → Flux<User> 자동 변환
    @GetMapping
    fun getUsers(): Flow<User> =
        service.findAll()

    // 병렬 처리: coroutineScope + async
    @GetMapping("/summary")
    suspend fun getSummary(): Summary = coroutineScope {
        val users = async { service.findAll().toList() }
        val count = async { service.count() }
        Summary(users.await(), count.await())
    }
}
```

### coRouter DSL + Handler

함수형 라우팅 방식인 `coRouter`를 사용하면 라우팅 선언과 핸들러 로직을 깔끔하게 분리할 수 있다.

```kotlin
// Router
@Configuration
class RouterConfig(private val handler: ProductHandler) {

    @Bean
    fun routes() = coRouter {
        "/products".nest {
            GET("", handler::findAll)
            GET("/{id}", handler::findById)
            POST("", handler::create)
            DELETE("/{id}", handler::delete)
        }
    }
}

// Handler
@Component
class ProductHandler(private val repository: ProductRepository) {

    suspend fun findAll(request: ServerRequest): ServerResponse =
        ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .bodyAndAwait(repository.findAll())

    suspend fun findById(request: ServerRequest): ServerResponse {
        val id = request.pathVariable("id").toLong()
        return repository.findById(id)
            ?.let { ServerResponse.ok().bodyValueAndAwait(it) }
            ?: ServerResponse.notFound().buildAndAwait()
    }

    suspend fun create(request: ServerRequest): ServerResponse {
        val product = request.awaitBody<Product>()
        return ServerResponse.status(HttpStatus.CREATED)
            .bodyValueAndAwait(repository.save(product))
    }

    suspend fun delete(request: ServerRequest): ServerResponse {
        val id = request.pathVariable("id").toLong()
        repository.deleteById(id)
        return ServerResponse.noContent().buildAndAwait()
    }
}
```

`bodyValueAndAwait()`가 내부적으로 하는 일은 간단하다.

```kotlin
// 확장 함수 내부 원리 (개념적 표현)
suspend fun ServerResponse.BodyBuilder.bodyValueAndAwait(body: Any): ServerResponse =
    this.bodyValue(body).awaitSingle()

suspend fun ServerResponse.HeadersBuilder<*>.buildAndAwait(): ServerResponse =
    this.build().awaitSingle()
```

Reactor `Mono`를 반환하는 빌더 메서드 뒤에 `awaitSingle()`을 체이닝하는 것이 `AndAwait` 접미사 패턴의 전부다.

### CoroutineCrudRepository

Repository 계층도 `CoroutineCrudRepository`를 사용하면 Reactor 타입이 완전히 사라진다.

```kotlin
interface ProductRepository : CoroutineCrudRepository<Product, Long> {
    fun findByCategory(category: String): Flow<Product>
    suspend fun findByName(name: String): Product?
}
```

- `findAll()` → `Flow<Product>` 반환
- `findById(id)` → `Product?` 반환 (nullable, Optional 불필요)
- `save(entity)` → `Product` 반환
- `deleteById(id)` → `Unit` 반환

---

## Reactive Streams vs 코루틴 실무 비교

복잡한 비즈니스 로직에서 두 방식의 차이가 가장 명확하게 드러난다.

### Before: 중첩된 Reactive 체인

```kotlin
// 오븐 예열 완료 여부에 따라 분기하는 로직
fun bakeHoney(): Mono<Honey> {
    return ovenService.heatOven(temperature)
        .flatMap { ovenReady ->
            honeyService.getHoney()
                .zipWith(butterService.getButter())
                .flatMap { (honey, butter) ->
                    heaterService.heatButterWithHoney(honey, butter)
                        .flatMap { heated ->
                            if (ovenReady) ovenService.bake(heated)
                            else Mono.just(heated)
                        }
                }
        }
}
```

### After: 코루틴으로 절차형 표현

```kotlin
fun bakeHoney(): Mono<Honey> = mono {
    val ovenReady = ovenService.heatOven(temperature).awaitFirstOrDefault(false)
    val honey = honeyService.getHoney().awaitFirst()
    val butter = butterService.getButter().awaitFirst()
    val heated = heaterService.heatButterWithHoney(honey, butter).awaitFirst()
    if (ovenReady) ovenService.bake(heated).awaitFirst() else heated
}
```

같은 로직이지만 코루틴 버전에서는 각 변수의 의미와 흐름이 한눈에 들어온다. null 처리도 Kotlin의 nullable 타입이 그대로 작동한다.

```kotlin
// Reactive: Optional 래핑 강요
fun checkAccess(ip: String): Mono<Boolean> {
    return Mono.zip(
        userRepo.findById(ip).map { Optional.of(it) }.defaultIfEmpty(Optional.empty()),
        blockRepo.findById(ip).map { Optional.of(it) }.defaultIfEmpty(Optional.empty())
    ).map { (profile, block) -> isAllowed(profile.orElse(null), block.orElse(null)) }
}

// 코루틴: nullable 타입으로 자연스럽게
suspend fun checkAccess(ip: String): Boolean {
    val userProfile = userRepo.findById(ip).awaitFirstOrNull()
    val block = blockRepo.findById(ip).awaitFirstOrNull()
    return isAllowed(userProfile, block)
}
```

---

## 성능: 리액티브와 코루틴은 동등하다

코루틴이 Reactor 위에서 동작하기 때문에 처리량과 지연 시간 모두 WebFlux 리액티브 방식과 통계적으로 동등하다. Allegro Tech의 Vegeta 부하 테스트 결과도 이를 뒷받침한다.

| 방식 | 처리량 | 응답 지연 |
|---|---|---|
| 블로킹 RestTemplate | 낮음 | 높음 |
| WebFlux + Reactor | 매우 높음 | 낮음 |
| WebFlux + 코루틴 | 매우 높음 | 낮음 (동등) |

코루틴 어댑터 레이어의 오버헤드는 무시할 수 있는 수준이다. 성능을 포기하지 않고 가독성을 얻는 셈이다.

---

## 컨텍스트 전파

분산 추적(Distributed Tracing)에서 `traceId`가 코루틴 경계를 넘어 올바르게 전파되려면 별도 설정이 필요하다.

### Hooks.enableAutomaticContextPropagation()

애플리케이션 시작 시 한 번만 호출하면 Reactor Context와 ThreadLocal 기반 컨텍스트(Micrometer, MDC 등)가 코루틴 실행 중에 자동으로 전파된다.

```kotlin
@SpringBootApplication
class Application {
    companion object {
        @JvmStatic
        fun main(args: Array<String>) {
            Hooks.enableAutomaticContextPropagation() // 반드시 애플리케이션 시작 시 호출
            runApplication<Application>(*args)
        }
    }
}
```

### PropagationContextElement 수동 설정

자동 전파 대신 특정 코루틴에만 적용하고 싶다면 `PropagationContextElement`를 명시적으로 주입한다.

```kotlin
suspend fun tracedOperation() {
    withContext(PropagationContextElement()) {
        // 이 블록 안에서 traceId가 MDC에 전파됨
        delay(100)
        logger.info("traceId 포함된 로그")
    }
}
```

의존성은 다음과 같다.

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor:1.8.0")
    implementation("io.micrometer:context-propagation") // 컨텍스트 전파 (선택)
}
```

---

## 정리

- `CoroutinesUtils.invokeSuspendingFunction()`이 `suspend` 함수를 `Mono`로 변환하는 공식 어댑터 진입점이다.
- `mono { }` 빌더와 `awaitSingle()` 계열 확장 함수가 Reactive ↔ Coroutine 세계를 양방향으로 연결한다.
- `coRouter` DSL과 `CoroutineCrudRepository`를 사용하면 비즈니스 코드에서 `Mono`/`Flux` 타입이 완전히 사라진다.
- 복잡한 `flatMap` 체인 대신 절차형 코루틴 코드로 가독성을 높이면서 성능은 동등하게 유지된다.
- `Hooks.enableAutomaticContextPropagation()`으로 분산 추적 컨텍스트를 코루틴 경계 너머로 자동 전파할 수 있다.

---

## 참고 자료

- [Spring Framework 공식 문서 - Kotlin Coroutines](https://docs.spring.io/spring-framework/reference/languages/kotlin/coroutines.html)
- [Spring Blog - Going Reactive with Spring, Coroutines and Kotlin Flow](https://spring.io/blog/2019/04/12/going-reactive-with-spring-coroutines-and-kotlin-flow/)
- [Foojay.io - Non-blocking with Spring WebFlux, Kotlin and Coroutines](https://foojay.io/today/build-and-test-non-blocking-web-applications-with-spring-webflux-kotlin-and-coroutines/)
- [Allegro Tech Blog - Making WebFlux code readable with Kotlin coroutines](https://blog.allegro.tech/2020/02/webflux-and-coroutines.html)
- [nexocode Blog - Coroutines vs Reactive Streams](https://nexocode.com/blog/posts/reactive-streams-vs-coroutines/)
