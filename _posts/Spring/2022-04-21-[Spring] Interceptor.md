---
title: "[Spring] Interceptor"
excerpt:
published: true
categories:
    - Spring
tags:
    - [Spring]

toc: true
toc_sticky: true

date: 2022-04-21
last_modified_at: 2022-04-21
---

## 스프링 인터셉터

스프링 인터셉터는 웹과 관련된 공통 관심 사항을 효과적으로 해결할 수 있는 기술이다.

> 서블릿 필터 -> 서블릿이 제공하는 기술
> 스프링 인터셉터 -> 스프링 MVC 제공

스프링 인터셉터의 흐름

HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 스프링 인터셉터 -> 컨트롤러

스프링 인터셉터는 디스패처 서블릿과 컨트롤러 사이에서 컨트롤러 호출 직전에 호출 된다.
스프링 인터셉터는 스프링 MVC가 제공하는 기능이기 때문에 디스패처 서블릿 이후에 등장하게 된다.
스프링 MVC의 시작점은 디스패처 스블릿.

스프링 인터셉터 체인

HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 인터셉터1 -> 인터셉터2 -> 컨트롤러

스프링 인터셉터는 체인으로 구성되는데, 중간에 인터셉터를 자유롭게 추가할 수 있다.

<br>
<hr>

### 스프링 인터셉터 인터페이스

스프링의 인터셉터를 사용하려면 HandlerInterceptor 인터페이스를 구현하면 된다.

```
public interface HandlerInterceptor {
  default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {}

  default void postHandle(HttpServletRequest request, HttpServletResponse response,
  Object handler, @Nullable ModelAndView modelAndView) throws Exception {}

  default void afterCompletion(HttpServletRequest request, HttpServletResponse
  response, Object handler, @Nullable Exception ex) throws Exception {}

}
```

-   서블릿 필터의 경우 단순하게 doFilter() 하나만 제공된다. 인터셉터는 컨트롤러 호출 전( preHandle ), 호출 후( postHandle ), 요청 완료 이후( afterCompletion )와 같이 단계적으로 잘 세분화 되어 있다.

요청 로그를 남기는 인터셉터를 만들어보자.

LogInterceptor - 요청 로그 인터셉터

```
@Slf4j
public class LogInterceptor implements HandlerInterceptor {
  public static final String LOG_ID = "logId";


  @Override
  public boolean preHandle(HttpServletRequest request, HttpServletResponse
  response, Object handler) throws Exception {

    String requestURI = request.getRequestURI();
    String uuid = UUID.randomUUID().toString();
    request.setAttribute(LOG_ID, uuid);
    //@RequestMapping: HandlerMethod
    //정적 리소스: ResourceHttpRequestHandler
    if (handler instanceof HandlerMethod) {
      HandlerMethod hm = (HandlerMethod) handler; //호출할 컨트롤러 메서드의
      모든 정보가 포함되어 있다.
    }
    log.info("REQUEST [{}][{}][{}]", uuid, requestURI, handler);
    return true; //false 진행X
  }

  @Override
  public void postHandle(HttpServletRequest request, HttpServletResponse
  response, Object handler, ModelAndView modelAndView) throws Exception {
    log.info("postHandle [{}]", modelAndView);
  }

  @Override
  public void afterCompletion(HttpServletRequest request, HttpServletResponse
  response, Object handler, Exception ex) throws Exception {
    String requestURI = request.getRequestURI();
    String logId = (String)request.getAttribute(LOG_ID);
    log.info("RESPONSE [{}][{}]", logId, requestURI);
    if (ex != null) {
      log.error("afterCompletion error!!", ex);
    }
  }
}
```

WebConfig - 인터셉터 등록

```
@Configuration
public class WebConfig implements WebMvcConfigurer {
 @Override
 public void addInterceptors(InterceptorRegistry registry) {
  registry.addInterceptor(new LogInterceptor())
  .order(1)
  .addPathPatterns("/**")
  .excludePathPatterns("/css/**", "/*.ico", "/error");
 }

}
```

-   registry.addInterceptor(new LogInterceptor()) : 인터셉터를 등록한다.
-   order(1) : 인터셉터의 호출 순서를 지정한다. 낮을 수록 먼저 호출된다.
-   addPathPatterns("/\*\*") : 인터셉터를 적용할 URL 패턴을 지정한다.
-   excludePathPatterns("/css/\*_", "/_.ico", "/error") : 인터셉터에서 제외할 패턴을 지정한다.

스프링 인터셉터를 이용해 인증 체크 기능을 개발해보자.

LoginCheckInterceptor

```
@Slf4j
public class LoginCheckInterceptor implements HandlerInterceptor {
  @Override
  public boolean preHandle(HttpServletRequest request HttpServletResponse
  response, Object handler) throws Exception {

    String requestURI = request.getRequestURI();
    log.info("인증 체크 인터셉터 실행 {}", requestURI);
    HttpSession session = request.getSession(false);
    if (session == null || session.getAttribute(SessionConst LOGIN_MEMBER) == null) {
      log.info("미인증 사용자 요청");
      //로그인으로 redirect
      response.sendRedirect("/login?redirectURL=" + requestURI);
      return false;
      }
      return true;
    }
}
```

이 인터셉터를 webConfig에 추가해주자

```
@Configuration
public class WebConfig implements WebMvcConfigurer {
  @Override
  public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new LogInterceptor())
    .order(1)
    .addPathPatterns("/**")
    .excludePathPatterns("/css/**", "/*.ico", "/error");

    registry.addInterceptor(new LoginCheckInterceptor())
    .order(2)
    .addPathPatterns("/**")
    .excludePathPatterns(
    "/", "/members/add", "/login", "/logout",
    "/css/**", "/*.ico", "/error"
    );
  }
  //...
}
```

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
