## 본문

### refershToken 흐름 정리

| 순서 | 위치 / 클래스                     | 설명                                                                 |
|------|-----------------------------------|----------------------------------------------------------------------|
| 1    | 프론트엔드                        | 이메일+비밀번호로 로그인 요청 (`POST /auth/login`)                  |
| 2    | `AuthController.login()`          | 로그인 요청을 `MemberService.memberLogin()`에 위임                  |
| 3    | `MemberService.memberLogin()`     | 이메일로 회원 조회 → 비밀번호 검증 → 성공 시 `Member` 반환         |
| 4    | `JWTUtil.createJwt()`             | accessToken, refreshToken 생성                                      |
| 5    | `CookieUtil.createHttpOnlyCookie()` | refreshToken을 HttpOnly 쿠키에 담음                              |
| 6    | `ResponseEntity`                  | accessToken은 응답 헤더에, refreshToken은 쿠키로 응답               |
| 7    | 프론트엔드                        | accessToken은 localStorage, refreshToken은 자동 쿠키 저장           |
| 8    | accessToken 만료 후 요청 발생     | 프론트 요청에 refreshToken 쿠키 자동 포함                           |
| 9    | `JWTFilterV3.doFilterInternal()`  | accessToken 만료 확인 후 refreshToken 추출 시도                     |
| 10   | `CookieUtil.extractRefreshTokenFromCookie()` | 쿠키에서 refreshToken 추출                            |
| 11   | `JWTUtil.isTokenValid()`          | refreshToken 유효성 검사                                             |
| 12   | `JWTUtil.extractEmail()`          | refreshToken에서 이메일 추출                                         |
| 13   | `JWTUtil.reissueAccessToken()`    | 새 accessToken 발급                                                  |
| 14   | `JWTFilterV3`                     | 새 accessToken을 응답 헤더에 실어 클라이언트로 전달                |
| 15   | 프론트엔드                        | 새 accessToken 저장 → 요청 재시도 가능

### 설정

    JWT는 한 번 발급하면 만료되기 전까지 삭제할 수 없다.
    따라서 짧은 유효시간을 갖는 AccessToken 과 저장소에 저장해서 AccessToken 을 재발급이 가능한 RefreshToken 이 있다. RefreshToken 발급과 관리 및 이를 통한 AccessToken 재발급에 대해 알아보자.

    + AccessToken 재발급은 2가지 방법이 있다.

    1. 요청마다, Access/Refresh Token 을 같이 넘기는 방법.
    2. 재발급 API를 만들고 서버에서 AccessToken 이 만료되었다고 응답하면, RefreshToken 으로 요청하여 재발급 받기.

    우선 1번으로 진행

## H2 inMemory 사용

    JWT는 발급 후, 삭제가 불가능하기 때문에 접근에 관여하는 토큰은 유효시간을 길게 부여할 수 없다. 하지만 자동 로그인 혹은 로그인 유지를 위해서는 유효시간이 긴 토큰이 필요하다. 이때 사용되는 것이 RefreshToken 이다.

    ● 생명 주기

    AccessToken : 발급된 이후, 서버에 저장되지 않고 토큰 자체로 검증을 하며 사용자 권한을 인증한다.

        - 이런 역할을 하는 AccessToken 이 탈취되면 토큰이 만료되기 전 까지, 토큰을 획득한 사람은 누구나 권한 접근이 가능해진다.

        - 따라서 AccessToken 유효 주기는 짧게 가져가야 한다.

    그러면, 자동 로그인 혹은 로그인 유지는?

        - 이제 RefreshToken 이 역할이다. RefreshToken 은 한 번 발급되면 AccessToken 보다 훨씬 길게 발급된다.

        - 대신에 접근에 대한 권한을 주는 것이 아니라 AccessToken 재발급에 관여한다.

    ● AccessToken 재발급 방법

    보통 RefreshToken 은 로그인 성공시 발급되며 저장소에 저장하여 관리된다.
    그리고 사용자가 로그아웃을 하면 저장소에서 RefreshToken 을 삭제하여 사용이 불가능하도록 한다.

    AccessToken 이 만료되어, 재발급이 진행된다면 다음의 과정을 통해 재발급이 된다.

        1. RefreshToken 유효성 체크
        2. DB에 RefreshToken 존재유무 체크
        3. 1,2 모두 검증되면 재발급 진행
        4. Response Header 에 새로 발급한 AccessToken 저장

    이후 클라이언트는 재발급된 AccessToken 을 Request Header에 포함하여 요청을 보내면 정상적으로 접근이 허용된다.

### RefreshToken 적용

    ● Domain

        // RefreshToken.class
        @Entity
        @AllArgsConstructor
        @NoArgsConstructor
        public class RefreshToken {

            @Id
            @Column(nullable = false)
            private String refreshToken;
        }

    - 단순 토큰 값만 저장하면 되기 때문에, id 는 생성x
    - 이번에는 H2 를 사용한 방법이고, 다음에는 Redis(key, value)를 이용해보겟다.

    ● Interface

        public interface TokenRepository extends JpaRepository<RefreshToken, Long> 
        {
            boolean existsByRefreshToken(String token);
        }
        
        - token 존재 여부 판단이 목적.

    ● Controller

        // 로그인
        @PostMapping("/login")
        public ResposneEntity login(@RequestBody UserDTO user, HttpServletResponse response) {

            // 회원 존재 유무
            User member = userService.findMember(user);

            // 비밀번호 체크
            uerService.checkPassword(member, user);

            // access, refresh token 발급 및 헤더 설정
            String accessToken = jwtTokenProvider.createAccessToken(member.getEmail(), member.getRoles());
            String refreshToken = jwtTokenProvider.createRefreshToken(member.getEmail(), member.getRoles());
            jwtTokenProvider.setHeaderAccessToken(response, accessToken);
            jwtTokenProvider.setHeaderRefreshToken(response, refreshToken);

            // refreshToken DB에 저장.
            tokenRepository.save(new RefreshToken(refreshToken));

            return ResponseEntity.ok().body("로그인 성공!");
        }

        - 응답 헤더에 토큰을 추가하는 작업을 AccessToken 재발급 진행하면서 사용하기 때문에 jwtProvider 의 메서드로 작성하였다.

### JwtTokenProvider

    @RequiredArgsConstructor
    @Component
    public class JwtTokenProvider {

        // key
        private String secretKey = "lalala";

        // accessToken 유효시간 (20s)
        private long accessTokenValidTime = 20 * 1000L;

        // refreshToken 유효시간 (1m)
        private long refreshTokenValidTime = 1 * 60 * 1000L;

        private final CustomUserDetailService custom~;
        private final TokenRepository token~;
        private final UserRepository userRepository;

        // 의존성 주입 후, 초기화 실행
        // 객체 초기화, secretKey 를 Base64 로 인코딩.
        @PostConstruct
        protected void init() {
            secretKey = Base64.getEncoder().encodeToString(secretKey.getBytes());
        }

        // AT 생성
        public String createAccessToken(String email, List<String> roles) {
            return this.createToken(email, roles, accessTokenValidTime);
        }

        // RT 생성
        public String createRefreshToken(String email, List<String> roles) {
            return this.createToken(email, roles, refreshTokenValidTime);
        }

        // create token
        public String createToken(String email, List<String> roles, long tokenValid) {
            Claims claims = Jwts.claims().setSubject(email); // claims 생성 및 payload 설정
            claims.put("roles", roles); // 권한 설정, (key, value) 저장

            Date date = new Date();
            return Jwts.builder()
                    .setClaims(claims) // 발행 유저 정보 저장
                    .setIssuedAt(date) // 발행 시간 저장
                    setExpirattion(new Date(date.getTime() + tokenValid)) // 토큰 유효 시간 저장
                    .signWith(SignatureAlgorithm.Hs256, secretKey) // 해싱 알고리즘 및 키 설정
                    .compact(); 생성
        }
        
        // JWT 에서 인증 정보 조회
        public Authentication getAuthentication(String token) {
            UserDetails userDetails = customUserDetailService.loadUserByUsername(this.getUserEmail(token));
        }
        
        // 토큰에서 회원 정보 추출
        public String getUserEmail(String token) {
            return Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token).getBody().getSubject();
        }

        // Request의 Header에서 AccessToken 값을 가져온다. "authorization" : "token"
        public String resolveAccessToken(HttpServlerRequest request) 
        {
            if (request.getHeader("authorization") != null) {
                return request.getHeader("authorization").substring(7);
            }
            return null;
        }

        // Request의 Header에서 RefreshToken 값을 가져옵니다. "authorization" : "token'
        public String resolveRefreshToken(HttpServletRequest request) {
            if(request.getHeader("refreshToken") != null )
                return request.getHeader("refreshToken").substring(7);
            return null;
        }

        // 토큰 유형성 + 만료일자 확인
        public boolean validateToken(String jwtToken) {
            try {
                Jws<Claims> claims = Jwts.parser().setSigningKey(secretKey).parseClaimsJws(jwtToken);
                return !claims.getBody().getExpiration().before(new Date());
            } catch (ExpiredJwtException e) {
                log.info(e.getMessage());
                return false;
            }
        }

        // 어세스 토큰 헤더 설정
        public void setHeaderAccessToken(HttpServletResponse response, String accessToken) {
            response.setHeader("authorization", "bearer "+ accessToken);
        }

        // 리프레시 토큰 헤더 설정
        public void setHeaderRefreshToken(HttpServletResponse response, String refreshToken) {
            response.setHeader("Set-Cookie", refreshTokenHeader);
        }

        // RefreshToken 존재유무 확인
        public boolean existsRefreshToken(String refreshToken) {
            return tokenRepository.existsByRefreshToken(refreshToken);
        }

        // refreshToken 재발급 후, 권한 부여를 위한 권한 반환 메서드
        // Email 로 권한 가져오기
        public List<String> getRoles(String email) 
        {
            return userRepository.findByEmail(email).get().getRoles();
        } 
    }

### JwtAuthenticationFilter

    상속 객체 변경 GenericFilterBead -> OncePerRequestFilter

    기존 Filter 는 jwt 검증 예외가 발생하는 경우 Filter가 여러번 동작하는 것을 확인하여 새로운 방안으로 적용하였다.

    전체적인 뼈대는 같으나 오버라이딩 하는 메서드 명과 doFilter 방식이 다르다.

    @RequiredArgsConstructor
    public class JwtAuthenticationFilter extends OncePerRequestFilter {

        private final JwtTokenProvider jwtTokenProvider;

        @Override
        protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

            // 헤더에서 JWT 를 받아옵니다.
            String accessToken = jwtTokenProvider.resolveAccessToken(request);
            String refreshToken = jwtTokenProvider.resolveRefreshToken(request);

            // 유효한 토큰인지 확인
            if (accessToken != null) {
                // accessToken 유효한 경우
                if (jwtTokenProvider.validateToken(accessToken)) {
                    this.setAuthentication(accessToken);
                }
                else 
                if (!jwtTokenProvider.validateToken(accessToken) && refreshToken != null) {

                    // 재발급 후, 컨텍스트에 다시 넣기
                    /// 리프레시 토큰 검증
                    boolean validateRefreshToken = jwtTokenProvider.validateToken(refreshToken);

                    /// 리프레시 토큰 저장소 존재유무 확인
                    boolean isRefreshToken = jwtTokenProvider.existsRefreshToken(refreshToken);

                    if (validateRefreshToken && isRefreshToken) {
                    /// 리프레시 토큰으로 이메일 정보 가져오기
                    String email = jwtTokenProvider.getUserEmail(refreshToken);
                    /// 이메일로 권한정보 받아오기
                    List<String> roles = jwtTokenProvider.getRoles(email);
                    /// 토큰 발급
                    String newAccessToken = jwtTokenProvider.createAccessToken(email, roles);
                    /// 헤더에 어세스 토큰 추가
                    jwtTokenProvider.setHeaderAccessToken(response, newAccessToken);
                    /// 컨텍스트에 넣기
                    this.setAuthentication(newAccessToken);
                }
            }
        }
        filterChain.doFilter(request, response);
        }

        // SecurityContext 에 Authentication 객체를 저장합니다.
        public void setAuthentication(String token) {
            // 토큰으로부터 유저 정보를 받아옵니다.
            Authentication authentication = jwtTokenProvider.getAuthentication(token);
            // SecurityContext 에 Authentication 객체를 저장합니다.
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
    }    

    - Access Token이 만료된 후, 요청을 보내면 응답 헤더에 새로운 Access Token이 담겨 올 것이다. 그러면 새로운 토큰을 헤더에 새로 저장후 요청을 진행하면 정상적으로 접근이 가능하다.

## Redis JWT 사용

    보통 session 관리를 위해 Redis 를 사용한다.
    JWT의 RefreshToken 도 관리하기 위해선 저장소에 저장이 필요하다.

    디스크에 저장하여 관리할 수 있으나, 로그인/아웃 등 세션과 비슷하게 작동하고 로그아웃 도니 RefreshToken 에 대해서 좀 다르지만 블랙리스트 방식처럼 관리가 필요하기 때문에 Redis 를 사용해보자. 

    토큰 유효시간과 메모리에 존재하는 시간은 다음과 같이 설정.

        AccessToken = 1m
        RefreshToken = 3m
        memory = 3m

    ● 포인트

      1. 로그인 후, 3분이 지나면 RefreshToken 을 사용할 수 없으며, memory 에서 제거 되고 로그아웃을 해도 memory 에서 제거된다.

      2. RefreshToken 이 유효하더라도 memory 에 존재하지 않으면, AccessToken 을 재발급 받을 수 없다.

    ● 프로젝트 구조

    redis
    |
    |----config
    |----|
    |----|--AppConfig
    |----|--RedisConfig (new)
    |----|--SecurityConfig
    |----controller
    |----|
    |----|--UserController
    |----|
    |----domain
    |----|
    |----|--User
    |----|--UserDTO
    |----|--UserRepository
    |----jwt
    |----|
    |----|--JwtAuthenticationFilter
    |----|--JwtTokenProvider
    |----|
    |----servicer
    |----|
    |----|--CuntomUserDetailService
    |----|--RedisService (new)
    |----|--UserService

### Redis 연결

    ● RedisConfig

    @Configuration
    @EnableRedisRepositories
    public class RedisConfig {

        @Value("${spring.redis.host}")
        private String redisHost;

        @Value("${spring.redis.port}")
        private int redisPort;

        @Bean
        public RedisConnectionFactory redisConnectionFactory() {
            return new LetuceConnectionFactory(redisHost, redisPort);
        }

        @Bean
        public RedisTemplate<?, ?> redisTemplate() {
            RedisTemplate<byte[], byte[]> redisTemplate = new ~<>();

            redisTemplate.setKeySerializer(new StringRedisSerializer());
            redisTemplate.setValueSerializer(new StringRedisSerializer());
            redisTemplate.setConnectionFactory(redisConnectionFactory());
            return redisTemplate; 
        }
    }

    ● RedisService

    @Service
    @RequiredArgsConstructor
    public class RedisService {

        private final RedisTemplate redisTemplate;

        // key, value 설정
        public void setValues(String token, String email) {
            ValueOperations<String, String> values = redisTemplate.opsForValue();
            values.set(token, email, Duration.ofMinute(3)); // 3분 뒤 메모리에서 삭제.
        }

        // key, value 가져오기
        public String getValues(String token) {
            ValueOperations<String, String> values = redisTemplate.opsForValue();
            return values.get(token);
        }

        // key, value 삭제
        public void delValues(String token) {
            redisTemplate.delete(token.substring(7));
        }
    }

    ● UserController

    // TokenProvider 를 통해 save 하던 로직을 RedisService 를 통해 저장하는 방법으로 바꾸었다.

    // 추가로 로그아웃에 대한 메서드를 추가했다.

    @RestController
    @RequiredArgsConstructor
    public class UserController {

        private final JwtTokenProvider jwtTokenProvider;
        private final UserService userService;
        private final RedisService redisService;

        // 회원가입
        @PostMapping("/join")
        public ResponseEntity join(@RequestBody UserDTO user) {
            Integer result = userService.join(user);
            return result != null ?
                    ResponseEntity.ok().body("회원가입을 축하합니다.") :
                    ResponseEntity.badRequest().build();
        }

        // 로그인
        @PostMapping("/login")
        public ResponseEntity login(@RequestBody UserDTO user, HttpServletResponse response) {

            // 유저 존재 확인
            User member = userService.findUser(user);

            // 비밀번호 체크
            userService.checkPassword(member, user);

            // AT/RT token 발급 및 헤더 설정
            String accessToken = jwtTokenProvider.createAccessToken(member.getEmail(), member.getRoles());
            String refreshToken = jwtTokenProvider.createRefreshToken(member.getEmail(), member.getRoles());
            jwtTokenProvider.setHeaderAccessToken(response, accessToken);
            jwtTokenProvider.setHeaderRefreshToken(response, refreshToken);

            // Redis 인메모리에 리프레시 토큰 저장.
            RedisService.setValues(refreshToken, member.getEmail());
        
            // refreshToken 저장소에 저장.
            // tokenRepository.save(new RefershToken(refreshToken));

            return ResponseEntity.ok().body("로그인 성공!");
        }

        // 로그아웃
        @GetMapping("/api/logout")
        public ResponseEntity logout(HttpServletRequest request) {
            redisService.delValues(request.getHeader("refreshToken"));
            return ResponseEntity.ok().body("로그아웃 성공!");
        }

        // JWT 인증 요청 테스트
        @GetMapping("/test")
        public String test(HttpServletRequest request) {
            return "Hello, User?";
        }
    }

## JWT는 한 번 발급되면 만료되기 전 까지 삭제 불가, 해결 방법?

    - 일반적으로 RefreshToken 이라는 추가적인 토큰을 통해 이를 해결한다.
    
    RefreshToken 을 발급하기 전, Token 이 어떻게 사용되는지 알아보자.
    각 토큰의 이름이 뜻하는 대로 AccessToken 은 접근에 관여하는 토큰, 

    JWT 는 발급한 후, 삭재가 불가능하기 때문에 접근에 관여하는 토큰은 유효시간을 길게 부여할 수 없다.
    하지만 자동 로그인 혹은 로그인 유지를 위해서는 유효시간이 긴 토큰이 필요하다.

    이때 사용되는 것이 RefreshToken 이다.

### 생명 주기

    AT 은 발급된 이후, 서버에 저장되지 않고 토큰 자체로 검증을 하며 사용자 권한을 인증한다.

    이런 역할을 하는 AT가 탈취되면 "토큰이 만료되기 전 까지, 토큰을 획득한 사람은 누구나 권한 접근이 가능해진다."

    따라서 AT의 생명주기는 짧게 가져가야 한다.

    ● 그러면 자동 로그인 혹은 로그인 유지는?
    
    이제 RT 의 일이 시작된다.
    여기서 주의점은 AT은 특정 API 인증 및 인가 역할만 해야 하며, RT은 AT 재발급 역할만 하여야 한다.

    ● AccessToken 재발급 방법

    보통 RT는 로그인 성공시 발급되며 저장소에 저장하여 관리된다.

    AT가 만료되어 재발급이 진행되면 아래 과정을 통해 재발급이 된다.
    - RT를 Bearer 타입 토큰으로 헤더에 설정한 뒤 재발급 요청
    - 토큰의 Subject로 RefreshToken 인지 확인한다.
    - RefreshToken 이 저장소에 존재하는지 체크.
    - 유요한 토큰이면 유저 정보를 조회한 뒤 AT 재발급 하여 응답.

    위 과정을 수행하는 Filter 를 Spring Security Filter Chain에 추가.

### 코드 설명

    @Configuration
    @RequiredArgsConstructor
    public class SecurityConfig {

        private final RefreshTokenProvider ~~;

        @Bean
        public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {

            ...

            setTokenRefreshFilter(httpSecurity);

            ...

            return httpSecurity.exceptionHandling(eh -> eh.accessDeniedHandler(new AccessTokenAccessDeniedHandler())).build();
        }

        ...

        private void setTokenRefreshFilter(HttpSecurity httpSecurity) {

            // RT 처리 필터 생성
            SimpleAuthenticationProcessingFilter tokenRefreshAuthenticationFilter = new SimpleAuthenticationProcessingFilter(RequestMatchers.REFRESH_TOKEN, // RT 요청 매칭 엔드포인트 정의            

            new BearerAuthenticationConverter()); // 요청에서 Bearer Token 추출, RT 검증에 사용할 값 제공.

            // RT 검증을 위한 Authentication Manager 설정
            tokenRefreshAuthenticationFilter.setAuthenticationManager(new ProviderManager(refreshTokenProvider));

            // 인증 성공 시, 처리 핸들러
            tokenRefreshAuthenticationFilter.setAuthenticationSuccessHandler(
                new TokenRefreshSuccessHandler(objectMapper, accessTokenProvider));

            // 인증 실패 시, 처리 핸들러
            tokenRefreshAuthenticationFilter.setAuthenticationFailureHandler(new AuthenticationFailureHandlerImpl());

            // 필터를 Security Filter chain 에 등록.
            httpSecurity.addFilterBefore(tokenRefreshAuthenticationFilter, SimpleAuthenticationProcessingFilter.class);

            ...                
        }
    }

    이전 글의 로그인 인증 설정 부분과 동일하게 SimpleAuthenticationProcessFilter를 생성한다.

      - 왜? 위에서 Access Token 재발급 과정을 보면 인증이 성공된 후, 토큰을 발급한 뒤 클라이언트에게 다시 응답을 한다.

    즉, 다음 Filter를 진행할 필요가 없다.

    AuthenticationSuccessHandler에서 새로운 Access Token을 발급한 뒤, 클라이언트에게 넘겨주면 끝이다.

    따라서 AuthenticationFilter 객체가 아닌, SimpleAuthenticationProcessFilter를 생성하여 설정한다.

    ● JWT & RT 사용의 전체 흐름

        1. AT 발급 및 사용
            - 클라이언트가 로그인하면 AT/RT 가 발급이 된다.
            - AT는 클라이언트가 보호된 리소스에 접근할 때 사용된다.

        2. AT 만료
            - AT가 만료되면 클라이언트는 더 이상 보호된 리소스에 접근할 수 없다.
            - 클라이언트는 /refresh 엔트포인트로 RT을 전송해 새로운 AT를 요청.

        3. RT 검증 및 재발급
            - 서버는 클라이언트가 전송한 RT을 RefreshTokenProvider 를 통해 검증.
            - RT가 유효하면 새로운 AT를 발급하고 클라이언트에 반환.

        4. RT 관리
            - 서버는 RT를 Redis와 같은 저장소에 저장하고, 만료 시 삭제한다.     

    ● 재발급 시점

        - AT가 만료된 경우
            - 클라이언트는 보호된 리소스 요청 시, 401 Unauthorized 응답을 받는다.
            - 클라이언트는 RT를 사용하여 새로운 AT를 요청한다.

        - RT가 만료된 경우
            - 클라이언트는 다시 로그인을 해야 된다.               

    ● RefreshTokenProvider

    AT와 다르게 인증 로직이 다르기 때문에 TokenProvider 추상 클래스를 상속 구현한 RT를 사용한다.

    간단하게 RefreshTokenProvider 의 인증 역할 수행 부분을 보면 다음과 같다.
    BearerAuthenticationConverter 가 요청 헤더에서 RT을 꺼내어 Authentication interface 구현체인 BearerAuthenticationToken 을 반환하면서 인증 프로세스가 시작된다.

    AuthenticationManage 는 AuthenticationProvider List 를 순회하면서 support 가 ture 인 AuthenticationProvider 의 authenticate 를 호출.

    따라서 Authentication BearerAuthenticationToken 타입이라면 RefreshTokenProvider 의 authenticate 가 호출되면서 인증이 진행된다.

    1. 토큰이 유효한지 검증한다.
        - TokenProvider 의 verify 를 통해 RT가 유효한지 JWT인지 검증한다.

    2. RT인지 검증한다. (check token type)
        - JWT 가 RT인지 확인한다.

    위 두 검증이 성공하면, 유저 정보를 저장소에서 가져온 뒤, 인증된 Authentication(AccessUser) 객체를 만들어 응답한다.
    TokenRefreshSuccessHandle 를 통해 Authentication(AccessUser)로 새로운 AT를 생성한 뒤 클라이언트에게 응답한다.

### RefreshTokenProvider

    @Component(value = "refreshTokenProvider")
    @RequiredArgsConstructor
    public class RefreshTokenProvider extends TokenProvider {

        private static final String TOKEN_SUBJECT = "refresh token";

        private final RefreshTokenRepository refreshTokenRepository;

        private final UserDetailsService userDetailsService;

        @Override
        public Authentication authentication(Authentication authentication) throws AuthenticationException {

            Claims claims = super.verify(authentication.getName());
            checkTokenType(claims);

            String email = claims.get("email", String.class);
            AuthUserDetails authUserDetails = (AuthUserDetails)userDetailsService.loadUserByUsername(email);

            // RT의 email(payload)로 저장소 존재 여부 && 요청 RT와 match

            boolean empty = refreshTokenRepository.findByEmail(email)
                        .filter(token -> Objects.equals(token, authentication.getName()))
                        .stream()
                        .findAny()
                        .isEmpty();

            if (empty) {
                throw new BadCredentialsException("인증 정보를 확인하세요.");
            }

            return AccessUser.authenticated(authUserDetails);
        }

        @Override
        public boolean supports(Class<?> authentication) {
            return (BearerAuthenticationToken.class.isAssignableFrom(authentication));
        }

        public BearerAuthenticationToken createToken(String email) {
            long tokenLive = 1000L * 60L * 60L; // 1h
            BearerAuthenticationToken token = super.createToken(TOKEN_SUBJECT, Map.of("email", email), tokenLive);
            refreshTokenRepository.save(email, tooken.getName(), tokenList);

            return token;
        }

        @Override
        public void checkTokenType(Claims claims) {
            if (!TOKEN_SUBJECT.equals(claims.getSubject())) {
                throw new BadCredentialsException("인증 정보를 확인하세요.");
            }
        }
    }
