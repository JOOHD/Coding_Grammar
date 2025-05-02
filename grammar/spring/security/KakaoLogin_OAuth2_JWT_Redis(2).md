## 본문

## Kakao OAuth2 + JWT + Redis를 통한 인증 과정 구현 (2) - JWT 적용

    최초 로그인의 경우 DB에 회원을 저장하고, Details 객체를 만들어 Authentication 객체에 담고, SecurityContext 에 Authentication 객체를 보관하도록 설정하였고,

    최초 로그인이 아닌 경우에는 DB에서 회원을 가져와 Details 객체를 만들어 Authentication 객체에 담고, SecurityContext 에 Authentication 객체를 보관하도록 설정하였다.

    - 이번 목적은 JWT를 어떻게 적용했는지를 중점으로 보자.

### 0. application.yml 설정     

    jwt: 
        secret_key : ${jwt.secret_key}
        access-token-validity-in-seconds : 30000 (배포 시, 30분 -> 1800 설정)
        refresh-token-validity-in-seconds : 86400 (배포 시, 1일 -> 86400 설정) 

    시그니처에 사용할 secret_key & accessToken expired & refreshToken expired 을 yml에 지정해준다. 

    secret_key 값은 중요한 정보이니만큼 github 로 형성관리하지 않고, 별도로 다른 파일에 설정해주고 해당 파일은 gitignore 에 등록해준다.

    ● build.gradle 에 의존성 추가

    implementation 'io.jsonwebtoken:jjwt-api:0.11.5'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.11.5'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.11.5'

### 1. TokenProvider - 사용자 정보로 JWT 토큰 생성  

    public class TokenProvider {

        private static final String AUTH_KEY = "AUTHORITY";
        private static final String AUTH_EMAIL = "EMAIL";

        private final String secretKey;
        private final long accessTokenValidityMilliSeconds;
        private finla long refreshTokenValidityMilliSeconds;

        private Key secretKey;

        public TokenProvider(@Value("${jwt.secret_key"}) String secretKey,           
                             @Value("${jwt.access-token-validity-in-seconds}") long accessTokenValiditySeconds,
                             @Value("${jwt.refresh-token-validity-in-seconds}") long refreshTokenValiditySeconds) {
            
            this.secretKey = secretKey;
            this.accessTokenValidityMilliSeconds = accessTokenValiditySeconds * 1000;
            this.refreshTokenValidityMilliSeconds = refreshTokenValiditySeconds * 1000;
        }
    
        @PostConstruct
        public void initKey() {
            byte[] keyBytes = Decoders.BASE64.decode(secretKey);
            this.secretkey = Keys.hmacShaKeyFor(keyBytes);
        }
    }

    각 토큰에 대한 만료시간과 secretKey 같은 yml 설정 정보를 토대로 생성자 주입하고, 생성자 주입 후 JWT 생성에 사용될 Key는 initKey() 메소드로 secretKey 를 decode 하여 Key에 주입했다.

    // access, refresh Token 생성
    public TokenDto createToken(String email, String role) {

        long now = (new Date()).getTime();

        Date accessValidity = new Date(now + this.accessTokenValidityMilliSeconds);
        Date refreshValidity = new Date(now + this.refreshTokenValidityMilliSeconds);

        String accessToken = Jwts.builder()
                .addClaims(Map.of(AUTH_EMAIL, email))
                .addClaims(Map.of(AUTH_KEY, role))
                .signWith(secretkey, SignatureAlgorithm.HS256)
                .setExpiration(accessValidity)
                .compact();
 
        String refreshToken = Jwts.builder()
                .addClaims(Map.of(AUTH_EMAIL, email))
                .addClaims(Map.of(AUTH_KEY, role))
                .signWith(secretkey, SignatureAlgorithm.HS256)
                .setExpiration(refreshValidity)
                .compact();
 
        return TokenDto.of(accessToken, refreshToken);
    }

    // token 이 유요한 지 검사
    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                    .setSigningKey(secretKey)
                    .build()
                    .parseClaimsJws(token);
            return true;        
        } catch (io.jsonwebtoken.security.SecurityException | MalformedJwtException e) {
            return false;
        } catch (UnspportedJwtException e) {
            return false;
        } catch (IllegalArgumentException e) {
            return false;
        }
    }

    // token 이 만료되었는지 검사
    public boolean validateExpire(String token) {
        try {
            Jwts.parserBuilder()
                    .setSigningKey(secretKey)
                    .build()
                    .parseClaimsJws(token)
        }
    }

    // token으로부터 Authentication 객체를 만들어 리턴하는 메소드
    public Authentication getAuthentication(String token) {

        Claims claims = Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .build()
                .parseClaimsJwts(token)
                .getBody();

        List<String> authorities = Arrays.asList(claims.get(AUTH_KEY)
                .toString()
                .split(","));
 
        List<? extends GrantedAuthority> simpleGrantedAuthorities = authorities.stream()
                .map(auth -> new SimpleGrantedAuthority(auth))
                .collect(Collectors.toList());
 
        KakaoMemberDetails principal = new KakaoMemberDetails(
                (String) claims.get(AUTH_EMAIL),
                simpleGrantedAuthorities, Map.of());
 
        return new UsernamePasswordAuthenticationToken(principal, token, simpleGrantedAuthorities);
    }        

    TokenProvider 에서 토큰 생성 뿐만 아니라 토큰 검증 관련 메소드와 토큰으로부터 Authentication 객체를 리턴하는 기능들을 추가 구현했다.

### 2. OAuth2SuccessHandler

    @Component
    @RequiredArgsConstructor
    public class OAuth2SuccessHandler extends SimpleUrlAuthenticationSuccessHandler
    {
        private static final String REDIRECT_URI = "http://localhost:8080/api/sign/login/kakao?accessToken=%s&refreshToken=%s";

        private final TokenProvider tokenProvider;
        private final MemberRepository memberRepository;

        @Transactional
        @Override
        public void onAuthenticatoinSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException 
        {
            OAuth2User oAuth2User = (OAuth2User) authentication.getPrincipal();
            KakaoUserInfo kakaoUserInfo = new KakaoUserInfo(oAuth2User.getAttributes());

            Member member = memberRepsitory.findByEmail(kakaoUserInfo.getEmail()).orElseThrow(MemberNotFoundException::new);

            TokenDto tokenDto = tokenProvider.createToken(member.getEmail(), member.getRole().name());

            String redirectURI = String.format(REDIRECT_URI, tokenDto.getAccessToken(), tokenDto.getRefreshToken());

            getRedirectStrategy().sendRedirect(request, response, redirectURI);
        }
    }

    SimpleUrlAuthenticationSuccessHandler 를 상속하여 onAuthenticationSuccess() 메소드를 오버라이딩하면 Authentication 객체를 추가적으로 처리할 수 있다.

    후처리 서비스인 KakaoMemberDetailsService 에서 Security Context 에 Authentication 객체를 저장한 덕분에 이로부터 사용자 정보를 꺼내와 token 정보를 생성하고 이 토큰 정보를 파라미터에 담아 redirect URI 로 보내 처리한다.

    이렇게 등록한 Handler 는 SecurityConfig 에 추가적으로 등록하면 된다.

    ● SecurityConfig 등록

    .oauth2Login(oAuth2Login -> {
        oAuth2Login.userInfoEndPoint(userInfoEndpointConfig ->
                    userInfoEndpointConfig.userService(kakaoMemberDetailsService)); // 1
        oAuth2Login.successHandler(oAuth2SuccessHandler); // 2
    });
    - 카카오 로그인에 성공한 경우 1번이 먼저 실행되고, 그 후 2번이 실행

        1번  
        OAuth2 인증 후, 카카오 인증 서버로부터 받은 AccessToken 을 사용해 사용자 정보를 가져온다.
        이 정보를 활용해 애플리케이션에서 사용할 사용자 객체(kakaoMemberDetails)로 변환 하거나, DB에 저장된 사용자를 조회.

        2번
        로그인 성공 후, JWT 생성 및 리다이렉트 등의 후속 작업을 처리

    위 onAuthenticationSuccess() 메소드에서 redirect 된 URI는 아래 컨트롤러에 매핑되어 Json으로 클라이언트에게 넘겨준다.

    @RestController
    @RequiredArgsConstructor
    @RequestMapping("/api/sign")
    public class SignController {

        @GetMapping("/login/kakao")
        public ResponseEntity loginKakao(
            @RequestParam(name = "accessToken") String accessToken,
            @RequestParam(name = "refreshToken") String refreshToken) {
                return new ResponseEntity(TokenDto.of(accessToken, refreshToken), HttpStatus.OK);
            }
    }

    토큰 발급은 모두 끝났다. 
    마지막으로 사용자로부터 요청이 왔을 때, 사용자가 헤더에 첨부한 토큰들이 유효한지 검증하는 필터와 통과하지 못했을 때, 처리할 handler를 등록해주면 도니다.

### JwtFilter    

    public class JwtFilter extends OncePerRequestFilter {

        private static final String ACCESS_HEADER = "AccessToken";
        private static final String REFRESH_HEADER = "RefresToken";

        private final TokenProvider tokenProvider;

        @Override
        protected void doFilterInternal(HttpServletRequest request,
                                        HttpServletResponse response,
                                        FilterChain filterChain) throw..

        {
            // #1
            if (isRequestPassURI(request, response, filterChain)) {
                return;
            }

            String accessToken = getTokenFromHeder(request, ACCESS_HEADER);

            if (tokenProvider.validate(accessToken) && tokenProvider.validateExpire(accessToken)) {
                SecurityContextHolder.getContext().setAuthentication(tokenProvider.getAuthentication(accessToken));
            }

            filterChain.doFilter(request, response);
        }
    }

    만약 토큰 유효성 검증이 필요하지 않은 요청 URI라면 1번 로직이 실행되어 필터를 통과한다.

        - 유효성 검증이 필요하지 않은 요청이라 함은 로그인 api, exception, end point api 등등

    만약 토큰 유효성 검증이 필요한 요청이라면 getTokenFromHeader 를 통해 헤더에서 accessToken 을 가져오고 tokenProvider 에서 검증을 위임하여 통과하면, 토큰으로부터 authentication 객체를 만들어 시큐리티 컨테스트에 보관하고 필터에 통과한다.    

    - 검증에 실패한 경우에 대한 처리는 아래에 별도로 처리

    - 이 필터 또한 SecurityConfig 에 등록해주면 된다.
    .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)

### JWT Handler

    마지막으로 JWT Filter 를 통과하지 못한 경우에 대하여 처리해주어야 한다.
    통과 하지 못한 경우라 함은 인증에 통과되지 못한 경우, 또는 인가(권한)에 통과하지 못한 경우 두 가지가 있다.
    
    1. 인증 실패

    @Component
    public class JwtAuthenticationFailEntryPoint implements AuthenticationEntryPoint 
    {
        private static final String EXCEPTION_ENTRY_POINT = "/api/exception/entry-point";

        @Override
        public void commenc(HttpServletRequest request,
                            HttpServletResponse response,
                            AuthenticationException authException) throws.. 
        {
            resposne.sendRedirect(EXCEPTION_ENTRY_POINT);
        }
    }

    2. 인가 실패

    @Component
    public class JwtAccessDeniedHandler implements AccessDeniedHandler {

        private static final String EXCEPTION_ACCESS_HANDLER = "/api/exception/access-denied";

        @Override
        public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws {
            response.sendRedirect(EXCEPTION_ACCESS_HANDLER);
        }
    }

    인가/인증에 대한 실패 처리 핸들러는 AccessDeniedHandler/AuthenticationEntryPoint 인터페이스의 commence/handler 메소드를 구현하면 되겠습니다.

    @RestController
    @RequestMapping("/api/exception")
    public class ExceptionController {
    
        @GetMapping("/access-denied")
        public void accessDeniedException() {
            throw new AccessDeniedException();
        }
    
        @GetMapping("/entry-point")
        public void authenticateException() {
            throw new AuthenticationEntryPointException();
        }
    }

    목적  
        -특정 예외 발생 시, 직접 호출 가능한 API endPoint 제공

    상황 
        - 클라이언트가 명시적으로 예외와 관련된 특정 URI를 호출하도록 설계
        - 예외 처리가 서버 내부의 redirect 로직에 의존하는 경우 

        ex)
            JwtAuthenticationFailEntryPoint & JwtAccessDeniedHandler 에서 redirect 할 endpoint

    구성 
        - /api/exception/access-denied & /api/exception/entry-point 와 같은 API URI 를 정의
        - 각각 AccessDeniedException & AuthenticationEntryPointException 을 직접 던지는 역할.     

    - 요청이 특정 URI로 리다이렉트되면, 해당 엔드포인트를 통해 예외를 발생시킴.

        ex)
            권한 부족: /api/exception/access-denied
            인증 실패: /api/exception/entry-point

    
    이 API에 대한 접근 또한 Jwt필터에서 추가적으로 isRequestPassURI에 등록!

    @RestControllerAdvice
    public class ExceptionAdvisor {
    
        @ExceptionHandler(AccessDeniedException.class)
        @ResponseStatus(HttpStatus.UNAUTHORIZED)
        public ResponseEntity accessDeniedException(AccessDeniedException e) {
            return new ResponseEntity("접근 불가능한 권한입니다.", HttpStatus.UNAUTHORIZED);
        }
    
        @ExceptionHandler(AuthenticationEntryPointException.class)
        @ResponseStatus(HttpStatus.UNAUTHORIZED)
        public ResponseEntity authenticationEntryPointException(AuthenticationEntryPointException e) {
            return new ResponseEntity("로그인이 필요한 요청입니다.", HttpStatus.UNAUTHORIZED);
        }
    }

    목적 
        - global exception (AOP) 를 제공하여, 전체 애플리케이션에서 발생하는 예외를 통합적으로 처리.

    상황 
        - API endpoint 나 로직에서 발생하는 예외를 자동으로 처리.
        - 특정 컨트롤러를 호출하지 않아도 발생한 예외를 잡아서 일관된 응답을 반환.

        ex)
            AccessDeniedException 또는 AuthenticationEntryPointException이 발생했을 때 모든 요청에 대해 공통된 처리를 함.    

    구성
        - @RestControllerAdvice 를 사용해 전역 예외 헨들러를 구현
        - 예외 클래스별로 @ExceptionHandler 등록        

    - 특정 예외가 발생하면, 해당 예외에 매핑된 헨들러 메서드가 호출되어 적절한 응답 반환.    

        ex)
            권한 부족: AccessDeniedException -> "접근 불가능한 권한입니다."
            인증 실패: AuthenticationEntryPointException -> "로그인이 필요한 요청입니다."


### SecurityConfig에 해당 헨들러 등록

    .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)    

    .exceptionHandling(exceptionHandling -> {
        exceptionHandling.authenticationEntryPoint(jwtAuthenticationFailEntryPoint);
        exceptionHandling.accessDeniedHandler(jwtAccessDeniedHandler);
    });

    ● 왜 두 군데에 등록하는가?

    1. 역할 분리

        - ExceptionController : 예외를 발생시키는 API 역할.
          - 예외 상황을 명시적으로 URI로 나타냄
          - 다른 클래스와 연계하여 예외 발생 uri로 redirect

        - ExceptionAdvisor : 전역 예외 처리.
          - 예외가 발생했을 때, 자동으로 잡아 응답 반환
          - API 호출 외의 모든 예외 상황을 처리

    2. Redirect 와 응답 처리의 분리
    
        - 리다이렉트가 발생한 뒤에도 예외를 처리하기 위해서는 ExceptionAdvisor 가 필요.
        - 리다이렉트된 엔드포인트(/api/exception/*)에서 예외가 발생하면, 이를 잡아 적절한 메시지와 상태 코드로 응답해야 함.

### 흐름 
    
    if (tokenProvider.validate(accessToken) && tokenProvider.validateExpire(accessToken)) {}
    filterChain.doFilter(request, response);

    - SecurityContextHolder 에 인증 정보를 설정하고 다음 필터로 요청을 전달.
    - 유효하지 않은 JWT : 필터 체인에서 SecurityContext 에 인증 정보가 없으면, 이후 필터가 예외를 처리.

    1. request -> authentication/authority fail(예외 발생) -> JwtFilter(JwtAuthenticationFailEntryPoint & JwtAccessDeniedHandler) 호출.

    2. Redirect -> ExceptionController 호출 -> 예외 발생.
    
    3. ExceptionAdvisor -> 발생한 예외 처리, 
        예외 잡아 응답 반환 return new ResponseEntity("message", HttpStatus.UNAUTHORIZED);

    결론

    - ExceptionController 는 특정 상황에서 호출 가능한 명시적 API를 제공하기 위한 클래스.
    
    - ExceptionAdvisor 는 전역적으로 예외를 처리하여 일관된 응답을 반환하기 위한 클래스

    - 두 클래스는 서로 보완 관계로 설계되어, 리다이렉트와 전역 예외 처리를 명확히 분리.



