# Spring MVC 요청 처리 흐름 (4) - ViewResolver


이전 글에서는 `HandlerAdapter`가 선택된 `HandlerMethod`를 실행하고,  
`ArgumentResolver`로 파라미터를 생성한 뒤  
`ReturnValueHandler`를 통해 반환값을 처리하는 구조까지 살펴봤다.

이번 글에서는 그 다음 단계, **Controller의 반환값은 어떻게 실제 HTTP 응답으로 변환되는가?** 이 질문에 집중해본다.



## 1. 실행 이후 남는 것: 반환값

Controller 메서드 실행이 끝나면  
Spring에 남아 있는 것은 하나다.

- 반환 객체 (return value)

이 반환값은 아직 HTTP 응답이 아니다.  
자바 객체일 뿐이다.

이 객체를 **실제 응답 바디 또는 View 렌더링 결과로 변환하는 과정**이 필요하다.

이 과정은 크게 두 갈래로 나뉜다.

1. View 기반 응답 (템플릿 렌더링)
2. Body 기반 응답 (JSON 등 직렬화)

---

## 2. View 기반 응답 흐름

Controller가 다음과 같이 반환한다고 가정하자.

~~~java
@GetMapping("/orders")
public String orderPage(Model model) {
    model.addAttribute("name", "spring");
    return "orderView";
}
~~~

여기서 `"orderView"`는 실제 HTML이 아니다.  
**논리적인 view name**이다.

### 흐름은 다음과 같다.

1. `HandlerAdapter`가 반환값을 `ModelAndView`로 정리한다.
2. `DispatcherServlet`이 `ViewResolver`를 호출한다.
3. `ViewResolver`가 view name을 실제 View 객체로 변환한다.
4. View가 Model 데이터를 사용해 렌더링한다.
5. 렌더링 결과가 HTTP Response로 작성된다.

---

## 3. ViewResolver는 무엇을 하는가

ViewResolver의 역할은 **논리적 view name → 실제 View 객체로 변환**이다.

대표 인터페이스는 다음과 같다.

~~~java
public interface ViewResolver {
    View resolveViewName(String viewName, Locale locale) throws Exception;
}
~~~

예를 들어 Thymeleaf를 사용하는 경우,

- `"orderView"`  
  → `/templates/orderView.html`

처럼 실제 템플릿 파일로 연결된다.

즉 ViewResolver는  
“어디에 있는 어떤 View를 사용할 것인가”를 결정하는 전략 객체다.

---

## 4. Body 기반 응답 흐름 (@ResponseBody)

이번에는 JSON 응답을 보자.

~~~java
@GetMapping("/users/{id}")
@ResponseBody
public UserResponse getUser(@PathVariable Long id) {
    return new UserResponse(id, "spring");
}
~~~

이 경우 ViewResolver는 호출되지 않는다.

왜냐하면 `@ResponseBody`가 붙어 있기 때문이다.

이 어노테이션은 반환값을 View로 해석하지 말고 **HTTP Body에 직접 쓰라는 의미**다.

---

## 5. HttpMessageConverter: 객체 → HTTP Body 변환기

JSON 응답을 처리하는 핵심 전략은 `HttpMessageConverter`다.

대표 인터페이스는 다음과 같다.

~~~java
public interface HttpMessageConverter<T> {

    boolean canWrite(Class<?> clazz, MediaType mediaType);

    void write(T t, MediaType contentType, HttpOutputMessage outputMessage)
            throws IOException;
}
~~~

동작 흐름은 다음과 같다.

1. 반환 객체 타입 확인
2. 요청의 Accept 헤더 확인
3. `canWrite()`가 true인 Converter 선택
4. `write()`를 호출해 실제 바이트로 직렬화
5. HTTP Response Body에 기록

JSON의 경우 보통 `MappingJackson2HttpMessageConverter`가 선택된다.

즉,

UserResponse 객체  
→ Jackson 직렬화  
→ JSON 문자열  
→ HTTP Body 작성

이라는 흐름을 따른다.

---

## 6. View 방식 vs Body 방식의 차이

두 방식의 차이를 정리하면 다음과 같다.

### View 기반

- 반환값: view name 또는 ModelAndView
- 처리 전략: ViewResolver
- 결과: 템플릿 렌더링된 HTML

### Body 기반

- 반환값: 객체
- 처리 전략: HttpMessageConverter
- 결과: JSON / XML 등 직렬화된 데이터

---

## 7. DispatcherServlet에서의 응답 처리 위치

`doDispatch()` 내부에서 실행 이후 흐름은 대략 다음과 같다.

~~~java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    
    // HandlerAdapter를 통해 실제 Controller 메서드 호출
    ModelAndView mv = ha.handle(request, response, mappedHandler.getHandler());
    
    // 반환값을 해석하여 최종 HTTP 응답 생성
    processDispatchResult(request, response, mappedHandler, mv, null);
}
~~~

`processDispatchResult()` 내부에서

- ViewResolver를 통한 View 결정
- View.render()
- 또는 HttpMessageConverter를 통한 Body 작성

이 수행된다.

즉, 응답 처리 역시 `DispatcherServlet`이 직접 모든 것을 처리하는 것이 아니라 전략 객체에 위임하는 구조다.

---

## 8. 응답 구조 한 줄 정리

응답 단계는 다음과 같이 이해하면 된다.

1. Controller 반환값 확보
2. `ReturnValueHandler`가 반환 타입 해석
3. View 방식인지 Body 방식인지 결정
4. `ViewResolver` 또는 `HttpMessageConverter` 실행
5. 최종 HTTP Response 생성

---

## 9. 정리

Spring MVC의 응답 구조는

- View 기반 렌더링 전략
- 메시지 변환 기반 직렬화 전략

두 전략으로 나뉜다.

그리고 이것 또한 전략 객체 위임 구조와 확장 가능한 플러그인 방식으로 설계되어 있다.

이로써

1. 매핑 구조 (HandlerMapping)
2. 실행 구조 (HandlerAdapter)
3. 응답 구조 (ViewResolver)

Spring MVC 요청 처리의 전체 흐름이 완성된다.  
Spring MVC 호출 스택을 총정리하면 다음과 같다.

### Spring MVC 호출 스택
```less
HttpServlet.service()
  └── FrameworkServlet.service()
        └── FrameworkServlet.doGet()/doPost()
              └── DispatcherServlet.doService()
                    └── DispatcherServlet.doDispatch()  // (1)
                          ├── getHandler()
                          │     └── HandlerMapping.getHandler() // 누가 처리할지 결정 (2)
                          │           └── AbstractHandlerMethodMapping.getHandlerInternal()
                          │                 └── lookupHandlerMethod()
                          ├── applyPreHandle()  // Pre-Interceptor
                          ├── getHandlerAdapter()
                          ├── HandlerAdapter.handle() // 어떻게 실행할지 결정/실행 (3)
                          │     └── RequestMappingHandlerAdapter.handleInternal()
                          │           └── invokeHandlerMethod()
                          │                 └── ServletInvocableHandlerMethod.invokeAndHandle()
                          │                       ├── ArgumentResolver.resolveArgument()
                          │                       ├── Controller Method invoke()
                          │                       └── ReturnValueHandler.handleReturnValue()
                          ├── applyPostHandle()  // Post-Interceptor
                          ├── processDispatchResult() // 응답 생성 (4)
                          │     ├── HandlerExceptionResolver // 예외 발생 시
                          │     ├── render()
                          │     │     ├── ViewResolver.resolveViewName()
                          │     │     └── View.render()
                          │     │
                          │     └── HttpMessageConverter.write() // @ResponseBody인 경우
                          └── applyAfterCompletion()  // After-Interceptor

```

아래에서 인터셉터와 관련한 내용은 [Interceptor.md](링크주소)에서 확인할 수 있다.
