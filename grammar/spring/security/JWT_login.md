## part1 SpringBoot JWT login - Filter에 대한 이해

### 역할 축약

    ● Authentication  
    
    사용자의 인증정보[id/password]를 저장하는 token을 가짐.(인증 후 최종 결과는 User객체/권한정보)담고 SecurityContextHolder에 저장.

        1. usernamePasswordAuthenticationFilter 가 AuthenticationManager 에게 Authentication 객체를 전달

        2. AuthenticationManager 는 인증의 전반적인 관리를 하고, 인증을 처리할 수 있는 AuthenticationProvider 를 찾는다.

        3. AuthenticationProvider 가 UserDetailsService(loadUserByUsername()) 를 통해 Service 계층에서 Repository 계층에 접근하여 id에 해당하는  회원 객체를 Authentication 객체에 저장하여 권한 정보와 저장한다.

        4. Authentication 객체를 AuthenticationManager 에 전달 후 다시,
        UsernamePasswordAuthenticationFilter 에 전달 후 SecurityContext 에 저장한다.

    ● AuthenticationProvider   

    실제 인증에 대한 부분을 처리, 인증 전의 Authentication 객체를 받아서, 인증이 완료된 객체를 반환하는 역할을 한다.

    ● AuthenticationManager 

    인자로 Authentication 객체를 받는다.
    Authentication 객체에 id, pw를 저장 후, AuthenticationManager에 보내 인증 과정을 거친다. (구현체는 ProviderManager)    

    ● SecurityContext

    Authentication 객체가 저장되는 보관소, 인증이 완료되면 HttpSession 에 저장되어 어플리케이션 전반에 걸쳐 전역적인 참조가 가능.

    ● SecurityContextHolder

    SecurityContext를 저장하는 저장소
    (SecurityContextHolder - SecurityContext - Authentication - User)
    위의 순서대로 앞의 객체가 뒤의 객체를 보관한다.

    ● TokenProvider

    JWT 토큰 관련 기능을 제공하는 클래스이다.

### Spring Security Configuration

    스프링 시큐리티를 사용하기 위해 기본적으로 설정파일을 작성해줘야 한다.
    config 패키지를 생성하고 해당 패키지에 SecurityConfig 클래스를 생성한다.

    @Configuration
    @EnableWebSecurity
    @RequiredArgsConstructor
    public class SecurityConfig extends WebSecurityConfigurerAdapter {

        private final CorsFilter corsFilter;

        @Override
        protected void configure(HttpeSecurity http) throws Exception {
            http.csrf().disable();
            http
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS) // 세션 사용 안함
                .and()
                .addFilter(corsFilter) // 인증(O), security Filter에 등록
                .formLogin().disable() // formLogin 사용 안함
                .httpBasic().disable()
                .authorizeRequests()
                // 이 부분은 커스터마이징
                .antMatchers("/v1/api/member", "/v1/api/member/all", "/v1/api/member/email")
                .access("hasRole('ROLE_USER') or hasRole('ROLE_MANAGER')") or hasRole('ROLE_ADMIN')
                .antMatchers("/v1/api/manager/**")
                .access("hasRole('ROLE_MANAGER') or hasRole('ROLE_ADMIN')")
                .antMatchers("v1/api/admin/**")
                .access("hasRole('ROLE_ADMIN')")
                .anyRequest()
                .permitAll();
        }
    }

    .httpBasic().disable() 
        - JWT은 httpBearer 방식이므로 httpBasic 방식은 disable() 처리한다.
        
    .antMatchers & access
        - 어느 요청이냐에 따라 맞는 권한 여부를 확인

    CORS 차단을 해제하고, JWT 방식으로만 검증할 것이므로 CorsFilter를 추가한다.
        - CorsFilter 클래스를 생성해서 넣어준다.

### CorsFilter class

    @Configuration
    public class CorsConfig {

        @Bean
        public CorsFilter corsFiler() {
            UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
            CorsConfiguration cofig = new CorsConfiguration();
            config.setAllowCredentials(true) // json 서버응답을 자바스크립트에서 처리할 수 있게 해줌
            config.addAllowedOrigin("*");    // 모든 ip에 응답을 허용
            config.addAllowedMethod("*");    // 모든 HTTPMETHOD에 허용
            cofing.addAllowedHeader("*");    // 모든 HTTPHEADER에 허용
            source.registerCorsConfiguration("/v1/api/**", config);

            return new CorsFilter(source);
        }
    }

### 정리

    지금 JWT를 사용하기 위해 스프링 시큐리티를 설정하고, CORS로 인한 차단을 해제하기 위해 CorsConfig 클래스 내에 CorsFilter 클래스를 새로 만들어줬으며, JWT 이외의 방식에서 사용되는 session, http basic, loginForm 등의 인증방식을 제거해줬다.
    스프링 시큐리티를 사용하고자 한다면 기본적으로 세팅해야 하는 부분이다.    

![HttpRequest&Response_map](/grammer/img/HttpRequest&Response_map.png)    

    위의 사진은 HttpRequest&Response에 대한 Spring의 처리 절차를 보여준다.
    Filter가 request, response를 가장 먼저 만가게 되고, Filter에 우리가 사용하는 Spring Security Filter도 포함이 되는 것이다.

![Filter_map](/grammer/img/Filter_map.png)    

    정리 해보면, 서비스에 접근하는 사람들을 검증하기 위해 요청자가 Login 시, 발급 받았던 토큰 jwt를 확인하므로 Authentication(인증)을 하는 것이고,
    이후 권한을 확인하는 Authorization(인가)을 하게 되는 것이다.

## part2 SpringBoot JWT login - Filter 적용 테스트

    Filter 
    - 필터는 외부 요청을 가장 먼저 검증하는 곳이다. 커스텀 필터를 만들어 테스트 해보자.

    @Sl4fj
    public class MyFilter implements Filter {
        @Override
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException {
            log.info("Filter");
            chain.doFilter(request, response);
        }
    }

    - Filter 인터페이스를 상속받아서 구현해주면 된다. 이때 메소드는 init(), destroy(), doFilter()가 있다.
     
    - Parameter로 웹 요청, 응답과 필터 체인이 있는 것을 확인할 수 있다.

    - chain.doFilter()를 통해 현재 필터에서 로직을 수행하고 다음 필터로 넘기게 된다.
    
### JWT 임시 검증

    @Slf4j
    public class MyFilter4 implements Filter {
        @Override
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException {

            HttpServletRequest req = (HttpServletRequest) request;
            HttpServletResponse res = (HttpServletResponse) response;
        }

        /*
            token이 필요하다. id/pw가 정상적으로 들어와서 로그인이 완료되면 토큰을 만들고 반환해준다.
            클라이언트는 요청할 때 마다 header에 Authorization - value 쌍으로 토큰을 넣으면 된다.
            이때 토큰이 서버가 갖고있는 토큰인지 검증많 하면 된다.
        */

        if (req.getMethod().equals(HttpMethod.POST.name())) {
            log.info("POST 요청");
            String headerAuth = req.getHeader("Authorization");
            if (headerAuth.equals("token")) {
                log.info("Filter4");
                log.info(headerAuth);
                chain.doFilter(req, res);
            } else {
                log.error("인증 안됨");
            }
        }   
    }

    - 인가 코드가 확인이 되면 Fitler Chain을 이어서 진행하고, 그렇지 않으면 "인증안됨"을 출력하고 요청을 종료시켜버린다.

    ● Test (잘못된 토큰)

    - 요청 시, Authorization에 "token2"를 넣어서 보내는 경우 

![authorization_token2](/grammer/img/authorization_token2.png)

![token2_console](/grammer/img/token2_console.png)

    - security filter에 token2 걸러진 것을 확인할 수 있다.

    ● Test (올바른 토큰)

    - 요청 시, Authorization에 "token"을 넣어서 보내는 경우

![authorization_token](/grammer/img/authorization_token.png)

![token_console](/grammer/img/token_console.png)

    - 정상적인 필터에서 처리되어 모든 필터가 순차적으로 진행된 것을 확인할 수 있다.

    - 이러한 플로우로 최초 로그인 요청 시, ID/PW 검증 후 Token을 발급해주고, 이후 다른 요청들에 대해 Token을 확인하고 confirm/reject 결정을 해주면 되겠다.

## part3 SpringBoot JWT login - UserDetails, UserDetailsService 이해

### 사용하는 이유?

    - 먼저 Spring Security의 동작을 이해해보자.

    1. 시큐리티는 "~/login" 주소로 요청이 오면 가로채서 로그인을 진행한다.

    2. 로그인 진행이 완료되면 시큐리티_session을 만들고 SecurityContextHeader에 저장한다.
    (SecurityContextHolder = 시큐리티 인메모리 세션 저장소)  

    3. 시큐리티가 갖고있는 시큐리티_session에 들어갈 수 있는 Object는 정해져있다.
    (Object == Authentication 타입 객체)

    4. Authentication 안에 User 정보가 있어야 한다.

    5. User 객체 타입은 UserDetails 타입 객체이다.

### 정리

    - Security Session에 객체를 저장해준다.
    - 세션에 저장될 수 있는 객체는 Authentication 타입이다.
    - Authentication 객체는 User 객체를 저장한다.
    - User 객체는 UserDetails 타입이다.
    
    -> 따러서 우리 서비스는 UserDetails 를 상속받는 User를 만들어야 한다.

    Security_Session -> Authentication -> UserDetails 관계를 갖는 것이므로 차례로 접근해서 꺼내면 된다.

    시큐리티_Session에 접근해서 Authentication을 꺼내고 거기서 UserDetails 타입 User를 꺼내면 우리가 원하는 User 객체를 꺼낼 수 있게 된다.

### 정리

    - 스프링 시큐리티는 오는 모든 접근 주제에 대해 Authentication를 생성.
    - Authentication은 SecurityContext에 저장된다.
    - SecurityContext는 SecurityContextHolder가 관리한다.

    -> 따라서, 시큐리티 세션들을 SecurityContextHolder 메모리 저장소에 저장하고, 꺼내서 사용하게 되는 것이다.

### SecurityContext

        - Authentication 객체가 직접 저장되는 저장소로 필요 시, 언제든지 Authentication 객체를 꺼내서 사용할 수 있도록 제공되는 클래스이다.

        - TheadLocal에 저장되어 같은 스레드라면 전역적으로 어디서든 접근이 가능하도록 설계되어있다.

### SecurityContextHolder

        - Authentication을 저장하는 SecurityContext를 관리하는 객체이다.    

### Authentication (인증)

        - 접근자의 존재를 증명
        - Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
            - 이런식으로 Authentication 객체를 SecurityContextHolder 저장소에서 꺼낼 수 있다.

        - 내부 구조
        1. principal : id, User 객체 저장.
        2. credentials : pw
        3. authorities : 인증된 사용자의 권한 목록
        4. details : 인증 부가 정보
        5. Authenticated : 인증 여부 (Boolean)    
        
![authentication_flow](/grammer/img/Authentication_flow.png)

### UserDetails를 상속받는 PrincipalDetails 클래스 생성

    @Data
    public class PrincipalDetails implements UserDetails {

        private final Member member;

        public PrincipalDetails(Member member) {
            this.member = member;
        }

        // 해당 유저의 권한을 리턴.
        @Override
        public Collection<? extends GrantedAuthority> getAuthorities() {
            Collection<GrantedAuthority> authorities = new ArrayList<>();
            this.member.getRoleList().forEach(R -> {
                authorities.add(() -> R);
            });
            return authorities;
        }
    }

    - Member는 필자 서비스에서 정의한 회원 Entity이다. 각자 자신의 회원 Entity를 맞게 넣어주면 되겟다.
    
    - getAuthorities()는 USER_ROLE, ADMIN_ROLE,,,, 같은 권한 리스트를 가져오게 된다.

    AuthenticationManager.class가 authenticate()를 통해 호출하는 loadUserByUsername() 메소드를 갖고있는 클래스는 UserDetailsService 클래스이다. 이를 상속받는 PrincipalDetailsService 클래스를 생성한다.

### PrincipalDetailsService

    @Slf4j
    @Service
    @RequiredArgsConstructor
    public class PrincipalDetailService implements UserDetailsService {

        private final MemberRepository memberRepository;

        // 시큐리티 session -> Authentication -> UserDetails
        // 시큐리티 세션(내부 Authentication(내부 UserDetails(PrincipalDetails)))
        @Override
        public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
            log.info("PrincipalDetailService.loadUserByUsername");
            log.info("LOGIN");
            Member member = memberRepository.findByUsername(username)
                    .orElseThrow(() -> new BussinessException(ExMessage.MEMBER_ERROR_NOT_FOUND));

            return new PrincipalDetails(member);
        }
    }

    - UsernamePasswordAutnenticationFilter 클래스에서 인증을 위해 위 메서드를 호출하게 된다.

    이렇게 필요한 객체 및 클래스를 생성했다. 또한 인증 과정에서 객체 타입이 어떠한 관계들로 되어있는지 정리가 되었으니 실제 로그인 요청을 받아서 Authentication을 발급받고 인증을 하고 JWT 토큰까지 발급받아 보자.



## part4 SpringBoot JWT login - UsernamePasswordAuthenticationFilter를 통한 로그인 로직 구현

    이전 포스터에서 정리한 내용을 기반으로 로직을 작성했다. 회원 로그인을 검증하기 위한 UsernamePasswordAuthenticationFilter를 상속받아,
    로그인 검증을 하고 토큰을 발급받는 JwtAuthenticationFilter 클래스를 구현해본다.

![login_authentication](/grammer/img/login_authentication.png)    

### login_authentication_flow

    1. 로그인 요청 (username, password)

    2. UsernamePasswordAuthenticationFilter에서 [username, password]를 이용해서 정상적인 로그인 여부를 검증
        - DI로 받은 AuthenticationManager 객체를 통해 로그인을 시도한다.

        - UserDetailsService를 상속받은 PrincipalDetailsServicve 클래스가 호출되고, loadUserByUsername() 가 실행된다.

        - 제정의된 loadUserByUsername() 메서드에서 회원_Repository에 접근하여 회원을 찾고 검증 후, UserDetails를 상속받은 PrincipalDetail 객체에 회원을 담아서 반환한다.

        - 스프링 시큐리티가 PasswordEncoder를 통해 password를 검증하고 확인이 되면 Authentication을 반환한다.

    3. UsernamePasswordAuthFilter에서 Authentication을 반환받고 이를 스프링 시큐리티에 다시 반환해 준다.
       -  authentication을 반환하므로 시큐리티_session에 저장한다.
            (권한 관리를 위해서 세션에 저장ㅅ)    
    
    4. 검증이 완료되었으므로 JWT를 발급받는다.
 
### JwtAuthenticationFilter

    @RequiredArgsConstructor
    public class JwtAuthenticationFilter extends UsernamePasswordAuthenticationFitler {

        private final AuthenticationManager authenticationManager;

        // login 요청을 하면 로그인 시도를 위해서 실행되는 함수
        @Override 
        public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
            log.info("로그인 시도 : JwtAuthenticationFilter.attemptAuthentication");

            ObjectMapper om = new ObjectMapper();

            try {
                // 1. username, password 받는다.

                // 2. 정상적인 로그인 여부를 검증한다.

                // 3. 로그인 성공

                // 4. authentication을 반환해준다.

                return super.attemptAuthentication(request, response);
            }

            // attemptAuthentication()에서 인증이 성공되면 다음 수행되는 메서드, JWT를 발급해준다.
            @Override
            protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response) {
                log.info("인증완료 : JwtAuthenticationFilter.successfulAuthentication");
                // 5. Jwt 발급

                super.successfulAuthentication(request, response, chain, authResult);
            }
        }
    }

### 1. Username, Password를 받는다.

    request로 넘어오는 [username, password]를 받아서 로그인 요청 객체를 생성 후 Authenticate를 위한 UsernamePasswordAuthenticationToken을 발행한다.

    JwtAuthenticationFilter class {

        ObjectMapper om = new ObjectMapper();

        // 1. username, password 받는다.
        LoginReq login = om.readValue(request.getInputStream(), LoginReq.class);

        // username, password를 이용해서 token 발급
        UsernamePasswordAuthenticationToken authenticationToken = 
                new UsernamePasswordAuthenticationToken(login.getUsername(), login.getPassword());

    }

    - ObjectMapper를 이용하여 HttpServletRequest 요청으로부터 LoginReq 객체를 생성
    
    - 생성된 LoginReq 객체에 접근하여, username/password를 꺼내서 
    UsernamePasswordAuthenticationToken 발급

        - 이때 username = principal, password = credentials 된다.

        ex) 
            login.getusername(), login.getPassword()

            authenticationToken.getPrincipal().toString();
            authenticationToken.getCredentails().toString();

            {
                "username" : "joo",
                "password" : "1234"
            }

### 2. 정상적인 로그인 여부를 검증한다.

    전달받은 로그인 정보를 이용해서 생성한 토큰을 가지고 로그인이 유효한지 검증하면 된다.
    회원의 존재 여부와 존재할 경우 해당 토큰의 (Principal == username && credentials == password)를 검증하면 된다.

    하지만, 패스워드를 비교하는 로직은 시큐리티 내부에서 검증되므로 따로 넣지않아도 된다.

    아이디/패스워드가 일치하면 알아서 authentication을 반환해주고, 아니면 연결이 종료된다.

    - authenticationManager class의 authenticate()에 토큰을 넘기면 자동으로 UserDetailsService class -> loadUserByUsername() 메소드가 실행된다.

    JwtAuthenticationFilter class {

        // 2. 정상적인 로그인 시도 여부를 검증한다.
        // -> 로그인 정보를 가지고 임시로 Auth token을 생성해서 인증을 확인한다.
        // -> DI 받은 authenticationManager 로그인 시도.
        // -> DetailsService를 상속받은 PrincipalDetailsService가 호출되고 loadUserByusername() 함수가 실행된다.
        // authentiacate()에 토큰을 넘기면 PrincipalDetailsService.class -> loadUserByusername() 메소드 실행된다.
        // DB에 저장되어있는 username & passwrod가 일치하면 authentication이 생성된다.
    }

### PrincipalDetailsService class

    @Service
    @RequiredArgsConstructor
    public class PrincipalDetailService implements UserDetailsService {

        private final MemberRepository member~~;

        // SecuritySession -> Authentication -> UserDetails
        // SecuritySession(Authentication(UserDetails(PrincipalDetails)))
        @Override
        public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
            log.info("PrincipalDetailService.loadUserByUsername");
            log.info("LOGIN");
            Member member = memberRepository.findByUsername(username)
                .orElseThrow(() -> new BussinuessException(ExMessage.MEMBER_ERROR_NOT_FOUND));

            return new PrincipalDetails(member);
        }

    }

### 3. 로그인 성공

    이 부분이 수행된다는 것은 loadUserByUsername() 메서드에서 성공적으로 회원조회 및 username, password를 통한 검증이 이루어졌다는 것을 의미.

    이젠 직접 확인해 보기 위해, 반환받은 authentication 객체에서 PrincipalDetails 객체를 꺼내서 username/password를 출력해보자.

    public class JwtAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

        private final AuthenticationManager authentication~~;

        // Login 요청을 하면 로그인 시도를 위해서 실행되는 함수
        @Override
        public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) {
            log.info("로그인 시도 : JwtAuthenticationFilter.attemptAuthentication");

            ObjectMapper om = new ~~~;

            try {

                // 1. username, password 받는다.

                // 2. 정상적인 로그인 시도를 해본다.
                Authentication authentication = 
                        authenticationManager.authenticate(authenticationToken);

                // 3. 로그인
                // Authentication에 있는 인증된 Principal 객체를 PrincipalDetails 객체로 꺼낸다.
                PrincipalDetails principalDetails = (PrincipalDetails) authenticatoin.getPrincipal();

                // 4. authentication을 반환해준다.
                return authentication;
            } catch (IOException e) {
                e.printStackTrace();
            }
            return null;
        }
    }

    // attemptAuthentication() 실행 후 인증이 정상적으로 완료되면 실행된다.
    // 따라서, 여기서 JWT토큰을 만들어서 request 요청한 사용자에게 JWT토큰을 response 해준다.
    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        log.info("인증 완료 : JwtAuthenticationFilter.successfulAuthentication");

        super.successfulAuthentication(request, response, authResult);
    }

![authentication_encoderPassword](/grammer/img/authentication_encoderPassword.png)    

    - 처음 요청 받았을 때 출력했던 principal, credentials가 암호화된것을 확인할 수 있다. 이는 실제 DB에 저장된 회원을 잘 조회해서 가져왔다는 것으로 회원 객체는 저장될 때 PasswordEncoder에 의해 password를 암호화해서 저장하기 때문이다.
  
### 4. authentication을 반환해준다.

    authentication 객체를 SecuritySession에 저장해야 하므로 반환한다.
    세션에 저장하면 편리하게 권한관리를 할 수 있다.
    
    public class JwtAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

        private final AuthenticationManager authenticationManager;

        // login 요청을 하면 로그인 시도를 위해서 실행되는 함수
        @Override
        public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {

            ObjectMapper om = new ~~;
            try {

                // 1. username, password

                // 2. 정상인지 로그인 시도를 해본다.

                // 3. 로그인이 되었다.

                // 4. authentication을 반한해준다.
                log.info("4. authenticagtion 반환");
                return authentication;
            } catch (IOException e) {
                e.printStackTrace();
            }
            return null;
        }
    }

![authentication_return](/grammer/img/authentication_return.png)

    - 정상저긍로 모든 인증이 되고 자동적으로 successfulAuthentication() 메소드가 이어서 수행되는 것을 볼 수 있다.

    - 여기서 JWT 토큰을 최종적으로 발행해서 반환해주면 클라이언트는 그것을 가지고 요청을 주면 된다.

### 5. 최종 전체 로직

    public class JwtAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

        private final AuthenticationManager ~~ ;

        // login 요청을 하면 로그인 시도를 위해서 실행되는 함수
        // Authentication 객체를 만들어서 리턴(AuthenticationManager를 통해)
        @Override
        public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
            log.info("로그인 시도 : JwtAuthenticationFilter.attemptAuthentication");

            // 로그인 요청 시, 들어온 데이터를 객체로 변환
            ObjectMapper om = new ~~;

            try {

                // 1. username, password
                log.info("1. username, password 받는다.");
                LoginReq login = om.readValue(request.getInputStream(), LoginReq.class);
                log.info("login.toString());

                // username, password를 이용해서 token 발급
                UsernamePasswordAuthenticationToken authenticationToken = new Username~~(login.getUsername(), login.getPassword());
                log.info(authenticationToken.getPrincipal().toString());
                log.info("==========================================\n")

                // 2. 정상인지 로그인 시도를 해본다. 
                log.info("2. 정상인지 로그인 시도를 해본다.");
                // -> 로그인 정보를 가지고 임시로 Auth 토큰을 생성해서 인증을 확인한다.
                // -> DI받은 authenticationManager로 로그인 시도한다.
                // -> DetailsService를 상속받은 PrincipalDetailsService가 호출되고 loadUserByUsername() 함수가 실행된다.
                // authenticate()에 토큰을 넘기면 PrincipalDetailsService.class -> loadUserByUsername() 메소드 실행된다.
                // DB에 저장되어있는 username & password가 일치하면 authentication이 생성된다.
                log.info("-> Authenticate Start");
                Authentication authentication =
                    authenticationManager.authenticate(authenticationToken);
                log.info("<- Authenticate End");
                log.info("==========================================\n");

                // 3. 로그인이 되었다.
                log.info("3. 로그인 성공");
                // 로그인이 되었다.
                // Authentication에 있는 인증된 Principal 객체를 PrincipalDetails 객체로 꺼낸다.

                // 4. authentication을 반환해준다.
                // authentication 객체를 session에 저장해야 하므로 반환한다. 세션에 저장하면 편리하게 권한관리를 할 수 있다.
                // 반환된 Authentication 객체가 세션에 저장된다-> 로그인 성공.
                log.info("4. authentication 반환");
                return authentication;
            } catch(IOException e) {
                e.printStackTrace();
            }
        }

        // attemptAuthentication() 실행 후 인증이 정상적으로 완료되면 실행되는 메소드.
        // 따라서, 여기서 JWT 토큰을 만들어서 request 요청한 사용자에게 JWT 토큰을 response 해준다.
        @Override
        protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) AuthenicateException {
            log.info("인증 완료 : JwtAuthenticationFilter.successfulAuthentication");

            super.successfulAuthentication(request, response, authResult);
        }
    }

![JwtAuthentication_AllResponse](/grammer/img/JwtAuthentication_AllResponse.png)    

## part5 SpringBoot JWT login - client request, BasicAuthenticationFilter 이용한 JWT 검증.

    username, Password를 이용한 검증은 완료된 상태로 다음 수행될 successfulAuthentication() 메서드에서 JWT 토큰을 발급해보자.

### JwtAuthenticationFilter class
    @Slf4j
    @RequiredArgsConstructor
    public class JwtAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

        private final AuthenticationManager authenticationManager;

        // login 요청을 하면 로그인 시도를 위해서 실행되는 함수
        @Override
        public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) {
            log.info("로그인 시도 : JwtAuthenticationFilter.attemptAuthentication");
            // 검증 완료
        }

        @Override
        public Authentication successfulAuthentication(HttpServletRequest request, HttpServletResponse response) {
            log.info("인증 완료 : JwtAuthentication.successfulAuthentication");
        }   
    }

    이전에 생성해서 반환된 Authentication 객체가 authResult Parameter로 들어오고 있다. 해당 Auth 객체를 이용해서 토큰을 생성해본다.
    
    - PrincipalDetails principal = (PrincipalDetails) authResult.getPrincipal();

    JWT 토큰을 만드는건 gradle에 추가한 implementation libaray를 통해 쉽게 생성할 수 있다.

    String jwt = JWT.create()
            .withSubject("JWT_토큰")
            .withExpired(new Date(System.currentTimeMillis() + 6000 + 10))  // 만료시간
            .withClaim("id", principal.getMember().getSeq()) // 회원 구분용 seq
            .withClaim("username", principal.getMember().getUsername())
            .sign(Algorithm.HMAC512("SecretKey@@!!!")); // signature를 생성하기 위한 SecretKey

    - jwt가 생성되었고 이를 응답헤더에 추가해서 반환해준다.
    - 클라이언트 전달받은 Jwt 토큰을 가지고 요청 때 마다 토큰을 가지고 요청하면된다.

    응답헤더에 HTTP Bearer 방식이므로 Authorization : "Bearer_jwt_"를 추가한다. key-value 쌍으로 들어간다. (주의) Bearer 다음에 공백이 꼭 들어가야 한다.

    response.addHeader("Authorization", "Bearer " + jwt); // jwt 응답 헤더에 추가
    - 응답 메시지 Header에 jwt가 추가된다.

![authentication_Header](/grammer/img/authentication_Header.png)

    body에는 딱히 내용이 없고 Header에 정상적으로 Authorizaion - jwt 토큰이 담겨있는것을 확인할 수 있다.

![authentication_encoded](/grammer/img/authentication_encoded.png)

    해당 토큰을 해싱해보면, 이런식으로 header에는 해시기법, payload에 저장된 값 (=signature)를 확인할 수 있다.

    HASH{ 인코딩(jwt_header) +.+ 인코딩(jwt_payload) +.+ 서버_SecretKey} == jwt_Signature 여부

### BasicAuthenticationFilter를 이용한 JWT 검증

    근데, 여기서 JWT 인증 방식과 Session 인증 방식의 차이점이 또 존재한다.

    ● Session 검증
        1. username&password LOGIN 요청  
        2. 서버에서 세션ID 생성  
        3. 클라이언트에서 쿠키에 세션ID를 저장 
        4. 요청 때마다 쿠키값에 세션ID 유효여부 검증과 같은 방식으로 이뤄진다. 

        이때 서버에서 세션ID 유효여부 검증은 HttpSession에서 제공하는 session.getAttribute("session_key")를 통해 내부적으로 알아서 이뤄지고 값을 가져올 수 있다.
        
    ● JWT 로직 검증
        1. username&password LOGIN 요청 
        2. Authentication 생성 후 검증  
        3. 로컬 저장소 or 쿠키에 저장.
        4. Authentication으로 JWT 토큰 발급 후 응답 헤더에 넣어서 반환

    - 클라이언트가 요청 때 전달한 JWT 토큰에 대해 서버가 JWT 토큰이 유효한지 판단하는 부분이 없다. 따라서 해당 필터를 만들어줘야 한다.
    이를 위해 인증 or 권한이 필요한 부분에서 실행되는 시큐리티 필터 중 BasicAuthenticationFilter를 상속하여 필터를 구현해본다.

### JwtAuthorizationFilter extends BasicAuthenticationFilter

    // 권한이나 인증이 필요한 특정 주소를 요청했을 때, BasicAuthenticationFilter를 타게 된다.
    // 권한이나 인증이 필요하지 않다면 거치지 않는다.
    @Slf4j
    public class JwtAuthenticationFilter extends BasicAuthenticationFilter {

        private MemberRepository ~~ ;

        public JwtAuthorizationFilter(AuthenticationManager authenticationManager, MemberRepository memberRepository) {
            super(authenticationManager);
            this.memberRepository = memberRepository;
        }

        // 인증이나 권한이 필요한 주소요청이 있을 때, 해당 필터를 거친다.
        @Override
        protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException {
            log.info("CHECKJWT : JwtAuthorizationFilter.doFilterInternal");

            // 1. 권한이나 인증이 필요한 요청이 전달됨
            String jwtHeader = request.getHeader("Authorization"); // Header에 들어있는 Authorization을 꺼낸다.(SpringSecurity Config) 

![authorization_header_request](/grammer/img/authorization_header_request.png)

            // 2. Header 확인
            if (jwtHeader == null || !jwtHeader.startWith("Bearer")) {
                chain.doFilter(request, resposne);
                return; // 헤더가 비어있거나, 비어있지는 않지만 Bearer 방식이 아니라면 반환.
            }

            // 3. JWT 토큰을 검증해서 정상적인 사용자인지 확인
            log.info("3. JWT 토큰을 검증해서 정상적인 사용자인지, 권한이 맞는지 확인");
            String jwtToken = request.getHeader("Authorization").replace("Bearer", "");
            String username = null;
            try {
                username = JWT
                        .require(Algorithm.HMAC512("SecretKey@@!!!"))
                        .build()
                        .verify(jwtToken)
                        .getClaim("username")
                        .asString();
            } catch (Exception e) {
                throw new BussinessException(ExMessage.JWT_ERROR_FORMAT);
            }
            - JWT 라이브러리를 이용해서 검증을 진행해본다.
            - 적용했던 Hash 알고리즘으로 SecretKey를 해시하고, 
            - 토큰에서 username 키에 해당하는 value를 문자열로 꺼낸다.
          
![authorication_wrongToken](/grammer/img/authorication_wrongToken.png)

            JWT 토큰 검증이 제대로 진행되었다면 Signature는 우리가 한 JWT임이 검증된것이다. 꺼낸 username을 가지고 Athentication 객체에 넣기위한 UserDetails 객체를 생성한다.

            if (username != null) {
                // 서명이 정상적으로 됨
                log.info("서명이 정상적으로 됨");

                // 4. 정상적인 서명이 검증되었으므로 username으로 회원을 조회한다.
                log.info("4. 서명이 검증되었다.");
                log.info("Athentication 생성을 위해 username으로 회원 조회 후 PrincipalDetails 객체로 감싼다.");
                Member member = memberRepository.findByUsername
                (username)
                        .orElseThrow(() -> new BusinessException(ExMessage.MEMBER_ERROR_NOT_FOUND));
                PrincipalDetails principalDetails = new PrincipalDetails(member);
                - JWT의 payload에서 꺼낸 username 값으로 회원을 조회한다.
                - 해당 회원을 PrincipalDetails 객체로 감싼다.

![authentication_signature_response](/grammer/img/authentication_signature_response.png)

                // 5. jwt토큰 서명을 통해서 정상이면 Authentication 객체를 만들어준다.
                log.info("5. Authentication 객체 생성");
                Authentication authentication = new UsernamePasswordAuthenticationToken(
                    principalDetails, null, principalDetails.getAuthorities());
                - UsernamePasswordAuthenticationToken()을 통해 Authentication을 만들어준다.
                - 로그인 때 username, password를 이용해서 Authentication을 만들었던것과 동일하다.
                다만, Principal에 조회로 구한 PrincipalDetils 회원을 담고,
                Credentials는 null로 비우고, 권한을 넣어준다.
                (이미 검증되었기에, 비밀번호는 필요가 없다.)

                // 6. 강제로 Security_Session에 접근항 Authentication 객체를 저장해준다.
                log.info("6. SecuritySession에 접근하여 Authentication 객체 저장");ㄴ
                SecurityContextHolder.getContext().setAuthentication(authentication); 
                - SecurityContextHolder에 전달받은 JWT로 만든 Authentication을 저장해준다. Authentication에는 현재 권한이 들어있으므로 권한이 필요한 곳에 조회할 때, 해당 권한을 체크해 줄 것이다.

                chain.doFilter(request, response);
            }
        }
    }

    로그인 요청을 통해 JWT 발급
    
![authentication_test](/grammer/img/authentication_test.png)

    발급 받은 JWT를 이용해서 요청
    1. MEMBER

![authentication_member_test](/grammer/img/authentication_member_test.png)    
    - 정상적으로 요청/응답이 된다.
    
    2. MANAGER

![authentication_manager_test](/grammer/img/authentication_manager_test.png)    
    - 권한 부족으로 403 Forbidden이 발생한다.
    - HTTP 403 Forbidden 클라이언트 오류 상태 응답 코드는 서버에 요청이 전달되었지만, 권한 때문에 거절되었다는 것을 의미.

    3. AMIND
    
![authentication_admin_test](/grammer/img/authentication_admin_test.png)    
    - 동일하게 403 Forbidden 발생

### 참고

    로그인을 통해 발급받은 JWT 토큰을 가지고 요청을 보내고, 권한에 따른 접근 제어까지 확인해보았다. 참고로 설정관련 값들을

    public interface JwtProperties {
        int EXPIRATION_TIME = 1000 * 60 * 10; // 10m
        String TOKEN_PREFIX = "Bearer";
        String HEADER_PREFIX = "Authorization";
    }

    이런식으로 인터페이스 클래스에 보관해서 사용하면 더욱 실수를 줄일 수 있다.

    또한, 시크릿 값은 Github와 같은 저장소에 올리면 안되므로 gitignore에 추가해준 properties 파일에 담았다.

![gitignore_secret](/grammer/img/gitignore_secret.png)

## ADD - HttpHeader에 있는 JWT 처리

    JWT는 HttpHeader에 담겨서 요청이 넘어오게 되고 이를 처리하는 방법 2가지를 알아보자. 

### 1. UsernamePasswordAuthenticationFilter

    UsernamePasswordAuthenticationFilter를 상속받은 CustomAuthenticationFilter에서 토큰을 통해 인증.

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpSerlvetResponse response) throws AuthenticationException {
        log.info("인증 시도");
        ObjectMapper om = new ~~();

        try {
            LoginDto loginDto = om.readValue(request.getInputStream(), LoginDto.class);
            Authentication authentication = new UserPasswordAuthenticationToken(loginDto.getUsername(), loginDto.getPassword());
            return authenticationManager.authenticate(authentication);
        } cathc (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {
        log.info("인증 성공");
        PrincipalUserDetails principal = (PrincipalUserDetails) authResult.getPrincipal();

        String accessToken = jwtService.createAccessToken(principal.getUsername());
        String refreshToken = jwtService.createRefreshToken();

        Member memberByUsername = jwtService.getMemberByUsername(principal.getUsername());
        jwtService.setRefreshTokenToUser(memberByUsername, refreshToken);

        jwtService.setResponseOfAccessToken(resposne, accessToken);
        jwtService.setResponseOfRefreshToken(response, refreshToken);
        jwtService.setResponseMessage(true, resposne, "로그인 성공");
    }

    @Override
    protected void unsuccessfulAuthentication(HttpServletRequest request, ServletResponse response, Authentication authResult) {
        log.info("인증 실패");
        ExMessage failMessage = failed.getMessage().equals(ExMessage.MEMBER_ERROR_NOT_FOUND);
            ExMessage.MEMBER_ERROR_NOT_FOUND;
            ExMessage.MEMBER_ERROR_PASSWORD;
        jwtService.setResponseMessage(false, response, "로그인 실패" + ":" + failMessage);
    }

    attemptAuthentication에서 먼저 authenticate를 통해 인증을 진행하고 성공/실패에 따라 적절한 메소드에서 이어서 진행.

### 2. SecurityContextHolder

    public class JwtFilter extends GenericFilterBean {

        private final JwtTokenProvider jwtTokenProvider;

        public JwtFilter(JwtTokenProvider jwtTokenProvider) {
            this.jwtTokenProvider = jwtTokenProvider;
        }

        @Override
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {

            HttpServletRequest httpServletRequest = (HttpServletRequest) request;
            String token = jwtTokenProvider.resolveToken(httpServletRequest);
        
            if (token != null && jwtTokenProvider.validateToken(token)) {
                SecurityContextHolder.getContext().setAuthentication(jwtTokenProvider.getAuthentication(token));
            }

            chain.doFilter(request, response);
        }
    }

### 2-1 JwtProvider
    public class JwtTokenProvider {

        private final SecurityProperties securityProperties;

        public JwtTokenProvider(SecurityProperties securityProperties) {
            this.securityProperties = securityProperties;
        }

        public String resolveToken(HttpServletRequest request) {
            String header = request.getHeader(securityProperties.getHeaderString());

            if (header != null && header.startsWith(securityProperties.getTokenProperties().getTokenPrefix())) {
                return header.substring(securityProperties.getTokenProperties().getTokenPrefix().length());
            } else {
                return null;
            }
        }

        public boolean validateToken(String token) {
            // 토큰 유효성 검사 로직 추가 (예시)
            return true; // 유효할 경우 true 리턴
        }

        public Authentication getAuthentication(String token) {
            // 토큰에서 인증 정보 추출 (예시)
            return null; // 실제 인증 정보를 반환
        }
    }
    header에서 jwt 헤더를 추출하고, 전역 객체인 SecurityContextHolder에 authentication 객체를 저장한다. 이후 Controller에서 해당 SecurityContext를 꺼내서 인가를 검증한다.

### 2-2 userController

    @Slf4j
    @RestController
    public class UserController {

        private final UserJpaRepository userJpaRepository;

        public UserController(UserJpaRepository userJpaRepository) {
            this.userJpaRepository = userJpaRepository;
        }

        @GetMapping("/user")
        public UserDto getUser() {

            logger.info("GET USER");

            Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
            
            if (principal instanceof UserDetails) {
                String username = ((UserDetails) principal).getUsername();
                logger.info("principal: {}", username);

                User user = userJpaRepository.findByUsername(username)
                        .orElseThrow(() -> new NotFoundException("멤버가 없음"));

                return UserDto.from(user);
            } else {
                throw new InternalAuthenticationServiceException("Can not found matched User Principal");
            }
        }
    }



