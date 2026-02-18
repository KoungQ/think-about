# Spring 트랜잭션 전파

트랜잭션은 시작-실행-종료(commit/rollback)로 끝나는 단순한 개념처럼 보인다.

하지만 실무에서는 서비스 메서드가 서로를 호출한다.

- 주문 생성 → 결제 → 포인트 적립
- 게시글 작성 → 파일 업로드 기록 → 알림 발송
- 회원 가입 → 약관 동의 저장 → 웰컴 쿠폰 발급

여기서 생기는 질문은 이것이다.

> 이미 트랜잭션이 있는 상태에서, 또 다른 `@Transactional` 메서드를 호출하면  
> 트랜잭션은 새로 열릴까? 기존 트랜잭션에 참여할까?

이 참여/새로 생성/중단 전략이 바로 **트랜잭션 전파(Propagation)** 다.

---

## 1. 전파(Propagation)란 무엇인가

전파는 한 문장으로 정리하면 다음이다.

> 현재 스레드에 트랜잭션이 존재할 때, 새로운 트랜잭션 요청을 어떻게 처리할지에 대한 정책

스프링은 트랜잭션을 ThreadLocal 기반으로 관리한다.

따라서 전파는 단순한 옵션이 아니라  
**ThreadLocal에 바인딩된 트랜잭션 문맥을 유지/교체/중단**하는 전략이다.

---

## 2. 왜 전파가 필요한가

예를 들어, 외부 트랜잭션에서 내부 메서드를 호출하는 상황을 보자.

```java
@Transactional
public void placeOrder() {
    orderRepository.save(...);
    paymentService.pay(); // 내부 호출
}
```

여기서 `paymentService.pay()`가 `@Transactional`이면?

- 같은 트랜잭션으로 묶일까?
- payment는 별도 트랜잭션으로 분리될까?
- payment가 실패하면 주문 저장도 같이 롤백될까?

이 동작을 결정하는 게 전파다.

---

## 3. 전파는 어디서 결정되는가

전파 옵션은 `getTransaction()` 단계에서 결정된다.

```less
TransactionAspectSupport.createTransactionIfNecessary()
  └── AbstractPlatformTransactionManager.getTransaction(...)
        ├── doGetTransaction()
        ├── isExistingTransaction(...)
        ├── handleExistingTransaction(...)   // 기존 트랜잭션 있을 때 전파 분기
        └── startTransaction(...)            // 없을 때 신규 시작
```

즉,

- 기존 트랜잭션이 있으면 → `handleExistingTransaction(...)`
- 없으면 → `startTransaction(...)`

전파는 DB 기능이 아니라 스프링이 트랜잭션 경계를 구성하는 애플리케이션 정책이다.

---

## 4. 대표 전파 옵션 5가지

전파 옵션은 많지만, 실무에서 핵심은 아래 5개다.

### 4.1 REQUIRED (기본값)

```java
@Transactional(propagation = Propagation.REQUIRED)
```

- 트랜잭션이 있으면 참여
- 없으면 신규 생성

가장 일반적인 “같은 트랜잭션으로 묶는다” 옵션이다.

---

### 4.2 REQUIRES_NEW

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
```

- 항상 새 트랜잭션 생성
- 기존 트랜잭션이 있으면 suspend(중단)했다가 끝나면 resume(복구)

“외부 트랜잭션과 운명을 분리하고 싶을 때” 사용한다.

예:
- 주문 저장은 성공시키되, 알림 발송 실패는 주문 롤백과 무관하게 처리하고 싶다
- 실패해도 되는 로그/감사(Audit)를 별도 트랜잭션으로 남기고 싶다

---

### 4.3 SUPPORTS

```java
@Transactional(propagation = Propagation.SUPPORTS)
```

- 트랜잭션이 있으면 참여
- 없으면 트랜잭션 없이 실행

읽기 로직 등에서 “있으면 타고, 없어도 됨” 같은 성격에 사용된다.

---

### 4.4 NOT_SUPPORTED

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
```

- 트랜잭션 없이 실행
- 기존 트랜잭션이 있으면 suspend

트랜잭션을 “의도적으로 끊고” 실행해야 할 때 사용한다.

다만 DB 일관성을 깨기 쉬워서 빈번한 사용은 피하는 편이 좋다.

---

### 4.5 MANDATORY / NEVER

```java
@Transactional(propagation = Propagation.MANDATORY)
@Transactional(propagation = Propagation.NEVER)
```

- MANDATORY: 트랜잭션이 없으면 예외
- NEVER: 트랜잭션이 있으면 예외

특정 계층/정책을 강제하고 싶을 때 사용한다.

---

## 5. 전파 옵션이 롤백에 미치는 영향

전파를 이해하기 어려운 이유는 rollback-only 때문이다.

### 5.1 REQUIRED의 롤백 동작

REQUIRED는 대부분 같은 트랜잭션으로 묶인다.

그래서 내부에서 예외가 터지면 외부도 같이 영향받는다.

- 내부에서 RuntimeException 발생 → 트랜잭션 rollback-only 마킹
- 외부는 catch로 잡아도 최종 commit 시점에 rollback됨

즉, 코드상으로는 commit을 호출해도 최종 결과는 rollback이 될 수 있다.

---

### 5.2 REQUIRES_NEW의 롤백 동작

REQUIRES_NEW는 트랜잭션이 분리된다.

- 내부 트랜잭션 rollback → 내부만 롤백
- 외부 트랜잭션은 계속 진행 가능

따라서 “실패가 외부를 전파하지 않게” 만들 수 있다.

---

## 6. 실무에서 가장 많이 나오는 패턴 3개

### 6.1 전체 원자성이 필요한 경우 (REQUIRED)

- 주문, 결제, 재고, 포인트가 모두 하나의 비즈니스 단위
- 중간 실패 시 전부 롤백

→ REQUIRED 기본값이 가장 자연스럽다.

---

### 6.2 감사 로그 / 알림은 분리하고 싶을 때 (REQUIRES_NEW)

- 주문 로직은 롤백되면 안 됨
- 알림/로그는 실패해도 전체를 망치면 안 됨

→ REQUIRES_NEW로 분리해서 외부 트랜잭션과 운명을 끊는다.

---

### 6.3 읽기 로직은 트랜잭션이 있어도 없어도 된다 (SUPPORTS)

- 서비스가 트랜잭션을 열고 있으면 참여
- 아니면 그냥 조회

→ SUPPORTS가 옵션상 적절하다.

---

## 7. 정리

트랜잭션 전파는 DB 기능이 아니다.

> 현재 스레드의 트랜잭션 문맥(ThreadLocal)을 기준으로  
> “참여할지, 새로 만들지, 중단할지”를 결정하는 스프링의 정책이다.

가장 중요한 선택지는 사실 2개로 압축된다.

- REQUIRED: 하나로 묶는다
- REQUIRES_NEW: 분리한다

실무에서 전파 옵션을 결정할 때는 항상 이 질문부터 하면 된다.

> 이 로직은 외부 트랜잭션과 운명을 같이해야 하는가, 분리해야 하는가?
