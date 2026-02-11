# Spring MVC 요청 처리 흐름 (2) - HandlerMapping


이전 글에서는 DispatcherServlet이 어떻게 요청을 받고,  
getHandler()를 통해 적절한 handler를 찾는지까지 살펴봤다.

이번 글에서는 그 다음 단계 **HandlerMapping은 어떻게 요청에 맞는 Controller 메서드를 선택하는가?** 이 질문에 집중해본다.

## 1. HandlerMapping의 역할

`HandlerMapping`의 역할은 단 하나다.

- **현재 요청을 처리할 handler를 찾아 반환한다.**

인터페이스는 매우 단순하다.

~~~java
public interface HandlerMapping {
    HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
~~~

반환 타입이 `HandlerExecutionChain`인 것이 포인트다.

- 실행할 handler(대부분 `HandlerMethod`)
- 적용될 `HandlerInterceptor` 목록

을 함께 묶어서 반환한다.

---

## 2. 대표 구현체: RequestMappingHandlerMapping

Spring MVC에서 우리가 가장 흔히 사용하는 매핑은 `@RequestMapping` 기반이다.  
이를 담당하는 구현체는 `RequestMappingHandlerMapping`이다.

상속 구조는 다음과 같다.

~~~java
public class RequestMappingHandlerMapping extends RequestMappingInfoHandlerMapping
            ↓
public abstract class RequestMappingInfoHandlerMapping extends AbstractHandlerMethodMapping<RequestMappingInfo>
            ↓
public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping
            ↓
public abstract class AbstractHandlerMapping implements HandlerMapping
~~~

애노테이션 기반 메서드 매핑의 핵심 로직은 `AbstractHandlerMethodMapping` 클래스에 있다.

---

## 3. HandlerMapping의 공통 흐름: getHandler → getHandlerInternal

`HandlerMapping` 구현체들은 공통적으로 다음 흐름을 따른다.

- `getHandler(request)` 호출
- 내부에서 `getHandlerInternal(request)`를 통해 handler 탐색
- handler를 `HandlerExecutionChain`으로 감싸 반환

대표적인 구조는 이런 형태다.

~~~java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    Object handler = getHandlerInternal(request);
    // ...
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
    // ...
    return executionChain;
}
~~~

여기서 핵심은 **handler를 실제로 찾는 로직**이 들어있는 `getHandlerInternal()`이다.

---

## 4. getHandlerInternal: lookupPath 추출과 조회 진입

`@RequestMapping` 기반 매핑은 `AbstractHandlerMethodMapping<T>`의 흐름을 탄다.  
여기서 `getHandlerInternal()`은 아래처럼 동작한다.

~~~java
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
    String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);

    this.mappingRegistry.acquireReadLock();
    try {
        HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
        return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
    }
    finally {
        this.mappingRegistry.releaseReadLock();
    }
}
~~~

여기서 중요한 포인트는 세 가지다.

1. request로부터 `lookupPath`를 만든다.
2. 매핑 정보를 담고 있는 `mappingRegistry`에 read lock을 걸고 조회한다.
3. `lookupHandlerMethod(lookupPath, request)`로 최종 handler를 찾는다.

즉, 실제 매칭의 중심은 `lookupHandlerMethod()`에 있다.

---

## 5. MappingRegistry: 매핑 정보가 저장되는 곳

`MappingRegistry`는 요청과 컨트롤러 메서드의 연결 정보를 미리 저장해두는 내부 저장소다.

Spring은

- 애플리케이션 시작 시점에 Controller를 스캔하면서
- `@RequestMapping` 메타데이터를 읽고
- “요청 조건 -> HandlerMethod” 형태로 매핑 정보를 등록해둔다.

런타임(요청 처리 시점)에는 이 정보를 기반으로 조회가 이뤄진다.

`AbstractHandlerMethodMapping` 내부에는 다음과 같은 필드가 존재한다.

```java
private final MappingRegistry mappingRegistry = new MappingRegistry();
```

MappingRegistry 내부의 핵심 구조는 Map이다.

```java
class MappingRegistry {
    private final Map<T, MappingRegistration<T>> registry = new HashMap<>();
}
```

여기서 T는 보통 `RequestMappingInfo`다.  
즉 매핑 정보는 `RequestMappingInfo` -> `HandlerMethod` 형태로 저장된다.

실제 등록 흐름은 다음과 같다.

```java
protected void registerHandlerMethod(Object handler, Method method, T mapping) {
    this.mappingRegistry.register(mapping, handler, method);
}
```

그리고 내부적으로 다음처럼 저장된다.

```java
public void register(T mapping, Object handler, Method method) {
    HandlerMethod handlerMethod = createHandlerMethod(handler, method);
    this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod));
}
```

여기서 요청 조건은 단순히 URL 문자열만 의미하지 않는다.

RequestMappingInfo 안에는 다음 조건이 함께 포함된다.

- path 패턴
- HTTP method
- params / headers 조건
- consumes / produces 조건

즉 단순한 URL 문자열 매핑이 아니라
**요청 조건 묶음 -> 실행 메서드**
형태의 Map 구조로 메모리에 저장된다.

---

## 6. lookupHandlerMethod: 후보 탐색과 최종 선택

`lookupHandlerMethod()`의 역할은 다음과 같다.

1. `lookupPath` 기준으로 일치 가능한 후보들을 찾는다.
2. HTTP method 등 추가 조건으로 후보를 좁힌다.
3. 여러 후보가 남으면 매칭 조건의 구체성을 비교한다.
4. 단 하나로 결정 가능할 경우 해당 `HandlerMethod`를 반환한다.

즉 매칭은 단순히 문자열 equals가 아니라,

- path pattern 매칭
- HTTP method 비교
- params / headers 조건 비교
- consumes / produces 조건 비교
- 매핑의 구체성(specificity) 비교

를 거쳐 후보를 점점 좁혀가는 과정이다.

중요한 점은 다음과 같다.

- 매칭 후보가 하나도 없으면 → 404
- 여러 후보가 남았는데 우열을 가릴 수 없으면 → Ambiguous mapping 예외 발생
- 비교를 통해 단 하나로 결정 가능할 때만 최종 handler가 선택된다

즉 Spring은 명확한 비교 규칙에 따라 단 하나로 결정 가능할 때만 handler를 반환한다.

## 7. HandlerMethod와 HandlerExecutionChain: 최종 산출물

HandlerMapping이 최종적으로 선택하는 handler는 대개 `HandlerMethod`다.

`HandlerMethod`는 단순히 Method 객체 하나가 아니라,

- 어떤 bean의 어떤 메서드인지
- 파라미터/애노테이션 메타데이터
- 실행을 위한 정보

를 포함하는 실행 단위다.

그리고 HandlerMapping의 반환값은 `HandlerExecutionChain`이며,

- 선택된 handler(`HandlerMethod`)
- 적용할 interceptor 목록

까지 포함한 실행 체인이다.

## 8. 정리

HandlerMapping 단계에서 일어나는 일을 한 줄로 정리하면 다음과 같다.

- 시작 시점에 `@RequestMapping` 기반 매핑 정보를 등록해두고
- 요청이 들어오면 `lookupPath`와 조건 비교를 통해
- 최종 `HandlerMethod`를 선택한 뒤
- interceptor를 포함한 `HandlerExecutionChain`으로 반환한다.

[다음 장](Spring%20MVC%20%EC%9A%94%EC%B2%AD%20%EC%B2%98%EB%A6%AC%20%ED%9D%90%EB%A6%84%20%283%29%20-%20HandlerAdapter.md)에서는 선택된 `HandlerMethod`를 실제로 실행하는 과정인 `HandlerAdapter`와 `ArgumentResolver` 중심의 **실행 구조**를 다룰 예정이다.
