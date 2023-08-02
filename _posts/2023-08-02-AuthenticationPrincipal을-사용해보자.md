---
title: 'AuthenticationPrincipal을 사용해보자'
categories:
    - Spring
tags:
    - Spring 
    - Security
    - Test
toc: true
toc_sticky: true
---

### 들어가며

오늘은 `@AuthenticationPrincipal` 어노테이션을 테스트하는 방법에 대해 정리해보려고 합니다.  
`@AuthenticationPrincipal` 어노테이션은 Spring Security에서 제공되며, 현재 인증된 유저의 정보를 가져오는데 사용되는 어노테이션입니다.   

저의 경우에는 로그인한 유저의 정보가 필요한 상황이었고 이를 해당 어노테이션으로 해결할 수 있었습니다.    
프로젝트의 코드를 보기 전에 앞서 `@AuthenticationPrincipal` 어노테이션을 사용하는 이유에 대해 간단하게 알아보겠습니다. 

### 왜 AuthenticationPrincipal을 사용했을까

현재 로그인을 한 사용자의 정보를 가져오는 방법은 여러가지가 있습니다.   

1. 클라이언트가 요청할 때마다 우리가 필요한 정보(이메일, 닉네임 등)를 함께 보내기
2. 로그인한 정보를 세션에 저장하고 필요할 때마다 세션에서 가져오기
3. 컨트롤러에서 `Principal` 객체로 주입받기
4. Spring Security가 제공하는 기능을 사용하기

클라이언트가 매번 요청할 때마다 사용자 정보를 함께 보내는 것은 비효율적이고, 보안에도 좋지 않다고 생각했습니다.   
현재 프로젝트에서는 Spring Security가 적용되어 있고, 세션이 아닌 토큰을 사용한 인증과 인가 처리를 하고 있습니다.  
때문에 사용자 정보를 세션에 저장하는 것보다는 다른 방법을 고려했고, Spring Security가 제공하는 기능을 사용하기로 했습니다.    
Spring Security를 사용하지 않았다면 `Principal`을 사용하는 것도 고려해봤을 것 같습니다.

### Spring Security 커스텀하기

`@AuthenticationPrincipal`은 `UserDetailsService`에 의해 생성된 `UserDetails` 객체에 적용할 수 있습니다.   
`@AuthenticationPrincipal`을 사용하기에 앞서 Spring Security를 커스텀한 과정을 보겠습니다.

```java
public class User implements UserDetails, CredentialsContainer {
    private String password;
    private final String username;
    private final Set<GrantedAuthority> authorities;
    ...
}
```

Spring Security에는 `User`라는 클래스가 존재합니다.     
이 클래스는 `UserDetails` 인터페이스를 구현하고 있으며, `UserDetails` 인터페이스는 사용자의 정보를 담고 있는 인터페이스입니다.     
하지만 진행 중인 프로젝트에서는 이메일 정보가 필요하고, `username` 필드에 이메일을 담는 것은 부적절하다고 생각했습니다.

```java
public class UserAdapter extends User {

    private final Member member;

    public UserAdapter(final Member member) {
        super(member.getEmail(), member.getPassword(), authorities(member.getRole()));
        this.member = member;
    }

    private static Collection<? extends GrantedAuthority> authorities(final Role role) {
        return Set.of(new SimpleGrantedAuthority(role.getValue()));
    }

    public Member getMember() {
        return member;
    }
}
```

어댑터 패턴을 사용하여 `User`를 상속한 `UserAdapter`를 만들어서 사용자의 정보를 담고 있는 `Member` 엔티티를 필드로 가지도록 했습니다. 
이 `UserAdapter` 클래스는 `UserDetailsService` 인터페이스에서 사용자의 정보를 가져오는 `loadUserByUsername`에 의해 생성됩니다.

```java
@Component("userDetailsService")
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final MemberRepository memberRepository;

    @Override
    public UserDetails loadUserByUsername(final String email) throws UsernameNotFoundException {
        final Member member = memberRepository.findByEmail(email)
            .orElseThrow(NotFoundMemberException::new);

        return new UserAdapter(member);
    }
}
```

여기까지 적용하면 `@AuthenticationPrincipal` 어노테이션을 사용할 준비가 되었습니다.  

### AuthenticationPrincipal 사용하기

```java
@RequiredArgsConstructor
@RestController
@RequestMapping("/coupons")
public class CouponController {

    private final CouponFacade facade;

    @ResponseStatus(HttpStatus.OK)
    @PreAuthorize("hasAnyRole('USER', 'TEACHER')")
    @GetMapping
    public Response<CouponViewResponse> read(@AuthenticationPrincipal final UserAdapter adapter) {
        final var result = facade.getCoupons(adapter.getMember().getId(), LocalDateTime.now());

        return Response.success(CouponViewResponse.from(result));
    }
}
```

본인이 등록한 쿠폰 목록을 조회하려면 물론 본인의 정보가 필요할 것입니다.   
이 정보는 아까 작성한 `UserAdapter` 클래스에 `@AuthenticationPrincipal` 어노테이션을 적용해서 조회할 수 있습니다. 

<p align="center">
    <img width="600" src="https://github.com/chicori3/algorithm/assets/40778768/fdd10317-4922-4378-a80c-38ad4537ba11">
</p>

제가 원하던 대로 `Member` 엔티티를 가져와 사용할 수 있었습니다.    
여기서 살짝 개선할 수 있는 부분은 `@AuthenticationPrincipal` 어노테이션을 커스텀하는 것입니다.    

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@AuthenticationPrincipal(expression = "#this == 'anonymousUser' ? null : member")
public @interface CurrentUser {

}
```

이처럼 `@AuthenticationPrincipal`을 커스텀하여 SPEL을 작성해주면 번거롭게 `UserAdapter` 클래스를 사용하지 않아도 `Member` 엔티티를 사용할 수 있게 됩니다.  

`@AuthenticationPrincipal`으로 편하게 로그인한 유저 정보를 받을 수 있게 되었습니다.     
이 어노테이션이 어떻게 유저 정보를 가져올 수 있는지 살펴보겠습니다.      

### HandlerMethodArgumentResolver

`HandlerMethodArgumentResolver`는 요청에 넘어온 데이터를 컨트롤러의 메서드에 사용되는 파라미터로 바인딩하는 역할을 하는 스프링의 인터페이스입니다.     
`@AuthenticationPrincipal` 어노테이션을 사용하면 `HandlerMethodArgumentResolver`를 구현한 `AuthenticationPrincipalArgumentResolver`가 사용됩니다.   

```java
public final class AuthenticationPrincipalArgumentResolver implements HandlerMethodArgumentResolver {

    private SecurityContextHolderStrategy securityContextHolderStrategy = SecurityContextHolder
        .getContextHolderStrategy();

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
        
        // 인증 객체 가져오기
        Authentication authentication = this.securityContextHolderStrategy.getContext()
            .getAuthentication();
        if (authentication == null) {
            return null;
        }
        Object principal = authentication.getPrincipal();

        // AuthenticationPrincipal 어노테이션과 SPEL 처리
        AuthenticationPrincipal annotation = findMethodAnnotation(AuthenticationPrincipal.class, parameter);
        String expressionToParse = annotation.expression();
        if (StringUtils.hasLength(expressionToParse)) {
            StandardEvaluationContext context = new StandardEvaluationContext();
            context.setRootObject(principal);
            context.setVariable("this", principal);
            context.setBeanResolver(this.beanResolver);
            Expression expression = this.parser.parseExpression(expressionToParse);
            principal = expression.getValue(context);
        }
        
        ... // ClassCastException 처리
        
        return principal;
    }
}
```

`AuthenticationPrincipalArgumentResolver`는 `SecurityContextHolder`에서 `Authentication` 객체를 가져옵니다.    
여기서 `Authentication` 객체는 `UsernamePasswordAuthenticationToken` 객체로 `Object` 타입의 `principal` 필드를 가지고 있는데 이 필드에 아까 작성한 `UserAdapter` 객체가 저장됩니다.   

만약 로그인하지 않았다면 `SecurityContextHolder`에 저장된 `Authentication` 객체는 `null`이므로 `null`을 반환되겠죠.

후에는 `@AuthenticationPrincipal` 어노테이션이 적용된 것을 찾고, SPEL이 적용되어 있다면 `principal` 필드에 저장된 `UserAdapter` 객체에서 `Member` 엔티티를 가져와 반환하게 됩니다.

### 테스트는 어떻게 할까?

저는 처음에 테스트할 때 굉장히 헤맸습니다.   
가능하면 Spring 컨테이너를 띄우지 않고 컨트롤러만 슬라이스 테스트를 하고 싶어서 `@ExtendWith(MockitoExtension.class)`를 사용했습니다.   

<p align="center">
    <img width="600" src="https://github.com/chicori3/algorithm/assets/40778768/cb347171-f154-4b51-a392-d45be3d60ae9">
</p>

네 이 에러가 저를 굉장히 괴롭게 했는데요.    
암만 디버거를 찍어서 어디가 문제인지 보려해도 파악하기가 힘들었습니다.     
해결한 지금 다시 보면 너무 미숙하지 않았나라는 생각이 들어요.    

문제는 `@AuthenticationPrincipal` 어노테이션을 사용하면 `HandlerMethodArgumentResolver`가 필요한데, `MockitoExtension`을 사용하면 `HandlerMethodArgumentResolver`가 등록되지 않기 때문입니다.  
또 Spring Security 관련 빈들도 당연히 등록되지 않게 되는데 이를 해결하기 위해 저는 `@WebMvcTest`를 적용했습니다.

<p align="center">
    <img width="600" src="https://github.com/chicori3/algorithm/assets/40778768/8308589f-327b-4373-9525-eefecacda960">
</p>

`@WebMvcTest`는 제가 테스트하려는 컨트롤러 계층과 Spring Security 등 만을 빈으로 등록하기 때문에 `@SpringBootTest`에 비해 테스트 속도가 빠르기 때문에 선택했습니다.   

```java
@WebMvcTest(controllers = CouponController.class)
final class CouponControllerTest {
    
    ...
    @Autowired
    private MockMvc mockMvc;
    @MockBean
    private CouponFacade couponFacade;

    @Test자
    @DisplayName("쿠폰을 조회하면 200 상태코드와 쿠폰 리스트를 반환한다")
    void getCoupons() throws Exception {
        ...

        final var result = mockMvc.perform(get(COUPON_URI)
                .with(user(createUser()))
            )
            .andDo(print());

        ...
    }

    private UserAdapter createUser() {
        Member member = Member.of("test", "test", "test");
        member.assignId(1L);

        return new UserAdapter(member);
    }
}
```

현재 컨트롤러에는 `@PreAuthorize`가 걸려있기 때문에 인증과 인가가 필요합니다.  
회원가입, 로그인 과정을 거치지 않는다면 401 또는 403이 발생하게 됩니다.

여기서 `with(user())`는 Spring Security Test 라이브러리에서 제공되는 기능이며, `RequestPostProcessor`를 사용하여 테스트에 필요한 인증 정보를 추가할 수 있습니다.        
저는 `@AuthenticationPrincipal`이 적용되어 있기 때문에, `UserAdapter` 타입을 만들어 사용하도록 했습니다.   
<p align="center">
    <img width="600" src="https://github.com/chicori3/algorithm/assets/40778768/5277c654-96dd-426c-b3df-8443e9db2924">
</p>

요청할 때 Spring Security Context에 위에서 언급한 `UserAdapter`가 저장된 것을 확인할 수 있습니다.    
덕분에 `@AuthenticationPrincipal`이 적용될 때도 에러가 발생하지 않고, 테스트가 무사히 통과되었습니다. 

### 마무리

이번 글에서는 Spring Security에서 제공하는 `@AuthenticationPrincipal` 어노테이션을 사용하는 방법에 대해 알아보았습니다.   
`@AuthenticationPrincipal` 어노테이션은 `HandlerMethodArgumentResolver`를 사용하여 구현되어 있기 때문에, 커스텀하게 구현할 수도 있습니다.     

`ExtendWith(MockitoExtension.class)`만을 계속 고집했다면 아마 아직도 끙끙대고 있지 않았을까해요.      
테스트를 작성할 때는 테스트 대상이 되는 코드와 의존 관계뿐이 아니라 테스트 환경도 고려해야 한다는 것을 배웠습니다.

잘못된 정보나 피드백이 있다면 편하게 댓글로 남겨주세요. 감사합니다! 😇

### 참고

> <https://pupupee9.tistory.com/137>    
> <https://docs.spring.io/spring-security/site/docs/5.0.x/reference/html/test-mockmvc.html>