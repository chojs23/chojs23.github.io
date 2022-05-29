---
title: "[Spring] Typeconverter"
excerpt:
published: true
categories:
  - Spring
tags:
  - [Spring]

toc: true
toc_sticky: true

date: 2022-04-18
last_modified_at: 2022-04-18
---

## 스프링 타입 컨버터

문자를 숫자로 변환하거나, 반대로 숫자를 문자로 변환해야 하는 것 처럼 애플리케이션을 개발하다 보면 타입을 변환해야 하는 경우가 상당히 많다.

```
@GetMapping("/hello-v1")
  public String helloV1(HttpServletRequest request) {
  String data = request.getParameter("data"); //문자 타입 조회
  Integer intValue = Integer.valueOf(data); //숫자 타입으로 변경
  System.out.println("intValue = " + intValue);
  return "ok";
 }
```

```
String data = request.getParameter("data")
```

HTTP 요청 파라미터는 모두 문자로 처리된다. 따라서 요청 파라미터를 자바에서 다른 타입으로 변환해서 사용하고 싶으면 다음과 같이 숫자 타입으로 변환하는 과정을 거쳐야 한다.

```
Integer intValue = Integer.valueOf(data)
```

<br>
이번에는 스프링 MVC가 제공하는 @RequestParam 을 사용해보자

```
@GetMapping("/hello-v2")
public String helloV2(@RequestParam Integer data) {
 System.out.println("data = " + data);
 return "ok";
}
```

HTTP 쿼리 스트링으로 전달하는 data=10 부분에서 10은 숫자 10이 아니라 문자 10이다.

스프링이 제공하는 @RequestParam 을 사용하면 이 문자 10을 Integer 타입의 숫자 10으로 편리하게 받을 수 있다.

이것은 스프링이 중간에서 타입을 변환해주었기 때문이다.

스프링의 타입 변환 적용 예

- 스프링 MVC 요청 파라미터
  - @RequestParam , @ModelAttribute , @PathVariable
- @Value 등으로 YML 정보 읽기
- XML에 넣은 스프링 빈 정보를 변환
- 뷰를 렌더링 할 때

<hr>

### 타입 컨버터 - Converter

타입 컨버터를 사용하려면 org.springframework.core.convert.converter.Converter
인터페이스를 구현하면 된다.

컨버터 인터페이스

```
package org.springframework.core.convert.converter;
public interface Converter<S, T> {
  T convert(S source);
}
```

먼저 가장 단순한 형태인 문자를 숫자로 바꾸는 타입 컨버터를 만들어보자.

StringToIntegerConverter - 문자를 숫자로 변환하는 타입 컨버터

```
@Slf4j
public class StringToIntegerConverter implements Converter<String, Integer> {
  @Override
  public Integer convert(String source) {
    log.info("convert source={}", source);
    return Integer.valueOf(source);
  }
}
```

String Integer 로 변환하기 때문에 소스가 String 이 된다. 이 문자를
Integer.valueOf(source) 를 사용해서 숫자로 변경한 다음에 변경된 숫자를 반환하면 된다.

<br>

IntegerToStringConverter - 숫자를 문자로 변환하는 타입 컨버터

```
public class IntegerToStringConverter implements Converter<Integer, String> {
 @Override
  public String convert(Integer source) {
    log.info("convert source={}", source);
    return String.valueOf(source);
  }
}
```

ConverterTest - 타입 컨버터 테스트 코드

```
class ConverterTest {
  @Test
    void stringToInteger() {
    StringToIntegerConverter converter = new StringToIntegerConverter();
    Integer result = converter.convert("10");
    assertThat(result).isEqualTo(10);
  }
  @Test
    void integerToString() {
    IntegerToStringConverter converter = new IntegerToStringConverter();
    String result = converter.convert(10);
    assertThat(result).isEqualTo("10");
  }
}
```

#### 사용자 정의 타입 컨버터

127.0.0.1:8080 과 같은 IP, PORT를 입력하면 IpPort 객체로 변환하는 컨버터를 만들어보자.

IpPort

```
@Getter
@EqualsAndHashCode
public class IpPort {
  private String ip;
  private int port;
  public IpPort(String ip, int port) {
    this.ip = ip;
    this.port = port;
  }
}
```

롬복의 @EqualsAndHashCode 를 넣으면 모든 필드를 사용해서 equals() , hashcode() 를 생성한다.
따라서 모든 필드의 값이 같다면 a.equals(b) 의 결과가 참이 된다.

StringToIpPortConverter

```
@Slf4j
public class StringToIpPortConverter implements Converter<String, IpPort> {
  @Override
  public IpPort convert(String source) {
    log.info("convert source={}", source);
    String[] split = source.split(":");
    String ip = split[0];
    int port = Integer.parseInt(split[1]);
    return new IpPort(ip, port);
  }
}
```

127.0.0.1:8080 같은 문자를 입력하면 IpPort 객체를 만들어 반환한다.

IpPortToStringConverter

```
@Slf4j
public class IpPortToStringConverter implements Converter<IpPort, String> {
  @Override
  public String convert(IpPort source) {
    log.info("convert source={}", source);
    return source.getIp() + ":" + source.getPort();
  }
}
```

IpPort 객체를 입력하면 127.0.0.1:8080 같은 문자를 반환한다.

ConverterTest - IpPort 컨버터 테스트

```
@Test
void stringToIpPort() {
  StringToIpPortConverter converter = new StringToIpPortConverter();
  String source = "127.0.0.1:8080";
  IpPort result = converter.convert(source);
  assertThat(result).isEqualTo(new IpPort("127.0.0.1", 8080));
}
@Test
void ipPortToString() {
  IpPortToStringConverter converter = new IpPortToStringConverter();
  IpPort source = new IpPort("127.0.0.1", 8080);
  String result = converter.convert(source);
  assertThat(result).isEqualTo("127.0.0.1:8080");
}
```

그런데 이렇게 타입 컨버터를 하나하나 직접 사용하면, 개발자가 직접 컨버팅 하는 것과 큰 차이가 없다.
타입 컨버터를 등록하고 관리하면서 편리하게 변환 기능을 제공하는 컨버전 서비스를 이용해보자.

<br>
<hr>

### 컨버전 서비스 - ConversionService

스프링은 개별 컨버터를 모아두고 그것들을 묶어서 편리하게 사용할 수 있는 기능을 제공하는데, 이것이 바로 컨버전 서비스( ConversionService )이다.

```
public interface ConversionService {
  boolean canConvert(@Nullable Class<?> sourceType, Class<?> targetType);
  boolean canConvert(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType);
  <T> T convert(@Nullable Object source, Class<T> targetType);
  Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType);
}
```

사용 예를 확인해보자.

ConversionServiceTest - 컨버전 서비스 테스트 코드

```
public class ConversionServiceTest {
  @Test
  void conversionService() {
    //등록
    DefaultConversionService conversionService = new
    DefaultConversionService();
    conversionService.addConverter(new StringToIntegerConverter());
    conversionService.addConverter(new IntegerToStringConverter());
    conversionService.addConverter(new StringToIpPortConverter());
    conversionService.addConverter(new IpPortToStringConverter());
    //사용
    assertThat(conversionService.convert("10", Integer.class)).isEqualTo(10);
    assertThat(conversionService.convert(10, String.class)).isEqualTo("10");
    IpPort ipPort = conversionService.convert("127.0.0.1:8080", IpPort.class);
    assertThat(ipPort).isEqualTo(new IpPort("127.0.0.1", 8080));

    String ipPortString = conversionService.convert(new IpPort("127.0.0.1", 8080), String.class);
    assertThat(ipPortString).isEqualTo("127.0.0.1:8080");
  }
}
```

<br>

**등록과 사용 분리**

컨버터를 등록할 때는 StringToIntegerConverter 같은 타입 컨버터를 명확하게 알아야 한다. 반면에 컨버터를 사용하는 입장에서는 타입 컨버터를 전혀 몰라도 된다. 타입 컨버터들은 모두 컨버전 서비스 내부에 숨어서 제공된다. 따라서 타입을 변환을 원하는 사용자는 컨버전 서비스 인터페이스에만 의존하면 된다. 물론 컨버전 서비스를 등록하는 부분과 사용하는 부분을 분리하고 의존관계 주입을 사용해야 한다.

**인터페이스 분리 원칙 - ISP(Interface Segregation Principal)**

인터페이스 분리 원칙은 클라이언트가 자신이 이용하지 않는 메서드에 의존하지 않아야 한다.

<hr>
<br>

#### 스프링에 Converter 적용

WebConfig - 컨버터 등록

```
@Configuration
public class WebConfig implements WebMvcConfigurer {
  @Override
  public void addFormatters(FormatterRegistry registry) {
    registry.addConverter(new StringToIntegerConverter());
    registry.addConverter(new IntegerToStringConverter());
    registry.addConverter(new StringToIpPortConverter());
    registry.addConverter(new IpPortToStringConverter());
  }
}
```

WebMvcConfigurer 가 제공하는 addFormatters() 를 사용해서 추가하고 싶은 컨버터를 등록하면 된다. 이렇게 하면 스프링은 내부에서 사용하는 ConversionService 에 컨버터를 추가해준다.

?data=10 의 쿼리 파라미터는 문자이고 이것을 Integer data 로 변환하는 과정이 필요하다. 실행해보면 직접 등록한 StringToIntegerConverter 가 작동한다.

그런데 생각해보면 StringToIntegerConverter 를 등록하기 전에도 이 코드는 잘 수행되었다. 그것은 스프링이 내부에서 수 많은 기본 컨버터들을 제공하기 때문이다. 컨버터를 추가하면 추가한 컨버터가 기본 컨버터 보다 높은 우선순위를 가진다.

?ipPort=127.0.0.1:8080 쿼리 스트링이 @RequestParam IpPort ipPort 에서 객체 타입으로 변환된 것을 확인할 수 있다.

<br>
<hr>

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-dark"
        crossorigin="anonymous"
        async>
</script>
