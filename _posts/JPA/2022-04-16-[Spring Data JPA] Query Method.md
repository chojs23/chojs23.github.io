---
title: "[Spring Data JPA] Query Method"
excerpt:
published: true
categories:
  - JPA

toc: true
toc_sticky: true

date: 2022-04-16
last_modified_at: 2022-04-16
---

## 쿼리 메소드

쿼리 메소드 기능 3가지

1. 메소드 이름으로 쿼리 생성
2. 메소드 이름으로 JPA NamedQuery 호출
3. @Query 어노테이션을 사용해서 리파지토리 인터페이스에 쿼리 직접 정의

### 메소드 이름으로 쿼리 생성

메소드 이름을 분석해서 JPQL 쿼리 실행

ex)이름과 나이를 기준으로 회원을 조회하려면?

순수 JPA 리포지토리

```
public List<Member> findByUsernameAndAgeGreaterThan(String username, int age) {
    return em.createQuery("select m from Member m where m.username = :username and m.age > :age")
    .setParameter("username", username)
    .setParameter("age", age)
    .getResultList();
}
```

스프링 데이터 JPA

```
public interface MemberRepository extends JpaRepository<Member, Long> {
 List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
}
```

스프링 데이터 JPA는 메소드 이름을 분석해서 JPQL을 생성하고 실행

쿼리 메소드 필터 조건
스프링 데이터 JPA 공식 문서 참고: (https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation)

### JPA NamedQuery

JPA의 NamedQuery를 호출할 수 있음

@NamedQuery 어노테이션으로 Named 쿼리 정의

```
@Entity
@NamedQuery(
 name="Member.findByUsername",
 query="select m from Member m where m.username = :username")
public class Member {
 ...
}
```

JPA를 직접 사용해서 Named 쿼리 호출

```
public class MemberRepository {
 public List<Member> findByUsername(String username) {
 ...
 List<Member> resultList =
 em.createNamedQuery("Member.findByUsername", Member.class)
 .setParameter("username", username)
 .getResultList();
 }
}
```

스프링 데이터 JPA로 NamedQuery 사용

```
List<Member> findByUsername(@Param("username") String username);
```

### @Query, 리포지토리 메소드에 쿼리 정의하기

메서드에 JPQL 쿼리 작성

```
public interface MemberRepository extends JpaRepository<Member, Long> {
    @Query("select m from Member m where m.username= :username and m.age =:age")
    List<Member> findUser(@Param("username") String username, @Param("age") int
    age);
}
```

단순히 값 하나를 조회

```
@Query("select m.username from Member m")
List<String> findUsernameList();
```

#### 파라미터 바인딩

파라미터 바인딩
위치 기반
이름 기반

```
select m from Member m where m.username = ?0 //위치 기반
select m from Member m where m.username = :name //이름 기반
```

#### 반환 타입

스프링 데이터 JPA는 유연한 반환 타입 지원

```
List<Member> findByUsername(String name); //컬렉션
Member findByUsername(String name); //단건
Optional<Member> findByUsername(String name); //단건 Optional
```

조회 결과가 많거나 없으면?

- 컬렉션
  - 결과 없음: 빈 컬렉션 반환
- 단건 조회
  - 결과 없음: null 반환
  - 결과가 2건 이상: javax.persistence.NonUniqueResultException 예외 발생
    > 참고: 단건으로 지정한 메서드를 호출하면 스프링 데이터 JPA는 내부에서 JPQL의
    > Query.getSingleResult() 메서드를 호출한다. 이 메서드를 호출했을 때 조회 결과가 없으면
    > javax.persistence.NoResultException 예외가 발생하는데 개발자 입장에서 다루기가 상당히
    > 불편하다. 스프링 데이터 JPA는 단건을 조회할 때 이 예외가 발생하면 예외를 무시하고 대신에 null 을
    > 반환한다.

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-dark"
        crossorigin="anonymous"
        async>
</script>
