---
title: "Spring Cookie, Session"
excerpt:
published: true
categories:
  - Spring
tags:
  - [Spring]

toc: true
toc_sticky: true

date: 2022-04-16
last_modified_at: 2022-04-16
---

## 로그인 처리하기 - 쿠키 사용

로그인 상태를 유지하기 위해 쿠키를 사용할 수 있다. 서버에서 로그인에 성공하면 HTTP 응답에 쿠키를 담아서 브라우저에 전달하자. 그러면 브라우저는 앞으로 해당 쿠키를 지속해서 보내준다.

> 영속 쿠키: 만료 날짜를 입력하면 해당 날짜까지 유지
> 세션 쿠키: 만료 날짜를 생략하면 브라우저 종료시 까지만 유지

브라우저 종료시 로그아웃이 되길 기대하므로, 우리에게 필요한 것은 세션 쿠키이다

```
@PostMapping("/login")
public String login(@Valid @ModelAttribute LoginForm form, BindingResult bindingResult, HttpServletResponse response) {
 if (bindingResult.hasErrors()) {
  return "login/loginForm";
 }
 Member loginMember = loginService.login(form.getLoginId(), form.getPassword());
 log.info("login? {}", loginMember);
 if (loginMember == null) {
  bindingResult.reject("loginFail", "아이디 또는 비밀번호가 맞지 않습니다.");
  return "login/loginForm";
 }
 //로그인 성공 처리
 //쿠키에 시간 정보를 주지 않으면 세션 쿠키(브라우저 종료시 모두 종료)
 Cookie idCookie = new Cookie("memberId",String.valueOf(loginMember.getId()));
 response.addCookie(idCookie);
 return "redirect:/";
}
```

쿠키 생성

```
Cookie idCookie = new Cookie("memberId", String.valueOf(loginMember.getId()));
response.addCookie(idCookie);
```

로그인에 성공하면 쿠키를 생성하고 HttpServletResponse 에 담는다. 쿠키 이름은 memberId 이고, 값은 회원의 id 를 담아둔다. 웹 브라우저는 종료 전까지 회원의 id 를 서버에 계속 보내줄 것이다.

로그인에 성공하면 로그인 한 사용자 전용 홈 화면을 보여주자.

```
@Controller
@RequiredArgsConstructor
public class HomeController {

 private final MemberRepository memberRepository;
// @GetMapping("/")
 public String home() {
 return "home";
 }
 @GetMapping("/")
 public String homeLogin(
 @CookieValue(name = "memberId", required = false) Long memberId,
Model model) {
 if (memberId == null) {
 return "home";
 }
 //로그인
 Member loginMember = memberRepository.findById(memberId);
 if (loginMember == null) {
 return "home";
 }
 model.addAttribute("member", loginMember);
 return "loginHome";
 }
}
```

@CookieValue 를 사용하면 편리하게 쿠키를 조회할 수 있다.
로그인 하지 않은 사용자도 홈에 접근할 수 있기 때문에 required = false 를 사용한다.

로그인 쿠키( memberId )가 없는 사용자는 기존 home 으로 보낸다. 추가로 로그인 쿠키가 있어도 회원이
없으면 home 으로 보낸다.
로그인 쿠키( memberId )가 있는 사용자는 로그인 사용자 전용 홈 화면인 loginHome 으로 보낸다. 추가로
홈 화면에 회원 관련 정보도 출력해야 해서 member 데이터도 모델에 담아서 전달한다.

### 로그아웃

세션 쿠키이므로 웹 브라우저 종료시 서버에서 해당 쿠키의 종료 날짜를 0으로 지정

```
@PostMapping("/logout")
public String logout(HttpServletResponse response) {
 expireCookie(response, "memberId");
 return "redirect:/";
}
private void expireCookie(HttpServletResponse response, String cookieName) {
 Cookie cookie = new Cookie(cookieName, null);
 cookie.setMaxAge(0);
 response.addCookie(cookie);
}
```

### 쿠키의 보안문제

쿠키를 사용해서 로그인Id를 전달해서 로그인을 유지할 수 있었다. 그런데 여기에는 심각한 보안 문제가 있다.

- 쿠키 값은 임의로 변경할 수 있다.
  - 클라이언트가 쿠키를 강제로 변경하면 다른 사용자가 된다.
  - 실제 웹브라우저 개발자모드 Application Cookie 변경으로 확인
  - Cookie: memberId=1 Cookie: memberId=2 (다른 사용자의 이름이 보임)
- 쿠키에 보관된 정보는 훔쳐갈 수 있다.
  - 만약 쿠키에 개인정보나, 신용카드 정보가 있다면?
  - 이 정보가 웹 브라우저에도 보관되고, 네트워크 요청마다 계속 클라이언트에서 서버로 전달된다.
  - 쿠키의 정보가 나의 로컬 PC가 털릴 수도 있고, 네트워크 전송 구간에서 털릴 수도 있다.
- 해커가 쿠키를 한번 훔쳐가면 평생 사용할 수 있다.
  - 해커가 쿠키를 훔쳐가서 그 쿠키로 악의적인 요청을 계속 시도할 수 있다.

대안

- 쿠키에 중요한 값을 노출하지 않고, 사용자 별로 예측 불가능한 임의의 토큰(랜덤 값)을 노출하고, 서버에서
  토큰과 사용자 id를 매핑해서 인식한다. 그리고 서버에서 토큰을 관리한다.
- 토큰은 해커가 임의의 값을 넣어도 찾을 수 없도록 예상 불가능 해야 한다.
- 해커가 토큰을 털어가도 시간이 지나면 사용할 수 없도록 서버에서 해당 토큰의 만료시간을 짧게(예: 30분)유지한다. 또는 해킹이 의심되는 경우 서버에서 해당 토큰을 강제로 제거하면 된다.

## 로그인 처리하기 - 세션 사용

앞서 쿠키에 중요한 정보를 보관하는 방법은 여러가지 보안 이슈가 있었다. 이 문제를 해결하려면 결국
중요한 정보를 모두 서버에 저장해야 한다. 그리고 클라이언트와 서버는 추정 불가능한 임의의 식별자
값으로 연결해야 한다.
이렇게 서버에 중요한 정보를 보관하고 연결을 유지하는 방법을 세션이라 한다.

서버는 클라이언트에 mySessionId 라는 이름으로 세션ID 만 쿠키에 담아서 전달한다.
클라이언트는 쿠키 저장소에 mySessionId 쿠키를 보관한다.

여기서 중요한 포인트는 회원과 관련된 정보는 전혀 클라이언트에 전달하지 않는다는 것이다.
오직 추정 불가능한 세션 ID만 쿠키를 통해 클라이언트에 전달한다.

### 서블릿 HTTP 세션

서블릿을 통해 HttpSession 을 생성하면 다음과 같은 쿠키를 생성한다. 쿠키 이름이 JSESSIONID 이고,
값은 추정 불가능한 랜덤 값이다.
Cookie: JSESSIONID=5B78E23B513F50164D6FDD8C97B0AD05

```
public class SessionConst {
 public static final String LOGIN_MEMBER = "loginMember";
}
```

HttpSession 에 데이터를 보관하고 조회할 때, 같은 이름이 중복 되어 사용되므로, 상수를 하나 정의했다.

```
@PostMapping("/login")
public String login(@Valid @ModelAttribute LoginForm form, BindingResult
bindingResult, HttpServletRequest request) {
 if (bindingResult.hasErrors()) {
 return "login/loginForm";
 }
 Member loginMember = loginService.login(form.getLoginId(),
form.getPassword());
 log.info("login? {}", loginMember);
 if (loginMember == null) {
 bindingResult.reject("loginFail", "아이디 또는 비밀번호가 맞지 않습니다.");
 return "login/loginForm";
 }
 //로그인 성공 처리
 //세션이 있으면 있는 세션 반환, 없으면 신규 세션 생성
 HttpSession session = request.getSession();
 //세션에 로그인 회원 정보 보관
 session.setAttribute(SessionConst.LOGIN_MEMBER, loginMember);
 return "redirect:/";
}
```

세션을 생성하려면 request.getSession() 를 사용하면 된다. default=true

request.getSession(true)
세션이 있으면 기존 세션을 반환한다.
세션이 없으면 새로운 세션을 생성해서 반환한다.

request.getSession(false)
세션이 있으면 기존 세션을 반환한다.
세션이 없으면 새로운 세션을 생성하지 않는다. null 을 반환한다.

Logout

```
@PostMapping("/logout")
public String logoutV3(HttpServletRequest request) {
 //세션을 삭제한다.
 HttpSession session = request.getSession(false);
 if (session != null) {
 session.invalidate();
 }
 return "redirect:/";
}
```

HomeController

```
@GetMapping("/")
public String homeLogin(HttpServletRequest request, Model model) {
 //세션이 없으면 home
 HttpSession session = request.getSession(false);
 if (session == null) {
 return "home";
 }
 Member loginMember = (Member)
session.getAttribute(SessionConst.LOGIN_MEMBER);
 //세션에 회원 데이터가 없으면 home
 if (loginMember == null) {
 return "home";
 }
 //세션이 유지되면 로그인으로 이동
 model.addAttribute("member", loginMember);
 return "loginHome";
}
```

#### @SessionAttribute

이미 로그인 된 사용자를 찾을 때는 다음과 같이 사용하면 된다. 참고로 이 기능은 세션을 생성하지 않는다.

@SessionAttribute(name = "loginMember", required = false) Member loginMember

```
@GetMapping("/")
public String homeLoginV3Spring(
 @SessionAttribute(name = SessionConst.LOGIN_MEMBER, required = false)
Member loginMember,
 Model model) {
 //세션에 회원 데이터가 없으면 home
 if (loginMember == null) {
 return "home";
 }
 //세션이 유지되면 로그인으로 이동
 model.addAttribute("member", loginMember);
 return "loginHome";
}
```

세션을 찾고, 세션에 들어있는 데이터를 찾는 번거로운 과정을 스프링이 한번에 편리하게 처리해준다.
