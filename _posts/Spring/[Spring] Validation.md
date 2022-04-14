---
title: "스프링 Validator 사용"
excerpt:
published: true
categories:
  - Spring
tags:
  - [Spring]

toc: true
toc_sticky: true

date: 2022-04-14
last_modified_at: 2022-04-14
---

##Spring Validator

사용자의 입력값을 검증하는 로직을 Controller에 넣으면 코드가 복잡해지고 관리하기가 힘들어진다.

<pre>
<code>
@PostMapping("/add")
    public String addItem(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {
        
        if (!StringUtils.hasText(item.getItemName())) {
            bindingResult.rejectValue("itemName","required");
        }

        if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() >
                1)       {

            bindingResult.rejectValue("price","range",new Object[]{1000,1000000},null);

        }
        if (item.getQuantity() == null || item.getQuantity() >= 9999) {
            bindingResult.rejectValue("quantity","max",new Object[]{9999},null);
        }
        
        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();
            if (resultPrice < 10000) {
                bindingResult.reject("totalPriceMin",new Object[]{1000,resultPrice},null);
            }
        }
        
        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v2/addForm";
        }
        ...
    }

</code>
</pre>

컨트롤러에서 검증 로직이 차지하는 부분은 매우 크다. 이런 경우 별도의 클래스로 역할을 분리하는 것이 좋다. 그리고 이렇게 분리한 검증 로직을 재사용 할 수도 있다.

위 코드의 검증 로직을 별도로 분리하자.

<pre>
<code>
public interface Validator {
  boolean supports(Class<?> clazz);
  void validate(Object target, Errors errors);
}
</code>
</pre>

스프링은 검증을 위해 Validator 인터페이스를 제공한다.

supports() {} : 해당 검증기를 지원하는 여부 확인

validate(Object target, Errors errors) : 검증 대상 객체와 BindingResult

<pre>
<code>
@Component
public class ItemValidator implements Validator {
    @Override
    public boolean supports(Class<?> clazz) {
        return Item.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        Item item=(Item)target;

        if (!StringUtils.hasText(item.getItemName())) {
            errors.rejectValue("itemName","required");
        }
        if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() >
                1000000) {
            errors.rejectValue("price","range",new Object[]{1000,1000000},null);
        }
        if (item.getQuantity() == null || item.getQuantity() >= 9999) {
            errors.rejectValue("quantity","max",new Object[]{9999},null);
        }
        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();
            if (resultPrice < 10000) {
                errors.reject("totalPriceMin",new Object[]{1000,resultPrice},null);
            }
        }
    }
}
</code>
</pre>

Validator 인터페이스를 구현한 ItemValidator를 만들었다. @Component 애노테이션을 통해 ItemValidator가 컴포넌트 스캔의 대상이 되어 스프링 빈으로 등록시켰다.

<pre>
<code>
private final ItemValidator itemValidator;


@PostMapping("/add")
public String addItemV5(@ModelAttribute Item item, BindingResult bindingResult,
RedirectAttributes redirectAttributes) {

  itemValidator.validate(item, bindingResult);

  if (bindingResult.hasErrors()) {
    log.info("errors={}", bindingResult);
    return "validation/v2/addForm";
    }

</code>
</pre>

스프링 빈으로 등록한 ItemValidator를 주입받아서 직접 호출하였다. ItemValidator가 bindingResult에 검증오류결과를 자동으로 담긴다.

<hr>

#### WebDataBinder를 통해서 사용하기

WebDataBinder 는 스프링의 파라미터 바인딩의 역할을 해주고 검증 기능도 내부에 포함한다.

<pre>
<code>
@InitBinder
public void init(WebDataBinder dataBinder) {
 log.info("init binder {}", dataBinder);
 dataBinder.addValidators(itemValidator);
}

@PostMapping("/add")
public String addItemV6(@Validated @ModelAttribute Item item, BindingResult
bindingResult, RedirectAttributes redirectAttributes)
</code>
</pre>

이렇게 WebDataBinder 에 검증기를 추가하면 해당 컨트롤러에서는 검증기를 자동으로 적용할 수 있다. 컨트롤러에서 validator를 직접 호출하지 않고 검증대상 앞에서 @Validated 애노테이션을 붙혀 사용할 수 있다.

@Validated 는 검증기를 실행하라는 애노테이션이다.
이 애노테이션이 붙으면 앞서 WebDataBinder 에 등록한 검증기를 찾아서 실행한다. 그런데 여러 검증기를 등록한다면 그 중에 어떤 검증기가 실행되어야 할지 구분이 필요하다. 이때 supports() 가 사용된다.

여기서는 supports(Item.class) 호출되고, 결과가 true 이므로 ItemValidator 의 validate()가 호출된다.

> 검증시 @Validated @Valid 둘다 사용가능하다.
> javax.validation.@Valid 를 사용하려면 build.gradle 의존관계 추가가 필요하다.
> implementation 'org.springframework.boot:spring-boot-starter-validation'
> @Validated 는 스프링 전용 검증 애노테이션이고, @Valid 는 자바 표준 검증 애노테이션이다.
