# Spring MVC 요청 처리 흐름 (3) - HandlerAdapter


이전 글에서는 `DispatcherServlet`이 요청을 받아 `doDispatch()`로 진입하고,  
`getHandler()`를 통해 이 요청을 누가 처리할지를 결정하는 지점까지 살펴봤다.

이번 글에서는 그 다음 단계인 **선택된 handler를 Spring은 어떻게 실제로 호출하는가?** 에 집중해본다.



## 1. 왜 HandlerAdapter가 필요한가

`DispatcherServlet`은 handler를 직접 실행하지 않는다.

Spring MVC는 다양한 형태의 handler를 지원한다.

- `@RequestMapping` 기반 메서드
- `HttpRequestHandler`
- `Controller` 인터페이스 기반 구현체

handler마다 타입과 호출 방식이 다르기 때문에  
공통 실행 지점이 필요하다.  
그 역할을 하는 것이 `HandlerAdapter`다.

핵심 역할은 하나다.

- **주어진 handler를 실행 가능한 방식으로 호출한다**

---

## 2. HandlerAdapter 인터페이스

~~~java
public interface HandlerAdapter {

    boolean supports(Object handler);

    ModelAndView handle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) throws Exception;
}
~~~

- `supports(handler)`  
  → 이 어댑터가 해당 handler를 처리할 수 있는지 판단한다.

- `handle(...)`  
  → 실제 handler를 실행한다.

`DispatcherServlet`은 등록된 `HandlerAdapter`들을 순회하면서  
`supports()`가 true인 구현체를 찾고,  
해당 어댑터의 `handle()`을 호출한다.

---

## 3. DispatcherServlet 내부 실행 흐름

`doDispatch()` 내부에서 handler 실행 흐름은 다음과 같다.

~~~java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {

    HandlerExecutionChain mappedHandler = getHandler(request);

    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

    ModelAndView mv = ha.handle(request, response, mappedHandler.getHandler());

    // 이후 View 처리 로직 ...
}
~~~

실행 흐름은 다음과 같다.

1. 현재 handler를 처리할 수 있는 `HandlerAdapter`를 찾는다.
2. 해당 Adapter에게 실행을 위임한다.
3. 실행 결과를 `ModelAndView` 형태로 받는다.

즉 실행의 주체는 `DispatcherServlet`이 아니라 `HandlerAdapter`다.

---

## 4. 대표 구현체: RequestMappingHandlerAdapter

우리가 사용하는 `@RequestMapping`, `@GetMapping` 기반 Controller 실행은  
`RequestMappingHandlerAdapter`가 담당한다.

이 클래스는 단순히 리플렉션으로 메서드를 호출하는 역할만 하지 않는다.  
실행 과정은 내부적으로 세 단계로 나뉜다.

1. **파라미터 생성**
2. **Controller 메서드 호출**
3. **반환값 처리**

이 과정을 가능하게 하는 두 가지 핵심 전략이 있다.

- `HandlerMethodArgumentResolver`
- `HandlerMethodReturnValueHandler`

---

## 5. ArgumentResolver: 파라미터는 어떻게 채워지는가

예를 들어 Controller 메서드가 다음과 같다고 하자.

~~~java
@GetMapping("/users/{id}")
public UserResponse getUser(
        @PathVariable Long id,
        @RequestParam String name,
        HttpServletRequest request) {
    // ...
}
~~~

`id`, `name`, `request` 같은 파라미터는 자동으로 채워진다.  
이 역할을 하는 것이 `HandlerMethodArgumentResolver`다.

`RequestMappingHandlerAdapter` 내부에는 여러 Resolver를 모아둔 Composite 구조가 있다.  
실행 시에는 다음 흐름을 따른다.

1. 메서드 파라미터 목록을 순회한다.
2. 각 파라미터마다
    - `supportsParameter()`가 true인 resolver를 찾는다.
    - `resolveArgument()`를 호출해 실제 값을 생성한다.

즉 파라미터 바인딩은 **전략 객체 리스트를 순회하는 구조**로 이루어진다.  
이 과정이 끝나면 Controller 메서드를 호출하기 위한 인자 배열이 완성된다.

---

## 6. 실제 메서드 호출

파라미터가 모두 준비되면  
`RequestMappingHandlerAdapter`는 내부적으로 `HandlerMethod`를 감싼 실행 객체를 통해  
리플렉션으로 Controller 메서드를 호출한다.

이 단계는 비교적 단순하다.

- 준비된 인자 배열을 전달한다.
- 실제 Controller 메서드를 invoke 한다.
- 반환값을 받는다.

이제 남은 단계는 반환값 처리다.

---

## 7. ReturnValueHandler: 반환값은 어떻게 처리되는가

Controller 반환 타입은 다양하다.

- `String` (view name)
- `ModelAndView`
- 객체 (`@ResponseBody`)
- `ResponseEntity`
- `void`

이 반환값을 처리하는 것이 `HandlerMethodReturnValueHandler`다.

동작 흐름은 ArgumentResolver와 유사하다.

1. 반환 타입을 확인한다.
2. `supportsReturnType()`가 true인 handler를 찾는다.
3. `handleReturnValue()`로 최종 처리를 수행한다.

예를 들어:

- `String` → view name 처리
- `ModelAndView` → 그대로 사용
- `@ResponseBody` → 내부적으로 `HttpMessageConverter` 호출

즉 반환값 처리 역시  
전략 객체 기반 위임 구조다.

---

## 8. 실행 구조 한 줄 정리

실행 구조는 다음 순서로 이해하면 된다.

1. `HandlerMapping`이 `HandlerMethod`를 선택한다.
2. `DispatcherServlet`이 `HandlerAdapter`를 선택한다.
3. `HandlerAdapter`가 `ArgumentResolver`로 파라미터를 생성한다.
4. Controller 메서드를 호출한다.
5. `ReturnValueHandler`가 반환값을 처리한다.

---

## 9. 정리

실행 구조는 단순한 메서드 호출이 아니다.

- 어떤 Adapter가 실행할지 결정하고
- 파라미터를 전략적으로 생성하고
- 반환값을 전략적으로 처리하는

확장 가능한 실행 파이프라인 구조다.

다음 장에서는 반환값이 실제 HTTP 응답으로 변환되는 과정인 **응답 구조**를 다룰 예정이다.

다음 장: 
