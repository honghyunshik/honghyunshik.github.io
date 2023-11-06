---
title: Effective Java 스터디 2장
author: honghyunshik
date: 2023-11-02 14:30:00 +0800
categories: [JAVA]
tags: [effective,java]
---

이펙티브 자바(Joshua Bloch) 책을 읽고 정리 및 느낀점을 기술하는 포스트입니다.
이번 포스트는 2장입니다.

# 2장 - 객체 생성과 파괴

## 주요 내용

    1. 객체를 만들어야 할 때와 만들지 말아야 할때를 구분하는 법
    2. 올바른 객체 생성 방법과 불필요한 생성을 피하는 방법
    3. 제때 파괴됨을 보장하고 파괴 전에 수행해야 할 정리 작업을 관리하는 요령

## 1. 생성자 대신 정적 팩터리 메서드를 고려하라
***

클래스의 인스턴스를 얻는 대표적인 방법은 public 생성자다.
하지만 정적 팩토리 메서드(static factory method)를 제공할 수 있다.


다음 코드는 boolean primitive type -> Boolean Reference type으로 변환하는 코드다.
```java
public static Boolean valueOf(boolean b){
  return b ? Boolean.TRUE : Boolean.FALSE;
}
```

### 왜 굳이 이런 방식을 택할까?

- 장점
  - 이름을 가질 수 있다. 따라서 객체의 특징을 쉽게 묘사할 수 있다.
    - 입력 매개변수들의 순서를 다르게 생성자를 추가하면 가독성이 떨어진다 -> Builder 패턴 사용하면 되지 않나?
  - 호출될 때마다 인스턴스를 새로 생성하지 않아도 된다.
    - 불변 클래스는 인스턴스를 미리 만들어 놓거나, 캐싱하여 재활용 할 수 있다
  - 반환 타입의 하위 타입 객체를 반환할 수 있다.
  - 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다.
    - EunmSet 클래스는 원소의 수에 따라 두 가지 하위 클래스 중 하나의 인스턴스를 반환한다.
  - 정적 팩토리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.
- 단점
  - 상속을 하려면 public이나 protected 생성자가 필요하니 정적 팩토리 메서드만 제공하면 하위클래스를 만들 수 없다.
  - 정적 팩토리 메서드는 프로그래머가 찾기 어렵다.
    - 생성자처럼 API 설명에 명확히 드러나지 않는다.

### 주요 명명 방식들

- from : 매개변수를 하나 받아서 해당 타입의 인스턴스 반환
````java
Date d = Date.from(instance);
````

- of : 여러 매개변수를 받아 적합한 타입의 인스턴스를 반환
````java
Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);
````

- valueOf : from과 of의 더 자세한 버전
````java
BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);
````

- getInstance : 인스턴스를 반환하지만, 같은 인스턴스임을 보장하지는 않는다
````java
StackWalker luke = StackWalker.getInstance(options);
````

- new Instance : 매번 새로운 인스턴스를 생성해 반환함을 보장한다
````java
Object newArray = Array.newInstance(classObject, arrayLen);
````

## 2. 생성자에 매개변수가 많다면 빌더를 고려하라

***

만약 생성자가 100 개라면 어떨까? 100개의 매개변수를 순서를 맞춰 생성자를 생성하는 것은 매우 고역이다.
이때 사용할 수 있는 방법은

    1. 기본 생성자로 인스턴스 생성 + setter로 설정하기(자바빈즈 패턴)
    2. Builder 패턴 활용하기

하지만 자바빈즈 패턴은 심각한 문제점을 가지고 있다.

    1. 메소드를 매우 많이 호출해야 한다.
    2. 객체가 생성되기 전에는 일관성이 무너진 상태이다.
    3. 일관성이 무너지는 문제 때문에 클래스를 불변으로 만들 수 없다.
    4. 스레드의 안정성을 위해 추가적인 작업이 필요하다.

따라서 빌더 패턴을 사용하는 것이 권장된다!
````java
NutritionFacts cocaCola = new NutritionFacts.Builder()
                              .calories(100)
                              .sodium(35)
                              .build();
````
훨씬 쓰기 쉽고, 읽기 쉽다는 것을 알 수 있다.


빌더 패턴은 계층적으로 설계된 클래스와 함께 쓰기 좋다.
이 책에서는 Pizza > Cheeze, Pepperoni로 설명하고 있다.

### Pizza 클래스(부모 클래스)

````java
public abstract class Pizza{
  public enum Topping { HAM, MUSHROOM, ONION}
  final Set<Topping> toppings;    //Enum이 불변이므로 토핑은 불변이다

  abstract static class Builder<T extends Builder<T>> {
    EnumSet<Topping> toppings = EnumSet.noneOf(Toping.class);
    public T addTopping(Topping topping){
      toppings.add(Objects.requireNonNull(topping));
      return self();
    }

    abstract Pizza build();
    
    //하위 클래스는 이 메서드를 overriding 하여 this를 반환한다
    protected abstract T self();
  }

  Pizza(Builder<?> builder){
    toppings = builder.toppings.clone();
  }
}
````
추상 메서드인 self를 더해 하위클래스에서 형변환하지 않고 메서드 연쇄를 지원할 수 있다.

### CheezePizza 클래스(자식 클래스 1)

````java
public class CheezePizza extends Pizza{
    public enum Size{ SMALL, MEDIUM, LARGE }
    private final Size size;
    
    public static class Builder extends Pizza.Builder<Builder> {
        private final Size size;
        
        public Builder(Size size){
            this.size = Objects.requireNonNull(size);
        }
        
        @Override
        public CheezePizza build(){
            return new CheezePizza(this);
        } 
        
        @Override
        protected Builder self() {return this;}
    }
    
    private CheezePizza(Builder builder){
        super(builder);
        size = builder.size;
    }
}
````
치즈 피자는 size 매개변수를 필수로 받는다.

### 페퍼로니 피자 클래스(자식 클래스2)

````java
public class Pepperoni extends Pizza{
    private final boolean sauceInside;
    
    public static class Builder extends Pizza.Builder<Build>{
        private boolean sauceInside = false;
        
        public Builder sauceInsde(){
            sauceInside = true;
            return this;
        }
        
        @Override
        public Pepperoni build(){
            return new Pepperoni(this);
        }
        
        @Override
        protected Builder self() { return this;}
    }
    
    private Pepperoni(Builder builder){
        super(builder);
        sauceInside = builder.sauceInside;
    }
}
````
페퍼로니 피자는 소스를 안에 넣을지 선택(sauceInside)하는 매개변수를 필수로 받는다.

각 코드에 대해서 설명해보도록 하겠다.
우선 Pizza Class는 추상 클래스로 Builder 인스턴스를 매개변수로 받는 생성자를 갖는다.
또한 필드값으로 Enum 타입의 토핑과 이를 Set에 저장한 불변객체 toppings를 갖는다.
마지막으로 내부 클래스로 Builder를 가지고 있다.

추상 클래스인 Builder 클래스는 필드값으로 EnumSet 타입의 toppings를 가지고 있다. 
addToping method를 통해 toppings에 topping을 더하고, self()를 반환한다.
이때 self()와 build()는 추상 메소드로 자식 클래스에 구현을 위임한다.

자식 클래스에서는 추가적인 기능과 build()와 self()에 대한 구현을 담당한다.

이런식으로 빌더 패턴을 통해 유연한 개발이 가능하지만, 단점도 존재한다.

    1. 빌더 생성 비용이 발생한다
    2. 매개 변수가 4개 이상은 되어야 효율적이다

하지만 API는 시간이 지날수록 매개변수가 많아지므로, 처음부터 빌더 패턴을 사용하는 것이 효율적이다.


## 3. private 생성자나 열거 타입으로 싱글턴임을 보증하라

***

### Singleton 패턴이란?
- 인스턴스를 오직 하나만 생성할 수 있는 클래스 ex) 함수 객체, 시스템 컴포넌트
- 한계)
  - 싱글턴 클래스를 사용하는 클라이언트를 테스트하기 어렵다
  - 왜? 싱글턴 인스턴스를 mock 구현으로 대체할 수 없기 때문에
  - mocking하려면 새로운 인스턴스를 만들어야 한다 but 싱글턴 패턴의 경우 새로운 인스턴스를 생성하는 것이 불가능하다

### Singleton 패턴 만드는 방식 1- public static final

````java
public class Elivs{
  public static final Elvis INSTANCE = new Elvis();
  private Elvis(){ ... }
  
  public void leaveTheBuilding() { ... }
}
````
첫 번째 방식은 생성자를 private으로 감춰두고, 이는 Elvis.INSTANCE를 초기화할 때 딱 한번 호출된다.

public이나 protected 생성자가 없으므로 인스턴스가 전체 시스템에서 하나뿐임이 보장된다. 

예외) Reflection API를 통해 private 생성자를 호출할 수 있다

방어) 두 번째 객체가 생성되려 할 때 예외를 던진다


- 장점
  - 해당 클래스가 싱글턴임이 API에 명백히 드러난다.
  - 간결한다.

### Singleton 패턴 만드는 방식 2 - 정적 팩토리 메서드

````java
public class Elvis{ 
    private static final Elvis INSTANCE = new Elvis();
    private Elvis(){ ... }
    private static Elvis getInstance() {return INSTANCE;}
  
  public void leaveTheBuilding(){ ... }
}
````
두 번째 방식은 정적 팩터리 메소드를 public static 멤버로 제공한다.

Elvis.getInstance()는 항상 같은 객체의 참조를 반환하므로 인스턴스가 전체 시스테에서 하나뿐임이 보장된다.

첫 번째 방식과 마찬가지로 Reflection 예외는 존재한다.

- 장점
  - API를 변경하지 않고 싱글턴이 아니게 변경할 수 있다.
  - 정적 팩토리 -> 제너릭 싱글턴 팩토리로 만들 수 있다.
  - 정적 팩토리의 메서드 참조를 supplier로 사용할 수 있다.

- 위 두 방식의 단점
  - 직렬화된 인스턴스를 역직렬화 할 때마다 새로운 인스턴스가 만들어진다.
  - readResolve 메소드를 제공해야 하는데 복잡하다.

### Singleton 패턴 만드는 방식 3 - 열거 타입 선언

````java
public enum Elivs{
    INSTANCE;
    
    public void leaveTheBuilder(){ ... }
}
````
훨씬 간결하고, 추가 노력 없이 직렬화 할 수 있다. 하지만 싱글턴이 Enum 외의 클래스를 상속해야 한다면 사용할 수 없다.


## 4. 인스턴스화를 막으려거든 private 생성자를 사용하라
***
정적 메서드와 정적 필드만을 담은 클래스를 만들 때가 있다. 이러한 클래스는 인스턴스로 만들어 쓰려고 만든 것이 아니다.
하지만 생성자를 명시하지 않으면 컴파일러가 자동으로 기본 생성자를 만들어준다. 이때 생성자를 private을 만들어주면 된다!

````java
public class UtilityClss{
    //인스턴스화 방지
    private UtilityClss(){
        //throw new AssertionError();
    }
}
````

이 방식은 상속을 불가능하게 하는 효과도 있다! 
