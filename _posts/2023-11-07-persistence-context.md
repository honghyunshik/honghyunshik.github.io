---
title: 영속성 컨텍스트
author: honghyunshik
date: 2023-11-07 16:30:00 +0800
categories: [SpringBoot,JPA]
tags: [springboot, jpa, entity]
---

## 영속성 컨텍스트란?
***
- 엔티티를 영구 저장하는 환경을 말합니다.
- 엔티티의 생명 주기를 관리합니다.
- 애플리케이션 - DB 사이에서 객체를 보관하는 가상의 DB 역할을 합니다.
- JPA Interface인 Entity Manager를 통해 영속성 컨텍스트에 엔티티를 보관하고 관리합니다. 
- Entity Manager를 생성할 때 하나 만들어지고, Entity Manager가 종료될 때 사라집니다.
- Entity Manager는 DB와 통신하는 물리적인 개념이고, 영속성 컨텍스트는 엔티티 객체를 메모리에 캐싱하고
생명주기를 관리하는 논리적인 개념입니다.

````java
entityManager.persist(entity)
````

위 코드를 통해 Entity를 영속성 컨텍스트에 저장합니다. 해당 entity는 아직은 DB
에 반영되지 않은 상태이고, 트랜잭션이 끝나야 반영합니다.

## Entity Manager란?

- EntityManagerFactory는 Thread-safe 하고 Entity Manager의 역할을 할 수 있지만
생성하는 비용이 상당히 큽니다.
- 따라서 요청이 올 때 마다 EntityManagerFactory를 통해 생성 비용이 상대적으로 적은 EntityManager를 생성합니다.
- EntityManager은 Thread-safe 하지 않기 때문에 동시성 문제가 발생할 수 있습니다.
=> 따라서 요청(스레드)별로 한 개씩 할당합니다.
- 이후 DB 연결이 필요한 시점에 DB Connection을 통해 DB와 연결합니다.

### Entity 생명주기

<img src="/assets/img/2023-11-07-persistence-context/entity-lifecycle.png" alt="entity 생명주기" width="300" height="200">

**비영속(new)*** : 아직 영속성 컨텍스트나 DB와 연관되지 않은 상태

    Member member = new Member("유저");

**영속(managed)** : 메서드를 통해 영속성 컨텍스트에 저장된 상태. Entity는 이후
영속성 컨텍스트에 의해 관리되며 DB와 동기화됩니다.

    entityManager.persist(member);
**준영속(detached)** : 영속성 컨텍스트에서 분리된 상태

    entityManager.detach(member);
    entityManager.clear();  //entityManager 비워도 준영속 상태 유지
    entityManager.close();  //entityManager 종료해도 준영속 상태 유지

! 준영속 상태에서는 1차 캐시, 쓰기 지연, 변경 감지, 지연 로딩을 포함한 영속성 컨텍스트가
제공하는 어떠한 기능도 동작하지 않습니다. 

즉, 더 이상 Entity Manager가 관리하지 않습니다!

단, 식별자 값은 가집니다. 

**식별자 값이란?**

=> 영속성 컨텍스트는 엔티티를 식별자 값(@Id로 테이블의 기본키와 매핑한 값)
으로 구분합니다.

**언제 준영속 상태가 될까요?**

=> 트랜잭션 종료, entityManager.detach(entity), entityManager.clear(),
entityManager.close(), DB에 영구저장 등

**준영속 상태가 유지된다면?**

=> 메모리 누수가 발생할 수 있고, 영속성 컨텍스트의 관리를 받지 않으므로 데이터 일관성 문제가 발생할 수 있습니다!

**그럼 어떻게 해야될까요?**

=> merge() 메서드를 통해 다시 영속 상태로 만들기 or 가바지 컬렉션을 통해 메모리 해지하기

**삭제(removed)** : Entity를 삭제한 상태. 커밋 시 DB에 해당 Entity가 삭제됩니다.

    entityManager.remove(member);

**DB에 영구 저장하기**

JPA는 보통 트랜잭션을 커밋하는 순간 영속성 컨텍스트에 새로 저장된 엔티티를 DB에 반영하고
삭제된 엔티티를 DB에서 삭제합니다. 이후 해당 Entity는 준영속 상태가 됩니다.


### 영속성 컨텍스트가 제공하는 기능들

#### 1. 1차 캐시

- 영속성 컨텍스트 내부에 있는 캐시입니다.
- Map의 형태로 만들어지며 Key는 식별자 값, value는 해당 entity 인스턴스가 저장되어 있습니다.
- 1차 캐시에서 Entity를 우선 찾고, 없으면 DB를 조회합니다.
- 이때 DB를 조회했다면 해당 데이터로 엔티티를 생성하고, 1차 캐시에 저장(영속)한 뒤 반환합니다.

#### 2. 동일성 보장

- 동일성 : 실제 인스턴스가 같다(주소값이 같다, ==로 비교)
- 동등성 : 실제 인스턴스는 다르지만 필드값이 같다.(eqauls로 비교)
- 영속성 컨텍스트는 **동일성**을 보장합니다!

#### 3. 트랜잭션을 지원하는 쓰기 지연

- Entity를 영속성 컨텍스트에 저장하거나 삭제해도 바로 DB에 반영하지 않습니다.
- 내부 쿼리 저장소에 모아두다가 트랜잭션의 commit 시 쿼리를 보냅니다.

#### 4. Dirty Checking(변경 감지)

- 트랜잭션 안에서 Entity를 조회하면 영속성 컨텍스트가 유지된 상태입니다.
- 영속성 컨텍스트가 유지된 상태에서 엔티티의 값을 변경하면 트랜잭션이 끝나는 시점에
해당 테이블에 변경분을 반영합니다.
- JPA로 엔티티를 수정할 때 따로 Update 쿼리를 날리지 않고,
해당 엔티티를 조회 후 데이터만 변경하면 됩니다!

##### Dirty Checking 흐름

1. 트랜잭션을 커밋하면 entityManager 내부에서 flush()가 호출됩니다.
2. 엔티티와 스냅샷(최초 상태)을 비교하여 변경된 엔티티를 찾습니다.
3. 변경된 엔티티의 수정 쿼리를 생성해서 쿼리 저장소에 저장합니다.
4. 쿼리 저장소의 쿼리를 flush()합니다.
5. DB 트랜잭션을 커밋합니다.

##### flush()

- 영속성 컨텍스트의 변경 내용을 DB에 동기화합니다.

1. Dirty Checking으로 스냅샷과 비교해 수정된 엔티티를 찾습니다.
2. 수정 쿼리를 만들고 쿼리 저장소에 저장합니다.
3. 트랜잭션 Commit 후에 쿼리를 DB로 보냅니다.

flush() 호출되는 경우 =>
1. entityManager.flush()
2. 트랜잭션 Commit 시 자동 호출
3. JPQL 쿼리 실행 시 자동 호출

#### 5. Lazy Loading

연관된 엔티티나 컬렉션을 실제로 사용하는 시점에 로딩합니다.
