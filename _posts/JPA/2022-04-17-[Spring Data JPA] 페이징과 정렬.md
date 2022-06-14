---
title: "[Spring Data JPA] 페이징과 정렬"
excerpt:
published: true
categories:
    - JPA

toc: true
toc_sticky: true

date: 2022-04-17
last_modified_at: 2022-04-17
---

## 페이징과 정렬

### JPA 페이징과 정렬

먼저 순수 JPA에서 페이징을 어떻게 하는지 알아보자.

-   검색 조건: 나이가 10살
-   정렬 조건: 이름으로 내림차순
-   페이징 조건: 첫 번째 페이지, 페이지당 보여줄 데이터는 3건

JPA 페이징 리포지토리

```
public List<Member> findByPage(int age, int offset, int limit) {
    return em.createQuery("select m from Member m where m.age = :age order by
    m.username desc")
    .setParameter("age", age)
    .setFirstResult(offset)
    .setMaxResults(limit)
    .getResultList();
}
public long totalCount(int age) {
    return em.createQuery("select count(m) from Member m where m.age = :age",
    Long.class)
    .setParameter("age", age)
    .getSingleResult();
}
```

JPA 페이징 테스트 코드

```
@Test
public void paging() throws Exception {
    //given
    memberJpaRepository.save(new Member("member1", 10));
    memberJpaRepository.save(new Member("member2", 10));
    memberJpaRepository.save(new Member("member3", 10));
    memberJpaRepository.save(new Member("member4", 10));
    memberJpaRepository.save(new Member("member5", 10));
    int age = 10;
    int offset = 0;
    int limit = 3;
    //when
    List<Member> members = memberJpaRepository.findByPage(age, offset, limit);
    long totalCount = memberJpaRepository.totalCount(age);
    //페이지 계산 공식 적용...
    // totalPage = totalCount / size ...
    // 마지막 페이지 ...
    // 최초 페이지 ..
    //then
    assertThat(members.size()).isEqualTo(3);
    assertThat(totalCount).isEqualTo(5);
}
```

<hr>

### 스프링 데이터 페이징과 정렬

페이징과 정렬 파라미터

-   org.springframework.data.domain.Sort : 정렬 기능
-   org.springframework.data.domain.Pageable : 페이징 기능 (내부에 Sort 포함)

특별한 반환 타입

-   org.springframework.data.domain.Page : 추가 count 쿼리 결과를 포함하는 페이징
-   org.springframework.data.domain.Slice : 추가 count 쿼리 없이 다음 페이지만 확인 가능
    (내부적으로 limit + 1조회)
-   List (자바 컬렉션): 추가 count 쿼리 없이 결과만 반환

페이징과 정렬 사용 예제

Page<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용

Slice<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용안함

List<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용안함

List<Member> findByUsername(String name, Sort sort);

<hr>

위의 순수 JPA로 페이징한 예시를 스프링 데이터 JPA 페이징으로 바꿔보자.

Page 사용 예제 정의 코드

```
public interface MemberRepository extends Repository<Member, Long> {
    Page<Member> findByAge(int age, Pageable pageable);
}
```

Page 사용 예제 테스트 코드

```
//페이징 조건과 정렬 조건 설정
@Test
public void page() throws Exception {
    //given
    memberRepository.save(new Member("member1", 10));
    memberRepository.save(new Member("member2", 10));
    memberRepository.save(new Member("member3", 10));
    memberRepository.save(new Member("member4", 10));
    memberRepository.save(new Member("member5", 10));
    //when
    PageRequest pageRequest = PageRequest.of(0, 3, Sort.by(Sort.Direction.DESC,"username"));
    Page<Member> page = memberRepository.findByAge(10, pageRequest);
    //then
    List<Member> content = page.getContent(); //조회된 데이터
    assertThat(content.size()).isEqualTo(3); //조회된 데이터 수
    assertThat(page.getTotalElements()).isEqualTo(5); //전체 데이터 수
    assertThat(page.getNumber()).isEqualTo(0); //페이지 번호 페이지는 0부터 시작
    assertThat(page.getTotalPages()).isEqualTo(2); //전체 페이지 번호
    assertThat(page.isFirst()).isTrue(); //첫번째 항목인가?
    assertThat(page.hasNext()).isTrue(); //다음 페이지가 있는가?
}
```

Page 인터페이스

```
public interface Page<T> extends Slice<T> {
  int getTotalPages(); //전체 페이지 수
  long getTotalElements(); //전체 데이터 수
  <U> Page<U> map(Function<? super T, ? extends U> converter); //변환기
}
```

Slice 인터페이스

```
public interface Slice<T> extends Streamable<T> {
  int getNumber(); //현재 페이지
  int getSize(); //페이지 크기
  int getNumberOfElements(); //현재 페이지에 나올 데이터 수
  List<T> getContent(); //조회된 데이터
  boolean hasContent(); //조회된 데이터 존재 여부
  Sort getSort(); //정렬 정보
  boolean isFirst(); //현재 페이지가 첫 페이지 인지 여부
  boolean isLast(); //현재 페이지가 마지막 페이지 인지 여부
  boolean hasNext(); //다음 페이지 여부
  boolean hasPrevious(); //이전 페이지 여부
  Pageable getPageable(); //페이지 요청 정보
  Pageable nextPageable(); //다음 페이지 객체
  Pageable previousPageable();//이전 페이지 객체
  <U> Slice<U> map(Function<? super T, ? extends U> converter); //변환기
}
```

페이지를 유지하면서 엔티티를 DTO로 변환하기

```
Page<Member> page = memberRepository.findByAge(10, pageRequest);
Page<MemberDto> dtoPage = page.map(m -> new MemberDto());
```

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
