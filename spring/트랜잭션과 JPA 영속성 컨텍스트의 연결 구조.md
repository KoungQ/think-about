# 트랜잭션과 JPA 영속성 컨텍스트의 연결 구조

앞 글에서 우리는 스프링 트랜잭션이

- AOP로 트랜잭션 경계를 만들고
- `PlatformTransactionManager`가 begin/commit/rollback을 수행하며
- ThreadLocal을 통해 트랜잭션 자원을 관리한다는 것을 보았다.

이제 한 단계 더 들어가 보자.

> 트랜잭션 경계 안에서 JPA(Hibernate)는 정확히 무엇을 하는가?

이 글에서는 다음을 하나의 흐름으로 연결해 이해한다.

- EntityManager와 영속성 컨텍스트 생명주기
- 영속성 컨텍스트의 핵심 기능
- flush 발생 시점
- flush와 commit의 차이
- readOnly=true일 때의 동작 변화

---

## 1. 큰 그림: 트랜잭션과 영속성 컨텍스트의 역할 분리

먼저 개념을 정확히 분리해야 한다.

- **트랜잭션**: DB 레벨의 작업 경계
- **영속성 컨텍스트**: 객체 상태 관리 영역

둘은 동일한 개념이 아니다.  
하지만 Spring + JPA 환경에서는 일반적으로 **같은 생명주기**를 갖는다.

```less
@Transactional 진입
  └── JpaTransactionManager
        ├── EntityManager 생성/조회
        ├── 영속성 컨텍스트 활성화
        └── JDBC Connection 준비

(서비스 로직 실행)
  └── 엔티티 상태 변화 → 영속성 컨텍스트에 누적

commit 시점
  ├── flush (SQL 전송)
  └── commit (트랜잭션 확정)

cleanup
  └── EntityManager close + 자원 반환
```

핵심은 이것이다.

> 트랜잭션이 경계를 만들고  
> 영속성 컨텍스트가 그 안에서 객체를 관리한다.

---

## 2. EntityManager와 영속성 컨텍스트 생명주기

### 2.1 생성 시점

`@Transactional`이 붙은 메서드에 진입하면:

- `JpaTransactionManager`가 개입하고
- `EntityManager`를 생성하거나 가져오고
- 현재 스레드(`ThreadLocal`)에 바인딩한다.

즉, 서비스 메서드 실행 전에 이미 영속성 컨텍스트는 준비되어 있다.

Spring은 실제 `EntityManager`를 직접 주입하는 대신  
프록시를 주입하고, 런타임에 `ThreadLocal`에서 실제 `EntityManager`를 찾아 위임한다.

---

### 2.2 트랜잭션 실행 중

트랜잭션이 살아 있는 동안:

- 1차 캐시 유지
- 스냅샷 유지
- Dirty Checking 대상 관리
- SQL 쓰기 지연(write-behind)

영속성 컨텍스트는 이 구간에서 모든 엔티티 상태를 관리한다.

---

### 2.3 종료 시점

트랜잭션이 종료되면:

1. flush (필요 시)
2. commit 또는 rollback
3. EntityManager close
4. ThreadLocal unbind

즉, 속성 컨텍스트는 트랜잭션 종료와 함께 종료된다.

이 시점 이후 엔티티는 **준영속(detached)** 상태가 된다.

---

## 3. 영속성 컨텍스트의 핵심 기능 3가지

영속성 컨텍스트는 단순 캐시가 아니다.  
다음 세 가지 핵심 메커니즘을 가진다.

- 1차 캐시 (동일성 보장)
- 변경 감지 (Dirty Checking)
- 쓰기 지연 (Write-Behind)

이 세 가지가 결합되어 JPA의 트랜잭션 모델이 완성된다.

---

### 3.1 1차 캐시 (동일성 보장)

영속성 컨텍스트는 내부적으로 다음과 같은 구조를 가진다.

```
Key: (엔티티 타입 + 식별자)
Value: 엔티티 객체
```

예:

```java
Member m1 = em.find(Member.class, 1L);
Member m2 = em.find(Member.class, 1L);
```

이 경우:

```java
m1 == m2  // true
```

같은 트랜잭션 안에서는  
동일한 DB row는 동일한 Java 객체로 유지된다.

이를 **동일성(identity) 보장**이라고 한다.

---

### 3.2 변경 감지 (Dirty Checking)

JPA는 개발자가 update를 직접 호출하지 않아도 된다.

```java
Member member = em.find(Member.class, 1L);
member.setName("newName");
```

update를 호출하지 않았지만,  
트랜잭션 commit 시 update SQL이 실행된다.

그 이유는 Dirty Checking 때문이다.

#### 내부 구조

영속성 컨텍스트 내부에는 두 가지가 존재한다.

```
영속성 컨텍스트
 ├── 1차 캐시 → 엔티티 객체
 └── 스냅샷 저장소 → 최초 상태 값
```

엔티티를 처음 로딩할 때:

- 현재 상태를 스냅샷으로 복사해둔다.

flush 시점에:

- 현재 객체 상태 vs 스냅샷 비교

달라진 필드만 update SQL을 생성한다.

중요:

> 스냅샷은 1차 캐시에 저장되는 것이 아니라  
> 변경 감지를 위해 별도로 유지되는 초기 상태 복사본이다.

---

### 3.3 쓰기 지연 (Write-Behind)

JPA는 SQL을 즉시 실행하지 않는다.

```java
em.persist(member);
```

이 순간 insert SQL이 바로 나가지 않을 수 있다.

왜냐하면:

> SQL은 영속성 컨텍스트 내부에 쌓여 있다가  
> flush 시점에 한 번에 실행된다.

이 전략을 **Write-Behind**라고 한다.

장점:

- 트랜잭션 단위로 SQL을 묶어 처리 가능
- 배치 최적화 가능
- flush 타이밍 제어 가능

---

## 4. flush는 언제 발생하는가

flush는 **영속성 컨텍스트의 변경 내용을 DB에 동기화**하는 작업이다.

flush != commit

대표적인 flush 발생 시점:

1. 트랜잭션 commit 직전
2. JPQL 실행 직전 (FlushMode.AUTO)
3. 명시적 `em.flush()` 호출

예:

```java
member.setName("A");
em.createQuery("select m from Member m").getResultList();
```

JPQL 실행 전에 flush가 일어날 수 있다.  
그래야 조회 결과가 최신 상태와 일관성을 유지한다.

---

## 5. flush와 commit의 차이

많이 헷갈리는 부분이다.

실행 순서는 다음과 같다.

```less
commit 호출
  ├── flush (SQL 전송)
  └── commit (트랜잭션 확정)
```

- flush: SQL을 DB에 전송
- commit: 트랜잭션을 확정

flush가 되었어도 commit이 되지 않았다면  
다른 트랜잭션에서 보이지 않을 수 있다.

따라서 **flush는 “동기화”, commit은 “확정”** 이다.

---

## 6. `readOnly=true`일 때 동작 변화

```java
@Transactional(readOnly = true)
```

### 6.1 개념

readOnly는 “쓰기 의도가 없다”는 선언이다.

중요:

> 쓰기를 강제로 막는 기능이 아니라  
> 성능 최적화를 위한 힌트다.

---

### 6.2 Spring 레벨 변화

- 트랜잭션을 read-only 모드로 설정
- JDBC Connection에 readOnly 힌트 전달

DB/드라이버에 따라 추가 최적화가 가능하다.

---

### 6.3 Hibernate/JPA 레벨 변화

일반 트랜잭션에서는:

- 엔티티 로딩 시 스냅샷 생성
- Dirty Checking 대상 등록
- flush 시점에 상태 비교

readOnly=true일 경우:

- 일부 환경에서는 스냅샷 생성 최소화
- Dirty Checking 비용 감소
- flush 전략 최적화 가능

즉, 변경 감지 비용을 줄일 수 있다.

---

### 6.4 실제로 쓰기를 하면?

```java
@Transactional(readOnly = true)
public void test() {
    Member m = em.find(Member.class, 1L);
    m.setName("changed");
}
```

객체는 변경된다.

하지만:

- flush가 억제될 수 있고
- DB 반영이 되지 않거나
- 예기치 않은 동작이 발생할 수 있다.

따라서 readOnly는 쓰기 금지가 아니라 **쓰기 없음에 대한 계약**이다.

---

## 7. 전체 구조 연결 정리

이제 모든 개념을 연결해보자.

1. 트랜잭션이 시작된다.
2. EntityManager가 생성되고 바인딩된다.
3. 영속성 컨텍스트가 활성화된다.
4. 1차 캐시가 동일성을 보장한다.
5. 스냅샷이 생성된다.
6. Dirty Checking이 변경을 추적한다.
7. Write-Behind로 SQL이 지연 저장된다.
8. flush가 SQL을 전송한다.
9. commit이 트랜잭션을 확정한다.
10. EntityManager가 종료된다.

결국,

> 트랜잭션이 경계를 만들고 영속성 컨텍스트가 객체를 관리하며  
> flush가 DB와 동기화하고 commit이 최종 확정한다.

이 구조를 이해하면 JPA는 **트랜잭션 경계 안에서 객체 상태를 일관되게 관리하기 위해 설계된 메커니즘**이라는 것이 보인다.