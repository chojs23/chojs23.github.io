---
title: "[SpringSecurity] 인증 프로세스 ajax 인증 구현"
excerpt:
published: true
categories:
    - SpringSecurity

toc: true
toc_sticky: true

date: 2022-06-24
last_modified_at: 2022-06-24
---

## 흐름 및 개요

![1](../../images/msa/스크린샷%202022-06-23%20오후%2010.23.04.png)

스프링에서 인증 및 인가처리는 Filter로 시작해서 Filter로 끝난다. 그렇기에 사용자가 인증요청을 할 때 Filter에서 가장 먼저 받아서 각각의 클래스에 인증처리를 맡기게 된다. 그래서 그런 인증요청을 맡을 AjaxAuthenticationFilter 를 만들어야 한다.

**인증 처리**

필터가 인증객체(AjaxAuthenticationToken) 에 인증처리를 하기위해 보낸정보를 담아서 인증관리자 (AuthenticationManager)에게 인증객체를 전달해주고 객체를 전달받은 매니저는 실질적으로 인증처리를 담당하는 AjaxAuthenticationProvider에게 인증처리를 위임하게 된다.

실질적인 인증처리를 위임받은 AjaxAuthenticationProvider는 인증 로직 수행 UserDetailsSevice 수행

인증성공을 한다면 AjaxAuthenticationSuccessHandler 구현체를 만들어 인증 성공시 처리로직을 수행

인증실패를 한다면 AjaxAuthenticationFailureHandler 구현체를 만들어 인증 실패시 처리로직을 수행

**인가 처리**

인증된 사용자가 자원에 접근요청을 할 경우 인가처리 시작

FilterSecurityInterceptor에서 인가처리를 담당

인가 검사중 인증예외(AuthenticationException)나 인가예외(AccessDeniedException)이 발생할 경우 ExceptionTranslationFilter 에게 전달

ExceptionTranslationFilter 에서 인증이 실패했을 경우에는 AjaxUrlAuthenticationEntryPoint 구현체에서 로직을 수행

ExceptionTranslationFilter 에서 자원 접근이 거부되었을 경우에는 AjaxAccessDeniedHandler 구현체에서 로직을 수행

## AjaxAuthenticationFilter

AbstractAuthenticationProcessingFileter 상속

-   실제로 Form인증 처리를 하는 UsernamePasswordAuthenticationFilter도 해당 추상클래스를 상속받아 사용한다

필터 작동 조건

-   AntPathRequestMatcher('/api/login") 로 요청정보와 매칭하고 요청 방식이 Ajax이면 필터 작동
-   Ajax인지 아닌지 기준을 두어 Ajax일 경우 필터가 동작하도록 구성을 하자

AjaxAuthenticationToken 생성하여 AuthenticationManager에게 전달하여 인증처리

Filter 추가

```
http.addFilterBefore(AjaxAuthenticationFIlter(), UsernamePasswordAuthenticationFilter.class)
```

-   Form 방식의 인증처리를 담당하는 UsernamePasswordAuthenticationFilter앞에 위치하도록 한다.

## AjaxAuthenticationProvider

AuthenticationProvider 인터페이스를 구현

인증 작동 조건

-   supports(Class<?> authentication)
    ProviderManager로부터 넘어온 인증객체가 AjaxAuthenticationToken 타입이면 작동

인증 검증이 완료되면 AjaxAuthenticationToken을 생성하여 최종 인증 객체를 반환

## AjaxAuthenticationSuccessHandler, AjaxAuthenticationFailureHandler

AjaxAuthenticationSuccessHandler

-   AuthenticationSuccessHandler인터페이스 구현
-   Response Header 설정
    -   response.setStatus(HttpStatus.OK.value())
    -   response.setContentType(MediaType.APPLICATION_JSON_VALUE);
-   JSON형식으로 변환하여 인증 객체 리턴 함
    -   objectMapper.writeValue(response.getWriter(), ResponseBody.ok(userDto));

AjaxAuthenticationFailureHandler

-   AuthenticationFailureHandler 인터페이스 구현
-   Response Header 설정
    -   response.setStatus(HttpStatus.UNAUTHORIZED.value())
    -   response.setContentType(MediaType.APPLICATION_JSON_VALUE);
-   JSON 형식으로 변환하여 오류 메시지 리턴 함
    -   objectMapper.writeValue(response.getWriter(), ResponseBody.error(userDto));

## AjaxLoginUrlAuthenticationEntryPoint , AjaxAccessDeniedHandler

AjaxLoginUrlAuthenticationEntryPoint

-   인증을 받지못한 사용자가 허락되지 않은 자원에 접근시도를 할 경우 처리하는 클래스
-   ExceptionTranslationFilter 에서 인증 예외 시 호출
-   AuthenticationEntryPoint 인터페이스 구현
-   인증 오류 메시지와 401 상태 코드 반환
    -   response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized");

AjaxAccessDeniedHandler

-   인증을 받은 사용자가 허락되지 않은 자원에 접근시도시 호출되는 클래스
-   ExceptionTranslationFilter 에서 인가 예외 시 호출
-   AccessDeniedHandler 인터페이스 구현
-   인가 오류 메시지와 403 상태 코드 반환
    -   response.sendError(HttpServletResponse. SC_FORBIDDEN, "forbidden");

## Ajax Custom DSLs 구현

Custom DSL (도메인 특화 언어) : 특정한 도메인을 정용하는데 특화된 컴퓨터 언어

-   AbstractHttpConfigurer
    -   스프링 시큐리티 초기화 설정 클래스
    -   필터, 핸들러, 메서드, 속성 등을 한 곳에 정의하여 처리할 수 있는 편리함 제공
    -   public void init(H http) throws Exception - 초기화
    -   public void configure(H http) - 설정

HttpSecurity 의 apply(C configurer)메서드 사용

## Ajax 로그인 구현 & CSRF 설정

-   헤더 설정

    -   전송 방식이 Ajax 인지의 여부를 위한 헤더 설정

        -   xhr.setRequestHeader("X-Requested-With", "XMLHttpRequest")

    -   CSRF 헤더 설정

    ```
    <meta id="_csrf" name="_csrf" th:content="${_csrf.headerName}"/>
    <meta id="_csrf_header" name="_csrf_header" th:content="${_csrf.headerName}"/>
    ```

    ```
    var csrfHeader = $('meta[name=_csrf_header"]').attr('content')
    var csrfToken = $('meta[name="_csrf"]').attr('content')
    xhr.setRequestHeader(csrfHeader, csrfToken);
    ```

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
