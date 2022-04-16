---
title: "Spring Bean Validation"
excerpt:
published: true
categories:
  - Spring
tags:
  - [Spring]

toc: true
toc_sticky: true

date: 2022-04-15
last_modified_at: 2022-04-15
---

## Bean Validator

Item 클래스에 Bean Validator 애노테이션을 적용해보자

```
@Data
public class Item {
 private Long id;
 @NotBlank
 private String itemName;
 @NotNull
 @Range(min = 1000, max = 1000000)
 private Integer price;
 @NotNull
 @Max(9999)
 private Integer quantity;
 public Item() {
 }
 public Item(String itemName, Integer price, Integer quantity) {
 this.itemName = itemName;
 this.price = price;
 this.quantity = quantity;
 }
}
```

@NotBlank : 빈값 + 공백만 있는 경우를 허용하지 않는다.
@NotNull : null 을 허용하지 않는다.
@Range(min = 1000, max = 1000000) : 범위 안의 값이어야 한다.
@Max(9999) : 최대 9999까지만 허용한다.

> 참고
> javax.validation.constraints.NotNull
> org.hibernate.validator.constraints.Range
>
> javax.validation 으로 시작하면 특정 구현에 관계없이 제공되는 표준 인터페이스이고, org.hibernate.validator 로 시작하면 하이버네이트 validator 구현체를 사용할 때만 제공되는 검증 기능이다. 실무에서 대부분 하이버네이트 validator를 사용하므로 자유롭게 사용해도 된다.

<h5> 검증기 생성 </h5>
```
ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
Validator validator = factory.getValidator();
```

> 스프링과 통합하면 직접 이런 코드를 작성하지는 않는다.

```
Set<ConstraintViolation<Item>> violations = validator.validate(item);
```

검증 대상( item )을 직접 검증기에 넣고 그 결과를 받는다. Set 에는 ConstraintViolation 이라는 검증오류가 담긴다. 따라서 결과가 비어있으면 검증 오류가 없는 것이다.

<hr>

### 스프링에서 Bean Validator 사용

위의 검증기 생성 코드를 제거해도 Item클래스에서 정의한 Bean Validator가 정상적으로 동작한다.
스프링 부트가 spring-boot-starter-validation 라이브러리를 넣으면 자동으로 Bean Validator를 인지하고 스프링에 통합한다.

```
dependencies {
        ...

	implementation 'org.springframework.boot:spring-boot-starter-validation'

        ...
}
```

<h5>스프링 부트는 자동으로 글로벌 Validator로 등록한다.</h5>

LocalValidatorFactoryBean 을 글로벌 Validator로 등록한다. 이 Validator는 @NotNull 같은 애노테이션을 보고 검증을 수행한다. 이렇게 글로벌 Validator가 적용되어 있기 때문에, @Valid ,@Validated 만 적용하면 된다. 검증 오류가 발생하면, FieldError , ObjectError 를 생성해서 BindingResult 에 담아준다.

```
@PostMapping("/add")
    public String addItem(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {
        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v3/addForm";
        }
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v3/items/{itemId}";
    }
```

검증 순서

1. @ModelAttribute 각각의 필드에 타입 변환 시도
   1. 성공하면 다음으로
   2. 실패하면 typeMismatch 로 FieldError 추가
2. Validator 적용

> 바인딩에 성공한 필드만 Bean Validation 적용
> BeanValidator는 바인딩에 실패한 필드는 BeanValidation을 적용하지 않는다. 생각해보면 타입 변환에 성공해서 바인딩에 성공한 필드여야 BeanValidation 적용이 의미 있다.
> ex)
> itemName 에 문자 "A" 입력 타입 변환 성공 itemName 필드에 BeanValidation 적용
> price 에 문자 "A" 입력 "A"를 숫자 타입 변환 시도 실패 typeMismatch FieldError 추가
> price 필드는 BeanValidation 적용 X

#### Bean Validation - 에러 코드

Bean Validation이 기본으로 제공하는 오류 메시지를 좀 더 자세히 변경해보자.

Bean Validation을 적용하고 bindingResult 에 등록된 검증 오류 코드를 보자.
오류 코드가 애노테이션 이름으로 등록된다. 마치 typeMismatch 와 유사하다.

```
Field error in object 'item' on field 'itemName': rejected value []; codes [NotBlank.item.itemName,NotBlank.itemName,NotBlank.java.lang.String,NotBlank]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [item.itemName,itemName]; arguments []; default message [itemName]]; default message [공백X]
Field error in object 'item' on field 'quantity': rejected value [99999]; codes [Max.item.quantity,Max.quantity,Max.java.lang.Integer,Max]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [item.quantity,quantity]; arguments []; default message [quantity],9999]; default message [9999 이하여야 합니다]
Field error in object 'item' on field 'price': rejected value [1]; codes [Range.item.price,Range.price,Range.java.lang.Integer,Range]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [item.price,price]; arguments []; default message [price],1000000,1000]; default message [1000에서 1000000 사이여야 합니다]
```

NotBlank 라는 오류 코드를 기반으로 MessageCodesResolver 를 통해 다양한 메시지 코드가 순서대로
생성된다.

@NotBlank
NotBlank.item.itemName
NotBlank.itemName
NotBlank.java.lang.String
NotBlank

@Range
Range.item.price
Range.price
Range.java.lang.Integer
Range

<h4>errors.properties</h4>
```
#Bean Validation 추가
NotBlank={0} 공백X
Range={0}, {2} ~ {1} 허용
Max={0}, 최대 {1}
```
{0} 은 필드명이고, {1} , {2} ...은 각 애노테이션 마다 다르다.

<hr>

### Bean Validation의 한계

데이터를 등록할 때와 수정할 때의 요구사항이 다르면 어떨까?

**등록시 기존 요구사항**

- 타입 검증
  - 가격, 수량에 문자가 들어가면 검증 오류 처리
- 필드 검증
  - 상품명: 필수, 공백X
  - 가격: 1000원 이상, 1백만원 이하
  - 수량: 최대 9999
- 특정 필드의 범위를 넘어서는 검증
  - 가격 \* 수량의 합은 10,000원 이상
    <br>

**수정시 요구사항**

- 등록시에는 quantity 수량을 최대 9999까지 등록할 수 있지만 수정시에는 수량을 무제한으로 변경할 수 있다.
- 등록시에는 id 에 값이 없어도 되지만, 수정시에는 id 값이 필수이다.

이러한 문제를 해결하기 위해선 2가지 방법이있다.

- Item을 직접 사용하지 않고, ItemSaveForm, ItemUpdateForm 같은 폼 전송을 위한 별도의 모델 객체를 만들어서 사용한다.
- BeanValidation의 groups 기능을 사용한다.

> groups 기능을 사용해서 등록과 수정시에 각각 다르게 검증을 할 수 있다. 그런데 groups 기능을 사용하니 Item 은 물론이고, 전반적으로 복잡도가 올라간다. 사실 groups 기능은 실제 잘 사용되지는 않는데, 그 이유는 주로 등록용 폼 객체와 수정용 폼 객체를 분리해서 사용하기 때문이다.

#### Form 전송 객체 분리

보통 Item 을 직접 전달받는 것이 아니라, 복잡한 폼의 데이터를 컨트롤러까지 전달할 별도의 객체를 만들어서 전달한다.

예를 들면 ItemSaveForm 이라는 폼을 전달받는 전용 객체를 만들어서 @ModelAttribute 로 사용한다. 이것을 통해 컨트롤러에서 폼 데이터를 전달 받고, 이후 컨트롤러에서 필요한 데이터를 사용해서 Item 을 생성한다.

ItemSaveForm

```
@Data
public class ItemSaveForm {

    @NotBlank
    private String itemName;

    @NotNull
    @Range(min=1000,max=1000000)
    private Integer price;

    @NotNull
    @Max(value = 9999)
    private Integer quantity;

}
```

ItemUpdateForm

```
@Data
public class ItemUpdateForm {

    @NotNull
    private Long id;

    @NotBlank
    private String itemName;

    @NotNull
    @Range(min=1000,max=1000000)
    private Integer price;

    @NotNull
    private Integer quantity;

}
```

Controller

```
@PostMapping("/add")
    public String addItem(@Validated @ModelAttribute("item") ItemSaveForm form, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model)
    {
      Item item=new Item(form.getItemName(),form.getPrice(),form.getQuantity());
    }


@PostMapping("/{itemId}/edit")
    public String edit(@PathVariable Long itemId, @Validated @ModelAttribute("item") ItemUpdateForm form, BindingResult bindingResult) {
      Item item = new Item(form.getItemName(),form.getPrice(),form.getQuantity());
    }
```

위와 같이 등록과 수정 폼을 따로 만들어주고 각 컨트롤러에 해당 폼 객체로 전달받았다.

폼 객체의 데이터를 기반으로 Item 객체를 생성한다. 이렇게 폼 객체 처럼 중간에 다른 객체가 추가되면 변환하는 과정이 추가된다.

<hr>

### Bean Validation - HTTP 메시지 컨버터

@Valid , @Validated 는 HttpMessageConverter ( @RequestBody )에도 적용할 수 있다.

> @ModelAttribute 는 HTTP 요청 파라미터(URL 쿼리 스트링, POST Form)를 다룰 때 사용한다.
> @RequestBody 는 HTTP Body의 데이터를 객체로 변환할 때 사용한다. 주로 API JSON 요청을 다룰 때 사용한다.

API Controller

```
@Slf4j
@RestController
@RequestMapping("/validation/api/items")
public class ValidationItemApiController {
 @PostMapping("/add")
 public Object addItem(@RequestBody @Validated ItemSaveForm form,
 BindingResult bindingResult) {
 log.info("API 컨트롤러 호출");
 if (bindingResult.hasErrors()) {
 log.info("검증 오류 발생 errors={}", bindingResult);
 return bindingResult.getAllErrors();
 }
 log.info("성공 로직 실행");
 return form;
 }
}
```

성공요청

POST http://localhost:8080/validation/api/items/add
{"itemName":"hello", "price":1000, "quantity": 10}

<br>

API의 경우 3가지 경우를 나누어 생각해야 한다.

- 성공 요청: 성공
- 실패 요청: JSON을 객체로 생성하는 것 자체가 실패함
- 검증 오류 요청: JSON을 객체로 생성하는 것은 성공했고, 검증에서 실패함

실패 요청

POST http://localhost:8080/validation/api/items/add
{"itemName":"hello", "price":"A", "quantity": 10}

price의 값에 숫자가 아닌 문자를 담아서 보냈다.

실패 요청 결과

```
{
 "timestamp": "2021-04-20T00:00:00.000+00:00",
 "status": 400,
 "error": "Bad Request",
 "message": "",
 "path": "/validation/api/items/add"
}
```

실패 요청 로그

```
.w.s.m.s.DefaultHandlerExceptionResolver : Resolved
[org.springframework.http.converter.HttpMessageNotReadableException: JSON parse
error: Cannot deserialize value of type `java.lang.Integer` from String "A":
not a valid Integer value; nested exception is
com.fasterxml.jackson.databind.exc.InvalidFormatException: Cannot deserialize
value of type `java.lang.Integer` from String "A": not a valid Integer value
 at [Source: (PushbackInputStream); line: 1, column: 30] (through reference
chain: hello.itemservice.domain.item.Item["price"])]
```

HttpMessageConverter 에서 요청 JSON을 Item 객체로 생성하는데 실패한다.
이 경우는 Item 객체를 만들지 못하기 때문에 컨트롤러 자체가 호출되지 않고 그 전에 예외가 발생한다. Validator도 실행되지 않는다.

이와 같은 예외 발생시 필터나 인터셉터 등을 이용해서 예외처리를 해줘야 한다.

<br>

검증 오류 요청

POST http://localhost:8080/validation/api/items/add
{"itemName":"hello", "price":1000, "quantity": 10000}

이번에는 HttpMessageConverter 는 성공하지만 검증(Validator)에서 오류가 발생하는 경우를 확인해보자.

수량( quantity )이 10000 이면 BeanValidation @Max(9999) 에서 걸린다.

```

 {
 "codes": [
 "Max.itemSaveForm.quantity",
 "Max.quantity",
 "Max.java.lang.Integer",
 "Max"
 ],
 "arguments": [
 {
 "codes": [
 "itemSaveForm.quantity",
 "quantity"
 ],
 "arguments": null,
 "defaultMessage": "quantity",
 "code": "quantity"
 },
 9999
 ],
 "defaultMessage": "9999 이하여야 합니다",
 "objectName": "itemSaveForm",
 "field": "quantity",
 "rejectedValue": 10000,
 "bindingFailure": false,
 "code": "Max"
 }
]
```

return bindingResult.getAllErrors(); 는 ObjectError 와 FieldError 를 반환한다. 스프링이 이 객체를 JSON으로 변환해서 클라이언트에 전달했다.

> @ModelAttribute vs @RequestBody
> HTTP 요청 파리미터를 처리하는 @ModelAttribute 는 각각의 필드 단위로 세밀하게 적용된다. 그래서
> 특정 필드에 타입이 맞지 않는 오류가 발생해도 나머지 필드는 정상 처리할 수 있었다.
> HttpMessageConverter 는 @ModelAttribute 와 다르게 각각의 필드 단위로 적용되는 것이 아니라,
> 전체 객체 단위로 적용된다.
> 따라서 메시지 컨버터의 작동이 성공해서 Item 객체를 만들어야 @Valid , @Validated 가 적용된다.

@ModelAttribute 는 필드 단위로 정교하게 바인딩이 적용된다. 특정 필드가 바인딩 되지 않아도 나머지
필드는 정상 바인딩 되고, Validator를 사용한 검증도 적용할 수 있다.
@RequestBody 는 HttpMessageConverter 단계에서 JSON 데이터를 객체로 변경하지 못하면 이후
단계 자체가 진행되지 않고 예외가 발생한다. 컨트롤러도 호출되지 않고, Validator도 적용할 수 없다.
