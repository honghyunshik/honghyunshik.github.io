---
title: IT Blog 프로젝트 1일차 - 회원가입
author: honghyunshik
date: 2023-11-08 14:30:00 +0800
categories: [Project, SpringBoot]
tags: [springboot, jpa, entity]
---

## 프로젝트 시작

기존에 ITBlog 프로젝트를 진행하고 있었는데, 이번에 GitHub-flow를 처음으로 공부하게 되어 복습도 할겸 GitHub-flow에 맞춰
개발을 진행하고 이를 기록으로 남기고자 합니다.

제가 구현하고자 하는 기능은

    1. SpringSecurity + JWT + Redis를 활용한 회원가입/로그인 기능
    2. OAuth2.0을 활용한 소셜로그인
    3. 블로그 크롤링 자동화
    4. Elastic Search를 활용해 응답하기

대표적으로 이렇게 4개의 초점을 맞춰보도록 하겠습니다. 그 외에 좋아요 기능, 댓글 기능 등... 차례차례 구현해보도록 하겠습니다!

아 그리고... 프로젝트란게 목표를 확실히 잡지 않으면 흐지부지 되는 경우가 많더라구요! 12.23 크리스마스 전에 
프로젝트를 마무리하고 백엔드 서버를 배포해보도록 하겠습니다!

### ERD 
우선은 이전에 짜둔 ERD를 먼저 보여드리겠습니다.

![ERD](/assets/img/2023-11-08-itblog-project-1/erd.png)

ERD Cloud를 활용해서 기본적인 ERD를 구성했습니다. 그럼 이제 프로젝트를 새로 생성해보겠습니다!

### 프로젝트 환경

- JAVA 17
- SpringBoot 3.1.1
- PostgreSQL
- Redis
- ElasticSearch 8.7.1
- Springdoc 2.1.0
- SpringSecurity
- Lombok

## 회원가입

가장 기본적인 기능인 회원가입부터 만들어 보겠습니다.  
github-flow에 따라 feature/register 브랜치에서 작업하도록 하겠습니다.

가장 첫 번째로 할 일은 이메일 유효성 검사 및 중복 체크 기능입니다!

### EmailRequestDto

````java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class EmailRequestDto {
    @NotNull(message = "이메일은 필수 입력사항입니다")
    @NotBlank
    @Pattern(regexp = "^[A-Za-z0-9+_.-]+@(.+)$")
    private String email;
}
````

이메일을 입력받는 Dto입니다. 이후 Controller에서 @Valid를 통해 유효성 검사를 진행하겠습니다.

### MemberApiController

````java
@RequestMapping(value = "/api/v1/member")
@RestController
@RequiredArgsConstructor
public class MemberApiController {

    private final MemberService memberService;

    @PostMapping("/email")
    public ResponseEntity<?> emailCheck(@Valid @RequestBody EmailRequestDto emailRequestDto, Errors errors){
  
      //이메일 유효성 검사 실패시 bad request + error 내용 반환
      if (errors.hasErrors()) {
        return ResponseEntity.badRequest().body(Collections.singletonMap("errors", errors.getAllErrors()));
      }
  
      if(memberService.emailExist(emailRequestDto)) return ResponseEntity.badRequest()
        .body(Collections.singletonMap("error","중복된 이메일이 존재합니다"));
  
      return ResponseEntity.ok().body(Collections.singletonMap("message","중복된 이메일이 없습니다"));
    }
}
````

이메일 유효성 검사는 Controller에서, 중복체크는 Service가 담당하도록 했습니다.

@Valid를 통해 Error가 존재하면 bad Request, 아니면 Service에게 위임합니다.

### MemberServiceImpl

````java
@RequiredArgsConstructor
@Service
public class MemberServiceImpl implements MemberService {

    private final MemberRepository memberRepository;

    @Override
    @Transactional
    public boolean emailExist(EmailRequestDto emailRequestDto) {
      Optional<Member> existingUser = memberRepository.findByEmail(emailRequestDto.getEmail());
  
      //이메일에 중복이 있는지 체크
      return existingUser.isPresent();
    }
}
````
MemberService를 구현한 MemberServiceImpl입니다. Controller와 Repository 사이에서
데이터를 가공하는 역할을 합니다.

이미 DB에 이메일이 존재한다면 true를, 아니라면 false를 반환합니다.

### Member

````java
@Getter
@NoArgsConstructor
@Entity
public class Member {

    @Id
    @GeneratedValue
    @Column(name = "member_id")
    private Long id;


    @Column(name = "member_name")
    private String name;

    @Column(name = "member_gender")
    private String gender;

    @Column(name = "member_birth")
    private String birth;

    @Column(name = "member_email")
    private String email;
    
    @Column(name = "member_phone_number")
    private String phoneNumber;

    @Column(name = "member_password")
    private String password;

    @Enumerated(EnumType.STRING)
    @Column(name = "member_role")
    private Role role;
}
````

Member Entity입니다. application.yml에 DB 설정을 추가할 경우 JPA가 자동으로 스키마를 정의해줍니다.

MemberRepository는 별다른 점이 없어서 기록하지 않도록 하겠습니다. JpaRepostiory를 상속받으면 됩니다!

앞으로의 로직은 다음과 같습니다.

1. 클라이언트가 이메일을 작성 후 이메일 중복검사 버튼 클릭
2. 이메일을 저장한 EmailRequestDto가 Controller에서 유효성 검사를 거침
3. 이후 Service에서 Repository를 통해 DB와 통신하여 중복된 이메일이 있는지 체크
4. 클라이언트에게 응답

자 이제 이 간단한 API를 테스트해보도록 하겠습니다!

### MemberServiceImpleTest

````java
@ExtendWith(MockitoExtension.class)
class MemberServiceImplTest {

    @InjectMocks
    private MemberServiceImpl memberService;
    @Mock
    private MemberRepository memberRepository;

    @Test
    public void 중복된_이메일이_있는경우_true_반환한다(){
        EmailRequestDto emailRequestDto = new EmailRequestDto("admin@naver.com");
        when(memberRepository.findByEmail(emailRequestDto.getEmail())).thenReturn(Optional.of(new Member()));

        assertTrue(memberService.emailExist(emailRequestDto));
    }

    @Test
    public void 중복된_이메일이_없는경우_false_반환한다(){
        EmailRequestDto emailRequestDto = new EmailRequestDto("admin@naver.com");
        when(memberRepository.findByEmail(emailRequestDto.getEmail())).thenReturn(Optional.empty());

        assertFalse(memberService.emailExist(emailRequestDto));
    }
}
````

위의 2개의 테스트 코드를 통과했습니다! 

추가로 기술할만한 내용은 처음 코드는
````java
@InjectMocks
private MemberService memberService;
````
였는데요, Interface를 선언하면 알아서 Impl 구현체를 주입해주길 바랬지만, 아쉽게도 그건 불가능했습니다.

이제 Controller에 대한 테스트를 진행해볼게요!

### MemberApiControllerTest

````java
@ExtendWith(SpringExtension.class)
@WebMvcTest(MemberApiController.class)
class MemberApiControllerTest {

    @Autowired
    private MockMvc mvc;

    @MockBean
    private MemberService memberService;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    public void 유효성_검사를_실패하면_bad_request_반환한다() throws Exception {
        EmailRequestDto emailRequestDto = new EmailRequestDto("admin@");
        this.mvc.perform(post("/api/v1/member/email")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(emailRequestDto)))
                .andExpect(status().isBadRequest());
    }

    @Test
    public void 유효성_검사를_성공하고_중복된_이메일이면_bad_request_반환한다() throws Exception{
        EmailRequestDto emailRequestDto = new EmailRequestDto("admin@naver.com");
        given(this.memberService.emailExist(emailRequestDto))
                .willReturn(true);

        this.mvc.perform(post("/api/v1/member/email")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(emailRequestDto)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.error").value("중복된 이메일이 존재합니다"));
    }

    @Test
    public void 유효성_검사를_성공하고_중복된_이메일이_아니면_200_반환한다() throws Exception{
        EmailRequestDto emailRequestDto = new EmailRequestDto("admin@naver.com");
        given(this.memberService.emailExist(emailRequestDto))
                .willReturn(false);
        this.mvc.perform(post("/api/v1/member/email")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(emailRequestDto)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.message").value("중복된 이메일이 없습니다"));
    }
}
````

근데 한가지 문제가 생겼습니다! 바로 mock 객체가 제대로 주입되지 않아 두번째 테스트도 200을 반환한다는 것인데요...

[깃허브 이슈](https://github.com/honghyunshik/itblog-with-elastic-search/issues/1)

해결한 후 위의 링크에 추가하도록 하겠습니다... Test코드에 문제가 있는 것 같아요 일단 Postman을 통해 테스트를 마쳤습니다.

우선은 오늘은 여기까지! Email Check 기능을 구현해보았습니다. 내일까지 회원가입 기능을 완성해보도록 하겠습니다.









