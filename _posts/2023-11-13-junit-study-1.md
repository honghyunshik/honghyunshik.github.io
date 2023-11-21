---
title: 자바와 JUnit을 활용한 실용주의 단위 테스트 스터디 1일차
author: honghyunshik
date: 2023-11-13 18:30:00 +0800
categories: [Java, JUnit]
tags: [springboot, test, study]
---

## 1. 첫 번째 테스트 만들기

이 책의 예제코드는 JUnit 4.X 버전을 기반으로 작성되었습니다. 저는 JUnit5에 더 익숙하므로
예제코드를 변경해서 작성하도록 하겠습니다.

### 1. 단위 테스트를 작성하는 이유

1. 시간을 절약할 수 있습니다 -> 매번 빌드하고 실행하고... 오래 걸립니다.
2. 코드를 추가할 때 유용합니다 -> 수정하거나 추가할 때 어느 부분이 잘못됐는지 확인하기 편합니다.
3. 시스템을 이해하기 쉬워집니다.

### 2. JUnit의 기본 : 첫 번째 테스트 통과

````java
public class ScoreCollection{
    private List<Scoreable> scores = new ArrayList<>();
    
    public void add(Scoreable scoreable){
        scores.add(scoreable);
    }
    
    public int arithmeticMean(){
        int total = scores.stream().mapToInt(Scoreable::getScore).sum();
        return total/scores.size();
    }
}
````

위의 클래스에 대한 간단한 테스트 코드를 작성해보겠습니다.

````java
public class ScoreCollectionTest{
    
    @Test
    public void test(){
        fail("Not yet implemented");
    }
}
````

테스트 원칙
    
    1. 테스트 대상 클래스 + Test를 붙여서 테스트 클래스 이름을 만듭니다.
    2. JUnit은 @Test 어노테이션이 붙은 메서드만 테스토로 실행합니다.

### 3. 테스트 준비, 실행, 단언

````java
public class ScoreCollectionTest{
    @Test
    public void answersArithmeticMeanofTwoNumbers(){
        //준비
        ScoreCollecion collection = new ScoreCollection();
        collection.add(()->5);
        collection.add(()->7);
        
        //실행
        int actualResult = collection.arithmeticMean();
        
        //단언
        assertEquals(actualResult,6);
    }
}
````

**※Tip** : 가장 마지막에 단언문 넣기!

### 4. 테스트가 정말로 뭔가를 테스트하는가?

**정상적으로 동작하는지 증명하기 위해 의도적으로 테스트에 실패하세요!!**

### 5. 마치며

질문

    1. 코드가 정상적으로 동작하는지 확신하려고 추가적인 테스트를 작성할 필요가 있는가?
    2. 내가 클래스에서 결함이나 한계점을 드러낼 수 있는 테스트를 작성할 수 있을까?

답변

    1. scores.size()가 0이라면 어떻게 될까?
    2. scores의 합이 int 범위를 넘어갈 수 있다?

위의 상황들을 생각해서 테스트 코드를 작성할 필요가 있어보입니다. 


## 2. JUnit 진짜로 써 보기

### 1. 테스트 대상 이해: Profile 클래스

iloveyouboss 애플리케이션의 일부를 테스트합니다. 이 애플리케이션은 잠재적인 구인자에게 유망한
구직자를 매칭하고 반대 방향에 대한 서비스도 제공합니다. 

구인자와 구직자는 둘 다 다수의 객관식 혹은 yes-no 질문에 대한 대답을 하는 프로파일을 생성합니다. 

````java
public class Profile {
  private Map<String,Answer> answers = new HashMap<>();
  private int score;
  private String name;

  public Profile(String name) {
    this.name = name;
  }

  public String getName() {
    return name;
  }

  public void add(Answer answer) {
    answers.put(answer.getQuestionText(), answer);
  }

  public boolean matches(Criteria criteria) {
    score = 0;

    boolean kill = false;
    boolean anyMatches = false;
    for (Criterion criterion: criteria) {
      Answer answer = answers.get(criterion.getAnswer().getQuestionText());
      boolean match = criterion.getWeight() == Weight.DontCare ||
          answer.match(criterion.getAnswer());

      if (!match && criterion.getWeight() == Weight.MustMatch) {
        kill = true;
      }
      if (match) {
        score += criterion.getWeight().getValue();
      }
      anyMatches |= match;
    }
    if (kill)
      return false;
    return anyMatches;
  }

  public int score() {
    return score;
  }
}
````

Question 객체는 질문 내용 + 답변이 가능한 범위(ex - true/false)를 포함합니다.

Answer 객체는 질문에 대한 답을 포함합니다.

Criteria 인스턴스는 다수의 Criterion 객체를 담는 컨테이너입니다.

Criterion 객체는 Answer 객체와 그 질문의 중요도 Weight를 캡슐화합니다.

matches() 메서드는 Criteria의 각 Criterion이 프로파일에 있는 답변과 맞는지 결정합니다.

### 2. 어떤 테스트를 작성할 수 있는지 결정

어느정도의 테스트 코드를 작성해야 할까요? 

시작점은 반복문, if문과 복잡한 조건문들로 시작합니다. 이후에는 데이터 변형도 고려해봅니다.
데이터가 null이거나 0이라면?

Criteria 인스턴스가 단순히 Criterion 객체 한 개를 포함하는 것은 문제가 없겠죠. 하지만 그렇게 단순하지 않습니다.
다음은 고려해 볼만한 테스트 케이스입니다.

    1. Criteria 인스턴스가 Criterion 객체를 포함하지 않을 때
    2. Criteria 인스턴스가 다수의 Criterion 객체를 포함할 때
    3. answers.get()에서 반환된 Answer 객체가 null일 때
    4. criterion.getAnswer() 혹은 criterion.getAnswer().getQuestionText()의 반환값이 null일 때
    ...

단순히 객체가 null이라는 가정만 해도 수없이 많은 테스트 케이스가 존재할 수 있습니다. 이걸 다 작성해야 하나요?

### 3. 단일 경로 커버

matches() 메서드에서 필요한 로직은 대부분 for loop 안에 있습니다. 우선은 Profile 인스턴스와 Criteria 인스턴스가 필요하니 생성해 줍시다.

````java
public class ProfileTest {

  @Test
  public void test() {
    Profile profile = new Profile("Bull Hockey, Inc.");
    Question question = new BooleanQuestion(1, "Got bonuses?");
    Criteria criteria = new Criteria();
    Answer criteriaAnswer = new Answer(question, Bool.TRUE);
    Criterion criterion = new Criterion(criteriaAnswer, Weight.MustMatch);
    criteria.add(criterion);
  }
}
````

matches() 메서드에서 for loop를 돌면서 answers 해시 맵에서 각 Criterion에 대응하는 Answer 객체를 가져옵니다.
따라서 사전에 Profile 객체에 적절한 Answer 객체를 넣어줘야 하겠네요

````java
public class ProfileTest {

  @Test
  public void test() {
    Profile profile = new Profile("Bull Hockey, Inc.");
    Question question = new BooleanQuestion(1, "Got bonuses?");
    Answer profileAnswer = new Answer(question,Bool.False);
    profile.add(profileAnswer);     //추가
    Criteria criteria = new Criteria();     //추가
    Answer criteriaAnswer = new Answer(question, Bool.TRUE);
    Criterion criterion = new Criterion(criteriaAnswer, Weight.MustMatch);
    criteria.add(criterion);
  }
}
````

자 이제 1. 테스트 명을 적절하게 변경하고 2. 실행과 단언 문을 넣어주겠습니다.

````java
public class ProfileTest {

  @Test
  public void matchAnswersFalseWhenMustMatchCriteriaNotMet() {
    Profile profile = new Profile("Bull Hockey, Inc.");
    Question question = new BooleanQuestion(1, "Got bonuses?");
    Answer profileAnswer = new Answer(question,Bool.False);
    profile.add(profileAnswer);     //추가
    Criteria criteria = new Criteria();     //추가
    Answer criteriaAnswer = new Answer(question, Bool.TRUE);
    Criterion criterion = new Criterion(criteriaAnswer, Weight.MustMatch);
    criteria.add(criterion);
    
    boolean matches = profile.matches(criteria);
    
    assertFalse(matches);
  }
}
````

이제는 유지 보수성도 고려해야 합니다. 현재 하나의 조건에 코드 10줄이 필요하네요. 조건이 15개로 한다면
20줄의 코드에 테스트 코드가 150줄이 필요하다? 좀 넌센스입니다.

### 4. 두 번째 테스트 만들기


