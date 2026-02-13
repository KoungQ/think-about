# ThreadLocal 완전 정리

---

## 1. ThreadLocal이 왜 필요한가

멀티 스레드 환경에서는 각 요청이 **서로 다른 스레드**에서 처리된다.

Spring MVC에서 요청 하나가 들어오면:

- Tomcat이 스레드를 하나 할당하고
- 그 스레드 안에서 Controller → Service → Repository까지 실행된다.

이때 이런 요구가 생긴다.

- 현재 로그인한 사용자 정보를 어디서든 꺼내 쓰고 싶다
- 현재 요청의 traceId를 로깅에 자동으로 넣고 싶다
- 트랜잭션 정보를 요청 단위로 유지하고 싶다

이 정보를 모든 메서드에 파라미터로 넘기면 코드가 오염된다.  
그래서 등장한 것이 **ThreadLocal**이다.

---

## 2. ThreadLocal이란 무엇인가

ThreadLocal은 **스레드마다 독립적인 값을 저장할 수 있게 해주는 저장소**다.

일반 `static` 변수는 모든 스레드가 공유한다.
~~~java
private static String value;
~~~

반면 ThreadLocal은 다음과 같이 선언한다.

~~~java
private static final ThreadLocal<String> value = new ThreadLocal<>();
~~~

스레드 A가 `set("A")`  
스레드 B가 `set("B")`

를 해도 서로 영향을 주지 않는다.  
각 스레드 내부에 따로 저장된다.

---

## 3. 내부 동작 원리

ThreadLocal이 값을 **자기 자신** 안에 저장하는 게 아니다.

실제로는 다음 구조다.

- `Thread` 객체 내부에 `ThreadLocalMap`이 있고
- 그 Map에 `(ThreadLocal 인스턴스 → 값)` 형태로 저장된다.

즉,

- ThreadLocal은 key 역할
- 실제 값은 Thread 안에 저장된다

그래서 스레드가 다르면 값도 다르다.

---

## 4. 기본 사용 방법

### 값 저장

~~~java
ThreadLocal<String> userContext = new ThreadLocal<>();
userContext.set("kim");
~~~

### 값 조회

~~~java
String user = userContext.get();
~~~

### 값 제거 (매우 중요)

~~~java
userContext.remove();
~~~

---

## 5. remove()를 반드시 해야 하는 이유

ThreadLocal을 잘못 쓰면 **메모리 누수** 또는 **요청 간 데이터 오염**이 발생한다.  
특히 **스레드 풀 환경**에서 위험하다.

Tomcat은 스레드를 재사용한다.

- 요청 A 처리 → Thread-1 사용 → 값 set()
- 요청 A 종료
- 요청 B 처리 → Thread-1 재사용

만약 remove()를 안 하면 요청 A의 값이 요청 B에서 그대로 남아있을 수 있다.

그래서 반드시 다음 패턴을 지켜야 한다.

~~~java 
try {
    userContext.set("kim");
    // business logic
} finally {
    userContext.remove();
}
~~~

---

## 6. Spring에서 ThreadLocal이 쓰이는 곳

### 1) 트랜잭션 관리

`@Transactional`은 내부적으로 ThreadLocal을 활용해  
현재 트랜잭션/커넥션/동기화 정보를 **스레드 단위로 유지**한다.

같은 요청 흐름(같은 스레드)에서는

- 같은 트랜잭션
- 같은 커넥션

을 재사용할 수 있는 기반이 된다.

### 2) SecurityContextHolder

Spring Security는 현재 인증 정보를 ThreadLocal에 저장한다.  
그래서 어디서든 다음처럼 꺼낼 수 있다.

~~~java
SecurityContextHolder.getContext().getAuthentication();
~~~

### 3) RequestContextHolder

현재 요청의 `HttpServletRequest` 같은 정보를  
ThreadLocal로 보관하고 필요할 때 꺼내 쓴다.

---

## 7. ThreadLocal vs static 변수

| 구분 | static 변수 | ThreadLocal |
|------|-------------|------------|
| 스레드 간 공유 | 공유됨 | 공유되지 않음 |
| 요청 단위 데이터 | 불가능 | 가능 |
| 멀티스레드 안전성 | 위험 | 안전 |
| 누수/오염 위험 | 상대적으로 낮음 | remove 안 하면 위험 |

---

## 8. ThreadLocal의 한계

### 1) 비동기 환경에서는 깨진다

ThreadLocal은 “스레드 로컬”이다.  
즉 스레드가 바뀌면 값이 전달되지 않는다.

- `@Async`
- `CompletableFuture`
- WebFlux (Reactor)

이런 환경에서는 ThreadLocal이 기대대로 동작하지 않는다.

WebFlux는 ThreadLocal 대신 **Reactor Context**를 사용한다.

### 2) 숨겨진 의존성을 만든다

ThreadLocal은 편리하지만

- 메서드 시그니처에 드러나지 않는 의존성이 생기고
- 테스트/디버깅 난이도가 올라간다

그래서 “요청 단위로 정말 공유해야 하는 값”에만 제한적으로 쓰는 게 좋다.

---

## 9. 실무 예시 코드

요청 단위 사용자 컨텍스트 구현 예시

~~~java
public class UserContextHolder {

    private static final ThreadLocal<Long> userIdHolder = new ThreadLocal<>();

    public static void set(Long userId) {
        userIdHolder.set(userId);
    }

    public static Long get() {
        return userIdHolder.get();
    }

    public static void clear() {
        userIdHolder.remove();
    }
}
~~~

Filter에서 세팅

~~~ java
try {
    UserContextHolder.set(loginUserId);
    chain.doFilter(request, response);
} finally {
    UserContextHolder.clear();
}
~~~

Service 어디서든 사용 가능

~~~java
Long userId = UserContextHolder.get();
~~~

---

## 10. 정리

ThreadLocal은 요청 단위 상태를 스레드 단위로 저장해서, 계층을 가로질러 공유하기 위한 도구다.

그리고 Spring의 트랜잭션, 보안, 요청 컨텍스트 같은 핵심 기능이  
이 개념을 기반으로 동작한다.
