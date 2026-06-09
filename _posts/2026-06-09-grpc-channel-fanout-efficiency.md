---
layout: post
title: "gRPC Fan-Out 패턴과 채널 효율성: 단일 요청이 다수 요청으로 퍼질 때 gRPC는 어떻게 버티나"
description: "gRPC 클라이언트가 다수의 하위 서버를 동시에 호출하는 fan-out 시나리오에서, HTTP/2 멀티플렉싱 구조와 7가지 패턴 및 채널 풀링 전략을 통해 고가용성을 달성하는 방법을 정리한다."
date: 2026-06-09 09:00:00 +0900
categories: [gRPC, distributed-systems]
tags: [grpc, fanout, http2, channel, multiplexing, microservices]
image:
  path: https://grpc.io/img/logos/grpc-icon-color.png
  alt: gRPC Logo
---

## 개요

마이크로서비스 아키텍처에서 단일 클라이언트 요청이 여러 하위 gRPC 서버를 동시에 호출하는 **fan-out** 현상은 흔히 발생한다. 예를 들어 결제 서비스가 한 번의 요청을 처리하기 위해 사기 탐지, 재고 확인, 알림 서비스를 동시에 호출해야 할 수 있다. 이 포스트에서는 gRPC가 HTTP/2 멀티플렉싱을 통해 이 fan-out을 효율적으로 처리하는 원리와, 실전에서 검증된 7가지 패턴 및 채널 관리 전략을 다룬다.

---

## HTTP/2 멀티플렉싱과 gRPC 채널 구조

gRPC는 전송 계층으로 HTTP/2를 사용한다. HTTP/2의 핵심은 **단일 TCP 연결 위에서 여러 요청을 동시에 처리하는 멀티플렉싱**이다. gRPC는 이를 세 계층의 개념으로 추상화한다.

| 계층 | gRPC 개념 | HTTP/2 매핑 |
|------|-----------|-------------|
| 1계층 | **Channel** | 하나 이상의 HTTP/2 커넥션 |
| 2계층 | **RPC** | HTTP/2 Stream |
| 3계층 | **Message** | HTTP/2 Data Frame |

공식 문서에서 명시하듯, "each channel may have many RPCs while each RPC may have many messages." 즉 하나의 채널이 수백 개의 동시 RPC를 처리할 수 있고, 큰 메시지는 여러 데이터 프레임으로 분할되어 전송된다.

```
Client
  └── Channel (HTTP/2 Connection Pool)
        ├── RPC 1 (Stream #1) → Service A
        ├── RPC 2 (Stream #3) → Service B
        ├── RPC 3 (Stream #5) → Service C
        └── RPC N (Stream #N) → Service N
```

**채널을 반드시 재사용해야 하는 이유는 여기 있다.** 새 채널을 매 호출마다 생성하면 소켓 오픈 → TCP 연결 → TLS 협상 → HTTP/2 핸드셰이크의 여러 라운드트립 비용이 발생한다. 채널을 재사용하면 이 모든 비용이 첫 번째 연결에서만 발생하고, 이후 모든 RPC는 이미 수립된 연결 위에서 멀티플렉싱된다.

---

## Fan-Out의 7가지 패턴

fan-out이 필요한 서비스에서 무작정 병렬 호출을 보내면 타임아웃 누적, 재시도 폭풍, 중복 처리 등 심각한 문제가 발생한다. 다음 7가지 패턴은 이를 방지하기 위한 실전 접근법이다.

### 패턴 1: Bounded Scatter-Gather (세마포어 기반 동시성 제한)

**문제:** fan-out 수가 무제한이면 하위 서비스 전체를 압도한다.

**해법:** 세마포어로 최대 동시 in-flight 요청 수를 제한하고, 데드라인 안에 응답한 결과만 부분 반환한다.

```csharp
var semaphore = new SemaphoreSlim(8); // 최대 동시 8개
var tasks = shardEndpoints.Select(async endpoint =>
{
    await semaphore.WaitAsync(cancellationToken);
    try
    {
        return await client.QueryAsync(request, deadline: DateTime.UtcNow.AddMilliseconds(200));
    }
    catch (RpcException ex) when (ex.StatusCode == StatusCode.DeadlineExceeded)
    {
        return PartialResult.Timeout(endpoint);
    }
    finally
    {
        semaphore.Release();
    }
});

var results = await Task.WhenAll(tasks);
return MergeResults(results); // 응답한 결과만 집계
```

### 패턴 2: Bidirectional Streaming with Flow Control (백프레셔)

**문제:** 클라이언트가 서버 처리 속도보다 빠르게 요청을 보내면 버퍼가 넘친다.

**해법:** gRPC 양방향 스트리밍의 내장 flow control을 활용한다. 클라이언트는 서버가 전송한 크레딧 메시지를 받은 만큼만 요청을 보낸다.

```csharp
using var stream = client.ProcessStream();

// 서버로부터 credit 수신 후 그만큼만 전송
await foreach (var credit in stream.ResponseStream.ReadAllAsync())
{
    for (int i = 0; i < credit.AllowedRequests; i++)
    {
        await stream.RequestStream.WriteAsync(new WorkItem { Id = NextId() });
    }
}
```

### 패턴 3: Server-Streaming Fan-In (샤드 집계)

**문제:** 여러 샤드에서 데이터를 모아야 하는데, 모두 기다리면 메모리가 부족하다.

**해법:** 각 샤드에서 서버 스트리밍 RPC로 데이터를 받아 인크리멘털하게 머지한다. time-to-first-byte가 개선되고 메모리 사용량이 일정하게 유지된다.

```csharp
var streams = shards.Select(shard =>
    shard.Client.StreamResults(query).ResponseStream.ReadAllAsync());

await foreach (var item in MergeAsyncEnumerables(streams))
{
    yield return item; // 받는 즉시 downstream으로 흘려보냄
}
```

### 패턴 4: Hedged Requests with Quorum (꼬리 지연 완화)

**문제:** p99 레이턴시가 높아 SLA를 맞추기 어렵다.

**해법:** 일정 시간(예: 80ms) 내에 응답이 없으면 다른 서버에 동일 요청을 중복 발송한다. 먼저 도착한 쿼럼(예: 3개 중 2개) 응답을 사용하고 나머지는 취소한다.

```csharp
using var cts = new CancellationTokenSource();
var replicas = new[] { replicaA, replicaB, replicaC };

var primaryTask = replicas[0].ReadAsync(request, cts.Token);
await Task.Delay(80); // soft threshold

if (!primaryTask.IsCompleted)
{
    // hedge: 나머지 복제본에도 요청 발송
    var hedgeTasks = replicas[1..].Select(r => r.ReadAsync(request, cts.Token));
    var quorumResult = await FirstTwoAgree(primaryTask, hedgeTasks);
    cts.Cancel(); // 나머지 취소
    return quorumResult;
}
return await primaryTask;
```

### 패턴 5: Request Coalescing & Idempotency Keys (중복 제거)

**문제:** 동시에 동일한 데이터를 요청하는 여러 호출자가 있어 하위 서비스에 불필요한 부하가 생긴다.

**해법:** 클라이언트 사이드에서 동일 요청을 하나의 RPC로 합치고, 서버 사이드에서는 멱등성 키로 TTL 윈도우 안의 중복 요청을 드롭한다.

```csharp
// 클라이언트: 동일 키에 대한 요청을 하나의 Task로 합침
var result = await requestCoalescer.GetOrAddAsync(
    key: $"user:{userId}:profile",
    factory: () => userClient.GetProfileAsync(new GetProfileRequest { UserId = userId }),
    ttl: TimeSpan.FromMilliseconds(50)
);
```

### 패턴 6: Async Fan-Out via Outbox Pattern (비동기 분산)

**문제:** 동기 fan-out이 너무 많아 응답 시간이 길어진다.

**해법:** 처리 의도를 DB 아웃박스 테이블에 트랜잭션으로 기록하고, 이벤트를 발행해 워커들이 비동기로 처리하도록 위임한다. 클라이언트는 task ID를 받아 나중에 폴링한다.

```csharp
// 1. 트랜잭션 안에서 outbox에 기록
await using var tx = await db.BeginTransactionAsync();
await db.OutboxEvents.AddAsync(new OutboxEvent { Payload = request });
await db.SaveChangesAsync();
await tx.CommitAsync();

// 2. 클라이언트에게 task ID 반환 (즉시 응답)
return new FanOutResponse { TaskId = taskId };
```

### 패턴 7: Retry Budgets & Circuit Breakers (재시도 예산)

**문제:** 재시도 폭풍이 장애를 더 악화시킨다.

**해법:** 재시도 볼륨을 성공 호출의 일정 비율(예: 10%)로 제한하고, 지속적 장애 시 서킷 브레이커를 연다. per-RPC 데드라인 전파로 하위 서비스는 불필요한 작업을 스스로 취소한다.

```csharp
var policy = Policy
    .WrapAsync(
        Policy.Handle<RpcException>()
              .CircuitBreakerAsync(exceptionsAllowedBeforeBreaking: 5,
                                  durationOfBreak: TimeSpan.FromSeconds(30)),
        Policy.Handle<RpcException>(e => e.StatusCode == StatusCode.Unavailable)
              .RetryAsync(retryCount: 1) // 최대 1회 재시도
    );
```

> **결제 시스템 적용 사례:** 패턴 1(Bounded Scatter-Gather)과 패턴 4(Hedged Requests)를 결합 적용한 결과, p99 레이턴시가 약 900ms에서 420ms로 절반 이상 감소했다.

---

## 동시 스트림 한계와 채널 풀링 전략

HTTP/2 커넥션 하나에서 처리할 수 있는 동시 스트림 수는 기본적으로 **100개**로 제한된다. 100개를 초과하는 RPC는 클라이언트 큐에 대기하게 되어 성능 저하가 발생한다.

### 해결 방법 1: `EnableMultipleHttp2Connections` (.NET 5+)

```csharp
var channel = GrpcChannel.ForAddress("https://grpc-server:5001", new GrpcChannelOptions
{
    HttpHandler = new SocketsHttpHandler
    {
        EnableMultipleHttp2Connections = true, // 스트림 한계 도달 시 새 연결 자동 생성
        PooledConnectionIdleTimeout = Timeout.InfiniteTimeSpan,
    }
});
```

### 해결 방법 2: 채널 풀링

고부하 영역에는 복수의 채널을 미리 생성해 랜덤하게 분산한다.

```csharp
public class GrpcChannelPool
{
    private readonly GrpcChannel[] _channels;
    private readonly Random _random = new();

    public GrpcChannelPool(string address, int poolSize = 4)
    {
        _channels = Enumerable.Range(0, poolSize)
            .Select(i => GrpcChannel.ForAddress(address, new GrpcChannelOptions
            {
                // 채널마다 고유 args로 구분 (내부 캐시 히트 방지)
                ServiceConfig = new ServiceConfig { ChannelCredentials = ChannelCredentials.Insecure }
            }))
            .ToArray();
    }

    public GrpcChannel Get() => _channels[_random.Next(_channels.Length)];
}
```

> 주의: 단일 HTTP/2 연결에서 스트림 한계를 인위적으로 높이는 것은 오히려 역효과다. 스트림 간 쓰기 경합과 TCP 패킷 손실 시 전체 연결 블로킹이 발생한다.

---

## 실전 성능 튜닝

### Keepalive Ping 설정

유휴 연결이 프록시나 방화벽에 의해 끊기는 것을 방지한다. gRPC는 HTTP/2 PING 프레임을 주기적으로 전송해 연결 생존 여부를 확인하고 프록시 타임아웃을 예방한다.

```csharp
var handler = new SocketsHttpHandler
{
    PooledConnectionIdleTimeout = Timeout.InfiniteTimeSpan,
    KeepAlivePingDelay = TimeSpan.FromSeconds(60),   // 60초마다 PING
    KeepAlivePingTimeout = TimeSpan.FromSeconds(30), // 30초 내 응답 없으면 연결 종료
    EnableMultipleHttp2Connections = true
};
```

### Flow Control 윈도우 크기 조정

대용량 스트리밍 시나리오에서 flow control 버퍼가 가득 차면 전송이 멈춘다. 서버 측에서 윈도우 크기를 늘려 throughput을 확보한다.

```csharp
// Kestrel 서버 설정
builder.WebHost.ConfigureKestrel(options =>
{
    var http2 = options.Limits.Http2;
    http2.InitialConnectionWindowSize = 1024 * 1024 * 2; // 2 MB (커넥션 레벨)
    http2.InitialStreamWindowSize = 1024 * 1024;          // 1 MB (스트림 레벨)
});
```

### 스트리밍 vs 단방향(Unary) 트레이드오프

| 항목 | Unary RPC | Bidirectional Streaming |
|------|-----------|------------------------|
| 연결 오버헤드 | 매 RPC마다 새 HTTP/2 요청 | 최초 스트림 수립 후 재사용 |
| 클라이언트 로드밸런싱 | 매 호출마다 가능 | 스트림 활성 중에는 고정 |
| 디버깅 복잡도 | 낮음 | 높음 |
| 고처리량 시나리오 | 오버헤드 누적 | 유리 |

fan-out 요청 수가 많고 각 요청이 짧다면 **단방향 RPC + 채널 풀링** 조합이 단순하면서도 효율적이다. 반면 지속적인 스트림 처리(로그 집계, 실시간 검색 등)에는 양방향 스트리밍이 유리하다.

---

## 정리

- **채널은 재사용하라.** 매 호출마다 새 채널을 생성하면 TCP/TLS 핸드셰이크 비용이 반복 발생한다.
- **동시 스트림 한계(기본 100개)를 인지하라.** 고부하 시나리오에서는 `EnableMultipleHttp2Connections` 또는 채널 풀링으로 병목을 해소한다.
- **fan-out에는 반드시 상한을 걸어라.** Bounded Scatter-Gather로 동시 in-flight 수를 제한하지 않으면 하위 서비스 전체가 압도된다.
- **꼬리 지연은 Hedged Requests로 완화하라.** p99를 목표로 할 때 단일 서버의 응답 지연을 백업 호출로 보완한다.
- **Retry Budgets로 재시도 폭풍을 막아라.** 성공 호출 대비 10% 이하로 재시도를 제한하고 서킷 브레이커를 병행 적용한다.

---

## 참고 자료

- [7 gRPC Fan-Out Patterns (No Fan-Fires)](https://medium.com/@bhagyarana80/7-grpc-fan-out-patterns-no-fan-fires-23144b78b6f1)
- [Performance best practices with gRPC - Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/grpc/performance?view=aspnetcore-10.0)
- [gRPC on HTTP/2: Engineering a Robust, High-performance Protocol - gRPC Blog](https://grpc.io/blog/grpc-on-http2/)
- [Performance Best Practices - gRPC Official Docs](https://grpc.io/docs/guides/performance/)
- [gRPC on HTTP/2 - CNCF Blog](https://www.cncf.io/blog/2018/08/31/grpc-on-http-2-engineering-a-robust-high-performance-protocol/)
