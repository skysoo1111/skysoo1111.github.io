---
layout: post
title: "테스트 전략: 계약 테스트에 대하여"
description: "MSA 환경에서 서비스 간 API 호환성을 계약 테스트로 검증하는 원리와 소비자 주도 계약(CDC), Pact 활용법을 정리한다."
date: 2026-07-09 16:32:05 +0900
categories: [Testing, MSA]
tags: [Kotlin, Pact, Contract Test, MSA, Testing]
image:
  path: https://source.unsplash.com/featured/?testing,contract,api
  alt: 계약 테스트 개념 이미지
---

> 원문: [테스트 전략: 계약 테스트에 대하여](https://velog.io/@skysoo1111/%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%A0%84%EB%9E%B5-%EA%B3%84%EC%95%BD-%ED%85%8C%EC%8A%A4%ED%8A%B8%EC%97%90-%EB%8C%80%ED%95%98%EC%97%AC)

## 🔸테스트 전략

하지만 실질적으로는 테스트를 작성하는데 있어 시간과 리소스의 한계가 있으므로 별다른 소독이 없는 테스트를 작성하느라 시간을 허비하고 싶지는 않다. … 테스트의 범위와 다양한 종류의 테스트 사이의 균형을 영리하게 결정해야 한다. … 상황에 맞춰 올바른 테스트를 진행할 수 있도록 돕기 위해 **테스트 사분면**과 **테스트 피라미드**를 소개하고자 한다.

*   **테스트 사분면**
    *   "무엇을/왜 테스트하나" (목적의 분류) - 테스트를 두 축으로 4분면에 나눈다.
        *   가로축: 비즈니스 대면(business-facing) ↔ 기술 대면(technology-facing)
        *   세로축: 팀 지원(supporting the team) ↔ 제품 비판(critiquing the product)

*   **테스트 피라미드**

    *   "얼마나 많이" (계층별 분량) - 테스트를 계층으로 쌓되, 아래로 갈수록 많이, 위로 갈수록 적게 두라는 원칙이다.

        *   하단 (단위 테스트): 가장 많이. 빠르고(밀리초), 격리돼 있고, 안정적. 로직 검증의 주력.

        *   중간 (통합/컴포넌트 테스트): 적당히. 실제 의존성(DB 등)과의 연동 검증. 느리지만 신뢰도 높음.

        *   상단 (E2E 테스트): 최소한. 전체 시스템 관통. 신뢰도는 최고지만 느리고 깨지기 쉽고 유지비가 큼.

> "테스트는 많이 짤수록 좋은건 아니다. 테스트마다 작성·유지·실행 비용이 들고, 느리고 깨지기 쉬운 테스트를 잘못된 곳에 많이 두면 오히려 개발을 방해한다. 그래서 '어디에 어떤 테스트를 얼마나 둘 것인가'를 먼저 정하는 게 전략이고, 책은 이를 위한 두 도구 — 테스트 사분면과 테스트 피라미드 — 를 제시하고 이 둘은 경쟁 관계가 아니라 서로 다른 질문에 답하는 상호 보완적 틀이다."

## 🔸계약 테스트

MSA에서 가장 무서운 건 "내 서비스는 안 건드렸는데 다른 서비스 때문에 깨지는" 상황이다.

    contents-service가 응답 필드명을 title → contentTitle 로 바꿈
            ↓
    그걸 쓰던 BFF가 title을 못 찾아 런타임에 깨짐
            ↓
    그런데 이걸 배포하고 나서야 알게 됨

이걸 잡으려는 기존 방법들엔 한계가 있다:

*   단위 테스트: 각 서비스는 자기 안만 봄. 서비스 사이의 불일치는 못 잡음.
*   E2E 테스트: 잡긴 하는데 느리고, 모든 서비스를 다 띄워야 하고, 깨지기 쉬움. 조합이 폭발.

계약 테스트는 이 사이를 메웁니다. E2E처럼 전체를 안 띄우면서도, 서비스 간 인터페이스 호환성을 빠르게 검증한다.

소비자(Consumer)와 제공자(Provider)가 주고받는 API의 형식에 대한 합의를 코드로 명시한 것이다.

*   제공자(Provider): API를 제공하는 쪽 (예: contents-service)
*   소비자(Consumer): 그 API를 호출하는 쪽 (예: BFF)
*   계약(Contract): "소비자는 이런 요청을 보내고, 제공자는 이런 형식으로 응답한다"는 약속

계약 테스트는 이 약속을 양쪽이 각각 지키는지 독립적으로 검증한다. 양쪽을 동시에 띄워서 실제 통신시키는 게 아니라, 계약 문서를 매개로 따로따로 확인하는 게 핵심이다.

*   **계약 테스트 프레임워크**
    *   Pact
    *   Spring Cloud Contracts

### 소비자 주도 계약 (Consumer-Driven Contract, CDC)

책이 강조하는 방식으로 계약을 소비자가 먼저 정의한다.

왜 소비자 주도를 강조하는가:

*   제공자가 "내가 주는 응답 전부"를 계약으로 삼으면, 소비자가 실제로 안 쓰는 필드까지 계약에 묶입니다. 그 필드를 바꾸면 아무도 안 쓰는데도 계약 위반이 됨.
*   소비자가 "내가 실제로 쓰는 것만" 계약으로 선언하면, 제공자는 그것만 지키면 나머지는 자유롭게 바꿀 수 있다.

    Client(소비자): "나는 contents의 응답에서 code, title, label만 쓴다"
            ↓ (이게 계약)
    contents(제공자): "그 세 필드는 항상 이 형식으로 준다" 를 검증
            → contents가 다른 필드(내부용)를 바꿔도 BFF 계약은 안 깨짐

### 1단계: 소비자 측 — 계약 생성

소비자가 "이런 응답을 기대한다"는 테스트를 짜면, 그게 계약 파일(pact) 로 산출된다.

```kotlin
// (소비자) 측 테스트
@Pact(consumer = "client", provider = "contents-service")
fun getContentContract(builder: PactDslWithProvider): RequestResponsePact =
    builder
        .given("content P001 exists")           // 제공자가 갖춰야 할 상태
        .uponReceiving("get content P001")
        .path("/v1/contents/P001")
        .method("GET")
        .willRespondWith()
        .status(200)
        .body(newJsonBody { o ->
            o.stringType("code", "P001")
            o.stringType("title", "도깨비")
            o.`object`("label") { it.booleanType("isFree", true) }
        }.build())
        .toPact()

// 이 테스트를 실행하면 → client-contents.json (계약 파일) 생성
```

이때 소비자는 제공자를 모킹해서 테스트한다. 계약대로 응답하는 가짜 제공자로 BFF 로직을 검증하고, 동시에 계약 파일을 뽑아낸다.

### 2단계: 제공자 측 — 계약 검증

생성된 계약 파일을 제공자가 가져가서, 자기 실제 응답이 그 계약을 만족하는지 검증한다.

```kotlin
// contents-service(제공자) 측 테스트
@Provider("contents-service")
@PactFolder("pacts")   // 소비자가 만든 계약 파일들
class ContentsProviderTest {
    @State("content P001 exists")   // 계약의 given 상태를 셋업
    fun setupContent() {
        repo.save(Content("P001", "도깨비", ...))
    }

    @TestTemplate
    fun verifyContract(context: PactVerificationContext) {
        context.verifyInteraction()   // 실제 응답이 계약과 맞는지 검증
    }
}
```

제공자가 필드명을 바꾸거나 형식을 어기면 이 테스트가 실패 → 배포 전에 CI에서 잡힘.

### API 계약 저장소와 발행

*   git
*   Pact Broker

소비자가 만든 계약을 제공자가 가져가려면 중간 저장소가 필요하다. Pact Broker(또는 PactFlow)가 계약 파일을 저장·버전관리·공유한다.

*   소비자 CI: 계약 생성 → Broker에 발행(publish)
*   제공자 CI: Broker에서 계약 가져와(fetch) 검증
*   Broker: "이 버전 조합이 서로 호환되는가"를 매트릭스로 관리

> "can-i-deploy: Broker의 킬러 기능. '지금 이 제공자 버전을 배포하면 기존 소비자들이 다 안전한가?'를 배포 직전에 물어볼 수 있다. 호환 안 되면 배포 차단."

```
[소비자 CI 파이프라인]                 [제공자 CI 파이프라인]
   (BFF 저장소)                       (contents 저장소)
        │                                   │
   소비자 테스트 실행                    제공자 테스트 실행
        │                                   │
   가짜 제공자(mock)로                  Broker에서 계약 받아
   BFF 로직 검증                    실제 contents 응답과 대조
        │                                   │
   계약 파일 생성                              │
        │                                   │
        └──── Broker에 발행 ──────────▶ 구독/fetch
```

## 🔸API 컴포넌트 테스트

컴포넌트 테스트는 여러 단위가 협업해 완성하는 동작을 테스트하는 것으로 외부 의존성을 호출해서는 안된다.

테스트 사분면의 Q1, Q2에 속한다.

## 🔸API 통합 테스트

서비스 전체를 띄우되 외부는 stub으로, DB는 Testcontainers 테스트 수행

*   stub 서버 구성의 장점

    *   외부 결제사 같은 서드파티 API는 계약을 걸수조차 없기 때문에 stub을 구성해야한다.

    *   에러/엣지 시나리오 재현

## 🔸종단 간 테스트

실제 사용자의 여정 전체를, 모든 구성요소가 실제로 맞물린 상태에서 검증하는 테스트이다.

*   서비스 간 실제 연동 문제: 계약 테스트는 형식만, 컴포넌트는 stub만 봤다. 실제로 다 연결했을 때만 드러나는 문제 — 예: 인증 토큰이 서비스를 거치며 제대로 전파 안 됨, 타임아웃이 사슬 전체에서 누적됨.

*   환경/설정 문제: 실제 배포 환경의 네트워크, 로드밸런서(ALB), 서비스 디스커버리, 설정값이 맞물려야 나타나는 문제.

*   전체 흐름의 데이터 정합성: A 서비스가 쓴 데이터를 B가 읽어 C로 넘기는 전 과정이 맞는지.

> "아래 층에서 잡을 수 있는 건 절대 E2E로 올리지 마라. 로직 버그는 단위로, DB 연동은 통합으로, 경계 호환은 계약으로 잡고 — E2E는 '그 모든 게 다 맞물렸을 때만 드러나는 것'에만 쓴다. E2E가 많다는 건 아래 층이 부실하다는 신호이다."

## 🔸정리

계약 테스트가 재밌었다.

*   백엔드 서비스 간
*   클라이언트 ↔︎ 서버
*   Kafka 프로듀서 ↔︎ 컨슈머

## 참고 자료

- [테스트 전략: 계약 테스트에 대하여 - velog](https://velog.io/@skysoo1111/%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%A0%84%EB%9E%B5-%EA%B3%84%EC%95%BD-%ED%85%8C%EC%8A%A4%ED%8A%B8%EC%97%90-%EB%8C%80%ED%95%98%EC%97%AC)
