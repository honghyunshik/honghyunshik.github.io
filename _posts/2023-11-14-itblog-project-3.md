---
title: IT Blog 프로젝트 3일차 - Elastic Search
author: honghyunshik
date: 2023-11-14 14:30:00 +0800
categories: [Project, SpringBoot]
tags: [springboot, elastic search]
---

# Elastic Search

오늘은 Elastic Search의 환경 설정을 해볼 생각입니다!

Elastic Searcch는 Apache Lucene 기반의 Java 오픈소스 분산 검색 엔진이며,
대량의 데이터를 거의 실시간(NRT:Near Real Time)으로 검색 및 저장할 수 있습니다!

우선 현재 가장 최신 버전인 8.11.1 버전을 다운로드 해봤습니다

가장 먼저 할 일은 /bin 경로의 elasticsearch-reset-password를 통해 비밀번호를 reset 해주는 일입니다.

elasticsearch 서버를 다운로드 받으면 기본적으로 [elastic] user가 등록되어 있습니다. 이 유저의 비밀번호를 reset 해주도록 합니다.

우리가 필요한 정보는 총 5가지에요.

    1. 유저 이름
    2. 유저 비밀번호
    3. 인증서 경로
    4. 호스트 이름
    5. 포트 번호

이렇게 총 5가지를 application.yml 설정파일에 등록해주도록 하겠습니다.

````yml
elastic-search:
  user: elastic
  password: [your password]
  ssl: C:\elasticsearch-8.11.1/config/certs/http_ca.crt
  port: 9200
  host: localhost
````

저는 로컬디스크 C에 그대로 받았기 때문에 저런 경로에 인증서가 저장되어 있네요!! 이제 저 정보들을 통해 ElasticSearch 서버와 통신할
ElasticSearchClient 객체를 생성해주면 됩니다.

## RestClientConfig

````java
@Configuration
public class RestClientConfig{

    @Value("${elastic-search.user}")
    private String userName;

    @Value("${elastic-search.password}")
    private String userPassword;

    @Value("${elastic-search.ssl}")
    private String ssl;

    @Value("${elastic-search.host}")
    private String host;

    @Value("${elastic-search.port}")
    private int port;

    @Bean
    public ElasticsearchClient elasticsearchClient() throws IOException {

        File certFile = new File(ssl);

        SSLContext sslContext = TransportUtils
                .sslContextFromHttpCaCrt(certFile);

        BasicCredentialsProvider credsProv = new BasicCredentialsProvider();
        credsProv.setCredentials(
                AuthScope.ANY, new UsernamePasswordCredentials(userName, userPassword)
        );

        RestClient restClient = RestClient
                .builder(new HttpHost(host, port, "https"))
                .setHttpClientConfigCallback(hc -> hc
                        .setSSLContext(sslContext)
                        .setDefaultCredentialsProvider(credsProv)
                )
                .build();

        // Create the transport and the API client
        ElasticsearchTransport transport = new RestClientTransport(restClient, new JacksonJsonpMapper());
        return new ElasticsearchClient(transport);
    }
}
````

이렇게 알맞은 아이디와 비밀번호, 인증서를 집어넣을 경우 Https 인증을 마친 클라이언트가 생성됩니다!

[공식 문서](https://www.elastic.co/guide/en/elasticsearch/client/java-api-client/current/connecting.html)
의 코드를 그대로 사용한 것이므로 자세한 내용은 링크를 참조하시면 좋을 것 같아요.

다음은 간단합니다. Entity와 Repository를 생성할 것인데요, 각각 Elastic Search와 상호작용 하는 객체라는 것을 Application Context가
알아야 겠죠? @Annotaion만 추가해주면 됩니다.

## Post

````java
@Getter
@Setter
@Document(indexName = "post")
public class Post {

    @Id
    private String id;
    private String title;
    private Date regDate;
}
````

기존의 Entity 설정과 별로 다르지 않습니다. @Document 어노테이션만 추가해주면 Application Context가 이 Entity는 Elastic Search와
통신한다는 것을 알 수 있습니다.

아 그리고 추가적으로 Elastic Search는 기존 RDB의 Table을 Index라고 부릅니다.

## PostRepository

````java
public interface PostRepostiroy extends ElasticsearchRepository<Post,String> {
}
````

Repository도 마찬가지입니다.

이렇게 Elastic Search 환경설정을 마쳤구요. 내일은 크롤링을 통해 여러 블로그의 포스트들을 가져와 Elastic Search 서버에 저장하도록
하겠습니다!
