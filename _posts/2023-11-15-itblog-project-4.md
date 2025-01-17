---
title: IT Blog 프로젝트 4일차 - 크롤링
author: honghyunshik
date: 2023-11-14 14:30:00 +0800
categories: [Project, SpringBoot]
tags: [springboot, selenium]
---

# 개요

오늘은 기존의 블로그 포스트들을 크롤링해서 제 Elastic Search 서버에 저장하도록 하겠습니다.

Elastic Search의 성능을 느껴보고 싶어서 이 프로젝트를 진행하게 되었기 때문에 최대한 많은 데이터를
저장하는 것이 좋을 것 같아요. 

    1. Tistory
    2. velog
    3. Github Pages

검색해보니 대표적으로 저렇게 3개의 기술 블로그가 존재하는 것 같아요.

제가 블로그 플랫폼들을 보면서 느낀건 velog는 좋아요로 순위도 매기면서 UI가 굉장히
잘 되어 있지만, Tistory와 Github Pages는 독자적인 개별 블로그 성향이 강합니다.

검색엔진에 의존해서 블로그가 노출되다 보니 접근성이 매우 떨어진다고 판단했습니다. 좋은 글을
작성해서 많은 좋아요 수를 받아도 크게 메리트가 없는 것 같습니다. 

이 점을 개선하기 위해서 접근성이 떨어지는 Tistory와 GitHub Pages의 포스트들을 
통합하여 velog처럼 트렌딩, 순위, 카테고리 등의 UI를 구축하려고 합니다.

## GitHub Pages 크롤링

제가 현재 사용하고 있는 플랫폼입니다. 저는 마크다운 랭기지, GitHub flow 등 학습을 위해
이 플랫폼을 선택했지만, 검색하기 매우 어렵습니다... 

포스트 점유율 자체가 Tistory와 velog가 압도적이다 보니 키워드로 검색을 해도
GitHub Page 플랫폼으로 작성된 포스트들은 노출되지 않습니다. 키워드를 통해 포스트를
검색하는 것은 포기해야 합니다.

다른 방법은 깃허브 검색창에 Advanced search를 통해 한국의 GitHub 블로그를 찾을 수 있긴 한데...

대부분의 GitHub Page는 정보성의 IT 관련 포스트를 올리는 것이 아닌, 자신을 PR하기 위한
포트폴리오 형식이 강했습니다. 

그리고 가장 중요한 것은 테마가 모두 달라 크롤링이 불가능합니다... 어쩔 수 없이 GitHub Pages의
포스트들은 포기하고 Tistory 스토리들로만 작업을 진행해야 할 것 같습니다.

## Tistory

결국 Tistory 만으로 DB를 구축하기로 했습니다. Tistory는 검색창에 검색을 하면 Daum
검색 엔진으로 이동 후 검색 결과가 반환되는 구조입니다. 결국 Daum 에다가 제가 직접
키워드를 입력하고, 직접 사이트를 방문하여 데이터를 뽑아내야 합니다.

뽑아낼 데이터는  작성자, 작성일, 제목, HTML 정도가 되겠습니다.

### 키워드

결국 다음에 검색 후 티스토리 블로그들을 크롤링 하는 것은 구현할 수 있지만, 어떤 것을 검색하느냐는
결국 제가 수작업으로 작성할 수 밖에 없습니다.

백엔드 관련 키워드를 챗 GPT에게 물어봤습니다. 200개를 해달라고 하니 진짜 200 개를 해주네요...
키워드 200개에 대해서 각각 10개의 검색결과 총 2000개를 DB에 저장하려고 합니다. 이 중에서 URL이 같은
포스트들도 분명 존재할 것이기 떄문에 최소 1500 개 이상의 데이터를 뽑아내도록 하겠습니다.

키워드가 궁금하신 분들은 아래의 더보기를 클릭해주세요!
<details>
<summary>키워드 더보기</summary>

String[] javaSpringBootKeywords = {
"스프링 프레임워크",
"스프링 부트",
"자바",
"메이븐/그레이들",
"RESTful API",
"마이크로서비스",
"JPA (자바 영속성 API)",
"하이버네이트",
"JDBC (자바 데이터베이스 연결)",
"SQL",
"NoSQL",
"몽고DB",
"MySQL/PostgreSQL",
"JUnit",
"Mockito",
"TDD (테스트 주도 개발)",
"깃",
"GitHub/GitLab",
"도커",
"쿠버네티스",
"CI/CD (지속적 통합/지속적 배포)",
"젠킨스",
"스프링 MVC",
"스프링 시큐리티",
"OAuth 2.0",
"JWT (JSON 웹 토큰)",
"Swagger/OpenAPI",
"타임리프",
"HTML/CSS",
"자바스크립트",
"JSON",
"XML",
"SOAP",
"웹 서비스",
"REST",
"톰캣",
"제티",
"아파치 카프카",
"래빗MQ",
"JMS (자바 메시지 서비스)",
"웹소켓",
"반응형 프로그래밍",
"스프링 WebFlux",
"클라우드 서비스 (AWS, Azure, GCP)",
"스프링 클라우드",
"API 게이트웨이",
"부하 분산",
"유레카",
"히스트릭스",
"캐싱",
"레디스",
"EhCache",
"로깅 (SLF4J, Logback)",
"AOP (관점 지향 프로그래밍)",
"의존성 주입",
"IOC (제어의 역전)",
"디자인 패턴 (싱글톤, 팩토리, 전략 등)",
"SOLID 원칙",
"ORM (객체 관계 매핑)",
"ACID 속성",
"트랜잭션 관리",
"스프링 배치",
"스프링 데이터",
"엘라스틱서치",
"단위 테스트",
"통합 테스트",
"모킹 프레임워크",
"데브옵스",
"애자일 방법론",
"스크럼/칸반",
"멀티쓰레딩",
"동시성",
"자바 메모리 관리",
"가비지 컬렉션",
"JVM (자바 가상 머신)",
"JRE (자바 런타임 환경)",
"JDK (자바 개발 키트)",
"자바 8 특징 (람다, 스트림)",
"자바 제네릭",
"예외 처리",
"REST 템플릿",
"Feign 클라이언트",
"웹소켓",
"서버-전송 이벤트 (SSE)",
"메시지 브로커",
"API 디자인",
"자료 구조",
"알고리즘",
"리눅스/유닉스 명령어",
"셸 스크립팅",
"빌드 도구 (앤트, 메이븐)",
"메이븐 저장소",
"버전 관리 시스템",
"스프링 부트 액추에이터",
"모니터링 도구 (프로메테우스, 그라파나)",
"로깅과 추적 (Zipkin)",
"보안 관행",
"OWASP Top 10",
"깔끔한 코드",
"소프트웨어 아키텍처 패턴 (모놀리스, 마이크로서비스, 서버리스)",
"운영 체제",
"네트워크 프로토콜",
"TCP/IP",
"HTTP/HTTPS",
"SSL/TLS",
"DNS",
"DHCP",
"IP 라우팅",
"서브넷팅",
"로드 밸런서",
"방화벽",
"VPN",
"SSH",
"FTP",
"SFTP",
"웹 서버",
"프록시 서버",
"CORS",
"OAuth",
"가상 머신",
"컨테이너",
"시스템 디자인",
"고가용성",
"장애 내성",
"확장성",
"성능 튜닝",
"데이터 복제",
"데이터 분할",
"속도 제한",
"API 버전 관리",
"API 보안",
"큐잉 시스템",
"분산 시스템",
"합의 알고리즘",
"CAP 이론",
"이벤트 소싱",
"CQRS",
"서버리스 아키텍처",
"블록체인 기초",
"머신 러닝 기초",
"인공 지능 기초",
"IoT 기초",
"클라우드 컴퓨팅 개념",
"엣지 컴퓨팅",
"빅데이터 기술",
"하둡",
"스파크",
"그래프 데이터베이스",
"시계열 데이터베이스",
"전문 검색 엔진",
"자바에서의 머신 러닝",
"자연어 처리",
"컴퓨터 비전",
"데이터 마이닝",
"그래프 알고리즘",
"동적 프로그래밍",
"탐욕 알고리즘",
"분할 정복",
"백트래킹",
"정렬 알고리즘",
"검색 알고리즘",
"트리 순회",
"그래프 순회",
"해싱",
"스택과 큐",
"연결 리스트",
"배열",
"행렬",
"문자열 알고리즘",
"비트 조작",
"재귀",
"복잡도 분석",
"Big O 표기법",
"공간-시간 트레이드오프",
"메모이제이션",
"동적 데이터 구조",
"그래프 이론",
"조합론",
"확률 이론",
"프로그래밍에서의 통계",
"윤리 해킹 기초",
"사이버보안 기본",
"침투 테스트 기초",
"암호학 기초"
};
</details>

### 크롤링

크롤링은 다음과 같은 순서로 진행될 거에요

    1. Chrome Driver 환경설정
    2. 키워드 검색 후 결과 URL Set으로 저장
    3. 해당 포스트 URL에서 작성일, 작성자, 제목, html 추출 후 Elastic Search Server에 저장

#### Chrome Driver 환경설정

Chrome Driver 환경설정은 간단해요. 자신의 Chrome Version과 맞는 Chrome Driver를 로컬에 저장하면 됩니다.

제 버전은 **119.0.6045.160**입니다. 이 버전에 맞는 ChromeDriver를 다운로드 하면 됩니다!

그리고 이 경로를 설정파일에 저장해 줍니다.

````yaml
selenium:
  web-driver-id: webdriver.chrome.driver
  web-driver-path: C:\javas\chromedriver.exe
````

이 경로를 통해 환경설정을 마무리 해주면 됩니다.

````java
@Configuration
public class ChromeDriverConfig {

    @Value("${selenium.web-driver-id}")
    private String web_driver_id;

    @Value("${selenium.web-driver-path}")
    private String web_driver_path;

    @Bean
    public ChromeDriver chromeDriver(){
        System.setProperty(web_driver_id,web_driver_path);
        ChromeOptions chromeOptions = new ChromeOptions();
        //chromeOptions.addArguments("headless");
        chromeOptions.addArguments("--start-maximized");
        chromeOptions.setPageLoadStrategy(PageLoadStrategy.NORMAL);
        return new ChromeDriver(chromeOptions);
    }
}
````

chromeOption은 취향껏 선택하면 됩니다. 제가 주석처리 해놓은 headless 옵션은 크롤링 시 웹 브라우저를
켜놓을 지 선택하는 옵션이에요. 저는 개발 당시 제가 작성한 대로 버튼이 클릭되는지 확인하기 위해 주석처리 한 상태로 개발을 진행했습니다.

#### 키워드 검색 후 결과 URL Set으로 저장

````java
@Service
@RequiredArgsConstructor
public class CrawlingServiceImpl implements CrawlingService{

    private final ChromeDriver chromeDriver;
    private final String DAUM_URL = "https://www.daum.net/";
    private Set<String> tistoryURLList = new HashSet<>();

    private void urlListInit() {
        final String[] keywords = SearchList.javaSpringBootKeywords;
        for(String keyword:keywords){
            searchKeyWord(keyword);
            clickTotalWebButton();
            setUrlList();
            break;
        }
    }

    private void searchKeyWord(String keyword){
        chromeDriver.get(DAUM_URL);
        WebElement searchBar = chromeDriver.findElement(By.className("tf_keyword"));
        searchBar.sendKeys(keyword);
        WebElement searchButton = chromeDriver.findElement(By.className("btn_search"));
        searchButton.click();
    }

    private void clickTotalWebButton(){

        List<WebElement> tabList = chromeDriver.findElements(By.className("txt_tab"));
        for(WebElement webElement:tabList){
            if(webElement.getText().equals("통합웹")) {
                webElement.click();
                return;
            }
        }
    }

    private void setUrlList(){
        List<WebElement> list = chromeDriver.findElements(By.className("item-title"));
        for(WebElement webElement:list){
            String url = getURL(webElement);
            if(isTistory(url)) tistoryURLList.add(url);
        }
    }

    private String getURL(WebElement webElement){
        return webElement.findElement(By.cssSelector("c-title")).getAttribute("data-href");
    }

    private boolean isTistory(String url){
        return url.contains("tistory.com");
    }

    @Override
    public void crawling() {
        urlListInit();
    }
}

````

로직은 다음과 같습니다.

    1. 검색창에 keyword 검색 후 검색 버튼 누르기
    2. 통합웹으로 이동
    3. tistory에서 작성된 URL 긁어오기
    4. set에 저장

#### 해당 포스트 URL에서 작성일, 작성자, 제목, html 추출 후 Elastic Search Server에 저장

