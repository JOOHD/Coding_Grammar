## 본문

## SpringSecurity를 이용한 로그인/회원가입


### build.gradle 설정 

    - 의존관계에 추가

    implementation 'org springframeworkboot:spring-boot-starter-security'

    testImplementation 'org.springframeworksecurity:spring-security-test'

    - 스프링 시큐리티 의존성 추가 시
    
    1. 서버 실행 시, SpringSecurity의 초기화 작업 및 보안 설정.
    2. 별도의 설정이나 구현을 하지 않아도 기본적인 웹 보안 기능이 서버에 연동.
    3. 모든 요청은 인증이 되어야 접근 가능.
    4. 기본 로그인 페이지 제공.

![Security_please_signin](/grammer/img/Security_signin.png) 

    + application.properties에 기본 name/password 설정이 가능하다.

### filter

    Request에 대해서 피러가 가로채어 인증 전 객체를 받을 수 있도록 특정 URI 접속 시, loginForm으로 이동하돌고 설정.

    spring security는 아래와 같이 SecurityFilterChain을 사용하도록 권장.

    @Configuration
    public class SecurityConfig {
    
        @Bean
        public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
            http.csrf().disable();
            http.authorizeRequests()
                    .antMatchers("/user/**").authenticated() // 로그인해야 들어올수 있음
                    .antMatchers("/manager/**").access("hasRole('ROLE_MANAGER')")
                    .antMatchers("/admin/**").access("hasRole('ROLE_ADMIN')")
                    .anyRequest().permitAll()
                    .and()
                    .formLogin()
                    .loginPage("/loginForm")
                    .loginProcessingUrl("/login") // login 주소가 호출이되면 시큐리티가 낚아채서 대신 로그인 진행해줌.
                    .defaultSuccessUrl("/");
            return http.build();
        }
    }

    ● 풀이

    - SecurityFilterChain 등록

        - http.csrf().disable() : CSRF 방지.  CSRF(Cross-Site Request Forgery) 로부터 보호

        - http.authorizeRequests()

            -antMatchers(String).authenticated() : String URI 에 대해 접근하기 위해서는  로그인 필수

            - antMatchers(String).access("ROLE") : 해당 ROLE을 가져야 접근할 수 있음

            - loginPage(String) : 스프링 시큐리티가 기본적으로 지원하는 로그인 페이지를 사용하지 않고 별도로 로그인 페이지 URI 등록

            - loginProcessUrl("/login") : /login 의 URI 요청이 오면 시큐리티가 낚아채서 로그인을 진행합니다.

            - defaultSuccessUrl() : 로그인 성공 시 이동할 URI, 지정하지 않으면 이전에 요청했던 URI로 자동으로 리다이렉트합니다.

### 회원가입 Controller

    @Slf4j
    @Controller
    @RequiredArgsConstructor
    public class IndexController {

        private final UserRepository userRepository;
        private final BCryptPasswordEncoder bCryptPasswordEncoder;
        
        @GetMapping("/joinForm")
        public String joinForm() {
            return "joinForm";
        }

        @PostMapping("/join")
        public String join(User user) {

            user.setRole("USER");

            String rawPassword = user.getPassword();
            String encPassword = bCryptPasswordEncoder.encode(rawPassword);

            user.setPassword(encPassword);
            userRepository.save(user);

            log.info("user = {} ", user);

            return "redirect:/loginForm";
        }
    }

    - joinForm에서 Post 전송된 파라미터를 user에 담고, BCryptPasswordEncoder를 통해 해상함수로 암호화된 비밀번호를 DB에 저장.

    + Security 로그인하기 위해서는 반드시 비밀번호를 위와 같이 암호화 해야 한다. "객체의 경우 자동으로 @ModelAttribute가 적용된다."

### 로그인 Controller
    
    @GetMapping("/loginForm")
    public String login() {
        return "loginForm";
    }

### loginForm

    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="UTF-8">
        <title>로그인 페이지</title>
    </head>
    <body>
    <h1>로그인 페이지</h1>
    <hr/>
    <!-- 시큐리티는 x-www-form-url-encoded 타입만 인식 -->
    <form action="/login" method="post">
        <input type="text" name="username" placeholder="Username" />
        <input type="password" name="password" placeholder="Password"/>
        <button>로그인</button>
    </form>
    <a href="/joinForm">회원가입을 아직 하지 않으셨나요?</a>
    </body>
    </html>

    입력하 ID와 PW를 /login 으로 Post 전송한다.
    이 때 위에서 등록한 필터에 의해 (http.loginProcessingUrl("/login")),
    login 주소가 호출이 되면 시큐리티가 낚아채서 대신 로그인 진행.

        " + loginProcessingUrl를 사용하면 스프링 시큐리티가 내부에서 AuthenticationManager와 AuthenticationProvider를 자동으로 생성.

        따라서, loginProcessingUrl을 사용하면 스프링 시큐리티는 내부에서 사용자가 해당 URL로 로그인을 시도할 때 인증 처리를 위해 AuthenticationManager와 AuthenticationProvider를 구성하지 않아도 기본적인 인증 처리를 수행할 수 있다.

    - 로그인 완료 후, Security Session에 세션 정보를 저장해야 한다.
    - 여기에 들어갈 수 있는 객체는 Authentication 객체 뿐이다. Authentication 객체 안에는 User 정보를 저장해야 하는데 이 때, User는 반드시 UserDetails 타입이어야 한다.

    쉽게 말해, 시큐리티가 /login 요청을 낚아채서 로그인을 진행한다.
    로그인 진행 완료가 되면, SecurityContextHolder에 시큐리티 세션을 저장.해야 한다.        

    이 과정에서 여기(SecurityContextHolder)에 들어갈 수 있는 객체는 Authentication 뿐이고, Authentication 객체 안에는 유저정보가 들어가 있어야 하는데, 유저 정보는 반드시 UserDetails 타입이어야 한다.
        
        "SecurityContextHolder(Session(Authentication(UserDetails)))"

    따라서 UserDetails를 구현한 구현체 PrincipalDetails를 Authentication 객체에 담아보자.

    + principal 
    - 현재 인증된 사용자를 나타내는 객체 (=현재 로그인한 사용자.)
    - 인증된 사용자를 식별하는 데 사용됨.

### UserDetails 구현 - PrincipalDetails

    public class PrincipalDetails implements UserDetails {

        private User user;

        public PrincipalDetails(User user) {
            this.user = user;
        }

        // 해당 User의 권한을 리턴하는 곳
        @Override
        public Collection<? extends GrantedAuthority> getAuthorities() {
            Collection<GrantedAuthority> collection = new ArrayList<>();
            collection.add(
                    new GrantedAuthority() {
                @Override
                public String getAuthority() {
                    return user.getRole();
                }
            });
            return collection;
        }

        @Override
        public String getPassword() {
            return user.getPassword();
        }

        @Override
        public String getUsername() {
            return user.getUsername();
        }
    
        // 계정 만료여부
        @Override
        public boolean isAccountNonExpired() {
            return true;
        }
    
        // 계정 잠김x 여부
        @Override
        public boolean isAccountNonLocked() {
            return true;
        }
    
        @Override
        public boolean isCredentialsNonExpired() {
            return true;
        }

        // 휴먼 계정
        @Override
        public boolean isEnabled() {
            // 1년동안 로그인 안하면 휴먼계정으로 
            // 현재 시간 - 마지막로그인날짜 => 1년 초과시, falas
            // else true
            return true;
        }
    }

### UserDetailsService 구현 - PrincipalDetailService

    // 시큐리티 설정에서 loginProcessingUrl("/login");
    // "/login" 요청이오면 자동으로 UserDetailsService 타입으로 Ioc 되어있는 loadUserByUsername 이 실행

    @Slf4j
    @Service
    @RequiredArgsConstructor 
    public class PrincipalDetailsService implements UserDetailsService {

        private final UserRepository userRepository;

        // 시큐리티 session(내부 Authentication(내부 UserDetails))
        @Override
        public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

            log.info("username = {}", username);

            User user = userRepository.findByUsername(username);

            if (user != null) {
                return new PrincipalDetails(user);
            }
            return null;
        }
    }

    - 시큐리티는 자동으로 UserDetailsService 타입으로 Ioc 되어있는 PrincipalDetailsService 가 실행되면서, loadUserByUsername 함수가 자동으로 실행된다.

    - 이 함수에서 DB에 쿼리메소드를 날려 일치하는 회원을 찾고 찾은 경우 PrincipalDetails(UserDetails)에 유저 정보를 담아 return.

    - 이제 시큐리티가 Session 안에 UserDetails 객체를 내장하고 있는 Authentication 객체가 들어가게 된다.(loadByUsername 함수에서 자동화)

    ● 의문점?

    UserDetailsService 인터페이스의 loadUserByUsername(String username)을 구현해서 사용자 정보를 DB에서 조회하고 반환한다.

    하지만 코드를 살펴보면 비밀번호를 체크하는 코드는 없다. 
    (잘못된 비밀번호를 입력하면 로그인에 실패합니다.)

    어느 부분에서 비밀 번호 체크를 하는 것 일까?
    -> AuthenticationProvider에서 검사한다.

### AuthenticationProvider    

    String presentedPassword = authentication.getCredentials().toString();

        if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
            log.info("Authentication failed: password doesnt match stored value");

            throw new BadCredentialsException(message.getMessage(
                "AbstractUserDetailsAuthenticationProvider.badCredentials",
                "Bad credentials"));
        }
