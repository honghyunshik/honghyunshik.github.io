---
title: Lazy Loading 및 N+1 문제 해결전략
author: honghyunshik
date: 2023-11-07 14:30:00 +0800
categories: [SpringBoot,JPA]
tags: [springboot, jpa, n+1]
---

## Lazy Loaindg이란?
***
- 당장 필요하지 않은 데이터나 객체를 추후에 로딩하게 하는 기술입니다.

### Lazy Loading의 장점
- 불필요한 자원의 사용을 방지해줍니다.
- 초기 로딩 시간을 줄여줍니다.

### Lazy Loading 사용처
- DB 쿼리 : 특정 연관 객체나 데이터를 액세스할때까지 로드하지 않습니다.
- 웹 페이지 이미지 : 스크롤이 내려갈 때마다 이미지를 로드합니다.
- 싱글턴 패턴 : 싱글턴 패턴을 적용한 인스턴스는 처음 접근될 때 생성됩니다.

## Fetch Type
- JPA가 Entity를 조회할 때, 연관관계에 있는 객체들을 어떻게 가져올것이냐 하는 설정값입니다.
- Eager 전략 : 연관 관계에 있는 Entity들을 모두 가져옵니다.
- Lazy 전략 : 연관 관계에 있는 Entity들을 가져오지 않고 getter로 접근할 때 가져옵니다.

### 구현

N:1 관계를 가지고 있는 Member과 Posts 테이블을 만들어보겠습니다.
하나의 Member는 여러개의 Posts를 작성할 수 있습니다.

Posts Entity는 @ManyToOne 어노테이션을 통해 N:1관계를 표현합니다.
````java
public class Posts{
  @ManyToOne
  @JointColumn(name = "member_id", referencedColumnName = "member_id")
  private Member member;
}
````

Member Entity는 @OneToMany 어노테이션을 통해 N:1관계를 표현합니다.
````java
public class Member{
    @OneToMany
    private List<Posts> postsList;
}
````

### 주인 vs 비주인

- 주인 : 양방향 관계에서 외래 키를 관리하는 쪽을 말합니다.
- 비주인 : 주인의 반대입니다. 자신의 PK가 외래키로서 관리받습니다.

주인 테이블에 외래키가 있기 때문에, 외래 키의 실제 변경 및 관리는 주인 테이블에서 발생합니다.

Member과 Posts 테이블의 경우 Posts 테이블에 Member 테이블에 대한 외래키가 존재합니다.

따라서 Posts 테이블이 주인, Member 테이블이 비주인입니다.

### 각 관계별 Default FetchType

- @OneToMany : Lazy Fetch
- @ManyToOne : Eager Fetch
- @OneToOne : Eager Fetch
- @ManyToMany : Lazy Fetch

=> 해당 Entity와 매핑되어 있는 Entity가 여러개라면 Lazy, 하나라면 Eager

### FetchType이 동작하는 시점

<img src="/assets/img/2023-11-07-lazy-loading/fetchtype.png" alt="fetchType 동작시점" width="350" height="250">

JPA Entity Manager에 의해 관리되는 Persistence Context에 Entity가 Managed 상태로 올라올 때 동작합니다!

### ManyToOne(Default = EAGER FETCH)
````roomsql
select
	posts.id,
	posts.member_id,
	member.id,
	member.name
from posts
outer join member
	on posts.member_id=member.id;
````
posts.getMember 메서드를 호출했다고 해볼게요. EAGER 전략은
이미 해당 posts와 매핑된 member 정보가 Entity Manager에 캐싱되어 있습니다.
한번에 posts와 join된 member을 읽어옵니다. 추가적인 쿼리는 필요하지 않아요.

### ManyToOne(fetch = FetchType.LAZY)
posts 테이블을 불러올 때 join 연산이 발생하지 않습니다.(posts.member_id 칼럼도 미리 불러옵니다) 하지만, posts.getMember를 통해 
join된 member Entity에 접근하려고 할 경우 **LazyInitializationException**이 발생합니다.

해당 Entity에 접근할 때 원래의 세션 혹은 영속성 컨텍스트가 닫혀있기 때문입니다.

이를 방지하기 위해선 @Transactional 어노테이션을 통해 트랜잭션 범위 내에 있게 해주어야 합니다.

만약 Transaction 내에서 posts.getMember Entity에 접근하면
````roomsql
select
    member.id,
    member.name
  from member
  where member.id=999;
````
이렇게 추가적인 쿼리가 발생합니다!

자, 그런데 이렇게 N개의 posts에 대해서 getMember을 한다면?

posts를 가져오는 쿼리 1개, 각각 getMember를 가져오는 쿼리 N개 총 N+1개의 쿼리가 실행되니다!

posts.getMember 쿼리 1개만 날렸는데 N+1개의 쿼리가 발생하는 **N+1문제** 라고 합니다.

### @OneToMany(Default=LAZY)

````roomsql
select
    member.id,
    member.name
  from member;
````

member에 대해서 가져올 때 쿼리가 하나 발생합니다.
이후 member.getPosts를 할 경우 N번의 쿼리가 더 발생하고 N+1 문제가 발생하겠네요.

### @OneToMany(fetch = FetchType.EAGER)

````roomsql
select
    member.id,
    member.name
  from member;
  
 select
    posts.id,
    posts.member_id
  from posts
  where member_id = 999;
  
select
    posts.id,
    posts.member_id
  from posts
  where member_id = 888;  
  
````

member를 가져온 후에 매핑된 posts도 같이 가져옵니다. 이때도 N+1개의 쿼리가 발생하네요.

대부분의 경우 N+1 문제가 발생할 수 있네요! 그럼에도 성능적인 문제 때문에 대부분의 경우 LAZY Loading을 사용한다고 합니다.

그럼 이번에는 N+1문제가 왜 발생하는지, 어떻게 해결하는지 알아보도록 하겠습니다!

### N+1 문제는 왜 발생할까?

예시를 들어보겠습니다.

1. **Fetch 전략이 EAGER인 경우(N:1관계에서 비주인 Entity가 주인 Entity를 참조하려는 경우)**
   1) findAll()을 한 순간 select m from Member m 이라는 JPQL 구문이 생성되고 해당 구문을 분석한 
select * from Member이라는 SQL이 생성되고 실행됩니다.
   2) DB의 결과를 받아 Member 엔티티의 인스턴스들을 생성합니다.
   3) 영속성 컨텍스트에서 연관된 Posts가 있는지 확인합니다.(있다면 추가적인 쿼리 없이 이를 반환합니다)
   4) 없다면 (2)에서 반환된 Member 인스턴스 개수에 맞게 select * from
posts where member_id=? 이라는 SQL 구문이 생성됩니다.(N+1 발생!!)

   2. **Fetch 전략이 LAZY인 경우
      1) findAll()을 한 순간 select m from Member m 이라는 JPQL 구문이 생성되고
   해당 구문을 분석한 select * from member이라는 SQL이 생성되고 실행됩니다.
      2) DB의 결과를 받아 Member 엔티티의 인스턴스들을 생성합니다.
      3) 이후 Member의 Posts 객체를 사용하려고 하는 시점에 영속성 컨텍스트에서 연관된 Posts가 있는지 확인합니다
         (있다면 추가 쿼리 발생 X)
      4) 없다면 (2)에서 반환된 Member 인스턴스 개수에 맞게 select * from posts where member_id=?
   라는 SQL 구문이 생성됩니다.(N+1 발생!!)

         
또한, JPA가 JPQL을 분석해서 SQL을 생성할 때 글로벌 Fetch전략을 참고하지 않고 오직 JPQL 자체만을 사용합니다.

이 말이 무엇이냐 하면, 글로벌 Fetch 전략이 LAZY로 설정되어 있더라도 JPQL 쿼리가 별개의 로딩 전략을 명시하면, 이를 우선시 한다는 말입니다.

### N+1 문제 해결 전략

#### 1. Fetch Join

JPQL을 사용하여 DB에서 데이터를 가져올 때 처음부터 연관된 데이터까지 같이 가져옵니다.

@Query 어노테이션을 통해 별도의 메소드를 만들어주어야 합니다!

````java
public interface MemberRepository extends JpaRepository<Member,Long>{
    @Query("select m from Member m join fetch m.posts")
    List<Member> findAllFetchJoin();
}
````

이는 마치 @ManyToOne EAGER 전략처럼 작동합니다.

한계)
  - N:1 관계가 두 개 이상인 경우 사용이 불가합니다.
  - 패치 조인 대상에게 별칭(as) 부여 불가능합니다.

발생 가능한 문제)
  - 카테시안 곱이 발생하여 중복이 발생할 수 있습니다.

**카테시안 곱이란?** 

두 테이블 사이에 유효 join 조건을 적지 않을 경우
해당 테이블의 모든 데이터를 전부 결합하여 CROSS JOIN으로 동작한다.

해결 방법)
  - JPQL에 DISTINCT를 추가하여 중복을 제거합니다
    - @Query("select DISTINCT m from Member m join fetch m.posts)
  - OneToMay 필드 타입을 Set으로 선언합니다
    - private Set<Posts> posts;

#### 2. Batch Size

application.yml
````yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 1000
````

이렇게 설정을 해주면 selct * from posts where member_id=?
대신에 select * from posts where member_id in (?,?,?) 방식으로 동작합니다.

N+1 문제를 완전히 해결하지는 못하지만 쿼리가 1번만 더 발생하게 해줍니다.

#### 결론

1. FetchType은 성능을 위해 LAZY 전략을 사용합니다.
2. 성능 최적화를 할 때 fetch join 사용합니다.
3. 기본적인 Batch Size를 1000 이하로 설정합니다.
