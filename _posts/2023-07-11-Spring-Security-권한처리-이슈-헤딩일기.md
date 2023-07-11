---
title: 'Spring Security 권한처리 이슈 헤딩일기'
categories:
    - Spring
tags:
    - Spring 
    - Security
    - Authorization
toc: true
toc_sticky: true
---

### 들어가며

<p align="center">
  <img src="https://github.com/f-lab-edu/infrun/assets/40778768/9bd1a233-5832-41ba-8563-41dac223e4e4" width="600"/>
</p>

현재 진행 중인 팀 프로젝트에서 이런 이슈가 올라왔습니다.    
이슈의 내용은 `int`로 설정한 `lecutreLevel`을 정수로 변환 불가능한 문자열 타입으로 요청을 보냈을 때, 인증 관련 에러가 반환된다는 것이었는데요.

프로젝트에서 Spring Security 관련 설정은 제가 했다보니 이 이슈가 왜 발생했는지, 어떻게 해결할 수 있는지에 대해 정리해보려고 합니다.

### 예제 코드

```java
@Bean
public SecurityFilterChain filterChain(final HttpSecurity http) throws Exception {
    // TODO : 스프링 시큐리티 설정을 더 적절하게 작성해야 함
    http
        .csrf(AbstractHttpConfigurer::disable)

        .exceptionHandling(handler -> handler
            .authenticationEntryPoint(customAuthenticationEntryPoint)
            .accessDeniedHandler(customAccessDeniedHandler))

        .headers(headers -> headers.frameOptions(FrameOptionsConfig::sameOrigin))

        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

        .authorizeHttpRequests(request -> request
            .requestMatchers("/test/**").permitAll()
            .anyRequest().authenticated()
        )

        .apply(jwtSecurityConfig());

    return http.build();
}
```

현재의 FilterChain은 위와 같이 구성했습니다.  
우선 Jwt를 사용하여 인증을 진행하고 있기 때문에 csrf를 비활성화했고, 세션 상태는 `STATELESS`로 설정했습니다.

예외 핸들링의 경우 `AuthenticationEntryPoint`와 `AccessDeniedHandler`를 커스텀한 클래스를 사용하고 있습니다.      
두 객체 모두 미리 작성한 `ControllerAdvice` 에서 공통적으로 예외를 처리하기 위해 발생한 예외를 위임하는 로직만 존재합니다.

여기서 중요한 것은 authorizeHttpRequests 메서드입니다.    
requestMatchers에 `/test` 하위 경로는 권한이 없어도 요청할 수 있도록 했고, 그 외의 요청은 모두 인증된 사용자만 요청할 수 있도록 설정했습니다.   

이 부분을 테스트하기 위해 간단한 컨트롤러 메서드를 보겠습니다.

```java
@RestController
@RequestMapping("/test")
public class MemberController {

    @GetMapping
    public String test() {
        return "test";
    }
    
    @PostMapping
    public int test(@RequestBody @Valid final Dto dto) {
        return dto.num();
    }
}

record Dto(
    @Positive
    int num
) {
}
```

### 실험해보기
<p align="center">
  <img src="https://github.com/f-lab-edu/infrun/assets/40778768/8ccd60f7-5340-4ac9-b0fd-c49f1fbd0210" width="49%" align="center">
  <img src="https://github.com/f-lab-edu/infrun/assets/40778768/2654037c-9357-44cf-9b53-6076e0114a8f" width="49%" align="center">
</p>

이슈에서 언급한 것처럼 `int`로 설정한 `num` 값을 정수로 변환 불가능한 문자열 타입으로 요청을 보냈을 때, 인증 관련 에러가 반환되는 것을 확인할 수 있습니다.   
근데 GET 요청은 성공한 것도 이상하네요.

제가 어려웠던 부분은 `permitAll()` 메서드를 사용했음에도 불구하고 인증 관련 에러가 발생한다는 것이었습니다.  
그래서 이번에는 모든 요청에 대해서도 `permitAll()`을 설정하고 테스트 해봤습니다.

<p align="center">
  <img width="400" alt="image" src="https://github.com/f-lab-edu/infrun/assets/40778768/9723b35a-c21e-448e-b9fd-dbdecbfeca95">
</p>

이번에는 적어도 권한 관련 에러가 발생하지 않았습니다.  
여기서 `authenticated()` 부분에 뭔가 놓치고 있구나를 알았습니다.

<p align="center">
  <img width="600" alt="image" align="center" src="https://github.com/f-lab-edu/infrun/assets/40778768/325139bd-de75-4e14-a20e-072947f9c9af">
</p>

네 여기서 뭔가 깨달은 느낌이었습니다.😅  

### 검증해보자

```java
@Bean
public SecurityFilterChain filterChain(final HttpSecurity http) throws Exception {
    // TODO : 스프링 시큐리티 설정을 더 적절하게 작성해야 함
    http
        .csrf(AbstractHttpConfigurer::disable)

        .exceptionHandling(handler -> handler
            .authenticationEntryPoint(customAuthenticationEntryPoint)
            .accessDeniedHandler(customAccessDeniedHandler))

        .headers(headers -> headers.frameOptions(FrameOptionsConfig::sameOrigin))

        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

        .authorizeHttpRequests(request -> request
            .requestMatchers("/test/**").permitAll()
            .dispatcherTypeMatchers(DispatcherType.ERROR).permitAll() // 추가한 부분
            .anyRequest().authenticated()
        )

        .apply(jwtSecurityConfig());

    return http.build();
    }
```


`DispatcherType`은 서블릿이 제공하는 Enum 타입으로 해당 요청이 어느 요청인지 구분하는 역할을 합니다.      
제가 의심스러운 부분은 바로 스프링부트의 에러 페이지인데요.       
저는 에러가 났을 때 이 에러에 대한 요청에 대한 권한 때문에 발생한 문제라고 가정했기 때문에 `DispatcherType.ERROR`를 추가했습니다.    

<p align="center">
  <img width="600" alt="image" src="https://github.com/f-lab-edu/infrun/assets/40778768/908adfc0-98bc-43c2-98dc-cdfd715378a9">
</p>

다행히 가정이 들어맞는 것을 확인할 수 있었습니다!

### 스프링 MVC의 에러 처리

<p align="center">
  <img width="600" src="https://github.com/f-lab-edu/infrun/assets/40778768/b185377e-acf6-4bef-926f-5d79bcbd58ab">
</p>

스프링 MVC는 기본적으로 처리가 되지 않은 에러가 발생하면 `/error` 뷰를 사용자에게 반환하기 위해 내부적으로 다시 요청을 하게 됩니다.    
하지만 제 Spring Security 설정에는 `/test` 경로 이외의 요청은 모두 인증이 필요하도록 설정이 되어 있었습니다.    
때문에 사실 서버에서는 제대로 400 에러가 발생했지만, 내부적으로 다시 요청을 하게 되면서 인증이 필요한 요청이 되어버린 것입니다.

### 직렬화는 왜 실패했을까?

마무리하기 전에 한 가지 더 궁금한 점이 있었습니다.   
이번 이슈는 정수 타입에 변환할 수 없는 문자열을 넣었고, 이에 대한 검증을 확인해보고자 했는데 왜 권한 관련 에러가 발생하는 지에 대한 원인 파악이었습니다.              
저희는 요청 데이터의 유효성 검사를 hibernate validator를 사용해서 검증을 하고 있습니다.     

팀원분도 사실 해당 검증을 통한 예외가 반환되기를 바랐을 것이고, 저도 그렇게 생각했습니다.
이쯤에서 로그를 살펴보면

<img width="1562" src="https://github.com/f-lab-edu/infrun/assets/40778768/9b7466b6-480c-482a-af4f-eab9ab56e044">

`HttpMessageNotReadableException` 예외가 발생하는데요.   
이 예외는 `HttpMessageConverter`의 구현체인 `AbstractJackson2HttpMessageConverter`에서 발생하는 예외입니다. 

```java
private Object readJavaType(JavaType javaType, HttpInputMessage inputMessage) throws IOException {

    ...
    
    ObjectMapper objectMapper = selectObjectMapper(javaType.getRawClass(), contentType);
    
    try {
        ...
        ObjectReader objectReader = objectMapper.reader().forType(javaType);
        objectReader = customizeReader(objectReader, javaType);
    }
    catch (InvalidDefinitionException ex) {
        throw new HttpMessageConversionException("Type definition error: " + ex.getType(), ex);
    }
    catch (JsonProcessingException ex) {
        throw new HttpMessageNotReadableException("JSON parse error: " + ex.getOriginalMessage(), ex, inputMessage);
    }
}
```

Jackson 라이브러리가 데이터를 자바 객체로 변환하는 메서드를 간추려서 가져와봤습니다.  
해당 데이터를 검증하기 위해서는 직렬화 작업이 선행되어야 하는데, 직렬화하는 과정에서 예외가 발생했기 때문에 검증을 할 수 없었던 것입니다.      

```java
@ResponseStatus(HttpStatus.BAD_REQUEST)
@ExceptionHandler(HttpMessageConversionException.class)
public Response<ErrorCode> handleHttpMessageConversionException(
    HttpMessageConversionException e
) {
    return Response.fail(ErrorCode.INVALID_PARAMETER, e.getMessage());
}
```

이 예외를 처리하는 핸들러를 작성해서 응답을 통일하여 반환할 수 있었습니다.

### 마무리
 
도구를 사용할 땐 적어도 도구의 사용법은 잘 알고 써야할텐데, 일단 헤딩해보자라는 생각으로 부딪혀서 좀 많이 헤맸던 것 같습니다.

처음부터 페이지를 띄워봤다면 더 빠르게 파악할 수 있었을텐데 아쉽기도 하고, 또 어떻게든 원인을 파악하고 이해할 수 있어서 좋은 것도 같습니다.    

잘못된 정보나 피드백이 있다면 편하게 댓글로 남겨주세요! 😇

> 참고    
> <https://spring.io/blog/2013/11/01/exception-handling-in-spring-mvc>   
> [스프링 MVC 2편 - 백엔드 웹 개발 활용 기술](https://inf.run/uA2g)