## 본문

## Kakao OAuth2 + JWT + Redis를 통한 인증 과정 구현 (3) - Redis 적용

    로그인 성공 시, AccessToken & RefreshToken 을 클라이언트에게 Json 으로 응답하도록 구현하고, 추가로 JWT 필터를 생성하여 AccessToken 을 검증하도록 구현.

    JwtFilter 에서 AccessToken 이 유효하지 않은 경우에는 헨들러를 사용하여 로그인 요청 메시지를 클라이언트에게 응답으로 내려준다.

    유효성 검사와 별개로 AccessToken 의 기한이 만료된 경우에 RefreshToken 을 통해 AccessToken 을 재발급 받는 로직을 구현해 보자.

    AccessToken 을 재발급하는 방법은 크게 2가지가 있다.

    1. 요청마다 AccessToken & RefreshToken 을 넘기고 AccessToken 이 만료된 경우 RefreshToken 을 검증하여 재발급 받기

    2. 재발급 API를 만들고 서버에서 AccessToken 이 만료되었다고, 응답하면 RefreshToken 으로 요청하여 재발급 받기.

    이번에는 1번 방법으로 진행하겠다.

    우선 RefreshToken 이 유효한 지 검증하기 위해서는 로그인 시 발급받은 RefreshToken 을 어딘가에 저장해두어야 하는데, 속도가 빠르고 별도의 쿼리문이 필요하지 않은 Redis 를 사용.

### 0. 설정

    redis:
        host: localhost
        port: 6379(default)

    build.gradle
        implementation 'org.springframework.boot:spring-boot-starter-data-redis'

    RedisConfig

    @EnableRedisRepositories
    @Configuration
    public class RedisConfig {

        @value("${spring.datasource.redis.host}")
        private String redisHost;

        @value("${spring.datasource.redis.port}")
        private String redisPort;

        @Bean
        public RedisConnectionFactory redisConnectionFactory() {
            return new LettuceConnectionFactory(redisHost, redisPort);
        }
    }

    - @EnableRedisRepositories : Redis 를 사용.
    - @Bean 으로 yml 에 등록한 host, port 를 사용해 빈 등록.

### RefreshToken Entity (Redis 에 등록할)

    @Builder
    @Getter
    @AllArgsConstructor
    @NoArgsConstructor
    @RedisHash(value = "refresh", timeToLive = 604800)
    public class RefreshToken {

        private String id;
        private Collection<? extends GrantedAuthority> authorities;

        @Indexed
        private String refreshToken;

        public String refreshToken;

        public String getAuthority() 
        {
            return authorities.stream()
                    .map(authority -> new SimpleGrantedAuthority(authority.getAuthority()))
                    .collect(Collectors.toList())
                    .get(0)
                    .getAuthority();
        }
    }

    - @RedisHash : Spring Data Redis 프레임워크에서 사용되는 어노테이션으로 Entity class 를 Redis 해서 형식의 데이터로 매핑한다.

        - 이 어노테이션을 명시하면 클래스의 인스턴스가 redis 해시로 매핑되고,
        Java 객체와 Redis DB간 데이터를 변환하고 저장할 수 있다.

    - @Indexed : 필드에 붙이게 되면 필드 값으로도 데이터를 빠르게 검색할 수 있다.

    ● RefreshTokenRepository

    public interface RefreshTokenRedisRepository extends CrudRepository<RefreshToken, Long> {

        RefreshToken findByRefreshToken(String refreshToken);
    }

    - JPA 처럼 Redis 와의 데이터 엑세스를 담당하는 RedisRepository 는 CrudRepository 를 상속받아 기본적인 메소드들을 사용할 수 있다.
    RefreshToken Entity 에는 PK 가 따로 지정되어 있지 않기 때문에 Long 으로 지정.

### onAuthenticationSuccess Refactoring    

    기존 로그인 성공 헨들러 리펙토링에서 RefreshTokenRedisRepository 의 CRUD 속성 중 하나인 SaveRefreshTokenOnRedis 메서드를 추가해준다.

    private void saveRefreshTokenOnRedis(Member member, TokenDto tokenDto) {

        List<SimpleGrantedAuthority> simpleGrantedAuthorities = new ArrayList<>();

        SimpleGrantedAuthorities.add(new SimpleGrantedAuthority(member.getRole().name()));

        refreshTokenRedisRepository.save(RefreshToken.builder()
                .id(member.getEmail())
                .authorities(simpleGrantedAuthorities)
                .refreshToken(tokenDto.getRefreshToken())
                .build());
    }

    - 로그인 성공 시, 발급받은 refreshToken - 회원 이메일을 refreshToken 리포지토리에 저장하도록 수정.

### TokenProvider 에 at/rt 재발급 메소드 추가.

    @Transactional
    public TokenDto reIssueAccessToken(String refreshToken)
    {
        RefreshToken findToken = refreshTokenRedisRepository.findByRefreshToken(refreshToken);

        TokenDto tokenDto = createToken(findToken.getId(), findToken.getAuthority());
        refreshTokenRedisRepository.save(RefreshToken.builder()
                .id(findToken.getId())
                .authorities(findToken.getAuthorities())
                .refreshToken(tokenDto.getRefreshToken())
                .build());
    
        return tokenDto;
    }

    - refreshToken 값을 받아서 refreshToken 이 유효하고 만료되지 않은 경우, redis에서 일치하는 RefreshToken 을 찾아 새로 accessToken & refreshToken 을 발급받고 Redis에도 새로 발급받은 refreshToken 을 업데이트해준다.

### JwtFitler 수정 - AccessToken 값이 유효, 기간 만료

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throw ..
    {

        if (!validateExpire(accessToken) && validate(accessToken)) { 
            String refreshToken = getTokenFromHeader(request, REFRESH_HEADER);
            if (validate(refreshToken) && validateExpire(refreshToken)) {
                // accessToken, refreshToken 재발급
                TokenDto tokenDto = tokenProvider.reIssuedAccessToken(refreshToken);
                SecurityContextHolder.getContext().setAuthentciation(tokenProvider.getAuthentication(tokenDto.getAccessToken()));

                redirectReissueURI(request, response, tokenDto);
            }
        }    

        filterChain.doFilter(request, response);
    }
    
    - JwtFilter 에서 AccessToken 유효하지만 만료된 경우, 요청 헤더의 refreshToken 값을 받아 값이 유효하고 만료되지 않았는지 검사하고 tokenProvider 에게 재발급을 위임한다.

        - refreshToken이 유효하지 않거나 만료된 경우, 또는 accessToken이 유효하지 않거나 만료된 경우에는 doFilter로 타고 들어가 JwtAccessDeined 핸들러에서 에러 메시지로 응답하도록 동작합니다.

        ● redirectReissueURI method

        private static void redirectReissueURI(HttpServletRequest request, HttpServletResponse response, TokenDto tokenDto)
                throws IOException {
            HttpSession session = request.getSession();
            session.setAttribute("accessToken", tokenDto.getAccessToken());
            session.setAttribute("refreshToken", tokenDto.getRefreshToken());
            response.sendRedirect("/api/sign/reissue");
        }

        - 재발급을 통해 토큰을 받아 Authentication 객체를 만들어 SecurityContext 에 보관하고, 발급받은 토큰들은 session에 저장하여 클라이언트에게 redirect 한다.

### SignController

    @RestController
    @RequiredArgsConstructor
    @RequestMapping("/api/sign")
    public class SignController {

        /**
         * 엑세스 토큰 재발급 API
         */
        @GetMapping("/reissue") 
        public ResponseEntity reissueToken(HttpServletRequest request)
        {
            HttpSession session = request.getSession();
            String accessToken = (String) session.getAttribute("accessToken");
            String refreshToken = (String) session.getAttribute("refreshToken");
 
            return new ResponseEntity(new TokenDto(accessToken, refreshToken), HttpStatus.OK);
        }
    }

    - redirect 된 URI는 해당 컨트롤러와 매핑되어 클라이언트는 Json 형태로 새로 발급받은 AccessToken, RefreshToken 의 정보를 알 수 있다.
    클라이언트는 이제 이 값들을 헤더에 설정해서 서버에 요청하면 된다.

### AccessToken 유효성 상태에 따른 코드 추가 

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        // 유효성 검사가 필요 없는 요청은 필터를 통과
        if (isRequestPassURI(request, response, filterChain)) {
            return;
        }

        // 토큰 가져오기
        String accessToken = getTokenFromHeader(request, ACCESS_HEADER);
        String refreshToken = getTokenFromHeader(request, REFRESH_HEADER);

        // 1. 액세스 토큰 유효 && 리프레시 토큰 유효
        if (validateExpire(accessToken) && validate(accessToken)) {

            SecurityContextHolder.getContext()
                    .setAuthentication(tokenProvider.getAuthentication(accessToken));
        }

        // 2. 액세스 토큰 만료 && 리프레시 토큰 유효
        else if (!validateExpire(accessToken) && validate(accessToken)) {

            String refreshToken = getTokenFromHeader(request, REFRESH_HEADER);

            if (validate(refreshToken) && validateExpire(refreshToken)) {
                TokenDto tokenDto = tokenProvider.reIssueAccessToken(refreshToken);
                SecurityContextHolder.getContext()
                        .setAuthentication(tokenProvider.getAuthentication(tokenDto.getAccessToken()));
                redirectReissueURI(request, response, tokenDto);
                return;
            }
        }

        // 4. 액세스 토큰 만료 && 리프레시 토큰 만료
        else {
            log.warn("Both access token and refresh token are expired or invalid.");
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Authentication required. Both tokens are invalid.");
            return;
        }

        // 다른 필터로 요청 전달
        filterChain.doFilter(request, response);
    }

    validate: 토큰의 기본 구조나 서명이 유효한지 검사.
    validateExpire: 토큰의 만료 시점(exp)이 유효한지 검사.
    getTokenFromHeader: HTTP 헤더에서 토큰 값을 추출.
    redirectReissueURI: 새로 발급된 토큰 정보를 포함해 클라이언트를 특정 URI로 리다이렉트.

###  KakaoLogin 최종 흐름 요약

    1. 사용자가 kakaoLogin 버튼 클릭 시,

        - 클라이언트는 카카오 인증 서버로 redirect
        - kakao OAuth2 인증 서버
        - 카카오 서버로 직접 이동

    2. 카카오 인증 완료 후, redirect callback

        - 카카오 인증 서버가 클라이언트로 Authentication Code 를 전달.
        - 클라이언트는 Authentication Code 를 서버의 kakaoLoginController 로 전송.
        - 카카오 서버 -> 클라이언트 -> 백엔드

    3. AccessToken 발급 및 사용자 정보 요청

        - 백엔드가 Authorization Code 를 사용해 카카오 서버에서 AccessToken 요청.
        - 카카오 서버가 AccessToken 과 사용자 정보(kakaoUserInfo) 반환.
        - 백엔드 -> kakao OAuth2 서버
        - kakaoOAuth2Service.getKakaoUserInfo(accessToken)/

    4. JWT token 생성.

        - 백엔드에서 사용자의 카카오 정보를 기반으로 JWT AccessToken & RefreshToken 을 생성
        - RefreshToken 은 Redis 에 저장, AccessToken 은 클라이언트로 응답.
        - userInfo -> JWT 생성 -> RefreshToken 저장 (Redis).
        - JWT AccessToken -> 클라이언트
        - kakaoTokenProvider.createToken(userInfo)
        - RedisTemplate.save(refreshToken)

    5. 클라이언트 요청에 JWT 토큰 사용

       - 클라이언트는 이후 요청에 JWT AccessToken 을 Authorization Header 에 포함.
       - 백엔드의 JwtFilter 가 AccessToken 을 검증하고, 필요 시, RefreshToken 을 통해 토큰 재발급
       - 클라이언트 -> 벡엔드 -> Redis(RefreshToken 확인)        
       - JwtFilter.doFilterInternal(requet, response, filterChain)
       - tokenProvider.validate(token) & validateExpire(token)

    6. Redis 만료 및 갱신 처리

        - RefreshToken 의 만료 기간이 Redis 에서 관리
        - 만료된 RefreshToken 이 확인되면, 클라이언트에게 재로그인 요청.
        - 백엔드 -> Redis -> 클라이언트
        - RedisTemplate.delete(refreshToken) (만료 시)      
        - TokenProvider.reIssueAccessToken(refreshToken) (재발급 시)

    Final

    1. 사용자 클릭 -> 카카오 서버로 이동
    2. 인증 완료 후 -> 서버에서 Authorization Code 처리.
    3. AccessToken & 사용자 정보 -> 카카오에서 응답.
    4. JWT 생성 -> AccessToken(클라이언트) & RefreshToken(Redis)    
    5. 이후 요청 -> JWT 검증 (AccessToken), 필요 시 재발급 (RefreshToken)