---
layout: post
title: "실시간 중계 서비스에서 SSE를 택한 이유와 Kafka·Redis Fan-Out 설계"
description: "스포츠 중계처럼 서버에서 다수 사용자로 이벤트를 push해야 할 때, gRPC 대신 SSE를 선택해야 하는 근거와 Kafka·Redis를 조합한 fan-out 설계 트레이드오프를 정리한다."
date: 2026-06-09 16:43:28 +0900
categories: [SSE, distributed-systems]
tags: [SSE, Kafka, Redis, fanout, real-time, architecture]
image:
  path: https://source.unsplash.com/featured/?realtime,streaming
  alt: 실시간 스트리밍 아키텍처
---

## 개요

실시간 스포츠 중계처럼 "서버에서 발생한 이벤트를 다수의 사용자에게 계속 흘려보내야 하는" 서비스를 설계할 때, 전송 방식으로 무엇을 골라야 할까. 이 글에서는 gRPC server streaming과 SSE(Server-Sent Events)를 비교하며 **최종 사용자 대면 실시간 push에는 왜 SSE가 더 합리적인지**, 그리고 그 위에서 다수 서버 인스턴스로 이벤트를 fan-out 할 때 **Kafka 직접 구독 방식과 Redis pub/sub 방식의 장단점**을 정리한다.

## gRPC server streaming은 왜 사용자 대면엔 부담인가

gRPC server streaming은 클라이언트가 한 번만 호출하면 서버가 시간차로 메시지를 계속 흘려보내는 구조다. 백엔드 내부(MSA 간) 실시간 이벤트 구독에는 타입 안전성과 성능 면에서 훌륭하지만, 최종 사용자(브라우저/앱) 대면 서비스에는 다음 제약이 크다.

- **장시간 연결 유지 비용**: 경기 한 판이 2시간이면 stream을 2시간 열어둬야 한다. 동시 시청자 10만 명이면 stream 10만 개를 서버가 동시에 붙들고 있어야 한다.
- **연결 끊김·재연결**: 모바일 네트워크가 끊기면 stream도 끊긴다. "어디까지 받았는지"를 클라이언트가 기억했다가 재호출 시 그 지점부터 다시 받도록 직접 설계해야 한다. gRPC 자체는 자동 resume을 제공하지 않는다.
- **브라우저 직접 호출 불가**: 웹 브라우저는 순수 gRPC를 직접 쓰지 못하고 gRPC-Web과 Envoy 같은 프록시 계층이 필요하다.

## SSE가 실시간 중계에 잘 맞는 이유

SSE는 HTTP 위에서 서버가 클라이언트로 이벤트를 단방향 push 하는 표준 기술이다. 중계처럼 "서버 → 클라" 한 방향 흐름과 정확히 일치한다.

- **브라우저 네이티브**: `EventSource` 객체 하나로 끝. gRPC-Web 같은 별도 프록시 계층이 필요 없다.
- **자동 재연결**: 연결이 끊기면 브라우저가 알아서 다시 붙고, `Last-Event-ID` 헤더로 "어디까지 받았는지"를 표준으로 지원한다. 중계의 "끊긴 지점부터 재개"가 거의 공짜로 해결된다.
- **기존 HTTP 인프라 호환**: 로드밸런서, 쿠키/헤더 인증, CDN, 방화벽과 자연스럽게 어울린다.

### 클라이언트 예시

```javascript
const source = new EventSource("/matches/123/events");

source.addEventListener("score", (e) => {
    const event = JSON.parse(e.data);
    render(event); // "후반 89분 골!"
});

// 연결이 끊겨도 브라우저가 자동 재연결하며
// 마지막 수신한 id를 Last-Event-ID 헤더로 보낸다.
source.onerror = () => console.log("reconnecting...");
```

### SSE의 약점도 알고 쓰자

- **단방향만 가능**: 클라이언트 → 서버 전송이 필요하면 별도 POST API를 둬야 한다. 양방향이 필요하면 WebSocket이 답이다.
- **텍스트(UTF-8) 기반**: 바이너리는 base64 인코딩이 필요하다. 중계 메시지는 보통 JSON이라 문제되지 않는다.
- **HTTP/1.1의 브라우저당 연결 수 제한(6개)**: 단, **HTTP/2를 쓰면 멀티플렉싱되어 사실상 해소**된다.

## 핵심: 전송과 메시지 분배는 별개 레이어다

SSE를 골랐어도 "여러 서버 인스턴스에서 동일 이벤트를 모든 시청자에게 fan-out 하는" 문제는 그대로 남는다. **전송 프로토콜(SSE)** 과 **메시지 분배(브로커)** 는 별개 레이어로 보고 설계해야 한다.

```
중계 입력 → Kafka / Redis → 각 API 서버가 구독
                              → 자기에게 붙은 SSE 클라들에게 push
```

이 분배 레이어를 어떻게 구성하느냐에서 두 가지 대표 방식이 갈린다.

## 방식 A — 각 host가 Kafka를 직접 구독 (host별 consumer group 분리)

같은 consumer group이면 파티션이 **분배(로드밸런싱)** 되므로, 모든 host가 **모든 메시지**를 받으려면 host마다 group id를 다르게 주어 각각이 독립 broadcast가 되게 한다.

```
              ┌→ host1 (group=host1) → 자기 SSE 클라들
Kafka topic ──┼→ host2 (group=host2) → 자기 SSE 클라들
              └→ host3 (group=host3) → 자기 SSE 클라들
```

**장점**

- 구조가 단순하고 Redis 같은 중간 계층이 필요 없다.
- Kafka 내구성을 그대로 활용한다. host가 재시작해도 offset부터 다시 읽어 **놓친 메시지 복구**가 가능하고, SSE의 `Last-Event-ID` 재개와 궁합이 좋다.
- 메시지 유실이 거의 없다(at-least-once).

**단점**

- **consumer group이 host 수만큼 무한 증식**한다. `__consumer_offsets`에 메타데이터가 쌓이고, 오토스케일로 host가 뜨고 죽을 때마다 유령 group offset이 남는다.
- host가 동적으로 변하면 rebalancing과 브로커 커넥션 부담이 커진다(커넥션 수 = host 수).
- 모든 host가 전체 파티션을 다 읽으므로 같은 메시지를 N번 끌어와 대역폭을 낭비한다.
- 쿠버네티스 pod처럼 ephemeral host와 상극이다.

> host 수가 적고 고정적이면 단순해서 좋지만, 오토스케일 환경에선 빠르게 무너진다.

## 방식 B — 단일 consumer → Redis pub/sub → 각 host 구독

Kafka 소비는 단일 consumer group이 한 번만 하고, 받은 메시지를 Redis로 publish 하면 각 host가 Redis를 구독한다.

```
Kafka topic ──→ [단일 consumer group] ──→ Redis PUBLISH
                                              │
                            ┌─────────────────┼─────────────────┐
                          host1 SUB        host2 SUB         host3 SUB
                            → SSE 클라        → SSE 클라         → SSE 클라
```

**장점**

- Kafka 입장에서 group이 1개로 고정되어 브로커 부담이 최소다.
- host는 Redis에 가볍게 SUB만 했다 떨어지므로 **오토스케일링에 강하다**. offset·group 관리가 없어 pod가 막 뜨고 죽어도 Kafka 구조에 영향이 없다.
- 관심사가 분리된다. Kafka는 "내구성 있는 이벤트 소스", Redis는 "실시간 fan-out 채널" 역할을 맡는다.

**단점**

- Redis pub/sub은 **at-most-once, fire-and-forget**이라 그 순간 SUB 안 돼 있으면 메시지를 놓친다. 버퍼·재전송·replay가 없다.
- 단일 consumer와 Redis가 SPOF가 된다. 단일 consumer를 HA로 이중화하면 이번엔 중복 발행 문제가 생긴다.
- 홉이 하나 늘어 latency와 운영 인프라가 증가한다.
- 끊겼던 클라가 재접속해도 Redis 단엔 과거 메시지가 없어 복구가 불가하다.

## 두 방식의 트레이드오프 한눈에

| 항목 | A. host별 Kafka 직접 구독 | B. Redis pub/sub fan-out |
|------|--------------------------|-----------------------------|
| Kafka 부담 | host 수만큼 group/커넥션 증가 | group 1개로 고정 |
| 오토스케일링 | 취약 (group 생성·소멸 반복) | 자유로움 |
| 메시지 내구성 | 강함 (offset replay 가능) | 약함 (유실 가능) |
| 중간 컴포넌트 | 없음 (단순) | consumer + Redis 추가 |
| Latency | 1홉 | 2홉 (Kafka→Redis) |
| SPOF | 분산되어 적음 | Redis·단일 consumer가 위험 |

> **A = 내구성↑ / 확장성↓**, **B = 확장성↑ / 내구성↓**

## 절충안 — Redis Streams

B의 유실 단점이 걸린다면 pub/sub 대신 **Redis Streams**를 쓰는 변형이 자주 쓰인다.

```
Kafka ──→ 단일 consumer ──→ Redis Streams (XADD)
                                  │
              각 host가 XREAD / consumer group으로 소비
```

```bash
# 발행: 이벤트를 스트림에 추가 (MAXLEN으로 메모리 관리)
XADD match:123 MAXLEN ~ 1000 * type score player "손흥민" minute 89

# 소비: 끊겼던 클라는 마지막 ID 이후부터 다시 읽어 replay
XREAD COUNT 100 STREAMS match:123 1718000000000-0
```

pub/sub과 달리 메시지가 일정 기간 보관되어 재접속 시 마지막 ID부터 **replay**가 가능하다. 이는 SSE `Last-Event-ID`와 직결되어, B의 확장성과 A의 내구성을 어느 정도 동시에 확보한다. 대신 Redis 메모리 관리(MAXLEN trim)가 필요하다.

## 정리

- 최종 사용자 대면 실시간 push는 **SSE**가 합리적이다(브라우저 네이티브, 자동 재연결, HTTP 인프라 호환). 백엔드 내부 통신은 gRPC streaming이 여전히 강력하다.
- **전송(SSE)과 메시지 분배(브로커)는 별개 레이어**로 설계한다.
- host 수가 적고 고정이며 유실이 절대 불가하면 **방식 A(host별 Kafka 직접 구독)**.
- 시청자 대규모 + 오토스케일 + 약간의 유실 허용이면 **방식 B(Redis pub/sub)**.
- 대규모인데 재접속 복구까지 챙기려면 **Redis Streams 기반 절충안**이 가장 균형이 좋다.

## 참고 자료

- [MDN — Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [Redis — Pub/Sub](https://redis.io/docs/latest/develop/interact/pubsub/)
- [Redis — Streams](https://redis.io/docs/latest/develop/data-types/streams/)
- [Apache Kafka — Consumer Group](https://kafka.apache.org/documentation/#intro_consumers)
