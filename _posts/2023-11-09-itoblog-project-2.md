---
title: IT Blog 프로젝트 2일차 - 회원가입,로그인
author: honghyunshik
date: 2023-11-09 14:30:00 +0800
categories: [Project, SpringBoot]
tags: [springboot, jpa, entity]
---

프로젝트 2일차입니다. 오늘은 회원가입을 구현해보도록 하겠습니다!

## 회원가입

### Member

````java
@Builder
    public Member(String birth, String email, String name, String password,String gender, Role role,
                  String phoneNumber){
        this.birth = birth;
        this.email = email;
        this.name = name;
        this.password = password;
        this.gender = gender;
        this.role = role;
        this.phoneNumber = phoneNumber;
    }
````

우선은 Member Entity에 Builder 패턴을 추가해주었습니다.

만약 생성자가 50개가 된다고 하면, 매개변수를 모두 기억해서 넣어주는 것은 고역이 될 것입니다. 실수가 매우 많이 발생하겠죠?
또 한가지 방법은 setter을 통해서 50개를 다 넣어주는거에요. 근데 이 방법은 메소드가 매우 많이 필요하고, 한번에 인스턴스화 되지 않기 때문에
thread-safe 하지 않다는 단점이 있어요. 따라서 Builder 패턴이 매우 효과적입니다.

### RegisterRequestDto

````java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class RegisterRequestDto {

    @NotNull(message = "생일은 필수 입력사항입니다")
    @NotBlank
    @Pattern(regexp = "(19|20)\\d{2}(0[1-9]|1[012])(0[1-9]|[12][0-9]|3[01])")
    private String birth;

    @NotNull(message = "이메일은 필수 입력사항입니다")
    @NotBlank
    @Pattern(regexp = "^[A-Za-z0-9+_.-]+@(.+)$")
    private String email;

    @NotBlank
    @NotNull(message = "이름은 필수 입력사항입니다")
    @Pattern(regexp = "^[가-힣]{2,4}$", message = "이름은 최소 두글자 이상, 최대 네글자 이하의 한글로만 가능합니다")
    private String name;

    @NotBlank
    @NotNull(message = "비밀번호는 필수 입력사항입니다")
    @Pattern(regexp = "^[a-zA-Z\\d`~!@#$%^&*()-_=+]{8,12}$")
    private String password;

    @NotNull(message = "성별은 필수 입력사항입니다")
    private String gender;

    @NotBlank
    @NotNull
    @Pattern(regexp = "^01(?:0|1|[6-9])-(?:\\d{3}|\\d{4})-\\d{4}$", message = "전화번호 양식을 확인해주세요")
    private String phoneNumber;

    public Member toEntity(){
        return Member.builder()
                .birth(this.birth)
                .email(this.email)
                .name(this.name)
                .gender(this.gender)
                .password(this.password)
                .role(Role.USER)
                .phoneNumber(this.phoneNumber)
                .build();
    }
}
````

EmailRequestDto와 마찬가지로 유효성 검사를 진행하였구요, 마지막에는 toEntity() 메소드가 있습니다.

저 메소드를 통해 Dto -> Entity로 변환 후 jpa를 통해 DB에 저장하도록 하겠습니다.

### MemberApiController

````java
@PostMapping("/register")
    public ResponseEntity<?> register(@Valid @RequestBody RegisterRequestDto registerRequestDto, Errors errors){

        //회원가입 Dto 유효성 검사 실패시 bad request + error 내용 반환
        if(errors.hasErrors()){
            return ResponseEntity.badRequest().body(Collections.singletonMap("errors", errors.getAllErrors()));
        }
        
        memberService.register(registerRequestDto);
        
        return ResponseEntity.ok().body(Collections.singletonMap("message","회원가입이 완료되었습니다"));
    }
````

유효성 검사만 통과하면 이미 이메일 중복 검사는 진행했으니까 그대로 DB에 저장하면 되겠습니다.

### WebSecurityConfig

````java
@EnableWebSecurity
@Configuration
public class WebSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception{

        http
                .httpBasic((httpBasic)->httpBasic.disable())
                .authorizeHttpRequests((authz)-> authz
                        .requestMatchers(WhiteList.WHITE_LIST).permitAll()
                        .anyRequest().authenticated()
                )
                .cors((cors) -> cors.disable())
                .csrf((csrf) -> csrf.disable())
                .sessionManagement((sessionManagement)->
                        sessionManagement.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }
    @Bean
    public BCryptPasswordEncoder bCryptPasswordEncoder(){
        return new BCryptPasswordEncoder();
    }
}
````
회원의 비밀번호를 DB에 그대로 저장하면 보안상 위험하므로 암호화를 한 후 저장하려고 합니다.

BCryptPasswordEncoder 클래스를 통해 암호화를 진행할 수 있고, spring security 라이브러리에 있으므로 spring boot starter security 의존성을 추가해주었습니다.

근데 SpringSecurity를 추가하면 갑자기 모든 요청에 401 Unauthorized를 반환할 것입니다. 인증이 완료되지 않았기 때문입니다. 인증은 로그인 페이지에서 진행하도록 하고
우선 지금은 WhiteList에 회원가입 페이지를 넣어주고 이 URL의 요청은 모두 인증을 받지 않는 것으로 진행하겠습니다

### WhiteList

````java
public class WhiteList {
    public static final String[] WHITE_LIST = {
            "/api/v1/member/**"
    };
}
````

### MemberServiceImpl

````java
@Override
    @Transactional
    public void register(RegisterRequestDto registerRequestDto) {
        registerRequestDto.setPassword(bCryptPasswordEncoder.encode(registerRequestDto.getPassword()));
        memberRepository.save(registerRequestDto.toEntity());
    }
````
Dto에서 비밀번호를 암호화 한 후 repository를 통해 DB에 저장했습니다.

자 이제 Test 코드를 작성해보도록 하겠습니다!

### MemberServiceImplTest

````java
@Test
public void BCrypt로_비밀번호를_암호화한다(){
    RegisterRequestDto registerRequestDto = new RegisterRequestDto();
    registerRequestDto.setPassword("password");
    when(bCryptPasswordEncoder.encode(registerRequestDto.getPassword())).thenReturn("encrypt");
    memberService.register(registerRequestDto);
    assertEquals(registerRequestDto.getPassword(),"encrypt");
}
````

이렇게 BCrypt로 암호화 한 후 저장하는 로직까지 테스트를 마쳤습니다. Controller의 유효성 검사는 EmailCheck 메서드에서 테스트를 마쳤기때문에 따로 진행하진 않을게요.

자 이렇게 간단히 회원가입을 마쳤구요... 이번에는 로그인을 구현해보도록 하겠습니다

## 로그인

로그인은 JWT 토큰을 통해서 인증을 하도록 하겠습니다. 최근에는 Session 인증보다 Token 인증을 더 선호하는 추세에요.

JWT 토큰의 장점은 다음과 같습니다

    1. Session 방식은 로그인 정브를 DB에 저장해야 합니다. 하지만 Jwt Token은 토큰 자체만으로 판단할 수 있기 때문에 따로 저장할 
    필요가 없어서 메모리를 절약할 수 있습니다.
    2. Session 방식은 로그인 정보를 확인하기 위해서는 DB 서버와 통신해야 합니다. 하지만 Jwt Token은 자체적으로 유효한 토큰인지 판별할
    수 있기 때문에 효율적입니다.
    3. Jwt는 서명을 통해 보안상 이점을 가집니다.

회원가입보다 난이도가 많이 올라간다는 점 유의해서 봐주세요!

### WebSecurityConfig

SpringSecurity가 인증을 하는 과정은 내부에 있는 FilterChain를 통해서 이루어집니다. FilterChain 중에는 UsernamePasswordAuthenticationFilter
가 존재하는데, 우리는 이 필터 대신에 Custom한 JWT Filter를 추가해 줄 겁니다. 

UsernamePasswordAuthenticationFilter 이전에 JWT Filter를 거쳤다면 SecurityContext에 인증된 사용자 정보가 저장될 것이고, 그렇다면 다음 필터는
통과됩니다.

````java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception{

    http
            .httpBasic((httpBasic)->httpBasic.disable())
            .authorizeHttpRequests((authz)-> authz
                    .requestMatchers(WhiteList.WHITE_LIST).permitAll()
                    .anyRequest().authenticated()
            )
            .cors((cors) -> cors.disable())
            .csrf((csrf) -> csrf.disable())
            .sessionManagement((sessionManagement)->
                    sessionManagement.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)       //추가
            .exceptionHandling(handler->handler.authenticationEntryPoint(customAuthenticationEntryPoint));;     //추가
    return http.build();
}
````

밑에 두 줄이 추가되었습니다. 

JwtAuthenticationFilter는 요청의 Header의 필드인 Authorization에 저장되어 있는 토큰이 유효한지 판별합니다.
유효할 경우 Filter를 통과시키고, 아니라면 인증 실패를 클라이언트에게 반환하겠습니다.

인증 실패를 처리하는 Handler는 CustomAuthenticationEntryPoint가 담당합니다.

### JwtAuthenticationFilter

이 필터의 목적은 Jwt Token이 유효한 지를 판별하는 것입니다. 우선, 이 클래스는 OncePerRequestFilter를 상속받습니다. 해석 그대로 요청이 올 때마다
이 필터를 동작하겠다는 얘기죠.

````java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final RedisTemplate<String,Object> redisTemplate;
    private final String[] WHITE_LIST = WhiteList.WHITE_LIST;
    @Override
    protected void doFilterInternal(@NonNull HttpServletRequest request,
                                    @NonNull HttpServletResponse response,
                                    @NonNull FilterChain filterChain) throws ServletException, IOException {

        String accessToken = resolveToken((HttpServletRequest) request);

        AntPathMatcher pathMatcher = new AntPathMatcher();
        for(String whiteList:WHITE_LIST){

            if(pathMatcher.match(whiteList,request.getRequestURI())){
                filterChain.doFilter(request,response);
                return;
            }
        }

        if(accessToken!=null) {

            try{
                jwtTokenProvider.isValidateToken(accessToken);
                //로그아웃 되어있지 않다면 filter 통과
                String logout = (String) redisTemplate.opsForValue().get(accessToken);
                if (ObjectUtils.isEmpty(logout)) {
                    Authentication authentication = jwtTokenProvider.getAuthentication(accessToken);
                    SecurityContextHolder.getContext().setAuthentication(authentication);
                }
                //로그아웃 되어있다면 filter 통과 못함 => 다시 로그인
                else {
                    unauthorized(response, "로그아웃 되었습니다");
                    return;
                }
            } catch (ExpiredJwtException e) {

                Authentication authentication = jwtTokenProvider.getAuthentication(accessToken);
                TokenInfo tokenInfo = jwtTokenProvider.refreshAccessToken(authentication);

                //토큰이 단순히 만료된거라면 새로운 access token 발급
                if (tokenInfo != null) {
                    response.addHeader(JwtConstants.HEADER, tokenInfo.getAccessToken());
                    SecurityContextHolder.getContext().setAuthentication(authentication);
                }
                //refresh token도 만료됐다면 filter 통과 못함 => 다시 로그인
                else {
                    unauthorized(response, "로그인한지 너무 오래됐습니다");
                    return;
                }
            }catch (SecurityException | MalformedJwtException e){
                unauthorized(response,"토큰이 변조되었습니다");
                return;
            }catch(UnsupportedJwtException e){
                unauthorized(response,"지원하지 않는 타입의 토큰입니다");
                return;
            }catch (IllegalArgumentException e){
                unauthorized(response,"토큰의 claim이 비었습니다");
                return;
            }catch(Exception e){
                unauthorized(response,"유효하지 않은 토큰입니다");
                return;
            }
        }else{
            unauthorized(response,"토큰이 존재하지 않습니다");
            return;
        }
        filterChain.doFilter(request,response);
    }

    private void unauthorized(HttpServletResponse httpServletResponse, String message) throws IOException {

        httpServletResponse.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        httpServletResponse.setCharacterEncoding("UTF-8");
        httpServletResponse.getWriter().write(message);
        httpServletResponse.getWriter().flush();
    }

    //Request Header 에서 Token 정보를 추출합니다
    private String resolveToken(HttpServletRequest httpServletRequest){
        String authHeader = httpServletRequest.getHeader(JwtConstants.HEADER);
        if(authHeader!=null && authHeader.startsWith(JwtConstants.TYPE)) {
            //Bearer을 제외한 Token 값을 추출합니다
            return authHeader.substring(7);
        }
        return null;
    }
}
````

매우 긴 코드입니다... 하나씩 설명해보도록 하겠습니다.

우선, 맨 밑의 resolveToken Method는 요청의 헤더 필드 중 Authorization 필드를 가공합니다.

Authorization은 Bearer + Token으로 이루어져 있기 때문에 앞의 Bearer을 제외한 Token만을 판별하겠습니다.

OncePerRequestFilter를 상속받았다면 doFilterInternal 메서드를 오버라이딩 해주어야 합니다. 이 메서드 안에 Filter의 기능을 구현해주면 됩니다.

우선, 우리가 앞에서 White List에 인증을 거치지 않았으면 하는 URL들을 명시했는데, 그렇다고 Filter Chain을 거치지 않는 것이 아닙니다.

따라서 White List에 대해서 필터를 통과시켜주는 메서드를 작성했습니다.
````java
AntPathMatcher pathMatcher = new AntPathMatcher();
for(String whiteList:WHITE_LIST){

    if(pathMatcher.match(whiteList,request.getRequestURI())){
        filterChain.doFilter(request,response);
        return;
    }
}
````

Filter는 3가지 경우가 존재합니다.

    1. 유효한 토큰으로 요청한 경우
    2. 만료된 토큰으로 요청한 경우
    3. 유효하지 않은 토큰으로 요청한 경우

#### 1. 유효한 토큰으로 요청한 경우

````java
String accessToken = resolveToken((HttpServletRequest) request);
jwtTokenProvider.isValidateToken(accessToken);
Authentication authentication = jwtTokenProvider.getAuthentication(accessToken);
SecurityContextHolder.getContext().setAuthentication(authentication);
````
우선 token을 추출하고, jwtToeknProvider에게 유효성 검사를 위임합니다. 유효하지 않다면 2,3번으로 넘어가고, 유효하다면 
로그아웃 되어 있는지 확인합니다. 로그아웃 되어 있다면 다시 로그인을 해야 하기 때문에 필터를 통과할 수 없고, 그렇지 않다면 
SecurityContext에 사용자 인증 정보를 저장합니다.

그리고 추가적으로 여기서 Redis를 활용했는데요, 그 이유는 데이터 저장 시간을 내가 정할 수 있기 때문입니다. 로그아웃된 토큰 정보를 계속해서
저장할 필요는 없겠죠? 토큰이 만료될 때까지만 가지고 있다가 알아서 삭제디면 그만입니다.

로그아웃을 했다면 해당 토큰을 토큰이 만료될 때까지 Redis에 저장할 것이므로 Redis에 해당 토큰이 존재하는지를 통해 로그아웃 여부를 확인합니다.


#### 2. 토큰이 만료된 경우

JwtTokenProvider는 토큰의 유효성 검사를 진행하고, 유효하지 않다면 예외를 Throw합니다. 가장 먼저 토큰이 만료됐는지 확인합니다.
토큰이 만료됐다면 자동으로 연장해주기 위함이에요. 

````java
Authentication authentication = jwtTokenProvider.getAuthentication(accessToken);
TokenInfo tokenInfo = jwtTokenProvider.refreshAccessToken(authentication);

//토큰이 단순히 만료된거라면 새로운 access token 발급
if (tokenInfo != null) {
    response.addHeader(JwtConstants.HEADER, tokenInfo.getAccessToken());
    SecurityContextHolder.getContext().setAuthentication(authentication);
}
//refresh token도 만료됐다면 filter 통과 못함 => 다시 로그인
else {
    unauthorized(response, "로그인한지 너무 오래됐습니다");
    return;
}
````

Jwt Token은 Access Token과 Refresh Token으로 구분됩니다. 왜 이렇게 토큰을 구분해 놓았느냐, 바로 보안 때문입니다.

토큰이 탈취당하더라도, 만료 기한이 짧다면 페이지를 최대한 보호할 수 있기 때문에 Access Token은 짧은 유효 기간을 가집니다.

하지만, 단순히 유효기간만 짧다면 로그인을 계속 해줘야 하기 때문에 번거롭겠죠? 따라서 Refresh Token을 통해 Access Token을 계속 연장해주는 방법을
택하는 겁니다.

따라서, Access Token이 만료됐더라도 Refresh Token이 만료되지 않았다면 새로운 Access Token을 발행해 주도록 하겠습니다. 이 Refresh Token 또한
Redis에 저장하도록 하겠습니다. 만약 Redis에 없다면? 너무 오래전에 로그인 했기 때문에 다시 로그인 해주어야 합니다.

#### 3. 토큰이 유효하지 않은 경우

필터를 통과하면 안됩니다. 하나 고민한 것은 토큰이 왜 이상한지 굳이 같이 반환할 필요가 있나 싶긴 했어요. 어차피 어떤 이유에서건 토큰이 이상하면 인증은
안되는거니까. 그래도 일단은 경우를 나눠서 반환해주었습니다.

자 이렇게 Filter 구현을 마쳤고, 이제 토큰의 유효성을 판별해주고, 새로운 토큰을 발급해주는 JwtTokenProvider를 구현해보겠습니다.

### JwtTokenProvider

````java
@Component
public class JwtTokenProvider {

    private RedisTemplate<String,Object> redisTemplate;
    private Key key;

    //기간 30분
    private static int ACCESS_EXPIRE_TIME = 30*60*1000;
    //기간 1주일
    private static int REFRESH_EXPIRE_TIME = 7*24*60*60*1000;

    public JwtTokenProvider(@Value("${jwt.secret}") String secretKey, RedisTemplate<String,Object> redisTemplate) {

        this.redisTemplate = redisTemplate;
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        this.key = Keys.hmacShaKeyFor(keyBytes);
    }

    //Access Token이 만료됐지만 refresh Token이 Redis에 존재할 경우 새로운 access Token 발급
    //발급하는 기간은 30분과 refresh Token의 min값
    public TokenInfo refreshAccessToken(Authentication authentication){

        String refreshToken = (String) redisTemplate.opsForValue().get(authentication.getName());
        if(ObjectUtils.isEmpty(refreshToken)) return null;
        int refreshTTL = Math.toIntExact(redisTemplate.getExpire(authentication.getName(), TimeUnit.MILLISECONDS));
        String accessToken = generateAccessToken(authentication,generateAuthorities(authentication),
                Math.min(ACCESS_EXPIRE_TIME,getExpirationTime(refreshToken)));

        return TokenInfo.builder()
                .grantType(JwtConstants.TYPE)
                .accessToken(accessToken)
                .refreshToken(refreshToken)
                .build();
    }

    //토큰의 만료기간 얻기
    public int getExpirationTime(String token){
        Date expiration = Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token).getBody().getExpiration();
        long now = new Date().getTime();
        return (int) (expiration.getTime()-now);
    }

    public String generateAuthorities(Authentication authentication){

        return authentication.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.joining(","));
    }

    //accessToken의 유지기간은 30분
    //refreshToken이 30분 이하로 남았을 경우 refresh Token이 만료된 이후 로그인 필요
    public String generateAccessToken(Authentication authentication, String authorities, int time){

        long now = new Date().getTime();

        return Jwts.builder()
                .setSubject(authentication.getName())
                .claim("auth",authorities)
                .setExpiration(new Date(now + time))
                .signWith(key, SignatureAlgorithm.HS256)
                .compact();
    }

    //refreshToken의 유지기간은 2주
    public String generateRefreshToken(Authentication authentication, String authorities){

        long now = new Date().getTime();

        return Jwts.builder()
                .setSubject(authentication.getName())
                .claim("auth",authorities)
                .setExpiration(new Date(now + REFRESH_EXPIRE_TIME))
                .signWith(key,SignatureAlgorithm.HS256)
                .compact();
    }

    public TokenInfo generateToken(Authentication authentication){

        String authorities = generateAuthorities(authentication);

        //accessToken 유지기간은 30분
        String accessToken = generateAccessToken(authentication,authorities,ACCESS_EXPIRE_TIME);

        //refreshToken 유지기간은 2주
        String refreshToken = generateRefreshToken(authentication,authorities);

        redisTemplate.opsForValue().set(authentication.getName(),refreshToken,REFRESH_EXPIRE_TIME,TimeUnit.MILLISECONDS);

        return TokenInfo.builder()
                .grantType(JwtConstants.TYPE)
                .accessToken(accessToken)
                .refreshToken(refreshToken)
                .build();
    }

    private Claims parseClaims(String accessToken){
        try{
            return Jwts.parserBuilder().setSigningKey(key).build().
                    parseClaimsJws(accessToken).getBody();
        }catch(ExpiredJwtException e){
            return e.getClaims();
        }
    }

    //예외를 상위 메서드로 Throw
    public void isValidateToken(String token) throws Exception{
        Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token);
    }

    public Authentication getAuthentication(String accessToken){

        Claims claims = parseClaims(accessToken);

        if(claims.get("auth")==null) throw new RuntimeException("권한 정보가 없는 토큰입니다");

        Collection<? extends GrantedAuthority> authorities = Arrays.stream(claims.get("auth").toString().split(","))
                .map(SimpleGrantedAuthority::new).collect(Collectors.toList());

        UserDetails principal = new User(claims.getSubject(),"",authorities);

        return new UsernamePasswordAuthenticationToken(principal,"",authorities);
    }
}
````
Jwt Token은 간단한 사용자 정보와 유효기간, 서명 등을 포함합니다. Access token의 유효기간은 30분, Refresh Token의 유효기간은 1주일로 잡았습니다.

### MemberApiController

````java
@PostMapping("/login")
public ResponseEntity<?> login(@Valid @RequestBody LoginRequestDto loginRequestDto,Errors errors){
    if(errors.hasErrors()){
        return ResponseEntity.badRequest().body(Collections.singletonMap("errors", errors.getAllErrors()));
    }
    TokenInfo tokenInfo = memberService.login(loginRequestDto);
    return ResponseEntity.ok().header(JwtConstants.TYPE,tokenInfo.getAccessToken()).build();
}
````
login 실패 시 bad Request를, 성공 시 200과 응답 header로 Access Token을 반환합니다.
프론트에서는 이 Access Token을 Http Only Cookie로 관리할 계획입니다.

### MemberServiceImpl

````java
@Override
public TokenInfo login(LoginRequestDto loginRequestDto) throws PasswordNotMatchingException {

    UsernamePasswordAuthenticationToken usernamePasswordAuthenticationToken =
            new UsernamePasswordAuthenticationToken(loginRequestDto.getEmail(),loginRequestDto.getPassword());
    Authentication authentication;
    try{
        authentication = authenticationManagerBuilder.getObject().authenticate(usernamePasswordAuthenticationToken);
    }catch(Exception e){
        throw new PasswordNotMatchingException("비밀번호가 틀렸습니다");
    }
    //login 시 토큰 발급 -> 로그인 기준으로 access Token : 30분   refresh Token : 1주
    return jwtTokenProvider.generateToken(authentication);
}

@Override
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

    return memberRepository.findByEmail(username)
            .map(member->{
                return User.builder()
                        .username(member.getEmail())
                        .password(member.getPassword())
                        .roles(String.valueOf(member.getRole()))
                        .build();
            }).orElseThrow(()->new UsernameNotFoundException("해당 이메일의 유저를 찾을 수 없습니다"));
}
````

이 클래스는 UserDetailsService를 상속받습니다. 그리고 loadUserByUsername 메서드를 오버라이딩 해주어야 하는데, 이 메서드는 따로 호출해줄 필요 없이
Spring Security가 자동으로 호출하며 DB와 통신해 사용자 정보를 가져와 UserDetails 객체로 변환해주는 역할을 합니다.

이렇게 변환된 UserDetails 객체와 사용자가 입력한 비밀번호를 매칭해줍니다. 따로 암호화를 할 필요 없이 알아서 비교해줍니다.
````java
authentication = authenticationManagerBuilder.getObject().authenticate(usernamePasswordAuthenticationToken);
````

자 이렇게 구현을 마쳤구요! 이번에는 테스트 코드를 작성해보도록 하겠습니다.

## 로그인 테스트

테스트 케이스는 다음과 같습니다.

    1. 비밀번호가 틀렸다   
    2. 이메일이 틀렸다   
    3. 유효한 토큰이지만 로그아웃 된 기록이 있다
    4. 유효한 토큰이고 로그아웃 된 기록이 없다
    5. Access Token과 Refresh Token이 모두 만료된 토큰으로 요청한다
    6. Access Token은 만료됐지만 Refresh Token이 만료되지 않은 토큰으로 요청한다
    7. 변조된 토큰으로 요청한다
    8. WHITE LIST가 아닌 URL로 토큰 없이 요청한다
    9. WHITE LIST인 URL로 토큰 없이 요청한다

Spring Security 테스트 코드는 @SpringBootTest 테스트로 진행했습니다. 테스트 코드가 매우 무거워지지만 Security 기능을 모두 체크하기 위해선
어쩔 수 없는 것 같습니다.

### 1. 비밀번호가 틀렸다 테스트 코드

````java
@Test
public void 로그인시_비밀번호를_틀리면_401_반환한다() throws Exception{
    LoginRequestDto wrongPasswordLoginRequestDto = new LoginRequestDto("wrongEmail@naver.com","password");
    postLoginWithUnAuthorized(wrongPasswordLoginRequestDto);
}
private void postLoginWithUnAuthorized(LoginRequestDto loginRequestDto) throws Exception {
    mockMvc.perform((post("/api/v1/member/login")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(loginRequestDto))))
            .andExpect(status().isUnauthorized());
}

@BeforeAll
public void init() throws Exception {

  when(memberRepository.findByEmail("rightEmail@naver.com")).thenReturn(Optional.ofNullable(Member.builder()
  .email("rightEmail@naver.com")
  .password("rightPassword")
  .role(Role.USER)
  .build()));

  when(memberRepository.findByEmail("wrongEmail@naver.com")).thenReturn(Optional.empty());
}
````

비밀번호가 틀릴 경우 401을 반환합니다. 이때 Respository 모킹을 통해 맞는 아이디와 비밀번호를 넣으면 Member 객체를 반환하고, 틀렸다면 Null을 반환하도록 합니다.

### 2. 이메일이 틀렸다 테스트 코드

````java
@Test
public void 로그인시_이메일을_틀리면_401_반환한다() throws Exception {
    LoginRequestDto wrongEmailLoginRequestDto = new LoginRequestDto("wrongEmail@naver.com","password");
    postLoginWithUnAuthorized(wrongEmailLoginRequestDto);
}   
````
1번 테스트와 같습니다. 다만 Dto의 아이디와 비밀번호를 바꿔주면 됩니다.

### 3. 유효한 토큰이지만 로그아웃 된 기록이 있다 테스트 코드

````java
@Test
public void 유효한_토큰이_로그아웃한_기록이_있으면_401_반환한다() throws Exception{
    doNothing().when(jwtTokenProvider).isValidateToken("token");

    RedisTemplate<String,Object> redisTemplate = mock(RedisTemplate.class);
    ValueOperations<String,Object> valueOperations = mock(ValueOperations.class);
    when(redisTemplate.opsForValue()).thenReturn(valueOperations);
    when(valueOperations.get("token")).thenReturn("logout");

    mockMvc.perform(get("/index")
                    .header("Authorization","Bearer token"))
            .andExpect(status().isUnauthorized());
}
````

redisTemplate을 모킹해서 [token]의 로그아웃 기록이 있는 것처럼 작성했습니다.
근데 또 다시 Mock 객체가 제대로 주입되지 않는 문제가 발생했습니다! 제가 같은 부분을 놓치고 있는 것 같아요.

### 4. 유효한 토큰이고 로그아웃 된 기록이 없다 테스트 코드

````java
@Test
public void 유효한_토큰이_로그아웃_기록이_없으면_200_반환한다() throws Exception {
    doNothing().when(jwtTokenProvider).isValidateToken("token");

    mockMvc.perform(get("/index")
            .header("Authorization","Bearer token"))
            .andExpect(status().isOk());
}
````

### 5. Access Token과 Refresh Token이 모두 만료된 토큰으로 요청한다 테스트 코드

````java
@Test
public void 만료된_토큰인뎨_refresh_token_이_있다() throws Exception{
    doThrow(ExpiredJwtException.class).when(jwtTokenProvider).isValidateToken("token");

    User principal = new User("username", "password", Collections.singletonList(new SimpleGrantedAuthority("USER")));
    Authentication authentication = new UsernamePasswordAuthenticationToken(principal, null, principal.getAuthorities());

    when(jwtTokenProvider.getAuthentication("token")).thenReturn(authentication);
    TokenInfo newToken = new TokenInfo();
    newToken.setAccessToken("newToken");
    when(jwtTokenProvider.refreshAccessToken(authentication)).thenReturn(newToken);

    mockMvc.perform(get("/index")
                    .header("Authorization","Bearer token"))
            .andExpect(status().isOk())
            .andExpect(header().string("Authorization","newToken"));
}
````

### 6. Access Token은 만료됐지만 Refresh Token이 만료되지 않은 토큰으로 요청한다 테스트 코드

````java
@Test
public void 만료된_토큰인데_refresh_token_이_없다() throws Exception{
    doThrow(ExpiredJwtException.class).when(jwtTokenProvider).isValidateToken("token");

    User principal = new User("username", "password", Collections.singletonList(new SimpleGrantedAuthority("USER")));
    Authentication authentication = new UsernamePasswordAuthenticationToken(principal, null, principal.getAuthorities());

    when(jwtTokenProvider.getAuthentication("token")).thenReturn(authentication);
    when(jwtTokenProvider.refreshAccessToken(authentication)).thenReturn(null);

    mockMvc.perform(get("/index")
                    .header("Authorization","Bearer token"))
            .andExpect(status().isUnauthorized())
            .andExpect(content().string("로그인한지 너무 오래됐습니다"));
}
````

### 7. 변조된 토큰으로 요청한다 테스트 코드

````java
@Test
public void 변조된_토큰으로_요청하면_401_반환한다() throws Exception{
    doThrow(MalformedJwtException.class).when(jwtTokenProvider).isValidateToken("token");

    mockMvc.perform(get("/index")
                    .header("Authorization","Bearer token"))
            .andExpect(status().isUnauthorized())
            .andExpect(content().string("토큰이 변조되었습니다"));
}
````

### 8. WHITE LIST가 아닌 URL로 토큰 없이 요청한다 테스트 코드

````java
@Test
public void WHITE_LIST_아닌_URL_로_토큰_없이_요청한다() throws Exception{
    mockMvc.perform(get("/index"))
            .andExpect(status().isUnauthorized())
            .andExpect(content().string("토큰이 존재하지 않습니다"));
}
````

### 9. WHITE LIST인 URL로 토큰 없이 요청한다 테스트 코드

````java
@Test
public void WHITE_LIST_인_URL_로_토큰_없이_요청한다() throws Exception{
    String white_list_url = WhiteList.WHITE_LIST[0];
    mockMvc.perform(get(white_list_url))
            .andExpect(status().isNotFound());
}
````

3번을 제외한 나머지 테스트 코드는 성공했습니다! 이렇게 로그인 기능 구현을 마칩니다. 내일부터는 Elastic Search를 사용해보도록 할게요!
