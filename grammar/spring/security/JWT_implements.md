## 본문 

### 1. JWT 구현 - 프로젝트 설정

    이 전에 Security에 Jwt를 도입하여 Authentication & authorization 방식은 인증 아키텍쳐를 따라 어떤 흐름으로 로직이 흘러가는지 알아보았다.
    이번에는 이 전과 조금 다른 방식으로 JWT를 Security에 도입해보자.

    ● 프로젝트 설정

    - Spring boot : 2.7.12 ver
    - dependencies에 추가해주고 Generate
        - Spring Web
        - Spring Security
        - Validation
        - Spring Data Jpa
        - Lombok
        - H2 Database

    ● Controller

    @RestController
    @RequestMapping("/api")
    public class HelloController {
    
        @GetMapping("/hello")
        public ResponseEntity<String> hello() {
            return ResponseEntity.ok("hello");
        }
    }    

![jwt_login_page](/grammer/img/jwt_login_page.png) 

    - Security가 자동으로 로그인 페이지를 띄워준다.
    - 로그에서 인증키를 찾아 ID에 user, Pw에 인증키를 입려하면 성공.

### 2. JWT 구현 - Security 설정, 데이터 삽입

    Spring Security의 기본 인증 방식(세션 기반) 대신 JWT를 이용한 토큰 기반 인증을 구현하기 위해 작성.

    @Configuration
    @EnableWebSecurity  // 1
    public class SecurityConfig {

        @Bean // 2
        public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
            http .authorizeRequests() // 3
                 .antMatches("/api/hello").permitAll() // 4
                 .antMatchers("/api/authenticate").permitAll() // 5
                 .antMatchers("/api/signup").permitAll() // 6
                 .anyRequest().authenticated(); // 7
        
            return http.build();
        }
    }

    1. SecurityConfig 클래스를 @Configuration으로 설정파일로 등록하고, @EnableWebSecurity으로 기본적인 보안 구성을 활성화 해준다.

    2. Security에서 사용하는 필터를 customizing하기 위해 빈으로 등록한다.

    3. 아래 체인으로 연결된 메소드에 한해, 접근 권한을 설정한다.

    4. "/api/hello/" 는 접근 권한 설정 x

    5. "/api/authenticate" 는 접근 권한 설저어 x

    6. "/api/signup" 은 접근 권한 설정 x

    7. 그 외의 요청에 한해서는 Authentication 인증 필요.

    - /api/hello와, 로그인, 회원가입 한해 인증 없이 접근 가능하도록, 그 외의 API에 대해서는 인증을 해야만 접근할 수 있게 설정.

    ● Entity class

    @Entity
    @Getter
    @Setter
    @Builder
    @AllArgsConstructor
    @NoArgsConstructor
    public class Member {
    
        @Id
        @Column(name = "member_id")
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;
    
        private String username;
    
        private String password;
    
        private String nickname;
    
        private boolean activated;
    
        @ManyToMany
        @JoinTable(
                name = "member_authority",
                joinColumns = {@JoinColumn(name = "member_id", referencedColumnName = "member_id")},
                inverseJoinColumns = {@JoinColumn(name = "authority_name", referencedColumnName = "authority_name")})
        private Set<Authority> authorities;
    }

    ● Authority class

    @Entity
    @Table(name = "authority")
    @Getter
    @Setter
    @Builder
    @AllArgsConstructor
    @NoArgsConstructor
    public class Authority {
    
        @Id
        @Column(name = "authority_name", length = 50)
        private String authorityName;
    }

    - Member (*) -> Authority (*) : 다대다 관계 매핑 by JoinTable

    ● DataSource 설정

    server:
      port: 8080
      servlet:
        context-path: /
        encoding:
            charset: UTF-8
            enabled: true
            force: true

    spring:
    datasource:
        driver-class-name: org.h2.Driver
        url: jdbc:h2:tcp://localhost/~/jwtserver
        username: sa
        password:
    
    jpa:
        database-platform: org.hibernate.dialect.H2Dialect
        hibernate:
    #      hbm2ddl-auto: create #create update none
        ddl-auto: create-drop
        properties:
        hibernate:
            show_sql: true

    ● JPA 설정

    insert into member (username, password, nickname, activated) values ('admin', '$2a$08$lDnHPz7eUkSi6ao14Twuau08mzhWrL4kyZGGU5xfiGALO/Vxd5DOi', 'admin', 1);
    insert into member (username, password, nickname, activated) values ('user', '$2a$08$UkVvwpULis18S19S5pZFn.YHPZt3oaqHZnDwqbCW9pft6uFtkXKDC', 'user', 1);  

    insert into authority (authority_name) values ('ROLE_USER');
    insert into authority (authority_name) values ('ROLE_ADMIN');      

    insert into member_authority (member_id, authority_name) values (1, 'ROLE_USER');
    insert into member_authority (member_id, authority_name) values (1, 'ROLE_ADMIN');
    insert into member_authority (member_id, authority_name) values (2, 'ROLE_USER');

    - 현재 설정이 ddl-auto: create-drop으로 되어 있으므로 애플리케이션을 실행할 때마다 데이터를 모두 지우게 되는데, 편의를 위해 데이터를 생성해두고 실행될 때마다 추가할 수 있도록 구현.

![jwt_db](/grammer/img/jwt_db.png)  

### 3. JWT 구현 - TokenProvider 

    이 전에는 UsernamePasswordAuthenticationFilter의 successfulAuthentication() 메소드에서 Token 정보를 지정하고 JWT를 만들어 주었지만, application.yml 파일에 JWT 관련 설정을 추가하여 JWT 생성 정보를 지정해줄 수 있다.

    ● application.yml

    jwt:
        header: Authorization
        secret: zklspefnlskne // secret key 
        token-validty-in-seconds: 86400 // 만료 시간(초)

    ● Dependency (build.gradle)       

    implementation group: 'io.jsonwebtoken', name: 'jjwt-api', version: '0.11.5'
    runtimeOnly group: 'io.jsonwebtoken', name: 'jjwt-impl', version: '0.11.5'
    runtimeOnly group: 'io.jsonwebtoken', name: 'jjwt-jackson', version: '0.11.5'
    
    ● TokenProvider

    토큰의 생성과 검증을 담당하는 클래스를 별도로 만들어 사용할 것이다.

    @Component
    public class TokenProvider implements InitializingBean {

        private final Logger log = LoggerFactory.getLogger(TokenProvider.class);

        private static final String AUTHORITIES_KEY = "auth";
        private final String secret;
        private final long tokenValidityInMilliseconds;
        private Key key;

        public TokenProvider(@Value("${jwt.secret}") String secret,
                             @Value("${jwt.token-validity-in-seconds}") long tokenValidityInseconds) {
            this.secret = secret;
            this.tokenValidityInMilliseconds = tokenValidityInseconds * 1000;
        } 

    - 생성자 및 초기화.
    - yml에 설정해둔 정보를 기반(secretKey, validity)으로 TokenProvider를 생성.

        @Override
        public void afterPropertiesSet() {
            byte[] keyBytes = Decoders.BASE64.decode(secret);
            this.key = Keys.hmacShaKeyFor(keyBytes);
        }
        - InitializingBean interface의 메소드로 BeanFacotry에 의해 모든 property 가 설정되고 난 뒤, 실행되는 메소드이다. 생성자가 실행된 이후 시크릿 키를 Base64로 인코딩 후 HMAC-SHA 알고리즘으로 암호화한다.

        // 토큰 생성 : Authentication 객체의 property를 기반으로 Token을 생성.
        public String createToken(Authentication authentication) {
            // 권한 정보 얻기
            String authorities = authentication.getAuthorities().stream()
                    .map(GrantedAuthority::getAuthority)
                    .collect(Collectors.joining(","));

            // 토큰 만료시간 
            long now = (new Date()).getTime();
            Date validity = new Date(now + this.tokenValidityInMilliseconds); 

            return Jwts.builder()
                .setSubject(authentication.getName())
                .claim(AUTHORITIES_KEY, authorities)
                .signWith(key, SignatureAlgorithm.HS512)
                .setExpiration(validity)
                .compact();       
        }

    ● 동작 방식

    1. Authentication.getAuthorities() 메소드를 통해 UserDetails의 authorities를 반환하여 권한 정보를 받아낸다.
    2. 현재 시간을 받아낸다. (=now)
    3. 권한 정보와 현재 시간을 토대로 Jwt Builder를 통해 토큰 생성.

        // 인증정보 생성 (토큰으로부터 Authentication 객체 추출)
        public Authentication getAuthentication(String token) {
            Claims claims = Jwts
                    .parserBuilder()
                    .setSigningKey(key)
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
    
            Collection<? extends GrantedAuthority> authorities =
                    Arrays.stream(claims.get(AUTHORITIES_KEY).toString().split(","))
                            .map(SimpleGrantedAuthority::new)
                            .collect(Collectors.toList());
    
            // claims를 기반으로 Authentication 객체 생성
            User principal = new User(claims.getSubject(), "", authorities);
    
            // Spring Security는 생성된 Authentication 객체를 SpringContext에 저장.
            return new UsernamePasswordAuthenticationToken(principal, token, authorities);
        }

    - Jwt-Parse를 이용하여 입력받은 token을 parsing하여 claims(payload + signature + header)라는 객체를 얻을 수 있따.
    - 이 claims 객체를 통해 권한정보를 추출한 후, 권한정보와 claims의 subject를 토대로 User를 하나 생성하여 UsernamePasswordAuthenticationToken로 만들어 리턴한다.
      
    ● 구성 

    claims : (payload, signature, header)
    UsernamePasswordAuthenticationToken : (principal, token, authorities)
    authority : 역할(Role), 권한(Permission)
    principal : 인증된 사용자의 정보 (=UserDetails 객체 id, pw, activities..)

        // 토큰 검증 
        public boolean validateToken(String token) {
            try {
                Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token);
                return true;
            } catch (io.jsonwebtoken.security.SecurityException | MalformedJwtException e) {
                logger.info("잘못된 JWT 서명입니다.");
            } catch (ExpiredJwtException e) {
                logger.info("만료된 JWT 토큰입니다.");
            } catch (UnsupportedJwtException e) {
                logger.info("지원되지 않는 JWT 토큰입니다.");
            } catch (IllegalArgumentException e) {
                logger.info("JWT 토큰이 잘못되었습니다.");
            }
            return false;
        }
    }

### 4. JWT 구현 - Filter & Handler

    이번에는 인증 시점에 자동으로 호출될 수 있는 JWT 필터를 구현해보고, JWT 필터를 통과하지 못한 경우 (권한 미달, 토큰 Validate 실패) 처리하는 JWT 핸들러를 구현해봅시다.

    ● JWT Filter

    @Slf4j
    public class JwtFilter extends GenericFilterBean {

        public static final String AUTHORIZATION_HEADER = "Authorization";

        private TokenProvider tokenProvider;

        public JwtFilter(TokenProvider tokenProvider) {
            this.tokenProvider = tokenProvider;
        }

        // 토큰의 인증정보 SecurityContext에 저장.
        @Override
        public void doFilter(ServletRequest request, 
                             ServletResponse response, 
                             FilterChain filterChain) 
                             throw IOException, ServletException {

            HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequeset;

            // request를 이용해서 jwt 추출
            String jwt = resolveToken(httpServletRequest);
            String requestURI = httpServletRequest.getRequestURI();

            if (StringUtils.hasText(jwt) && tokenProvider.validateToken(jwt)) {

                // jwt가 유효하다면 authentication 객체 생성.
                Authentication authentication = tokenProvider.getAuthentication(jwt);
                SecurityContextHolder.getContext().setAuthentication(authentication);
                log.debug("Security Context에 '{}' 인증 정보를 저장했습니다, uri: {}", authentication.getName(), requestURI);
            } else {
                log.debug("유요한 JWT 토큰이 없습니다, uri: {}", requestURI);
            }

            // 다음 필터로 요청 전달.
            filterChain.doFilter(servletRequest, serlvetResponse);
        }

        private String resolveToken(HttpServletRequest request) {
            String bearerToken = request.getHeader(AUTHORIZATION_HEADER);
    
            if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
                return bearerToken.substring(7);
            }
    
            return null;
        }
    }

    ● GenericFilterBean

    - Spring Security와 함께 많이 사용되며, 사용자 인증, 권한 부여, 로깅 등과 같은 공통된 작업을 처리하는 데 유용하게 사용된다.

    ● UsernamePasswordAuthenticationFilter
    
    - SpringSecurity에서 사용자 인증에 특화된 목적으로 사용하는 반면, GenericFilterBean은 Spring Framework에서 일반적인 필터닐 작업을 수행하고자 할 떼, 사용된다. GenericFilterBean은 다양한 필터링 작업을 구현할 수 있는 유연성을 제공하며, Spring의 다른 기능과 통합하여 사용할 수 있다.

    ● jwt = resolveToken(httpServletRequest);

    -resolveToken 메소드를 통해 request로부터 Authorization 헤더 정보를 꺼내와 베리어를 해제하고 리턴한다.

    ● authentication = tokenProvider.getAuthentication(jwt);

    - 요청 정보에 토큰이 있고, 이 전 포스팅에서 생성한 TokenProvider로 토큰 검증에 통과하면 토큰으로부터 Authentication 객체를 만든다.

    ● SecurityContextHolder.getContext().setAuthentication(authentication);

    - Authenticatoin 객체를 Security Context에 담는다.

    ● filterChain.doFilter(servletRequest, servletResponse);

    - 다음 필터로 넘어간다.

    ● doFilter

    - JwtFilter에서 요청 인터셉트
    - 요청 헤더 정보에서 토큰을 꺼내와 TokenProvider로 토큰 검증
    - 검증 통과 시, Authentication 객체 생성
    - Session 영역의 SecurityContext에서 Authentication 객체를 담고 필터 통과.

### 5. JWT 구현 - Authentication & Authority 실패 시, 예외처리

    ● AuthenticationEntryPoint : 인증 실패 처리 핸들러

    @Component
    public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {

        @Override
        public void commence(HttpServletRequest request,
                             HttpServleResponse response,
                             AuthenticationException authException) throws IOException {

            // 유요한 자격증명을 제공하지 않고 접근하려 할때, 401
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED);
        }
    }

    - 인증에 실패하였을 경우 401 응답을 리턴할 수 있도록 EntryPoint를 구현.

    ● AccessDeniedHandler : 권한 미달 처리 핸들러

    @Component
    public class JwtAccessDeniedHandler implements AccessDeniedHandler {

        @Override
        public void handle(HttpServletRequest request, HttpServletResposne response, AccessDeniedException accessDeniedException) throws IOException {
            // 필요한 권한이 없이 접근하려 할 때 403
            response.sendError(HttServletResponse.SC_FORBIDDEN);
        }
    }

    ● JwtFilter -> JwtSecurityConfig

    public class JwtSecurityConfig extends SecurityConfigureAdapter<DefaultSecurityFilterChain, HttpSecurity> 
    {
        private TokenProvider tokenProvider;

        public JwtSecurityConfig(TokenProvider tokenProvider) {
            this.tokenProvider = tokenProvider;
        }

        @Override
        public void config(HttpSecurity http) {
            http.addFilterBefore(
                new JwtFilter(tokenProvider),
                UsernamePasswordAuthenticationFilter.class
            );
        }
    }

    - 기존에 로그인 시점에서 호출되던 UsernamePasswordAuthenticationFilter를 JwtFilter로 Override 해준다.

    - addFilterBefore() : 스프링 시큐리티 필터링에 등록해주어야 하기 때문에 addFilterBefore에 등록해준다.

    ● JwtSecurityConfig & AuthenticationEndPoint & AccessDeniedHandler -> SecurityConfig 등록

    @Configuration
    @EnableWebSecurity
    @EnableMethodSecurity
    @RequiredArgsConstructor
    public class SecurityConfig {
    
        private final TokenProvider tokenProvider;
        private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
        private final JwtAccessDeniedHandler jwtAccessDeniedHandler;

        @Bean
        public BCryptPasswordEncoder bCryptPasswordEncoder() {
            return new BCrtyptPasswordEncoder();
        }

        @Bean
        public SecurityFilterChain filterChain(HttpSecurity http) throws Exception 
        {
            http. 
                // 토큰 사용하는 방식이므로 csrf disable
                csrf().disable()

                .exceptionHandling() // 예외 처리 핸들러 등록
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                .accessDeniedHandler(jwtAccessDeniedHandler)

                .and()
                .headers()
                .frameOptions()
                .sameOrigin()

                // 세션 사용하지 않기 위해 stateless로 설정
                .and()

            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)

                .and()
                .authorizeRequests()
                .antMatchers("/api/hello").permitAll()
                .antMatchers("/api/authenticate").permitAll() // 로그인
                .antMatchers("/api/signup").permitAll() // 회원가입
                .anyRequest().authenticated()
 
                .and()
                .apply(new JwtSecurityConfig(tokenProvider)); // JWT 필터 등록
 
            return http.build();
        }
    }

### 6. JWT 구현 - TEST    

    ● Oveview

    1. 외부와의 통신에 사용할 DTO 클래스 생성
    2. Repository 관련 코드 생성
    3. 로그인 api, 관련 로직 생성

    ● Login Dto 

    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    @Builder
    public class LoginDto {

        @NotNull
        @Size(min = 3, max = 50)
        private String username;

        @NotNull
        @Size(min = 3, max = 100)
        private String password;
    }

    ● Member Dto

    @Data
    @Builder
    @AllArgsConstructor
    @NoArgsConstructor
    public class MemberDto {
    
        @NotNull
        @Size(min = 3, max = 50)
        private String username;
    
        @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
        @NotNull
        @Size(min = 3, max = 100)
        private String password;
    
        @NotNull
        @Size(min = 3, max = 50)
        private String nickname;
    }

    ● Token Dto

    @Data
    @Builder
    @AllArgsConstructor
    @NoArgsConstructor
    public class TokenDto {
        private String token;
    }

    ● Repository (Spring Data JPA)

    public interface MemberRepository extends JpaRepository<Member, Long>
    {
        @EntityGraph(attributePaths = "authorities")
        Optional<Member> findOneWithAuthoritiesByUsername(String username); 
    }

    - @EntityGraph로 다대다 연관 테이블도 함께 fetch join하여 조회하여 Username으로 Authority까지 함께 조회

![entityGraph_selectDB](/grammer/img/entityGraph_selectDB.png)

### 7. JWT 구현 - 로그인 API 생성

    @RestController
    @RequestMapping("/api")
    @RequiredArgsConstructor
    public class AuthController {

        private final TokenProvider tokenProvider;
        private final AuthenticationManagerBuilder authenticationManagerBuilder;

        @PostMapping("/authenticate")
        public ResponseEnity<TokenDto> authorities(@Valid @RequestBody LoginDto loginDto)
        {
            // 1) usernamePasswordToken 생성
            UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken 
            {
                (loginDto.getUsername(), loginDto.getPassword());
            }

            // 2) 토큰기반 authenticate() -> loadUserbyUsername()
            Authentication authentication = authenticationManagerBuilder.getObject().authenticate(authenticationToken);

            // 3) Token 기반 권한
            SecurityContextHolder.getContext().setAuthentication(authenticationToken);

            // 4) JWT 생성
            String jwt = tokenProvider.createToken(authentication);

            // 5) 응답 헤더에 jwt 추가
            return ResponseEntity.ok()
                    .header(HttpHeaders.AUTHENTICATION, "Bearer" + jwt)
                    .body(TokenDto.builder().token(jwt).build());
        }
    }

    - "/api/authenticate"의 요청 바디에 LoginDto 정보를 넣고 Post 전송시, authorize 메서드가 호출된다.

    - login 요청시,

        1. header 정보를 토대로 UsernamePasswordAuthenticationToken을 생성.

        2. 생성한 토큰을 담아 AuthenticationManager에서 authenticate() 호출
              
            - AuthenticationProvider에서 등록한 DetailsService를 호출하여 loadUserByUsername 메소드를 실행 -> DB에서 정보 확인 후, Authentication 객체에 UserDetails를 담아 리턴한다.

        3. 리턴된 Authentication 객체를 SecurityContext에 담는다. (for 인가)            

        4. Authentication 정보를 기반으로 JWT를 생성한다.

        5. 응답 헤더에 JWT를 담아 클라이언트에 리턴한다.
     
    ● UserDetailsService

    위 API에서 Spring Security의 인증 방식을 동작시키기 위해 필요한 UserDetailsService의 구현체를 작성해보자.

    @Service
    @RequiredArgsConstructor
    public class MemberDetailsService implements UserDetailsService {
        
        private final MemberRepository memberRepository;

        @Override
        public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

            Optional<Member> member = Optional.ofNullable(memberRepository.findOneWithAuthoritiesByUsername(username).orElseThrow(() -> new UsernameNotFoundException(username + " -> 데이터베이스에서 찾을 수 없습니다.")));

            return createUser(username, member.get());
        }

        private User createUser(String username, Member member) {

            if (!member.isActivated()) {
                throw new RuntimeException(username + " -> non activateed!");
            }

            List<SimpleGrantedAuthority> grantedAuthorities = member.getAuthorities().stream()
                            .map(authority -> new SimpleGrantedAuthority(authority.getAuthorityName()))
                            .collect(toList());

            return new User(username, member.getPassword(), gratedAuthorities);                           
        }
    }    

![authenticate_postman](/grammer/img/authenticate_postman.png)    

    - 사전에 data.sql에 등록한 admin

![selectToken_postman](/grammer/img/selectToken_postman.png)

### 8. JWT 구현 - 회원가입 & Authorization validation

    회원 가입을 통해 데이터에비으 회원을 저장하고 로그인을 통해 JWT를 발급받아, 제한된 리소스에 접근 시 발급받은 JWT를 검증하는 로직을 개발해보자.

    ● MemberService

    @Slf4j
    @Service
    @RequiredArgsConstructor
    @Transactional(readOnly = true)
    public class MemberService {

        private final MemberRepository memberRepository;
        private final PasswordEncoder passwordEncoder;

        @Transactional
        public MemberDto signup(MemberDto memberDto) {

            Member memberInDb = memberRepository.findOneWithAuthoritiesByUsername(memberDto.getUsername()).orElse(null);

            if (memberInDb != null) {
                throw new RuntimeException("이미 가입되어 있는 유저입니다.");
            }

            Member member = Member.create(memberDto, passwordEncoder.encode(memberDto.getPassword()));
            
            return MemberDto.toDto(memberRepository.save(member));
        }

        public MemberDto getMemberWithAuthorities(String username) {
            return MemberDto.toDto
            (
                memberRepository.findOneWithAuthoritiesByUsername(username.orElse(null));
            )
        }

        // SecurityContext 내부 Authentication 객체의 username
        public MemberDto me() {
            reutrn MemberDto.toDto(SecurityUtil.getCurrentUsername().flatMap(memberRepository::findOneWithAuthoriteisByUsername).orElse(null));
        }
    }

    ● signup() 

    1. MemberDto를 인자로 받는다.

    2. findOneWithAuthorityByUsername() : JPA 쿼리 메소드,

       - memberDto의 username으로 Repository에서 member를 찾고, member의 authority까지 @EntityGraph로 함께 탐색해서 memberInDb에 리턴. 

    3. 만약 해당 멤버를 찾았다면 이미 가입되어 있는 유저이므로 RuntimeException을 던지고,

        public static Member create(MemberDto memberDto, String encodedPw) {
            return Member.builder()
                    .username(memberDto.getUsername())
                    .password(encodedPw)
                    .nickname(memberDto.getNickname())
                    .activated(true)
                    .authorities(Collections.singleton(Authority.builder().authorityName("ROLE_USER").build()))
                    .build();
        }

    4. 그렇지 않으면, Member의 static 메소드의 builder로 회원을 하나 만들어 Repository에 저장한다. 이 때, Password는 반드시 Encoding하여 저장.

    5. 저장한 Member -> MemberDto로 변환, 리턴.

    ● getMemberWithAuthoritiesForAdmin()

        public MemberDto getMemberWithAuthoritiesForAdmin(String username) {
            return MemberDto.toDto(
                    memberRepository.findOneWithAuthoritiesByUsername(username).orElse(null));
        }

    - 일반적으로 username으로 DB에서 member를 찾고 authority까지 함께 검색하는 메소드이다.
    (MemberController에서 admin권한의 사용자만 사용할 수 있도록 설정)

    ● getMemberWithAuthoritiesForUser()

        // SecurityContext 내부 Authentication 객체의 usrename
        public MemberDto me() {
            return MemberDto.toDto(SecurityUtil.getCurrentUsername()
                    .flatMap(memberRepository::findOneWithAuthoritiesByUsername).orElse(null));
        }

    ● getCurrentUsername()

        public static Optional<String> getCurrentUsername() 
        {
            Authentication authentication = SecurityContextHoldeer.getContext().getAuthentication();

            if (authentication == null) {
                log.info("no authentication info found");
                return Optional.empty();
            }

            Object principal = authentication.getPrincipal();

            if (principal instanceof UserDetails) {
                UserDetails userDetails = (UserDetails) principal;
                return Optional.ofNullable(userDetails.getUsername());
            }

            if (principal instanceof String) {
                return Optional.of(principal.toString());
            }
            throw new IllegalStateException("invalid authentication");
        } 

        - 클라이언트가 자신의 정보를 볼 수 있돌고 검색하는 메소드이다.
        - SecurityContext에서 Authentication 객체를 꺼내어 UserDetails의 Username을 꺼내 리턴.
        (MemberController에서 admin, user 권한의 사용자만 사용할 수 있도록 설정할 예정)

    ● MemberController

    @RestController
    @RequestMapping("/api")
    @RequiredArgsConstructor
    public class MemberController {

        private final MemberService memberService;

        // 회원가입 요청
        @PostMapping(value = "/signup")
        public ResponseEntity<MemberDto> signup(@Valid @RequestBody MemberDto memberDto) {
            return ResponseEntity.ok(memberService.signup(memberDto));
        }

        // 본인 정보 조회
        @GetMapping(value = "/member", produces = MediaType.APPLICATION_JSON_VALUE)
        @PreAuthorities("hasAnyRole('USER', 'ADMIN')")
        public ResponseEntity<MemberDto> findMyInfoForUserAdmin() {
            return ResponseEnity.ok(memberService.getMemberWithAuthoritiesForUser());
        }

        // username으로 회원 정보 조회
        @GetMapping(value = "/member/{username}", produces = MediaType.APPLICATION_JSON_VALUE)
        @PreAuthorities("hasRole('ADMIN')")
        @ResponseBody
        public ResponseEntity<MemberDto> findInfoForAdmin(@PathVariable String username) {
            return ResponseEntity.ok(memberService.getMemberWithAuthoritiesForAdmin(username));
        }
    }

![signup_postman](/grammer/img/signup_postman.png)

![login_postman](/grammer/img/login_postman.png)

![authenticate_valid_postman](/grammer/img/authenticate_valid_postman.png)

![authenticate_forbid_postman](/grammer/img/authenticate_forbid_postman.png)

### 9. JWT 구현 - 가상 데이터를 이용한 시나리오

    1. 사용자 정보

       - 사용자가 로그인 정보를 입력한다.

        username : "JOO"
        password : "1234"
        authorities : ["ROLE_USER", "ROLE_ADMIN"]

    2. 로그인 과정

        A. 사용자가 로그인 요청
            - 사용자가 username, password 입력 후, 서버에 요청

        POST /login
        Body : {"username" : "JOO", "password" : "1234"}

        B. 서버에서 인증 처리

        1) AuthenticationManager가 username & password 검증
           - 성공하면, Authentication 객체 생성
            Authentication authentication = new UsernamePasswordAuthenticationToken(principal, null, authorities);

        2)  TokenProvider가 인증된 정보를 바탕으로 JWT 생성
            - authentication.getName() -> JOO (토큰의 subject).
            - authentication.getAuthorities() -> [ROLE_USER, ROLE_ADMIN] (claims의 auth).
            - 유효 기간은 tokenValidityInMilliseconds에 설정된 값.
    
            ● JWT 생성 코드

            String token = Jwts.builder()
                .setSubject("JOO")
                .claim("auth", "ROLE_USER, ROLE_ADMIN")
                .setExpiration(new Date(System.currentTimeMillis() + 3600000)) // 1시간
                .signWith(key, SignatureAlogrithm.HS512)
                .compact();

            ● 생성된 토크

            eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJqb2huLmRvZSIsImF1dGgiOiJST0xFX1VTRVIsUk9MRV9BRE1JTiIsImV4cCI6MTYzMjQ2ODAwMH0
            .T9CsdvR-3f9k_93nUuHdXZc9T8o

        3)  서버는 클라이언트에 토큰을 반환.

            Response: { "token": "eyJhbGcioi..." }

    3.  인가 과정 (클라이언트 요청)

        A. 클라이언트 요청
            - 클라이언트는 JWT를 Authorization 헤더에 담아 요청
            
            GET /profile
            Authorization: Bearer eyJhbGcioi...

        B. 서버에서 토큰 해독
            - TokenProvider.getAuthentication(token) 호출:

            Claims claim = Jwt.parserBuilder()
                .setSigningKey(key)   
                .build()
                .parseClaimsJws(token)
                .getBody();

            - 추출된 claims 값
                subject : "JOO"    
                auth : "ROLE_USER, ROLE_ADMIN"

            - Claims를 기반으로 Authentication 객체 생성

                User principal = new User("JOO", "", authorities);
                Authentication auth = new UsernamePasswordAuthenticationToken(principal, token, authorities);

        C. 요청 처리

            - Spring Security는 생성된 Authentication 객체를 SecurityContext에 저장.
            - 이후 해당 사용자의 권한 (ROLE_USER, ROLE_ADMIN)을 기반으로 요청을 인가.

    ● 요약

        1. 로그인
            사용자 -> 서버: ID, PW 제출
            서버 -> 사용자: JWT 발급

        2. 요청
            사용자 -> 서버: JWT 포함 요청
            서버: 토큰 해독, 사용자 정보 확인
            서버: 권한 검사 및 요청 처리                

    ● 용어 정리

        Principal : 사용자 정보 (username, ID 등)
        Credentials : 인증 정보 (PW), 인증 후 null 처리.
        Authorities : 권한 정보 (ROLE_USER, ROLE_ADMIN).
        Token(JWT) : 인증 및 인가에 사용되는 토큰.
        Claims : JWT Payload의 데이터 (subject, auth).
        Header : 토큰 타입과 알고리즘 정보.
        Payload : JWT의 사용자 정보와 메타데이터.
        Signature : 토큰 위변조 방지.