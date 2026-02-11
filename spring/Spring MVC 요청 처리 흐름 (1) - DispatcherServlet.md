# Spring MVC 요청 처리 흐름 (1) - DispatcherServlet 

## 1. MVC란 무엇인가

MVC는 **Model–View–Controller**의 약자이다.

- **Model**은 데이터와 비즈니스 상태를 담당한다.
- **View**는 사용자에게 보여질 결과를 담당한다.
- **Controller**는 요청을 받아 전체 흐름을 제어하는 역할을 한다.

Spring MVC는 이 구조를 기반으로 하되, 모든 요청을 하나의 진입점으로 모으는 **Front Controller 패턴**을 사용한다.  
그 중심이 바로 `DispatcherServlet`이다.

---

## 2. 전체 처리 흐름

![Spring MVC 요청 처리 전체 흐름.png](../assets/Spring%20MVC%20%EC%9A%94%EC%B2%AD%20%EC%B2%98%EB%A6%AC%20%EC%A0%84%EC%B2%B4%20%ED%9D%90%EB%A6%84.png)

### 1. 요청은 어디로 들어오는가

클라이언트가 HTTP 요청을 보낸다.  
예를 들어 `GET /orders/1` 같은 요청이다.

이 요청은 가장 먼저 `DispatcherServlet`으로 들어온다.

Spring Boot 환경에서는 모든 요청이 기본적으로 이 서블릿으로 매핑된다.  
즉, 모든 웹 요청은 하나의 진입점을 통해 들어온다.

---

### 2. 어떤 Controller가 처리할지 찾는다

`DispatcherServlet`은 바로 Controller를 호출하지 않는다.  
먼저 `HandlerMapping`에게 묻는다.

> 이 URL을 처리할 Handler는 누구인가?

`HandlerMapping`은 요청 URL과 매핑된 Handler를 찾아 반환한다.

여기서 독특한 점은 Spring은 Controller 대신 Handler라는 표현을 쓴다는 것이다.  
즉, 매핑의 대상은 보통 “Controller 클래스”라기보다 **Controller 메서드**이다.

---

### 3. 찾았다고 바로 실행하지 않는다

Handler를 찾았다고 해서 바로 호출하지는 않는다.

Spring은 다양한 형태의 Handler를 지원한다.  
그래서 **찾아낸 Handler를 실제로 호출 가능한 방식**으로 맞춰주는 계층이 필요하다.

여기서 등장하는 것이 `HandlerAdapter`이다.

`DispatcherServlet`은 `HandlerAdapter`를 통해 Controller 메서드를 실행한다.  
이 구조 덕분에 Spring은 다양한 방식의 Controller를 유연하게 지원할 수 있다.

---

### 4. Controller가 실행된다

이제 실제 Controller 메서드가 호출된다.

이 안에서 보통 다음 작업이 수행된다.

- Service 호출
- Repository 접근
- 데이터 조회 및 가공
- Model 구성

여기까지가 전통적인 MVC 관점에서 “Controller → Model” 영역이다.

---

### 5. Controller는 무엇을 반환하는가

Controller는 보통 다음 중 하나를 반환한다.

- view name (문자열)
- `ModelAndView`
- JSON (`@ResponseBody`)

예를 들어 `return "orderDetail";` 처럼 문자열을 반환할 수 있다.

이 문자열은 아직 실제 View가 아니다.  
단지 **논리적인 view name**이다.

---

### 6. View를 실제 객체로 변환한다

`DispatcherServlet`은 이 view name을 가지고 `ViewResolver`에게 요청한다.

> `orderDetail`이라는 이름을 실제 어떤 View로 만들까?

`ViewResolver`는 이를 실제 `View` 객체로 변환한다.  
예를 들어 템플릿 기반이라면 `/templates/orderDetail.html` 같은 대상으로 이어질 수 있다.

이 단계에서 View가 결정된다.

---

### 7. View가 렌더링된다

이제 View는 Controller가 구성한 Model 데이터를 사용하여 최종 응답을 생성한다.

- HTML 생성
- JSON 직렬화
- 템플릿 렌더링

그리고 완성된 결과가 HTTP Response로 반환된다.

---

## 3. 이 흐름을 구조로 다시 보면

Spring MVC 요청 처리는 크게 세 단계로 나눌 수 있다.

1. **매핑 구조**
    - 어떤 메서드가 이 요청을 처리할지 결정한다.
      (`HandlerMapping`)

2. **실행 구조**
    - Controller를 호출하고 파라미터를 바인딩한다. (`HandlerAdapter`)

3. **응답 구조**
    - 반환값을 View 또는 JSON으로 변환한다. (`ViewResolver`, `HttpMessageConverter`)

이 세 구조 전체를 감싸고, 순서를 조율하는 중심에  
`DispatcherServlet`이 존재한다.

그렇다면 이제 자연스럽게 다음 질문이 나온다.

> 이 모든 흐름을 조율하는 DispatcherServlet은  
> 실제 코드에서는 어떻게 동작하는가?

이제 DispatcherServlet의 구현 코드부터 따라가 보자.


---

## 4. DispatcherServlet 내부 구조

먼저 상속 구조부터 보면 다음과 같다.

~~~java
public class DispatcherServlet extends FrameworkServlet
            ↓
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware
            ↓
public abstract class HttpServletBean extends HttpServlet
            ↓
public abstract class HttpServlet extends GenericServlet
~~~

즉 `DispatcherServlet`은 결국 `HttpServlet` 계열이다.  
그래서 `DispatcherServlet`도 서블릿 생명주기 흐름을 따른다.

- `init()`
- `service()`
- `doGet()`, `doPost()` ...

다만 Spring MVC에서는 `DispatcherServlet`이 Front Controller이기 때문에  
실제로는 MVC 흐름을 타기 위한 내부 디스패치 로직으로 이어진다.

---

### 1. doService → doDispatch로 이어진다

실제로 디버깅해보면, 요청은 `doService()`로 들어오고  
그 안에서 `doDispatch()`가 호출된다.

~~~java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
    try {
        doDispatch(request, response);
    }
    finally {
        // ...
    }
}
~~~

실제 요청 처리는 `doDispatch()`에서 이루어진다.

`doDispatch()`의 Javadoc에는 다음과 같이 적혀 있다.

> Process the actual dispatching to the handler.  
> The handler will be obtained by applying the servlet's HandlerMappings in order.  
> The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters.

핵심은 두 가지다.

1. HandlerMapping을 순서대로 적용해서 handler를 찾는다.
2. 해당 handler를 지원하는 HandlerAdapter를 찾아 실행한다.

즉, DispatcherServlet은 직접 매핑하거나 실행하지 않고 적절한 전략 객체를 찾아 **위임**할 뿐이다.

---

### 2. doDispatch는 getHandler를 통해 handler를 찾는다

`doDispatch()` 내부를 보면, 현재 요청에 대해 handler를 구하는 구간이 있다.

~~~java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HandlerExecutionChain mappedHandler = null;

    // Determine handler for the current request.
    mappedHandler = getHandler(request);

    // ...
}
~~~

결국 **handler를 어떻게 구하느냐**의 핵심은 `getHandler()`이다.

---

### 3. DispatcherServlet.getHandler()는 HandlerMapping을 순회한다

`getHandler()` 코드는 다음과 같다.

~~~java
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
~~~

DispatcherServlet은 초기화 과정에서 여러 `HandlerMapping`들을 등록하고,  
이를 `handlerMappings` 리스트로 관리한다.

따라서 위 코드는 다음 의미가 된다.

1. 등록된 `HandlerMapping`이 있다면 (`handlerMappings != null`)
2. 순서대로 하나씩 돌면서
3. 각 `HandlerMapping`에게 “이 요청을 처리할 handler를 찾아달라”고 요청하고
4. 처음으로 매칭되는 결과를 반환한다.

여기서 반환되는 타입이 `HandlerExecutionChain`인 것도 포인트다.

- handler 자체뿐 아니라
- 적용될 interceptor 목록까지

묶어서 들고 있는 구조다.

---

### 4. 정리

지금까지의 내용을 구조적으로 다시 묶어보면 다음과 같다.

Spring MVC 요청 처리는 겉으로 보면 단순하다.

Client → Controller → View

하지만 실제 내부에서는 이 흐름을 `DispatcherServlet`이 중앙에서 조율한다.

요청이 들어오면,

1. `DispatcherServlet`이 가장 먼저 요청을 받는다.
2. `doService()`에서 DispatcherServlet 전용 request attribute 등 처리 환경을 준비한다.
3. 실제 요청 처리는 `doDispatch()`에서 수행된다.
4. `doDispatch()`는 먼저 `getHandler()`를 호출해 현재 요청을 처리할 handler를 찾는다.
5. `getHandler()`는 등록된 `HandlerMapping`을 순서대로 순회하며, 처음으로 매칭되는 `HandlerExecutionChain`을 반환한다.
6. 이후 단계에서는 반환된 handler를 실행하기 위해 `HandlerAdapter`가 선택되고, 반환값은 응답 전략(`ViewResolver`, `HttpMessageConverter`)에 의해 최종 응답으로 만들어진다.

즉, Spring MVC 요청 처리는

- **매핑(누가 처리할지 결정)**,
- **실행(어떻게 호출할지 처리)**,
- **응답(어떤 형식으로 응답할지 결정)**

세 구조가 협력하는 흐름이고,  
이 전체 순서를 조율하는 중심이 `DispatcherServlet`이다.

다음 장에서는 `HandlerMapping` 내부에서 URL에 맞는 handler(HandlerMethod)가 어떻게 선택되는지를 이어서 살펴볼 것이다.

다음 장: [Spring MVC 요청 처리 흐름 (2) - MappingHandler.md](Spring%20MVC%20%EC%9A%94%EC%B2%AD%20%EC%B2%98%EB%A6%AC%20%ED%9D%90%EB%A6%84%20%282%29%20-%20MappingHandler.md)