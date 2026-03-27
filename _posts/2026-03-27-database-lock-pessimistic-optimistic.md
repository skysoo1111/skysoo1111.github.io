---
layout: post
title: "DB 락 완전 가이드 - 비관적 락과 낙관적 락, 데드락 해결 전략"
date: 2026-03-27 09:00:00 +0900
categories: [Database]
tags: [lock, pessimistic-lock, optimistic-lock, jpa, database, concurrency, deadlock]
---

## 개요

다중 사용자 환경에서 동일한 데이터에 여러 트랜잭션이 동시 접근하면 **데이터 불일치**가 발생할 수 있다. 이를 제어하는 핵심 메커니즘이 바로 **DB 락(Lock)**이다.

### 갱신 분실(Lost Update) 문제

락의 필요성을 이해하려면 먼저 갱신 분실 문제를 알아야 한다.

**시나리오: 재고 관리 시스템**

1. 관리자 A와 B가 재고 10개인 상품을 동시 조회
2. A가 2개 판매 → 재고 8로 수정 및 커밋
3. B가 조회 시점 기준(10개)으로 5개 판매 → 재고 5로 수정 및 커밋
4. 결과: 최종 재고는 3이어야 하나 **5가 됨** (A의 변경 사항 유실)

이처럼 `READ_COMMITTED`나 `REPEATABLE_READ` 같은 표준 격리 수준만으로는 **여러 요청에 걸친 갱신 분실을 막을 수 없다**. DB 락이 필요한 이유다.

---

## 비관적 락 (Pessimistic Lock)

### 개념과 동작 원리

**"데이터 경합이 빈번하게 발생할 것이라고 비관적으로 가정하고, 동시 수정을 원천적으로 차단한다."**

트랜잭션이 데이터를 읽는 시점에 DB 레벨의 락을 획득하고, 다른 트랜잭션이 해당 데이터에 접근하지 못하도록 막는다.

**동작 흐름:**
1. 트랜잭션 A가 데이터를 읽으며 배타적 락(X-Lock) 획득
2. 트랜잭션 B가 동일 데이터 접근 시도 → 대기 상태
3. 트랜잭션 A가 수정 완료 후 커밋 → 락 해제
4. 트랜잭션 B가 락 획득 후 갱신된 데이터로 작업 시작

### SQL 예시

```sql
-- 배타적 락 (쓰기 전용, 다른 트랜잭션의 읽기/쓰기 차단)
SELECT * FROM product WHERE id = 1 FOR UPDATE;

-- 공유 락 (읽기 허용, 쓰기만 차단)
SELECT * FROM product WHERE id = 1 FOR SHARE;
```

### JPA 구현 예시

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select p from Product p where p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    @Transactional
    public void decreaseStock(Long id, int quantity) {
        Product product = productRepository.findByIdForUpdate(id)
            .orElseThrow(EntityNotFoundException::new);
        product.decreaseStock(quantity);
    }
}
```

### 잠금 타임아웃 설정

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({
    @QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000")
})
@Query("select p from Product p where p.id = :id")
Optional<Product> findByIdForUpdate(@Param("id") Long id);
```

3초 이내에 락을 획득하지 못하면 `LockTimeoutException`이 발생한다.

### LockModeType 종류

| 모드 | 설명 | DB 락 | SQL |
|------|------|-------|-----|
| `PESSIMISTIC_WRITE` | 배타적 쓰기 락 | Exclusive Lock | `SELECT ... FOR UPDATE` |
| `PESSIMISTIC_READ` | 공유 읽기 락 | Shared Lock | `SELECT ... FOR SHARE` |
| `PESSIMISTIC_FORCE_INCREMENT` | 배타적 락 + 버전 강제 증가 | Exclusive Lock + Version | `SELECT ... FOR UPDATE` |

### 장단점

**장점:**
- 충돌을 사전에 방지하여 데이터 일관성을 강력하게 보장
- 충돌 발생 즉시 감지 가능 (락 대기 중 인지)
- 충돌 비용이 큰 금융 거래에 적합

**단점:**
- 락 보유 시간 동안 다른 트랜잭션이 대기 → 동시 처리 성능 저하
- 교착 상태(Deadlock) 발생 위험
- 웹 환경에서 HTTP 요청 간 락 유지 불가

---

## 낙관적 락 (Optimistic Lock)

### 개념과 동작 원리

**"데이터 충돌이 거의 발생하지 않을 것이라 가정하고, 수정 시점에 변경 여부만 확인한다."**

DB 레벨의 락을 사용하지 않고, **버전(Version) 컬럼**을 활용해 애플리케이션 레벨에서 충돌을 감지한다. Compare-And-Swap(CAS) 메커니즘과 유사하다.

**동작 흐름:**
1. 트랜잭션 A와 B가 데이터(version=1)를 동시 조회 (락 없음)
2. 트랜잭션 A가 먼저 커밋 → `UPDATE ... WHERE version=1` 성공 → version=2
3. 트랜잭션 B가 커밋 시도 → `UPDATE ... WHERE version=1` 실패 (이미 version=2)
4. **0 rows affected** → `OptimisticLockException` 발생 → 롤백

### @Version 기반 구현

```java
@Entity
public class Product {

    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private int stock;

    @Version
    private Long version; // Integer 또는 Timestamp도 가능

    public void decreaseStock(int quantity) {
        if (this.stock < quantity) {
            throw new IllegalStateException("재고가 부족합니다.");
        }
        this.stock -= quantity;
    }
}
```

`@Version` 필드는 JPA가 자동으로 관리한다. 엔티티 수정 시 자동으로 버전을 증가시키고 WHERE 절에 포함시킨다.

### 커밋 시 생성되는 SQL

```sql
UPDATE product
SET stock = 8, version = 2
WHERE id = 1 AND version = 1;
-- 영향받은 행이 0이면 OptimisticLockException 발생
```

### JPA 구현 예시

```java
// Repository - 별도 어노테이션 없이 @Version만으로 동작
public interface ProductRepository extends JpaRepository<Product, Long> {
    // 기본 findById 사용 가능
}

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    @Transactional
    public void decreaseStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow();
        product.decreaseStock(quantity);
        // 트랜잭션 커밋 시 버전 체크 자동 수행
    }
}
```

### 예외 처리 및 재시도 로직

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    @Transactional
    public void decreaseStock(Long productId, int quantity) {
        try {
            Product product = productRepository.findById(productId)
                .orElseThrow();
            product.decreaseStock(quantity);
        } catch (ObjectOptimisticLockingFailureException e) {
            log.warn("동시성 충돌 발생 - productId: {}", productId);
            throw new ConcurrencyConflictException(
                "재고가 다른 사용자에 의해 수정되었습니다. 다시 시도해주세요.");
        }
    }
}
```

충돌이 잦은 환경에서는 재시도(Retry) 로직을 별도로 구현하거나, Spring Retry를 활용할 수 있다.

### 장단점

**장점:**
- DB 직접 락 미사용으로 높은 동시 처리 성능
- 교착 상태(Deadlock) 발생 없음
- 상태 비저장(stateless) 방식으로 웹 애플리케이션, 마이크로서비스 환경에 적합
- HTTP 요청 간 경계를 넘어서도 충돌 감지 가능

**단점:**
- 충돌 발생 시 전체 트랜잭션 롤백
- 충돌이 빈번한 환경에서는 반복 롤백 비용으로 오히려 성능 저하
- `@Modifying @Query` 같은 벌크 업데이트는 버전 관리를 우회함

---

## 비관적 락 vs 낙관적 락 비교

| 특징 | 비관적 락 | 낙관적 락 |
|------|----------|----------|
| 철학 | 충돌 방지 (Prevent) | 충돌 감지 (Detect) |
| 잠금 시점 | 읽기 시 | 쓰기/커밋 시 |
| 잠금 주체 | DB 트랜잭션 | 애플리케이션 |
| 구현 방식 | DB 락 (`FOR UPDATE`) | 버전 컬럼 (`@Version`) |
| 교착 상태 | 발생 가능 | 발생 안 함 |
| 성능 (충돌 적을 때) | 낮음 | 높음 |
| 성능 (충돌 많을 때) | 높음 | 낮음 |
| HTTP 요청 간 적용 | 불가 | 가능 |
| 적합 환경 | 높은 충돌 환경 | 낮은 충돌 환경 |

> **오해 주의:** "비관적 락이 항상 느리고 낙관적 락이 항상 빠르다"는 잘못된 믿음이다. 충돌이 빈번한 환경에서는 반복 롤백 비용보다 락 대기 비용이 더 저렴할 수 있다.

---

## 데드락 (Deadlock)

### 개념

두 개 이상의 트랜잭션이 서로 상대방의 락 해제를 기다리며 **무한정 대기하는 상태**다.

```
트랜잭션 A: Lock(Item 1) 획득 → Lock(Item 2) 대기 중...
트랜잭션 B: Lock(Item 2) 획득 → Lock(Item 1) 대기 중...
→ 서로를 기다리며 영원히 대기 (Deadlock)
```

### 발생 조건 4가지

데드락은 다음 4가지 조건이 **동시에 만족**될 때 발생한다.

1. **상호 배제 (Mutual Exclusion)**: 자원은 한 번에 하나의 트랜잭션만 사용 가능
2. **점유와 대기 (Hold and Wait)**: 자원을 보유한 상태에서 다른 자원을 요청하며 대기
3. **비선점 (No Preemption)**: 다른 트랜잭션이 보유한 자원을 강제로 빼앗을 수 없음
4. **순환 대기 (Circular Wait)**: 트랜잭션들이 순환적 의존 관계를 형성

### 예방 및 해결 전략

**예방 (Prevention):**
- 4가지 조건 중 하나를 제거
- **항상 동일한 순서로 락 획득** → 순환 대기 조건 제거
  - 예: 여러 테이블 업데이트 시 항상 id 오름차순으로 락 획득

**회피 (Avoidance):**
- 자원 할당 전 데드락 가능성을 예측하고 회피
- 은행원 알고리즘(Banker's Algorithm) 활용

**탐지 및 복구 (Detection and Recovery):**
- DB가 자동으로 데드락을 감지하고 하나의 트랜잭션을 희생(Victim)으로 선택하여 롤백
- MySQL에서 데드락 확인:

```sql
SHOW ENGINE INNODB STATUS;
-- LATEST DETECTED DEADLOCK 섹션 확인
```

**타임아웃 설정:**
- 락 대기 타임아웃을 설정하여 무한 대기 방지
- `jakarta.persistence.lock.timeout` 힌트 활용

---

## 실무에서의 선택 기준

### 비관적 락이 적합한 시나리오

**충돌이 자주 발생하거나 충돌 비용이 높을 때**

| 시나리오 | 이유 |
|---------|------|
| 인기 상품 재고 차감 | 동시 구매 요청이 집중됨 |
| 항공권/공연 티켓 좌석 예약 | 좌석 중복 예약 절대 불가 |
| 금융 거래 (계좌 이체, 출금) | 충돌 시 롤백 비용이 매우 큼 |

### 낙관적 락이 적합한 시나리오

**충돌 가능성이 낮고 읽기 위주일 때**

| 시나리오 | 이유 |
|---------|------|
| 상품 설명/이름 수정 | 동시 수정 확률이 낮음 |
| 사용자 프로필 변경 | 본인만 수정하는 경우가 대부분 |
| 위키/문서 협업 도구 | 충돌 시 사용자에게 재시도 안내 가능 |

### 하이브리드 접근법

두 방식을 혼용하는 전략도 유효하다.

> **예시: 이커머스 주문 흐름**
> - 상품 조회 / 장바구니 담기 → 낙관적 락 (충돌 거의 없음)
> - 최종 결제 / 재고 차감 → 비관적 락 (충돌 허용 불가)

최선의 선택은 애플리케이션의 **데이터 접근 패턴**과 **비즈니스 요구사항**에 대한 깊은 이해에서 비롯된다.
