---
title: "[SpringSecurity] 인가 프로세스 DB 연동 서비스 계층 구현"
excerpt:
published: true
categories:
    - SpringSecurity

toc: true
toc_sticky: true

date: 2022-07-02
last_modified_at: 2022-07-02
---

## Method방식

1. 서비스 계층의 인가 처리 방식
   화면, 메뉴 단위가 아닌 기능 단위로 인가처리
   메소드 처리 전, 후로 보안 검사 수행하여 인가처리
2. AOP기반으로 동작
   프록시와 어드바이스로 메소드 인가처리 수행
3. 보안 설정 방식
   어노테이션 권한 설정 방식

    ```
    @PreAuthorize("hasRole('USER')"), @PostAuthorize("hasRole('USER')"), @Secured("ROLE_USER")
    ```

    맵 기반 권한 설정 방식

    - 맵 기반 방식으로 외부와 연동하요 메소드 보안 설정 구현

## Method 방식 - 주요 아키텍처

인가 처리를 위한 초기화 과정과 진행

![1](../../images/msa/스크린샷%202022-07-01%20오후%209.01.32.png)

초기화 과정

1. 초기화 시 전체 빈을 검사하면서 보안이 설정된 메소드가 있는지 탐색
2. 빈의 프록시 개체를 생성
3. 보안 메소드에 인가처리(권한심사) 기능을 하는 Advice 를 등록
4. 빈 참조시 실제 빈이 아닌 프록시 빈 객체를 참조

진행 과정

1. 메소드 호출 시 프록시 객체를 통해 메소드를 호출
2. Advice가 등록되어 있다면 Advice 를 작동하게 하여 인가 처리
3. 권한 심사 통과하면 실제 빈의 메소드를 호출한다.

인가 처리를 위한 초기화 과정

![1](../../images/msa/스크린샷%202022-07-01%20오후%209.03.38.png)

스프링 초기화 시 '빈 후 처리기 클래스'에서 전체 빈(Bean)을 검사하는데, 이 클래스의 구현체 중 하나가 자동 프록시 생성기(DefaultAdvisorAutoProxyCreator)이다.
자동 프록시 생성기는 빈들을 검사하면서 보안이 설정된 메소드를 찾아서 그 메소드를 가지고 있는 빈이 있으면 그 빈의 프록시객체를 생성

-   자동 프록시 생성기(DefaultAdvisorAutoProxyCreator)는 MethodSecurityMetadataSourceAdvisor를 통해서 빈(Bean)을 검사한다.
-   MethodSecurityMetadataSourceAdvisor 는 MethodSecurityMetadataSourcePointcut과 MethodSecurityInterceptor 클래스를 가지고 있다.
    -   MethodSecurityMetadataSourcePointcut
        메소드에 보안이 설정되있는지 탐색하는 클래스이고, 해당 클래스에서 현재 검사하는 빈(Bean)의 클래스 및 메소드 정보를 MethodSecurityMetadataSource클래스에 전달합니다. 그럼 MethodSecurityMetadataSource는 전달받은 정보를 파싱(parse())해서 권한설정 어노테이션등을 찾는다.
    -   MethodSecurityInterceptor
        MethodSecurityMetadataSource 에서 보안설정이 된 메서드를 찾았을 경우 MethodSecurityInterceptor 에 어드바이스 등록을 하는 클래스

보안 메소드가 설정된 빈일 경우 프록시 생성대상이 되고 자동프록시 생성기에서는 프록시 객체를 생성
**프록시 객체가 생성 된 다음 MethodSecurityInterceptor 에 어드바이스가 등록**

초기화 과정 이후

![1](../../images/msa/스크린샷%202022-07-01%20오후%209.13.52.png)

사용자가 order() 나 display() 메서드를 호출

-   order() 호출

    1. OrderServiceProxy 객체에서 요청을 받아서 Advice등록이 되어있는지 확인
    2. MethodSecurityInterceptor에서 인가처리를
    3. 승인되었을 경우 실제 객체(orderService) 에서 order() 메서드를 호출
    4. 거절되었을 경우 예외(AccessDeniedException)를 발생

-   display() 호출

    1. OrderServiceProxy 객체에서 요청을 받아서 Advice등록이 되어있는지 확인
    2. display()는 Advice 등록이 되어있지 않기 때문에 바로 실제객체 메소드(orderService.display())를 호출

## 어노테이션 권한 설정

![1](../../images/springsecurity/Screenshot%202022-07-01%20at%2020.08.05.jpg)

보안이 필요한 메서드에 설정한다.

-   @PreAuthorize, @PostAuthorize
    SpEL 지원
    **@PreAuthroize("hasRole('ROLE_USER') and (#account.username == principal.username)")**
    PrePostAnnotationSecuritymetadataSource가 담당

```
@Controller
public class AopSecurityController {

    @GetMapping("/preAuthorize")
    @PreAuthorize(value = "hasRole('ROLE_USER') AND (#accountDto.username == principal.username)")
    public String preAuthorize(AccountDto accountDto, Model model, Principal principal){
        model.addAttribute("method", "Success @PreAuthorize");

        return "aop/method";
    }

}
```

-   @Secured, @RolesAllowed
    SpEL 미지원
    @Secured("ROLE_USER"), @RolesAllowed("ROLE_USER")
    SecuredAnnotationSecurityMetadataSource, Jsr250MethodSecurityMetadataSource가 담당

```
@Override
@Secured("ROLE_MANAGER")
public void order() {
    System.out.println("order");
}
```

-   @EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
    해당 어노테이션을 통해 설정을 해줘야 위 항목의 어노테이션이 활성화가 된다.

```
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(securedEnabled = true, prePostEnabled = true)
@Slf4j
@Order(1)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
		...
}
```

## Method 방식 - Map 기반 DB 연동

MapBasedSecurityMetadataSource

![1](../../images/msa/스크린샷%202022-07-01%20오후%209.38.11.png)

Filter기반 Url 방식은 FilterSecurityInterCeptor 필터가 동작해서 인가처리를 하지만, AOP기반 Method방식은 MethodSecurityInterceptor라는 인터셉터에서 어드바이스가 작동해서 처리한다. 필터기반은 요청을 필터가 가로채 동작하고, Method방식은 메소드 호출시 프록시 객체가 메소드에 등록된 Advice를 작동하게 해서 인가처리를 한다.
두 방식 다 권한정보를 얻어야 하기 때문에 AccessDecisionManager에게 사용자의 권한정보를 전달한다.
각각 FilterInvocationSecurityMetadataSource , MethodSecurityMetadataSource 구현체로부터 권한정보를 가져와서 전달한다.

![1](../../images/msa/스크린샷%202022-07-01%20오후%209.40.59.png)

Method방식도 Map기반으로 DB를 연동해서 처리할 수 있는 방법을 지원한다.
MapBasedMethodSecurityMetadataSource 클래스는 MethodSecurityMetadataSource 인터페이스를 구현하고 있고, 해당 인터페이스도 최상위 인터페이스인 SecurityMetadataSource 인터페이스를 상속받고 있다.

-   어노테이션 설정 방식이 아니라 맵 기반으로 권한을 설정
-   기본적인 구현이 완성되어 있고, DB로부터 자원과 권한정보를 매핑한 데이터를 전달하면 메소드 방식의 인가처리가 이뤄지는 클래스

![1](../../images/msa/스크린샷%202022-07-01%20오후%209.42.13.png)

사용자가 admin()메소드 호출을 하는데 admin()메서드는 ROLE_ADMIN권한이 설정되어 있다.

-   스프링 초기화 단계에서 admin() 메서드를 가진 객체가 프록시 객체로 생성되었을 것이고, 권한이 설정된 admin()에 Advice가 등록
-   메서드에 걸린 Advice는 MethodSecurityInterceptor

MethodSecurityInterceptor는 인가처리중 해당 메서드에 어떤 권한이 설정되있는지에 대한 정보를 MapBasedMethodSecurityMetadataSource 클래스에 권한정보를 요청
MapBasedMethodSecurityMetadataSource 클래스는 요청을 처리하기위해 내부에 가지고 있는 MethodMap이라는 객체에서 Key(method name): Value(권한정보)를 통해서 권한정보를 추출
MapBasedMethodSecurityMetadataSource 클래스는 추출한 권한정보를 전달하고 권한 목록이 존재할 경우 AccessDecisionManager에게 전달해 인가처리를 진행

```
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class MethodSecurityConfig extends GlobalMethodSecurityConfiguration {

    @Override
    protected MethodSecurityMetadataSource customMethodSecurityMetadataSource() {
        return new MapBasedMethodSecurityMetadataSource();
    }
}
```

customMethodSecurityMetadataSource를 오버라이딩 해서 Map기반으로 메소드 인가 처리를 할 수 있는 클래스를 생성해서 리턴해줌
현재는 기본생성자(no parameter)지만 DB에서 얻은 권한정보를 인자값으로 넘겨서 등록할 수 있다.
GlobalMethodSecurityConfiguration 의 methodSecurityMetadataSource() 빈(Bean)에서customMethodSecurityMetadataSource 가 not null인 경우 source라는 리스트에 추가한다.

![1](../../images/msa/스크린샷%202022-07-01%20오후%209.51.21.png)

메서드 보안 설정 클래스(MethodSecurityConfig)을 만들고 Map기반으로 메소드 인가 처리를 할 수 있도록 MapBasedMethodSecurityMetadataSource 객체를 만들어서 설정클래스에 전달했다.(customMethodSecurityMetadataSource)

MapBasedMethodSecurityMetadataSource 객체는 methodMap이라는 Map객체에 자원/권한정보가 필요한데 DB로부터 데이터를 받아서 ResourceMap 이라는 객체를 만들어야한다.

MethodResourcesMapFactoryBean

-   DB로 부터 얻은 권한/자원 정보를 ResourceMap을 빈으로 생성해서 MapBasedMethodSecurityMetadataSource 에 전달

![1](../../images/msa/스크린샷%202022-07-01%20오후%209.58.35.png)

자동 프록시 생성기(DefaultAdvisorAutoProxyCreator)는 스프링 초기화 시점에 MethodSecurityMetadataSourceAdvisor 에서 빈들을 검사하는데, 내부적으로 MethodSecurityMetadataSourcePointcut 에서 프록시 생성 대상 클래스와 어드바이스 등록 대상 메소드를 탐색한다.

보안이 설정된 메서드를 탐색하며 현재 검사하는 빈의 클래스 및 메소드 정보를 MapBaseMethodSecurityMetadataSource에 전달하고 전달받은 정보를 파싱해서 권한 설정(인가처리 어드바이스)을 MehtodSecurityInterceptord에 등록

```
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)//메소드 보안 동작 어노테이션
public class MethodSecurityConfig extends GlobalMethodSecurityConfiguration {

    @Autowired
    private SecurityResourceService securityResourceService;

    @Override
    protected MethodSecurityMetadataSource customMethodSecurityMetadataSource() {
        return mapBasedMethodSecurityMetadataSource();
    }

    @Bean
    public MapBasedMethodSecurityMetadataSource mapBasedMethodSecurityMetadataSource() {
        return new MapBasedMethodSecurityMetadataSource(methodResourcesMapFactoryBean().getObject());
    }

    @Bean
    public MethodResourcesMapFactoryBean methodResourcesMapFactoryBean() {
        MethodResourcesMapFactoryBean methodResourcesMapFactoryBean = new MethodResourcesMapFactoryBean();
        methodResourcesMapFactoryBean.setSecurityResourceService(securityResourceService);
        return methodResourcesMapFactoryBean;
    }
}
```

```
@Slf4j
public class MethodResourcesMapFactoryBean implements FactoryBean<LinkedHashMap<String, List<ConfigAttribute>>> {

    private SecurityResourceService securityResourceService;

    private LinkedHashMap<String, List<ConfigAttribute>> resourcesMap;

    public void setResourcesMap(LinkedHashMap<String, List<ConfigAttribute>> resourcesMap) {
        this.resourcesMap = resourcesMap;
    }

    public void setSecurityResourceService(SecurityResourceService securityResourceService) {
        this.securityResourceService = securityResourceService;
    }

    @Override
    public LinkedHashMap<String, List<ConfigAttribute>> getObject() {
        if(resourcesMap == null){
            init();
        }
        return resourcesMap;
    }
    private void init(){
        resourcesMap = securityResourceService.getMethodResourceList();
    }

    @Override
    public Class<?> getObjectType() {
        return LinkedHashMap.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

```
ublic LinkedHashMap<String, List<ConfigAttribute>> getMethodResourceList() {

    LinkedHashMap<String, List<ConfigAttribute>> result = new LinkedHashMap<>();
    List<Resources> resourcesList = resourcesRepository.findAllMethodResources();
    resourcesList.forEach(re ->
            {
                List<ConfigAttribute> configAttributeList = new ArrayList<>();
                re.getRoleSet().forEach(ro -> {
                    configAttributeList.add(new SecurityConfig(ro.getRoleName()));
                });
                result.put(re.getResourceName(), configAttributeList);
            }
    );
    return result;
}
```

## ProtectPointcutPostProcessor

메소드 방식의 인가처리를 위한 자원 및 권한정보 설정 시 자원에 포인트 컷 표현식을 사용할 수 있도록 지원하는 클래스

빈 후처리기로서 스프링 초기화 과정에서 빈 들을 검사하여 빈이 가진 메소드 중 포인트 컷 표현식과 matching 되는 클래스, 메소드, 권한 정보를 MapBaasedMethodSecurityMetadataSource 에 전달하여 인가처리가 되도록 제공되는 클래스

DB 저장 방식

-   Method 방식
    -   io.security.service.OrderService.order: ROLE_USER
-   Pointcut 방식
    -   execution(* io.security.service.*Service.\*(..)):ROLE_USER

설정 클래스에서 빈 생성시 접근제한자가 package 범위로 되어 있기 때문에 리플렉션을 이용해 생성합니다.

ProtectPointcutPostProcessor 은 public class가 아니기 때문에 Java의 리플렉션 기능을 이용해야 한다.

```
@Bean
BeanPostProcessor protectPointcutPostProcessor() throws Exception{
    Class<?> clazz = Class.forName("org.springframework.security.config.method.ProtectPointcutPostProcessor");
    Constructor<?> declaredConstructor = clazz.getDeclaredConstructor(MapBasedMethodSecurityMetadataSource.class);
    declaredConstructor.setAccessible(true);
    Object instance = declaredConstructor.newInstance(mapBasedMethodSecurityMetadataSource());
    Method setPointcutMap = instance.getClass().getMethod("setPointcutMap", Map.class);
    setPointcutMap.setAccessible(true);
    setPointcutMap.invoke(instance,pointcutResourcesMapFactoryBean().getObject());

    return (BeanPostProcessor)instance;
}
```

![1](../../images/msa/스크린샷%202022-07-02%20오후%201.33.20.png)
MethodResourcesMapFactoryBean
DB로부터 얻은 권한 자원 정보를 ResourceMap 빈으로 생성해서 ProtectPointcutPostProcessor에 전달

![1](../../images/msa/스크린샷%202022-07-02%20오후%201.33.31.png)

DB에서 읽어온 포인트컷 권한 정보를 ProtectPointcutPostProcessor 에 전달하면 빈 후처리이기 때문에 자신이 검사하는 빈들 중에서 포인트컷 표현식에 해당하는 TargetClass, Method, ConfigAttribute 를 MapBaseMethodSecurityMetadataSource에 전달

```
private String resourceType;
...
public void init(){
    if("method".equals(resourceType)){
        resourcesMap = securityResourceService.getMethodResourceList();
    } else if("pointcut".equals(resourceType)){
        resourcesMap = securityResourceService.getPointcutResourceList();
    }
}
```

```
@Bean
public MethodResourcesMapFactoryBean pointcutResourcesMapFactoryBean() {
    MethodResourcesMapFactoryBean methodResourcesMapFactoryBean = new MethodResourcesMapFactoryBean();
    methodResourcesMapFactoryBean.setSecurityResourceService(securityResourceService);
    methodResourcesMapFactoryBean.setResourceType("pointcut");
    return methodResourcesMapFactoryBean;
}
```

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
