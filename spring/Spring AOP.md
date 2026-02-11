# Spring AOP

## 1. AOP란 무엇인가

AOP(Aspect Oriented Programming)는 공통 관심사를 분리하기 위한 프로그래밍 패러다임이다.

객체 지향 프로그래밍은 클래스와 객체 중심으로 책임을 나눈다.  
하지만 실제 애플리케이션에서는 특정 로직이 여러 계층에 반복적으로 등장한다.

대표적인 예시는 다음과 같다.

- 로깅
- 트랜잭션
- 권한 검사
- 예외 처리
- 성능 측정
- 캐싱

이 로직들은 특정 도메인의 핵심 기능이 아니다.  
그러나 여러 클래스에 반복적으로 적용된다.

이를 **횡단 관심사(Cross-cutting Concern)** 라고 한다.

---

## 2. 횡단 관심사와 핵심 관심사

핵심 관심사(Core Concern)

- 도메인 로직 그 자체
- 주문 생성, 결제 처리 같은 실제 비즈니스 기능

횡단 관심사(Cross-cutting Concern)

- 특정 도메인에 종속되지 않음
- 여러 계층과 여러 클래스에 공통 적용됨

![횡단관심사.png](../assets/%ED%9A%A1%EB%8B%A8%EA%B4%80%EC%8B%AC%EC%82%AC.png)

핵심 관심사는 수직 구조로 배치되지만  
횡단 관심사는 이 구조를 가로지른다.

---

## 3. AOP가 해결하려는 문제

공통 로직이 비즈니스 로직을 침범하면 다음과 같은 코드가 된다.

```java
@Service
public class OrderService {

    public void createOrder(Order order) {

        log.info("주문 시작");

        try {
            orderRepository.save(order);  // 핵심 로직
            log.info("주문 완료");
        } catch (Exception e) {
            log.error("주문 실패", e);
            throw e;
        }
    }
}
```

핵심 로직은 한 줄이지만 공통 로직이 이를 감싸고 있다.

이 구조는

- 코드 중복
- 응집도 저하
- 정책 변경 시 다중 수정
- 비즈니스 로직 가독성 저하

를 유발한다.

AOP는 이 결합을 구조적으로 끊는다.

---

## 4. AOP의 핵심 개념

### Aspect

횡단 관심사를 하나의 모듈로 묶은 단위이다.

Spring에서는 `@Aspect`가 붙은 클래스가 이에 해당하며,  
내부적으로는 [`AnnotationAwareAspectJAutoProxyCreator`](https://github.com/spring-projects/spring-framework/blob/main/spring-aop/src/main/java/org/springframework/aop/aspectj/annotation/AnnotationAwareAspectJAutoProxyCreator.java)
가 이를 분석해 Advisor로 변환한다.

Aspect는 **어디에(Pointcut) 어떤 로직(Advice)을 적용할지 선언하는 단위**이다.



### Advice

실제로 실행될 공통 로직이다.

Spring에서는 내부적으로 `MethodInterceptor` 기반으로 실행되며,  
실제 호출 흐름은 [`ReflectiveMethodInvocation`](https://github.com/spring-projects/spring-framework/blob/main/spring-aop/src/main/java/org/springframework/aop/framework/ReflectiveMethodInvocation.java)
의 `proceed()`에서 제어된다.

Advice는 **JoinPoint 실행 흐름에 삽입되는 코드 조각**이다.


### Pointcut

Advice를 적용할 대상을 정의하는 조건식이다.

Spring에서는 [`Pointcut`](https://github.com/spring-projects/spring-framework/blob/main/spring-aop/src/main/java/org/springframework/aop/Pointcut.java)
인터페이스로 표현된다.

Pointcut은 **어떤 JoinPoint에 Advice를 연결할지 결정하는 필터**이다.


### JoinPoint

Advice가 개입할 수 있는 실행 지점이다.

Spring AOP는 프록시 기반이므로 기본적으로 **메서드 실행 지점만 JoinPoint로 사용한다.**


### Target

실제 비즈니스 로직 객체이다.

AOP 적용 이전의 원본 객체이며 Proxy에 의해 호출이 위임되는 대상이다.


### Proxy

Target을 감싸 실행 흐름을 제어하는 객체이다.

AOP의 기본 실행 구조는 다음과 같다.

```
Client → Proxy → Target
```

---

## 5. AOP 적용 예제

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("execution(* com.example.service..*(..))")
    public Object logging(ProceedingJoinPoint joinPoint) throws Throwable {

        log.info("start");

        try {
            return joinPoint.proceed();
        } finally {
            log.info("end");
        }
    }
}
```

```java
@Service
public class OrderService {

    public void createOrder(Order order) {
        orderRepository.save(order);
    }
}
```

서비스 코드에는 핵심 로직만 남고 공통 로직은 실행 흐름에 의해 자동 적용된다.

---

## 6. Spring AOP의 내부 구조

Spring AOP는 **프록시 기반 구조**로 동작한다.

개념과 내부 객체 구조를 연결하면 다음과 같다.

```
@Aspect
   ↓
Advisor (Pointcut + Advice)
   ↓
MethodInterceptor
   ↓
Proxy
   ↓
Target
```

- Aspect는 Advisor로 변환된다.
- Advisor는 Pointcut + Advice를 포함한다.
- Advice는 MethodInterceptor 형태로 실행된다.
- Proxy가 이 실행을 감싼다.

---

## 7. Bean 생성 이후, AOP 적용 판단

Spring은 Bean 인스턴스 생성과 초기화(init)까지 마친 뒤,
`applyBeanPostProcessorsAfterInitialization()` 단계에서 등록된 `BeanPostProcessor`들을 순회한다.

![Bean 생성 시점.png](../assets/Bean%20%EC%83%9D%EC%84%B1%20%EC%8B%9C%EC%A0%90.png)

핵심은 다음 루프이다.

- `getBeanPostProcessors()`로 모든 BeanPostProcessor를 가져오고
- 각 Processor의 `postProcessAfterInitialization(result, beanName)`을 호출한다.
- 이때 반환된 객체가 다음 Processor의 입력으로 넘어간다.
- 최종적으로 반환된 `result`가 “해당 Bean의 최종 결과물”이 된다.

즉, 이 지점은 단순히 후처리 단계가 아니라 **Bean 객체를 다른 객체로 교체할 수 있는 지점**이다.

AOP는 여기서 개입한다.

- AOP 관련 BeanPostProcessor가 `postProcessAfterInitialization()`에서 동작하며
- 현재 Bean이 Advisor(Pointcut + Advice) 적용 대상인지 검사한다.
- 적용 대상이라면 원본 Bean 대신 Proxy를 만들어 반환한다.


---

## 8. Proxy 생성 지점

AOP 적용 대상으로 판단되면, AutoProxyCreator는 내부에서  
`wrapIfNecessary(bean, beanName, cacheKey)`를 통해 프록시 생성 여부를 최종 결정한다.

![Proxy 생성 시점.png](../assets/Proxy%20%EC%83%9D%EC%84%B1%20%EC%8B%9C%EC%A0%90.png)

`wrapIfNecessary()` 흐름은 크게 두 단계로 정리된다.

### 1) 적용할 Advisor(Interceptor) 목록 조회

```java
Object[] specificInterceptors =
        getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
```

- 이 Bean에 적용 가능한 Advisor(Pointcut + Advice)를 조회한다.
- 결과가 `DO_NOT_PROXY`이면 프록시를 생성하지 않고 원본 Bean을 반환한다.


### 2) 프록시 생성 및 반환

```java
Object proxy = createProxy(
        bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
return proxy;
```

- `createProxy(...)` 호출 결과로 Proxy 객체가 생성되어 반환된다.
- `SingletonTargetSource(bean)`는 프록시가 위임할 Target이 현재 bean임을 의미한다.


### “Spring 컨테이너에 등록된다”의 의미

“원본 Bean 대신 Proxy가 컨테이너에 등록된다”는 표현은 다음을 의미한다.

- `wrapIfNecessary()`가 Proxy를 반환하면
- 그 반환값이 `applyBeanPostProcessorsAfterInitialization()` 흐름을 통해 상위로 전파되고
- 최종적으로 Spring 컨테이너가 보관하고 외부에 노출하는 Bean 인스턴스가 Proxy로 확정된다.

즉, “컨테이너에 Proxy가 등록된다”는 말은 **BeanPostProcessor 체인의 반환값이 최종 Bean 인스턴스로 확정된다**는 뜻이다.

---

## 9. 메서드 호출 시점

이후 메서드 호출 시 흐름은 다음과 같다.

```
Client
   → Proxy
       → Advisor(Advice) 실행
           → Target 호출
```

Advice는 내부적으로 `MethodInterceptor` 형태로 실행된다.  
실행 제어는 `ReflectiveMethodInvocation.proceed()`가 담당한다.

즉,
- Bean 생성 단계에서 구조가 준비되고
- 호출 단계에서 Advice가 실행된다.

---

## 10. 생성 시점과 호출 시점 정리

Spring AOP는 두 단계로 동작한다.

### Bean 생성 시점
- Aspect 분석
- Advisor 생성
- Pointcut 매칭
- Proxy 생성

### 메서드 호출 시점
- Proxy가 호출 가로챔
- Advice 실행
- Target 호출

---

## 11. 정리

AOP는 횡단 관심사를 분리하는 기술이 아니라 **실행 구조를 제어하는 설계 방식**이다.

Spring AOP는 이를 프록시 기반으로 구현한다.

- 생성 단계에서 Proxy를 준비하고
- 호출 단계에서 실행 흐름을 제어한다.

Spring AOP의 본질은 실행 구조를 외부에서 조립하는 메커니즘에 있다.
