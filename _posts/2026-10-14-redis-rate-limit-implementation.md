---
layout: post
title: "rate limit 구현"
description: "Redis로 rate limit을 구현하는 네 가지 방식을 비교하고, 여러 명령을 이어 실행할 때 생기는 레이스 때문에 Lua 스크립트가 필요한 이유와 주의점을 정리한다."
date: 2026-10-14 11:34:30 +0900
categories: [Redis, Backend]
tags: [Redis, Rate Limit, Lua, Sliding Window, Token Bucket]
image:
  path: /assets/img/posts/redis-rate-limit-implementation/image-20261013-001520.png
  alt: Redis INCR 원자성과 Lua 스크립트
---

## 🙋‍♂️ Redis의 rate limit 구현을 알아보기 전에 다음 질문에 답해보자.

- 현재 값이 99인 키가 있다고 가정하고 아래와 같은 명령을 수행하면 최종 값은 뭐가 될까?
    - ![현재 값이 99인 키](/assets/img/posts/redis-rate-limit-implementation/image-20261013-004038.png)

      GET(A) → GET(B) → INCR(A) → INCR(B) 했을 때의 값?


## 📚 Redis의 rate limit 구현 알아보기

Redis로 rate limit을 구현하는 방식은 크게 네 가지가 있다.

- **Fixed Window**, **Sliding Window Log**, **Sliding Window Counter**, **Token Bucket/GCRA**.
- 무엇을 저장하고 어떤 원자 연산이 필요한지가 방식마다 다릅니다.

### 1. 비교 요약

| 방식 | 저장 구조 | 메모리/키 | 정확도 | 경계 버스트 | 적합한 곳 |
| --- | --- | --- | --- | --- | --- |
| Fixed Window | String + TTL | O(1) | 낮음 | 최대 2배 | 단순 쿼터, 일/월 단위 |
| Sliding Log | ZSET | O(limit) | 정확 | 없음 | 저 limit·보안성 (로그인, OTP) |
| Sliding Counter | 카운터 2개 (Hash) | O(1) | 근사 (오차 작음) | 사실상 없음 | 고트래픽 API 기본값 |
| Token Bucket / GCRA | Hash / 단일 값 | O(1) | 정확 (모델 기준) | capacity로 제어 | 버스트 허용이 필요한 경우 |

### 2. 구현 방식

<ol start="1"><li>Fixed Window</li></ol>

![Fixed Window 동작](/assets/img/posts/redis-rate-limit-implementation/image-20261013-010854.png)

- 위 동작처럼 INCR + EXPIRE NX를 수행하면 원자적으로 처리 가능
- 만약 GET 먼저 해서 판단을 하면  Lua가 필요하다

```
GET → 99 → 앱: 99 < 100 → INCR
          ↑ 이 사이에 다른 요청이 끼어듦
```

<ol start="2"><li>Sliding Log</li></ol>

![Sliding Log 동작](/assets/img/posts/redis-rate-limit-implementation/image-20261013-010938.png)

<ol start="3"><li>Sliding Counter</li></ol>

![Sliding Counter 동작](/assets/img/posts/redis-rate-limit-implementation/image-20261013-010959.png)

<ol start="4"><li>Token Bucket / GCRA</li></ol>

![Token Bucket / GCRA 동작](/assets/img/posts/redis-rate-limit-implementation/image-20261013-011016.png)


---

### 3. 왜 Lua가 필요한가

`INCR` 하나는 원래 원자적입니다. 문제는 **여러 명령을 이어서 실행할 때** 생깁니다.

![INCR 원자성과 여러 명령 조합의 레이스](/assets/img/posts/redis-rate-limit-implementation/image-20261013-001520.png)

Redis는 `INCR` 하나를 처리할 때 값 읽기, +1, 쓰기를 메인 스레드에서 쉬지 않고 한 번에 끝냅니다.

그 사이에 다른 클라이언트 명령이 들어갈 시점이 없어요.

| 시나리오 | 레이스 |
| --- | --- |
| `INCR` → `EXPIRE` | 두 명령 사이에 클라이언트가 죽으면 **TTL 없는 키**가 남아 영구 차단됩니다 |
| `GET`(확인) → `INCR`(실행) | 두 요청이 동시에 count=99를 읽고 둘 다 통과해서 limit을 넘깁니다 |
| `ZREMRANGEBYSCORE` → `ZCARD` → `ZADD` | 정리, 개수 확인, 추가 사이에 다른 요청이 끼어듭니다 |

**Lua가 원자성을 보장하는 이유:** Redis는 명령을 단일 스레드 이벤트 루프에서 실행합니다. `EVAL`/`EVALSHA`로 들어온 스크립트는 실행이 끝날 때까지 다른 클라이언트 명령이 끼어들 수 없습니다.


**✔️ 주의할 점:**

- **롤백은 없습니다.** 스크립트가 중간에 에러로 끝나도 그 전에 실행된 쓰기는 남습니다. MULTI와 같은 "격리"는 되지만 "트랜잭션 롤백"은 안 됩니다.
- **긴 스크립트는 Redis 전체를 막습니다.** `busy-reply-threshold`(구 `lua-time-limit`, 기본 5초)를 넘기면 다른 클라이언트가 `BUSY` 응답을 받습니다. 이미 쓰기를 한 스크립트는 `SCRIPT KILL`도 안 됩니다.
- **Cluster에서는 스크립트가 쓰는 키를 전부 `KEYS[]`로 넘겨야 하고, 같은 slot에 있어야 합니다.** 그래서 `rl:{user123}:curr`, `rl:{user123}:prev`처럼 **hash tag**로 묶습니다.
- **스크립트 안의 `redis.call('TIME')`:** Redis 5부터는 effects replication(스크립트 대신 결과 명령을 복제)이 기본이고 7부터는 그것만 남았습니다. 그래서 `TIME` 이후에 쓰기를 해도 안전합니다. 서버 시계를 기준으로 쓰면 앱 서버 간 clock skew 문제가 사라집니다.
- **Lua number는 Redis로 반환될 때 정수로 잘립니다.** 소수(가중치, 토큰 수)를 반환하려면 `tostring()`으로 감싸야 합니다.
- **Spring Data Redis의 `DefaultRedisScript`**는 `EVALSHA`를 먼저 시도하고, `NOSCRIPT`가 오면 `EVAL`로 자동 fallback합니다. 그래서 스크립트 본문을 매번 보내지 않습니다.
