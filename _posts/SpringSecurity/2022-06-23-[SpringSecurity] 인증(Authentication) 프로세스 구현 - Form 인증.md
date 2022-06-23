---
title: "[SpringSecurity] 인증(Authentication) 프로세스 구현 - Form 인증"
excerpt:
published: true
categories:
    - SpringSecurity

toc: true
toc_sticky: true

date: 2022-06-23
last_modified_at: 2022-06-23
---

## 프로젝트 구성

1. 프로젝트 명
   core-spring-security
2. 프로젝트 기본 구성
   의존성 설정, 환경설정 UI 화면 구성, 기본CRUD 기능
   스프링 시큐리티 보안 기능을 점진적으로 구현 및 완성
3. Springboot, Spring MVC, Spring Data JPA
4. DB - Postgresql Server

5. 사용하는 라이브러리 목록
   JPA
   Spring WEB
   Spring devtool
   thymeleaf
   postgresql
   lombok
   spring-security

## WebIgnore 설정

정적 자원 정리 - WebIgnore 설정
js / css / image 파일 등 보안 필터를 적용할 필요가 없는 리소스를 설정

```
@Override
public void configure(WebSecurity web) throws Exception {
    web.ignoring().requestMatchers(PathRequest.toStaticResources().atCommonLocations());
}
```

> .antMatchers("/","/user").permitAll() 이런 식으로 permitAll()을 통해 접근을 허가해줄 수도 있지만, 해당 방법으로는 허가를 해주더라도 보안검사를 한 뒤 통과 해주는 반면, ignoring()을 이용한 방법은 보안검사 자체를 안하기에 성능적으로 더 권장되는 방법이다.

## PasswordEncoder

PasswordEncoder

비밀번호를 안전하게 암호화 하도록 제공
Spring Security 5.0 이전에는 기본 PasswordEncoder 가 평문을 지원하는 NoOpPasswordEncoder (Deprecated)

생성

-   PasswordEncoder passwordEncoder = PasswordEncoderFactories.createDelegatingPasswordEncoder()
-   여러개의 PasswordEncoder 유형을 선언한 뒤, 상황에 맞게 선택해서 사용할 수 있도록 하는 Encoder이다

암호화 포맷 → {id}encodedPassword

-   기본 포맷은 Bcrypt:
    {bcrypt}\$2a\$10\$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1t
-   알고리즘 종류: bcrypt, noop, pbkdf2, scrypt, sha256

인터페이스
encode(password)

-   패스워드 암호화

matches(rawPassword, encodedPassword)

-   패스워드 비교

## CustomUserDetailsService

![1](../../images/msa/스크린샷%202022-06-23%20오전%2012.11.27.png)

1. CustomUserDetailsService 구현
   → 회원 가입시 유저정보를 UserDetails 타입으로 만들어서 반환해주기 위해 CustomUserDetailsService를 구현한다 UserDetailsService 를 상속(implements)받아 loadUserByUsername 메서드를 오버라이딩 해서 적절하게 개발자가 만든 Account와 매칭시켜준다.

2. AccountContext 구현
   → CustomUserDetailsService를 보면 최종적으로 UserDetails 타입으로 반환을 해줘야 한다. 그렇기에 AccountContext 를 만들어서 UserDetails 타입으로 객체를 만들어주는 구현체를 만들어야 한다. User 클래스를 상속받아 생성자를 구현한다.

3. SecurityConfig에 CustomUserDetailsService 등록
   → 스프링 시큐리티 설정클래스에 위에서 구현한 CustomUserDetailsService를 등록해서 사용하도록 한다.

## CustomAuthenticationProvider

AuthenticationProvider를 구현해서 비밀번호 및 사용자 정의 정책별 보안검사 후 인증객체를 반환하는 메서드

![1](../../images/msa/스크린샷%202022-06-23%20오전%2012.48.11.png)

## Custom Login Form Page

![1](../../images/msa/스크린샷%202022-06-23%20오후%201.54.10.png)

```
@Override
public void configure(HttpSecurity http) throws Exception {
    http.formLogin().loginPage("/customLogin")
}
```

## 로그아웃 및 화면 보안 처리

로그아웃 방법
\<form\> 태그를 사용해서 POST로 요청
\<a\> 태그를 사용해서 GET으로 요청 - SecurityContextLogoutHandler활용

인증 여부에 따라 로그인/로그아웃 표현

```
<li sec:authorize="isAnonymouse()"><a th:href="@{/login}">로그인</a></li>
<li sec:authorize="isAuthenticated()"><a th:href="@{/logout}">로그아웃</a></li>
```

> 시큐리티의 해당 기능 sec:authorize="isAnonymouse()" 을 사용하기 위해선 dependence 를 추가해줘야 한다.
> Gradle: implementation 'org.thymeleaf.extras:thymeleaf-extras-springsecurity5'
> html: \<html xmlns:sec="http://www.thymeleaf.org/thymeleaf-extras-springsecurity5"\>

> 주로 사용되는 security5 메서드 리스트(ex: \<li... sec:authorize="isAnonymouse()"></li\>)
> isAnonymouse() : 익명 사용자일 경우 true
> isAuthenticated(): 로그인한 사용자일 경우
> hasAuthority(ROLE): 권한 확인

## WebAuthenticationDetails, AuthenticationDetailsSource

![1](../../images/msa/스크린샷%202022-06-23%20오후%202.15.26.png)

인증필터( AuthenticationFilter )에서 인증처리를 하게 되면 인증객체(Authentication)을 생성해 username 와 password를 저장.
그리고 내부적으로 Object타입의 details속성을 가지고 있는데, 이 details의 속성에 AuthenticationDetailsSource 에서 WebAuthenticationDetails 객체를 만들어 저장.
이렇게 저장된 WebAuthenticationDetails 클래스에서는 사용자가 추가적으로 전달한 파라미터를 가져와 저장하는 역할을 한다.
(request.getparameter("param1"))

WebAuthenticationDetails

-   인증 과정 중 전달된 데이터를 저장
-   Authentication의 details속성에 저장

AuthenticationDetailsSource

-   WebAuthenticationDetails객체를 생성

## CustomAuthenticationSuccessHandler

인증 성공 이후에 후처리를 원하는대로 정의하여 로직을 수행하도록 하는 핸들러.

ex) 로그인 시도 → 실패 → 로그엔 페이지 이동 → 로그인 시도 → 성공 → 최초 로그인할 때 가려했던 페이지로 이동

```
@Component
public class CustomAuthenticationSuccesshandler extends SimpleUrlAuthenticationSuccessHandler {

    //이전에 요청했던 요청 저장 객체
    private RequestCache requestCache = new HttpSessionRequestCache();

    private RedirectStrategy redirectStrategy = new DefaultRedirectStrategy();

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        //default 이동 URL 페이지 설정
        setDefaultTargetUrl("/");

        SavedRequest savedRequest = requestCache.getRequest(request, response);
        if(savedRequest != null){
            String targetUrl = savedRequest.getRedirectUrl();
            redirectStrategy.sendRedirect(request,response,targetUrl);
        } else {
            redirectStrategy.sendRedirect(request,response, getDefaultTargetUrl());
        }
    }
}
```

SecurityConfig

```
@Autowired
private AuthenticationSuccessHandler customAuthenticationSuccessHandler;
...

@Override
protected void configure(HttpSecurity http) throws Exception {
    http
            .authorizeRequests()
            .antMatchers("/", "/users").permitAll()
            .antMatchers("/mypage").hasRole("USER")
            .antMatchers("/messages").hasRole("MANAGER")
            .antMatchers("/config").hasRole("ADMIN")
            .anyRequest().authenticated()
    .and()
            .formLogin()
            .loginPage("/login")
            .loginProcessingUrl("/login_proc")
            .authenticationDetailsSource(formAuthenticationDetailsSource)
            .defaultSuccessUrl("/")
            .successHandler(customAuthenticationSuccessHandler)
            .permitAll();
}
```

## CustomAuthenticationFailureHandler

인증에 실패했을 경우 후처리를 하는 핸들러를 사용자정의 구현

![1](../../images/msa/스크린샷%202022-06-23%20오후%203.38.46.png)

## Access Denied

인증을 시도하다가 발생한 예외는 인증 필터가 받게 된다.
하지만, 인증 성공 이후 특정 자원에 접근하려는 경우에 해당 자원에 접근권한이 없을 경우에는 인가예외가 발생하게 됩니다.
인가예외는 FilterSecurityInterceptor 에서 받아 ExceptionTranslationFilter로 보내게 되어 예외를 처리하게 된다.
핵심은 인증예외 중 발생한 예외는 인증처리필터(UsernamePaswordAuthenticationFilter) 에서 처리하지만, 인가처리는 예외 발생시
ExceptionTranslationFilter에서 처리한다는점 입니다.
그렇기에 AccessDeniedhandler의 구현체를 만든다.

```
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    private String errorPage;

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
        String deniedUrl = errorPage + "?exception="+accessDeniedException.getMessage();
        response.sendRedirect(deniedUrl);
    }

    public void setErrorPage(String errorPage) {
        this.errorPage = errorPage;
    }
}
```

```
@Override
public void configure(HttpSecurity http) throws Exception {

    http
        .exceptionHandling()
        .accessDeniedPage(“/accessDenied")
        .accessDeniedHandler(accessDeniedHandler)
}
```

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
