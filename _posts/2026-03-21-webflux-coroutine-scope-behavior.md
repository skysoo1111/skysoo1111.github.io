---
layout: post
title: "Webflux + Coroutine 에서의 Scope 동작"
date: 2026-03-21 00:00:00 +0900
categories: [Kotlin]
tags: [kotlin, coroutine, webflux, spring, reactor, netty]
---

> 원문: [https://velog.io/@skysoo/Webflux-Coroutine-%EC%97%90%EC%84%9C%EC%9D%98-Scope-%EB%8F%99%EC%9E%91](https://velog.io/@skysoo/Webflux-Coroutine-%EC%97%90%EC%84%9C%EC%9D%98-Scope-%EB%8F%99%EC%9E%91)

# coroutine scope
coroutine scope란 위에서 언급했 듯이 coroutine context를 포함한 coroutine을 실행하는 범위를 뜻한다. 

## 의문점: Webflux + kotlin coroutine 같이 사용하면 coroutine scope는 어떻게 될까?
springboot + kotlin 프로젝트로 구성도 많이하고 webflux + coroutine은 자연스런 구성일 것이다.
근데 개발하면서 coroutine scope를 별도로 지정해서 사용하지 않았는데, 위 구성시에는 내부적으로 어떻게 구성될까?
- Webflux는 스레드풀이 아니라 NIO 이벤트 루프 기반 동작인데, kotlin은 기본적으로 스레드풀 위에서 동작한다. 
- 조금 더 깊게 들어가면 둘 다 스레드를 효율적으로 사용하기 위함이지만 webflux는 java.nio를 직접 다루고 kotlin은 java.io에 의존한다. 

그럼 다시 본론으로 돌아가서 webflux + coroutine을 같이 사용하면 coroutine scope는 어떻게 되나?
- Spring이 요청 단위로 CoroutineScope를 제공하고 Dispatcher는 Reactor 이벤트 루프 전용 Dispatcher를 사용한다.
- 즉 kotlin의 전용 Dispatcher가 아닌 Reactor의 Scheduler가 만든 전용 Dispatcher 위에서 동작한다. 
- 이유는 위에서 언급한 바와 같이 kotlin은 스레드풀 위에서 동작하고 Reactor는 netty 이벤트 루프를 이용하므로 그 근간부터 다르다.
> val reactorDispatcher = Schedulers.parallel().asCoroutineDispatcher()

만약 reactor를 kotlin의 Dispatcher를 사용한다면 오히려 컨텍스트 단절이 생기면서 비효율적일 것이다.

| Dispatcher                                   | 소속                 | 스레드 종류       | 언제 쓰나              | 예시 Thread 이름                 |
| -------------------------------------------- | ------------------ | ------------ | ------------------ | ---------------------------- |
| `Dispatchers.Default`                        | Kotlin 표준          | CPU-bound 풀  | CPU 연산             | `DefaultDispatcher-worker-1` |
| `Dispatchers.IO`                             | Kotlin 표준          | I/O용 풀 (확장형) | 블로킹 I/O            | `DefaultDispatcher-worker-5` |
| **Reactor Dispatcher** *(Spring WebFlux 내부)* | Reactor (Netty 기반) | 이벤트 루프       | Non-blocking 요청 처리 | `reactor-http-nio-2`         |

## 내부 동작
[HTTP 요청] → Reactor pipeline 시작
   ↓
Reactor Context -> CoroutineContext (bridge)
   ↓
suspend 함수 실행 (실제 스레드는 reactor-http-nio-#)
   ↓
suspend -> resume 시점도 동일 이벤트 루프에서 스케줄됨

## coroutine suspend 시점에 Netty Scheduler 동작
suspend(함수 일시중단)의 기능을 kotlin은 coroutination이 reactor는 scheduler가 대신한다.
| 시점             | 이벤트 루프 스레드 상태          | 코루틴 상태      |
| -------------- | ---------------------- | ----------- |
| `suspend` 진입 전 | 코루틴 실행 중               | Active      |
| `suspend` 시    | subscribe 등록 → 스레드 반환  | Suspended   |
| I/O 완료 이벤트 수신  | Reactor가 resume 요청 스케줄 | Resuming    |
| resume 이후      | 다시 같은 Reactor 스레드에서 실행 | Active (재개) |


# 결론
webflux + coroutine을 사용하여 내부적으로 suspend 함수를 사용하더라도 CoroutineDispatcher를 감싼 형태의 Reactor 전용 dispatcher를 사용하고 그 근간은 netty의 이벤트 루프를 사용하여 동작된다. 

