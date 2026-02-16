# @Transactional

## 1. 왜 트랜잭션이 필요한가

데이터베이스 작업은 하나의 SQL로 끝나는 경우보다 여러 작업이 묶여 실행되는 경우가 더 많다.

예를 들어 주문 처리 로직:

- 주문 저장
- 재고 차감
- 결제 정보 저장
- 포인트 적립

이 모든 작업은 하나의 비즈니스 로직이다.  
하지만 DB 입장에서는 각각이 개별 SQL이다.

문제는 중간에 실패할 경우다.

```
주문 저장 - 성공
재고 차감 - 성공
결제 저장 - 실패
```

이 상태로 종료된다면 데이터 정합성이 깨진다.

그래서 필요한 것이 트랜잭션이다.

--- 

## 2. 트랜잭션이란 무엇인가

트랜잭션은 다음을 보장한다.

ACID

- Atomicity (원자성)
- Consistency (일관성)
- Isolation (격리성)
- Durability (지속성)

핵심은 다음 한 문장이다.

> 모두 성공하거나, 모두 실패하거나

--- 

## 3. 스프링에서 트랜잭션을 처리하는 방법

스프링은 두 가지 방식을 제공한다.

1. 프로그래밍 방식
2. 선언적 방식

### 프로그래밍 방식

```java
TransactionStatus status = transactionManager.getTransaction(def);

try {
    orderRepository.save(order);
    transactionManager.commit(status);
} catch (Exception e) {
    transactionManager.rollback(status);
}
```

문제점:

- 비즈니스 로직과 섞인다
- 반복 코드가 발생한다
- 가독성이 떨어진다

이것은 전형적인 횡단 관심사다.

그래서 등장한 것이 선언적 트랜잭션이다.

--- 

## 4. @Transactional은 무엇인가

```java
@Transactional
public void createOrder(OrderRequest request) {
    orderRepository.save(order);
    paymentRepository.save(payment);
}
```

이 한 줄이 의미하는 것:

- 메서드 실행 전에 트랜잭션 시작
- 정상 종료 시 commit
- 예외 발생 시 rollback

--- 

## 5. @Transactional의 동작 원리

핵심 키워드:

- [AOP](Spring%20AOP.md)
- [Proxy](CGLIB.md)
- TransactionInterceptor

### 1) 빈 생성 시점

스프링은 @Transactional이 붙은 빈을 발견하면  
원본 객체 대신 프록시 객체를 생성해 빈으로 등록한다.

즉, 우리가 주입받는 객체는 실제 서비스 객체가 아니라 프록시다.

### 2) 메서드 호출 시 흐름

```
Controller
   ↓
Proxy (TransactionInterceptor)
   ↓
Target (실제 Service 객체)
```

프록시는 내부적으로 다음과 같이 동작한다.

```java
beginTransaction();

try {
    target.method();
    commit();
} catch (Exception e) {
    rollback();
}
```

이 로직을 수행하는 것이 `TransactionInterceptor`이다.

--- 

## 6. 실제 호출 흐름

Spring MVC 흐름에 포함시키면 다음과 같다.

```
DispatcherServlet
   ↓
HandlerAdapter
   ↓
Controller
   ↓
Service (프록시)
   ↓
TransactionInterceptor
   ↓
실제 Service 메서드
```

트랜잭션은 서비스 로직을 감싸는 AOP Advice다.

---

## 7. 이 때 DB에서는 무슨 일이 일어나는가

`TransactionInterceptor`가 `TransactionManager`를 호출해 트랜잭션을 시작하면,
스프링 내부에서만 일이 벌어지는 것이 아니다.

실제로는 다음과 같은 일이 동시에 일어난다.

### 1) 커넥션 획득

`DataSource`로부터 DB 커넥션을 하나 가져온다.

```
DataSource → Connection 획득
```

이때 커넥션 풀(HikariCP 등)을 사용 중이라면
이미 생성된 커넥션을 하나 빌려오는 것이다.

### 2) autoCommit 비활성화

JDBC 기본 설정은 `autoCommit = true`다.

하지만 트랜잭션을 시작하면:

```
connection.setAutoCommit(false);
```

가 수행된다.

이 순간부터 SQL은 즉시 확정(commit)되지 않는다.

### 3) DB 세션에서 트랜잭션 시작

DB 입장에서는 다음과 같은 상태가 된다.

```
BEGIN;
```

MySQL 같은 경우 명시적으로 BEGIN을 날리지 않더라도,
autoCommit이 false인 상태에서 첫 쿼리가 실행되면
트랜잭션이 시작된다.

즉,

> 스프링에서 트랜잭션이 시작된다는 것은  
> 하나의 커넥션이 하나의 트랜잭션 세션을 점유한다는 의미다.

### 4) 하나의 커넥션 = 하나의 트랜잭션 단위

중요한 점은 이것이다.

- 트랜잭션은 커넥션 단위로 관리된다.
- 하나의 트랜잭션 동안 같은 커넥션이 사용된다.
- commit 또는 rollback 시점에 DB에 최종 반영된다.

### 5) commit / rollback 시점

정상 종료 시:

```
COMMIT;
```

예외 발생 시:

```
ROLLBACK;
```

그리고 트랜잭션이 끝나면:

- 커넥션은 autoCommit=true로 복구된다.
- 커넥션 풀로 반환된다.


결론적으로 스프링 트랜잭션은 하나의 커넥션을 묶어 하나의 작업 단위로 만들고,  
그 작업을 한 번에 확정하거나 되돌리는 메커니즘이다.

--- 

## 8. 롤백 규칙

기본 롤백 규칙:

- RuntimeException 발생 시 롤백
- Error 발생 시 롤백
- Checked Exception은 기본적으로 롤백하지 않음

설정을 변경하여 롤백 기준을 변경하거나

```java
@Transactional(rollbackFor = Exception.class)
```

특정 예외를 롤백 제외할 수 있다.

```java
@Transactional(noRollbackFor = IllegalArgumentException.class)
```

---

## 9. @Transactional 주요 속성 정리

### readOnly

```java
@Transactional(readOnly = true)
public List<Order> findOrders() {
    return orderRepository.findAll();
}
```

의미:

- 읽기 전용 트랜잭션
- JPA dirty checking 비활성화
- flush 방지
- 일부 DB 최적화 가능

조회 전용 로직에는 readOnly = true를 사용하는 것이 좋다.

### [트랜잭션 전파](Spring%20%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98%20%EC%A0%84%ED%8C%8C.md)

```java
@Transactional(propagation = Propagation.REQUIRED)
```

대표 옵션:

- REQUIRED → 있으면 참여, 없으면 생성
- REQUIRES_NEW → 항상 새 트랜잭션 생성
- SUPPORTS → 있으면 참여, 없으면 트랜잭션 없이 실행

### [트랜잭션 격리 수준 (Isolation)](../mysql/%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98%20%EA%B2%A9%EB%A6%AC%20%EC%88%98%EC%A4%80.md)

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
```

옵션:

- READ_UNCOMMITTED
- READ_COMMITTED
- REPEATABLE_READ
- SERIALIZABLE

### timeout 설정

```java
@Transactional(timeout = 5)
```

5초 초과 시 롤백된다.

### rollbackFor / noRollbackFor

```java
@Transactional(rollbackFor = IOException.class)
```

특정 Checked Exception도 롤백 대상으로 지정 가능하다.

---

## 10. 정리

@Transactionl은 개발자가 직접 begin, commit, rollback 코드를 작성하지 않아도 동작하도록 해준다.

동작 방식은 단순하다.

- 스프링은 `@Transactional`이 붙은 빈을 프록시로 감싼다.
- 메서드 호출이 프록시를 통과하면 `TransactionInterceptor`가 개입한다.
- `TransactionInterceptor`는 트랜잭션을 시작/종료하기 위해 `TransactionManager`를 호출한다.
- 정상 종료면 commit, 예외면 rollback을 수행한다.

여기까지는 **어떤 원리로 자동 트랜잭션이 걸리는지**를 설명한 것이다.

[다음 글](%EC%8A%A4%ED%94%84%EB%A7%81%EC%97%90%EC%84%9C%20%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98%EC%9D%B4%20%EB%8F%99%EC%9E%91%ED%95%98%EB%8A%94%20%EA%B3%BC%EC%A0%95.md)에서는 `TransactionInterceptor`가 실제로 무엇을 호출하고 어떤 순서로 트랜잭션을 시작, 커밋, 롤백하는지,
즉 스프링 내부에서 트랜잭션이 처리되는 전체 흐름을 `TransactionManager` 중심으로 정리한다.