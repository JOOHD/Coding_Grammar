## Form 로그인 & OAuth2 로그인 동시 사용 시 로그인 흐름과 구현

1. Form 로그인:
직접 만든 로그인 폼(/login)에서 아이디/비밀번호 입력 후,
UsernamePasswordAuthenticationFilter나 커스텀 LoginFilter가 인증 시도 → 성공 시 세션 생성(또는 JWT 발행) → 인가 처리

2. OAuth2 로그인:
구글, 카카오 등 외부 제공자의 로그인 연동,
로그인 버튼 클릭 시 외부 인증서버로 리다이렉트 → 인증 완료 후 OAuth2LoginAuthenticationFilter가 결과 받아 처리 → 사용자 정보 가져옴 → 토큰 생성 등 후속 처리

3. 두 방식을 같이 쓰는 이유:
- 사용자 편의성 (내부 계정, 외부 계정 둘 다 지원)
- 기존 회원은 form 로그인, 신규 회원은 SNS 로그인 등 다양성 제공

### 로그인 흐름 (filterChain, loginFilter 등) — Form 로그인과 OAuth2 로그인 동시 사용

- SecurityFilterChain 내부 흐름

-[ 클라이언트 요청 ]
        ↓
1. JWTFilterV3 (JWT 검증용 필터) - JWT 토큰 유효성 검사 → 인증 설정
        ↓
2. LoginFilter (커스텀 JSON 로그인 필터) - POST /api/login 요청에만 동작, ID/PW 인증 시도
        ↓
3. OAuth2LoginAuthenticationFilter - OAuth2 로그인 요청 처리 (예: /oauth2/authorization/**)
        ↓
4. UsernamePasswordAuthenticationFilter (기본 폼 로그인 필터) - 기존 폼 로그인 처리 (지금은 커스텀 필터로 대체됨)
        ↓
[ 인증 완료 → SecurityContextHolder에 인증 정보 저장 ]
        ↓
[ 인가 (Authorization) ]
        ↓
[ Controller or Resource 접근 ]

- Form 로그인 (LoginFilter)

1. 클라이언트 /api/login 경로로 JSON 형태로 이메일/비밀번호 전송
2. LoginFilter 가 요청 가로채 인증 시도
3. AuthenticationManager 에 위임해 유저 검증
4. 성공 시, JWT 토큰 발급 후 응답 헤더에 담거나, 세션 생성
5. 실패 시, 에러 처리 (커스텀 실패 핸들러 호출)

- OAuth2 로그인 흐름

1. 클라이언트가 /login 페이지에서 외부 로그인 버튼 클릭 -> /oauth2/authorization/{provider} 요청 발생
2. OAuth2AuthorizationRequestRedirectFilter 가 외부 인증서버로 리다이렉트
3. 인증 완료 후, OAuth2LoginAuthenticationFilter 가 콜백 응답 처리
4. CustomOAuth2UserService 가 사용자 정보 가져와 사용자 DB 처리 (회원가입 또는 로그인)
5. 성공/실패 헨들러가 후속 처리

###  두 로그인 방식 사용자 동시 이용 시, 흐름

| 상황                    | 처리 경로/필터                              | 흐름 설명                               |
| --------------------- | ------------------------------------- | ----------------------------------- |
| 1. 사용자 A - Form 로그인   | `/api/login` → `LoginFilter` 작동       | JSON 이메일/비밀번호 인증 → JWT 발급 → 접근 허용   |
| 2. 사용자 B - OAuth2 로그인 | `/login` → `/oauth2/authorization/**` | OAuth2 인증서버로 리다이렉트 → 콜백 처리 후 로그인 성공 |

### 실제 filterChain() 설정

http
    // CORS, CSRF 설정
    .cors().configurationSource(...)
    .and()
    .csrf(csrf -> csrf.ignoringRequestMatchers("/api/**")) // REST API는 CSRF 비활성화

    // 인증 요청 경로 설정
    .authorizeHttpRequests(...)
    .and()

    // Form 로그인 설정
    .formLogin(form -> form
        .loginPage("/login")
        .defaultSuccessUrl("/")
        .successHandler(loginSuccessHandler())
        .failureHandler(loginFailureHandler())
        .permitAll())

    // OAuth2 로그인 설정
    .oauth2Login(oauth2 -> oauth2
        .loginPage("/login")
        .authorizationEndpoint(auth -> auth
            .authorizationRequestResolver(new CustomAuthorizationRequestResolver(...)))
        .userInfoEndpoint(userInfo -> userInfo.userService(customOAuth2UserService))
        .successHandler(loginSuccessHandler())
        .failureHandler(loginFailureHandler()))

    // 세션 설정
    .sessionManagement(session -> session
        .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

    // 로그아웃 비활성화
    .logout(logout -> logout.disable())

    // 커스텀 필터 등록
    .addFilterBefore(new JWTFilterV3(...), UsernamePasswordAuthenticationFilter.class)
    .addFilterBefore(loginFilter(), JWTFilterV3.class);

✅ Form 로그인

- 목적: 브라우저에서 전통적인 로그인 폼 기반 인증 처리.

- 핵심 설정:
.loginPage("/login"): 커스텀 로그인 페이지
.successHandler(...): 로그인 성공 시 JWT 발급 등 후처리

- 활용 사례: OAuth2 로그인 불가능한 내부 사용자, 관리자 UI 등.

✅ OAuth2 로그인

- 목적: Google, Kakao 등 외부 소셜 로그인 지원.

- 핵심 설정:
.loginPage("/login"): 동일한 로그인 페이지 공유
.userService(...): 사용자 정보 매핑
.authorizationRequestResolver(...): redirect URI 등 커스터마이징

- 활용 사례: 사용자 경험 향상, 소셜 로그인 통합 처리.

✅ 세션 설정

.sessionCreationPolicy(SessionCreationPolicy.STATELESS)

- 의미: 서버에서 인증 상태(Session)를 저장하지 않음.
→ 클라이언트는 매 요청마다 JWT를 전송해야 함.

- 주의: formLogin(), oauth2Login()을 사용해도 세션 저장 안 됨.
→ 로그인 성공 시 서버가 JWT를 직접 발급하는 방식으로 연동 필요

✅ 로그아웃 설정 (CustomLogoutFilter)

http.logout(logout -> logout.disable())

- 목적: 
  - Spring Security 기본 로그아웃 기능을 끄고, 커스텀 로그아웃 로직 사용.
  - 클라이언트 쿠키/토큰 무효화
  - Refresh Token 제거 (Redis or DB 삭제)
  - Access Token 블랙리스트 등록

- 코드
public class CustomLogoutFilter extends OncePerReqeustFilter {
    private final JWTUtil jwtUtil;
    private final RedisTemplate<String, String>redisTemplate;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException {

        String path = request.getRequestURI();
        if (!path.equals("/logout")) {
            filterChain.doFilter(request, response);
            return;
        }

        String accessToken = jwtUtil.resolveToken(request);
        if (accessToken != null && jwtUtil.validateToken(accessToken)) {
            String userId = jwtUtil.getId(accessToken);
            redisTemplate.delete("RefreshToken:" + userId);

            long expiration = jwtUtil.getExpiration(accessToken);
            redisTemplate.opsForValue().set("Blacklist:" + accessToken, "logout", expiration, TimeUnit.MILLISECONDS);
        }

        response.setStatus(HttpServletResponse.SC_OK);
        response.getWriter().write("Logout success");
    }
}
- 핵심 포인트
  - "/logout" 요청만 가로채서 처리.
  - Redis에 저장된 Refresh Token 삭제
  - AccessToken 은 블랙리스트 처리 -> 만료 전까지는 요청이 들어와도 거부되도록

- 활용 예시:
  - Redis에서 RefreshToken 삭제
  - 블랙리스트 토큰 등록

✅ 필터 체인 구성 순서

🔸 JWTFilterV3 (JWT 인증 필터)
- 역할: 매 요청마다 Authorization 헤더의 JWT를 읽고 인증 처리.

- 목적:
  - 모든 요청에 대해 JWT 토큰을 검사하여 인증 처리
  - AccessToken 유효성 확인, 해당 유저 정보를 기반으로 SecurityContext에 인증 객체 설정
  - 블랙리스트 확이도 포함

- 기능:
  - 토큰 유효성 검사
  - Redis에 등록된 블랙리스트/Refresh Token 확인

- 코드
public class JWTFilterV3 extends OncePerRequestFilter {

    private final JWTUtil jwtUtil;
    private final RedisTemplate<String, String> redisTemplate;
    private final MemberRepository memberRepository;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        String token = jwtUtil.resolveToken(request);

        // 1. 토큰이 없거나 유효하지 않으면 다음 필터로
        if (token == null || !jwtUtil.validateToken(token)) {
            filterChain.doFilter(request, response);
            return;
        }

        // 2. 블랙리스트 체크
        String isBlacklisted = redisTemplate.opsForValue().get("Blacklist:" + token);
        if (isBlacklisted != null) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return;
        }

        // 3. 토큰에서 사용자 ID 추출 -> 사용자 인증 처리
        String userId = jwtUtil.getId(token);
        Member member = memberRepository.findById(Long.valueOf(userId)).orElse(null);
        if (member == null) {
            filterChain.doFilter(request, response);
            return;
        }

        UsernamePasswordAuthenticationToken auth =
            new UsernamePasswordAuthenticationToken(member, null, member.getAuthorities());

        SecurityContextHolder.getContext().setAuthentication(auth);

        filterChain.doFilter(request, response);
    }
}
- 핵심 포인트
  - resolveToken(request) -> 헤더에서 Bearer 토큰 추출
  - validateToken() -> 유효성 및 만료 확인
  - Redis에 블랙리스트가 존재하면 인증 거부
  - 유저 정보를 로딩하여 SecurityContext에 설정 -> 이후 컨트롤러에서 @AuthenticationPrincipal 사용가능

🔸 LoginFilter
- 역할: /login 경로에서 JSON 기반 로그인 처리 (email + password)

- 목적:
  - /login 요청에 대해 ID/PW(JSON 형식)를 받아 인증 처리
  - 인증 성공 시, JWT 발급 + Redis에 RefreshToken 저장 + 쿠키 설정등 

- 기능:
  - 인증 성공 시 JWT Access/RefreshToken 발급
  - Redis에 RefreshToken 저장

- 코드:
public class LoginFilter extends UsernamePasswordAuthenticationFilter {

    private final AuthenticationManager authenticationManager;
    private final JWTUtil jwtUtil;
    private final RedisTemplate<String, String> redisTemplate;

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) {
        try {
            LoginRequest loginRequest = new ObjectMapper().readValue(request.getInputStream(), LoginRequest.class);
            UsernamePasswordAuthenticationToken token =
                new UsernamePasswordAuthenticationToken(loginRequest.getEmail(), loginRequest.getPassword());

            return authenticationManager.authenticate(token);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
    
    @Override
    protected void successfulAuthentication(HttpServletRequest request,
                                            HttpServletResponse response,
                                            FilterChain chain,
                                            Authentication authResult) throws IOException {

        CustomUserDetails userDetails = (CustomUserDetails) authResult.getPrincipal();

        String accessToken = jwtUtil.createAccessToken(userDetails);
        String refreshToken = jwtUtil.createRefreshToken(userDetails);

        // RefreshToken Redis 저장
        redisTemplate.opsForValue().set("RefreshToken:" + userDetails.getId(), refreshToken,
                jwtUtil.getRefreshExpiration(refreshToken), TimeUnit.MILLISECONDS);

        // 쿠키 설정
        Cookie accessCookie = new Cookie("AccessToken", accessToken);
        accessCookie.setHttpOnly(true);
        accessCookie.setSecure(true);
        accessCookie.setPath("/");
        accessCookie.setMaxAge((int) jwtUtil.getExpiration(accessToken) / 1000);
        response.addCookie(accessCookie);

        response.getWriter().write("Login success");
    }
}
- 핵심 포인트
  - 요청 본문(JSON)에서 사용자 인증 정보 추출
  - AuthenticationManager.authenticate()로 실제 인증 수행
  - 성공 시,
    - JWT 생성
    - RefreshToken 을 Redis 에 저장
    - AccessToken 을 쿠키로 설정
  - 실패 시,
    - unsuccessfulAuthentication() 에서 처리

✅ 필터 순서
JWTFilterV3 → 먼저 실행되어 기존 JWT 인증 처리
LoginFilter → 이후 실행되어 로그인 요청을 처리

- 필터 순서의 목적:
로그인 API는 JWT 없이 접근되므로, JWTFilter보다 뒤에 배치하여 인증 필터 체인을 명확히 분리.

### 전체 구성 요약

| 기능 구분        | 방식                                | 목적/설정 이유          |
| ------------ | --------------------------------- | ----------------- |
| 브라우저 폼 로그인   | `formLogin()`                     | 전통 UI 인증 지원       |
| 소셜 로그인       | `oauth2Login()`                   | OAuth2 인증 지원      |
| 세션 무상태 처리    | `SessionCreationPolicy.STATELESS` | JWT 기반 인증 구조      |
| 로그아웃         | `.logout().disable()`             | 커스텀 로그아웃 처리       |
| 인증 필터        | `JWTFilterV3`, `LoginFilter`      | JWT 검증 & 토큰 발급    |
| 사용자 정보 저장 방식 | Redis + JWT                       | 상태 저장 없이 인증 상태 유지 |

### 마무리 : 왜 이렇게 복잡한 설정을 구성했는가

1. UI 사용자 (Form Login, Social Login)와 API 사용자 (JWT 기반 인증)를 모두 지원하기 위해.
2. 특히 서버가 stateless 방식을 기반으로 설계되어 있기 때문에, 로그인 성공 후에는 JWT를 통한 인증으로 모든 요청을 처리함
3. 세션 기반 인증과 JWT 인증을 동시에 설정하되, 최종적으로는 JWT를 통해 상태를 유지하도록 전체 로직을 구성함.

### 필터 내부 인증 처리 흐름(Form 로그인 & OAuth2 로그인)

1. 1단계 : 로그인 요청 감지 (/api/login)

- 클라이언트가 POST /api/login 으로 JSON (이메일, 비밀번호) 전송
- LoginFilter 의 attemptAuthentication() 호출

2. 2단계 : 입력값으로 인증 객체 생성   

ObjectMapper mapper = new ObjectMapper();
LoginRequestDTO loginDTO = mapper.readValue(request.getInputStream(), LoginRequestDTO.class);

UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
    loginDTO.getEmail(), loginDTO.getPassword()
);

3. 3단계 : AuthenticationManager 인증 위임

Authentciation authentication = auathenticationManager.authenticate(authToken);

-> 내부적으로 UserDetailsService 를 통해 사용자 조회
-> 비밀번호 검증 (PasswordEncoder.matches())
-> 성공하면 UsernamPasswordAuthenticationToken 에 인증된 사용자 담김

4. 4단계 : 인증 성공 처리

successfulAuthentication(...)
- JWT 생성
- 응답 헤더 또는 쿠키에 토큰 저장
- SecurityContextHolder 에 인증 정보 저장 (Spring Security)

### OAuth2 로그인 - OAuth2LoginAuthenticationFilter 흐름

1. 1단계 : /oath2/authorization/google 요청

- OAuth2 필터가 외부 로그인 페이지 (구글)로 Redirect

2. 2단계 : 사용자 로그인 성공 -> 콜백 /login/oauth2/code/google 도착

- Authentication Code + Redirect URI로 서버에 다시 도착
- 필터가 코드로 토큰 교환하고 사용자 정보 요청

3. 3단계 : 사용자 정보 수신 및 가공

CustomOAuth2UserService.loadUser(OAuth2UserRequest request)
- 구글 통해서 유저 정보 얻기
- 유저 DB에 존재하지 않으면, 자동 회원가입 처리
- 이후, OAuth2User 객체 생성

4. 4단계 : 인증 성공 처리

OAuth2AuthenticationToken → SecurityContextHolder 저장
- JWT 발급 또는 세션 설정
- 리디렉션 or 토큰 응답

### JWT 발급 및 인증 흐름 (JWTUtil 중심)

1. 인증 성공 시, JWT 발급 (LoginFilter or OAuth2SuccessHandler 등)

발급 로직 
String accessToken = jwtUtil.createAccessToken(userId, role);
String refreshToken = jwtUtil.createRefreshToken(userId);

코드 예시
public String createAccessToken(String userId, String role) {
    return Jwts.builder()
    .subject(userId)
    .claim("role", role)
    .expiration(new Date(now + accessTokenExpiry))
    .signWith(secretKey, SignatureAlgorithm.HS256)
    .compact();
}

2. JWT 저장 

쿠키 방식 : HttpOnly + Secure 쿠키에 저장 (보안 강화)
헤더 방식 : Authorization : Bearer {token} 응답으로 전달

3. 이후 요청에서 JWT 인증 흐름 (JWTFilterV3)

- JWTFilterV3 (extends OncePerRequestFilter)
1) 요청 헤더 or 쿠키에서 JWT 추출
    String token = tokenResolver.resolve(request); // 쿠키 or 헤더에서 토큰 추출
2) 유효성 검사
    if (jwtUtil.isExpired(token)) {...}
    Claims claims = jwtUtil.getClaims(token);
    - 서명 검증, 만료 여부 확인
    - 파싱 실패 시, 예외 발생
3) 인증 객체 생성 후, SecurityContext 설정
    String username = claims.getSubject();
    String role = claims.get("role");

    UsernamePasswordAuthenticationToken auth = 
        new UsernamePasswordAuthenticationToken(username, null, authorities);
    SecurityContextHolder.getContext().setAuthentication(auth);
4) 이후 컨트롤러 접근 허용됨
    - 인증 객체 존재 -> 인가 처리 성공
    - 없가나 만료 -> 401 Unauthorized

### 전체 인증 흐름 정리 (Form & OAuth2 + JWT 인증 포함)    

[ 클라이언트 로그인 요청 ]

1. Form Login (POST /api/login)       → LoginFilter
    ↳ AuthenticationManager.authenticate()
    ↳ UserDetailsService로 사용자 검증
    ↳ JWT 생성 및 응답

2. OAuth2 Login (GET /oauth2/...)     → OAuth2LoginAuthenticationFilter
    ↳ 외부 인증 서버 리다이렉트
    ↳ 사용자 정보 수신 → 자동 회원가입 or 로그인
    ↳ JWT 생성 or 세션 등록

[ 이후 모든 요청 (API 호출 등) ]

3. JWT 포함된 요청                    → JWTFilterV3
    ↳ 토큰 파싱 & 유효성 검사
    ↳ 인증 정보 생성 → SecurityContext 저장
    ↳ 인증된 사용자로서 컨트롤러 접근 가능

[ 인증 실패 시 ]
    ↳ 커스텀 예외 처리 → 401, 403, 로그인 페이지 redirect 등

LoginFilter (extends AbstractAuthenticationProcessingFilter)
JWTFilterV3 (extends OncePerRequestFilter)
OAuth2LoginAuthenticationFilter (스프링 내장)
