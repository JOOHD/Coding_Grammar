## 본문

## Kakao OAuth2 + JWT + Redis를 통한 인증 과정 구현 (1) - 카카오 로그인

    Spring Security + JWT + OAuth2 Client + Redis 를 사용하여 카카오 로그인을 프로젝트에 적용.

### OAuth2 

    - 인증 및 권한 위임을 위한 표준 프로토콜
    - 애플리케이션이 제3자 서비스(카카오, 구글, 네이버..)를 통해 사용자 리소스(프로필, 이메일등)에 접근하도록 허용.
    - OAuth2 에서는 AccessToken 을 발급받아 사용자의 데이터를 안전하게 요청.

### Spring Security 의 OAuth2User

    - Spring Security 에서 제공하는 interface, OAuth2 인증을 통해 가져온 사용자 정보를 표현

    ● 주요 메서드

    - getName() : 인증 주체 이름.
    - getAttributes() : 인증 제공자(카카오, 구글..)로부터 전달받은 사용자 정보(속성).
    - getAuthorities() : 사용자 권한 정보.

    ● OAuth2 사용자의 기본 흐름.

    1. 사용자가 카카오 로그인 페이지에서 인증을 완료.
    2. 카카오의 인증 서버가 AccessToken 을 반환
    3. 애플리케이션이 해당 AccessToken 으로 사용자 정보를 요청
    4. 사용자 정보를 Spring Security 의 OAuth2User 객체로 받아 처리.
    
### step1. 인가 코드 받기

![kakao_authority](/grammer/img/kakao_authority.png)

    1. 카카오 인증 서버로 /oauth/authorizze URI 로 GET 요청을 보낸다.
    2. 카카오 인증 서버에서 클라이언트에게 카카오 로그인 페이지를 통해 로그인을 요청한다.
    3. 사용자는 카카오계정으로 로그인
    4. 카카오계정이 유요한 경우 카카오 인증 서버에서 클라이언트에게 동의 화면을 통해 사용자 정보 수집 동의를 요청한다.
    5. 클라이언트가 동의한 항목을 카카오 인증 서버에게 요청한다.
    6. 카카오 인증 서버에서는 302 Redirect URI 로 서비스 서버에게 인가 코드를 전달한다.

    - 여기서 Redirect URI 는 Kakao Developers - 내 애플리케이션 - 카카오 로그인에서 추가한 URI 로 설정된다.

### step2. 토큰 받기

![kakao_token](/grammer/img/kakao_token%20.png)

    이제 서비스 서버로 받은 인가 코드를 가지고 사용자 정보에 대한 토큰을 발급받을 수 잇따.

    2. 카카오 인증 서버가 토큰을 발듭해서 서비스 서버에 전달.
    - 여기서 헷갈릴 수 있다. 여기서의 토큰은 서비스 서버에서 사용할 access Token 이 아니다.

### step3. 사용자 로그인 처리

![kakao_userLogin](/grammer/img/kakao_userLogin.png)

    카카오 서버가 해주어야 할 일은 모두 끝났다. 이 부분은 서비스 서버에 직접 구현해야 하는 부분이다.

### 0. application.yml 설정

![kakao_applicationyml](/grammer/img/kakao_applicationyml.png)

    - Redirect URI 는 Kakao Developers 에 설정한 URI 와 동일하게 지정해준다.

    - Client ID, Cient Secret Kakao Developers 에서 발급받은 값으로 지정해준다.

    - authorization-grant-type 은 인가 코드 방식을 사용하므로 authorization_code 지정해주고, client-authentication-method 는 반드시 client_secret_post 로 지정해주어야 한다.

    ● gradle

    - implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
    - implementation 'org.springframework.boot:spring-boot-starter-security'

    인증 필터를 사용하기 위해 Security 를, 카카오 로그인을 사용하기 위해 OAuth2 를 추가한다.

### 동작 흐름

    1. 사용자 로그인 버튼 클릭

    2. 카카오 OAuth2 인증 요청
        - 클라이언트가 카카오의 Authorization Server 로 인증 요청을 보냄
            ex) 
                http://kauth.kakao.com/oauth/authorize?
                client_id={REST_API_KEY}&       // application REST API KEY
                redirect_uri={REDIRECT_URI}&    // 인증 후 direct uri
                response_type=code              // code

    3. Authentication Code 반환
        - 사용자가 카카오 로그인 인증 성공 시, 카카오가 클라이언트의 redirect_uri 로 Authorization Code 전달
            ex)
                GET/callback?code={AUTHORIZATION_CODE}

    4. Access Token Request
        - 백엔드(Spring server) 가 Authroization Code 를 사용해 카카오의 인증 서버로 Access Token 요청.
            ex)
                URL : https://kauth.kakao.com/oauth/token
                HTTPS ; POST
                Param 
                    grant_type : authorization_Code
                    client_id : REST API KEY
                    redirect_uri : redirect URI
                    code : Authorization Code 

       Access Token Response
            ex)
                {
                    "access_token": "ACCESS_TOKEN",
                    "refresh_token": "REFRESH_TOKEN",
                    "expires_in": 3600,
                    "refresh_token_expires_in": 2592000
                }              

    5. 사용자 정보 요청
        - 서버가 카카오 API 를 호출해 사용자 정보를 가져온다.
            ex)
                URL : https://kapi.kakao.com/v2/user/me
                HEADER : Authorization = Bearer ACCESS_TOKEN

       사용자 정보 응답
            ex)
                {
                    "id": 123456789,
                    "kakao_account": {
                        "email": "user@example.com",
                        "profile": {
                        "nickname": "User"
                        }
                    }
                }           

    6. JWT 발급
        - 서버가 응답받은 사용자 정보를 기반으로 JWT를 생성
            - Access & Refresh Token 

        - Redis 활용
            - Access Token 은 사용자 요청에서 전달되므로 Redis 에 저장하지 않음
            - Refresh Token 은 Redis 에 저장
                - 키 : 사용자 ID
                - 값 : Refresh Token
                - TTL : Refresh Token expired

    7. 클라이언트로 JWT 전달
        - 서버가 JWT 를 클라이언트로 응답"
        ex)
            {
                "accessToken" : "JWT_ACCESS_TOKEN",
                "refreshToken" : "JWT_REFRESH_TOKEN"|
            }

    8. 클라이언트의 요청 처리
        -클라이언트가 로그인 후, API 요청 시, Access Token 을 Authorization header 에 포함.
        ex)
            Authorization Bearer JWT_ACCESS_TOKEN

        - 서버가 토큰의 서명을 검증하고, 필요한 경우 Redis 를 통해 블랙리스트 여부 확인.

    9. Refresh Token 을 이용한 Access Token 재발급
        1. Access Token 만료 시,
            - 클라이언트가 만료된 Access Token 과 함께 Refresh Token 을 서버로 전송
            - 서버가 Redis 에서 Refresh Token 을 확인하고 새, Accses Token 발급.
        2. Redis 블랙리스트 활용
            - 클라이언트가 로그아웃하거나 Refresh TOken 이 만료되면, 해당 Access Token 을 Redis 블랙리스트에 추가.


### 동작 흐름 요약 

    1. 클라이언트가 백엔드로 Authentication Code 전달
    2. 백엔드가 Authentication Code 를 카카오 인증 서버에 전달하고, Access Token 과 Refresh Token 을 발급받는다.
    3. 백엔드는 이 Access Token 을 사용해 카카오 API 서버에서 사용자 정보를 가져온다.
    4. 백엔드가 사용자 정보를 기반으로 자체적인 JWT를 생성해 클라이언트에 전달.
    5. 이후 클라이언트는 백엔드의 API 호출 시, JWT를 사용.

### 정리

    ● Authorization Code 란?

        - 클라이언트가 아닌 백엔드가 사용하는 일회용 코드
        - 카카오 인증 서버에서 발급해주는 (redirect_uri)
        - 클라이언트는 redirect 로 전달 받으면, 백엔드에 전달해 주어야 한다.
        - 카카오와 통신을 위한 key

    ● 백엔드의 Authorization Code 사용 방식

        1. 백엔드 Authorization Code 를 이용해 카카오 인증 서버에 POST 요청

            Request Param

            {            
                "grant_type": "authorization_code",
                "client_id": "{REST_API_KEY}",
                "redirect_uri": "{REDIRECT_URI}",
                "code": "{AUTHORIZATION_CODE}"
            }
            - client_id & redirect_uri 는 Authorization Code 발급 시, 사용했던 값과 동일해애ㅑ 한다.

        2. 카카오 인증 서버RK Authorization Code 를 검증한 뒤 Access/Refresh Token 을 발급

            Response Param

            {
                "access_token": "ACCESS_TOKEN",
                "refresh_token": "REFRESH_TOKEN",
                "expires_in": 3600,
                "refresh_token_expires_in": 2592000
            }

    ● 카카오 인증 서버에 Access Token 요청 이유?

        - 카카오 인증 서버에서 Access Token 을 발급 해준다.
            - OAuth2 의 기본 원칙은 Access Token 은 API 서버와 직접 통신을 위한 KEY 이다.
            - 카카오 API 서버는 카카오 인증 서버에서 발급한 Access Token 만 신뢰한다. 

        - 우리가 자체적으로 생성한 JWT는 우리 애플리케이션 내에서만 유효하고, 외부 API(Kakao 등)에 사용할 수 없다.    

    ● 핵심

        1. Access Token 은 카카오 API 서버와의 통신을 위한 것이다.
            - 카카오 API 서버는 카카오 인증 서버가 발급한 Access Token 만 인정.

        2. 백엔드는 자체적인 JWT 발급
            - 클라이언트와 통신할 때는 백엔드가 발급한 JWT를 사용.
            - JWT는 사용자 인증과 세션을 관리하기 위한 용도로 사용.
           

### KakaoMemberDetails - Authentication 객체안에 사용자 정보를 담기 위함

    @RequiredArgsConstructor
    public class KakaoMemberDetails implements OAuth2User {
    
        private final String email;
        private final List<? extends GrantedAuthority> authorities;
        private final Map<String, Object> attributes;
    
        @Override
        public String getName() {
            return email;
        }
    
        @Override
        public Collection<? extends GrantedAuthority> getAuthorities() {
            return authorities;
        }
    
        @Override
        public Map<String, Object> getAttributes() {
            return attributes;
        }
    }

    - step3 의 로그인 성공 후처리 서비스를 구현하기에 앞서 후처리 서비스에서 필요한 정보들을 먼저 구현.

    - 이 클래스는 인증 객체인 Authentication 객체안에 사용자 정보를 담기 위한 클래스로, 인스턴스는 카카오 계정의 email과 사용자의 권한인 authority 가 컬렉션 형태로 필드로 가지게된다. 
    
    - gradle 에 추가한 OAuth2 Client 가 제공하는 OAuth2User interface 의 구현체로 작성해야 Authentication 객체 안에 담을 수 있기 때문에 OAuth2User 를 구현하여 getter 를 오버라이딩 하였다.

### KakaoUserInfo

    ● 개념
    
    - 카카오로부터 받은 사용자 정보를 다루기 위한 커스텀 클래스.
    - Spring Security 의 OAuth2User.getAttributes() 에서 반환된 맵 데이터를 더 편리하게 사용할 수 있도록 캡슐화.

    1. 주요 속성

    카카오 로그인에서 받을 수 있는 사용자 정보

    - id, kakaoo_account, email, profile, nickname, profile_image_url

    2. kakaoUserInfo class

    @RequiredArgsConstructors
    public class KakaoUserInfo {

        private final Map<String, Onject> attributes;

        public String getId() {
            return attributes.get("id").toString();
        }

        public String getEmail() {
            Map<String, Object> kakaoAccount = (Map<String, Object>) attributes.get("kakao_account");
            return kakaoAccount != null ? kakaoAccount.get("email").toString() : null;
        }

        public String getNickname() {
            Map<String, Object> kakaoAccount = (Map<String, Object>) attributes.get("kakao_account");
            if (kakaoAccount != null) {
                Map<String, Object> profile = (Map<String, Object>) kakaoAccount.get("profile");
                return profile != null ? profile.get("nickname").toString() : null;
            }
        return null;
        }
    }

    3. 클래스 등록 및 설정 방법

    - 카카오 개발자 콘솔에 애플리케이션 등록.
    - Redirect URI 등록:
        ex) http://localhost:8080/login/oauth2/code/kakao
    - REST API Key 와 Client Secret Key 복사.

    4. Spring Security (application.yml)

        spring:
            security:
                oauth2:
                client:
                    registration:
                    kakao:
                        client-id: "YOUR_REST_API_KEY"
                        client-secret: "YOUR_CLIENT_SECRET"
                        scope:
                        - profile_nickname
                        - account_email
                        redirect-uri: "{baseUrl}/login/oauth2/code/kakao"
                        client-name: "Kakao"
                    provider:
                    kakao:
                        authorization-uri: https://kauth.kakao.com/oauth/authorize
                        token-uri: https://kauth.kakao.com/oauth/token
                        user-info-uri: https://kapi.kakao.com/v2/user/me
                        user-name-attribute: id

    5. Security Config 

    @EnableWebSecurity
    public class SecurityConfig {

        @Bean
        public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

            hhtp 
                .csrf().disable()
                .authorizeRequests()
                    .antMatchers("/login/**", "/oauth2/**).permitAll()
                    .anyRequest().authenticated()
                .and()
                .oauth2Login()
                    .userInfoEndpoint()
                    .userService(oAuth2UserService());
            return http.build();
        }

        @Bean
        public DefaultOAuth2UserService oAuth2UserService()
        {
            return new DefaultOAuth2UserService() {
                @Override
                public OAuth2User loadUser(OAuth2UserRequest userRequest) {
                    OAuth2User oAuth2User = super.loadUser(userRequest);
                    return oAuth2User; // 커스텀 로직 추가 가능.
                }
            }
        }
    }

    6. 동작 흐름

    - OAuth2 인증 : 카카오 서버로 redirect -> 인증 후 AccessToken 발급
    - 사용자 정보 요청 : DefaultOAuth2UserService 가 OAuth2User 객체를 바환.
    - 사용자 정보 매핑 : OAuth2User.getAttributes() 를 사용해 카카오 정보를 추출.
    - KakaoUserInfo 활용 : kakaoUserInfo 객체를 통해 필요한 사용자 정보를 쉽게 접근.

### KakaoUserInfo - 로그인 성공 시, kakaoMemberDetails 로 캐스팅 위한 정보

    public class KakaoUserInfo {

        public static final String KAKAO_ACOUNT = "kakao_account";
        public static final String EMAIL = "email";

        private Map<String, Object> attributes;

        public kakaoUserInfo(Map<String, Object> attributes) {
            this.attributes = attributes;
        }

        public String getEmail() {

            ObjectMapper objectMapper = new ObjectMapper();
            TypeRefrence<Map<String, Object>> typeReferencer = new TypeReference<Map<String, Object>>() {};
        

            Object kakaoAccount = attributes.get(KAKAO_ACCOUNT);
            Map<String, Object> account = objectMapper.convertValue(kakaoAccount, typeReferencer);

            return (String) account.get(EMAIL);
        }
    }

    로그인에 성공하게 되면 후처리 서비스에서 받은 사용자 정보들을 kakaoMemberDetails 로 캐스팅하기위한 정보들이다.

### KakaoMemberDetailsService - 카카오 로그인 성공 후처리 서비스 객체

    step2 에서 토큰을 통해 사용자 정보를 받게 되면 이렇게 DefaultOAuth2UserService 의 OAuth2UserRequest 객체에 사용자 정보가 담기게 된다.

    public class DefaultOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> 
    {
        private Coverter<OAuth2UserRequest, RequestEntity<?>> requestEntityConverter = new OAuth2UserRequestEntityConverter();
    }

    후처리 서비스는 사용자가 비즈니스 로직에 맞게 커스텀 구현해야 한다.
    떄문에 이 DefaultOAuth2UserService 를 상속받아 요구사항에 맞게 커스터마이징 해보자.

    @Service
    @RequiredArgsConstructor
    public class KakaoMemberDetailsService extends DefaultOAuth2UserService {

        private static final String PREFIX = "낯선";

        private final MemberRepository memberRepository;

        @Transactional
        @Override
        public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {

            OAuth2User oAuth2User = super.loadUser(userRequest);
            KakaoUserInfo kakaoUserInfo = new KakaoUserInfo(oAuth2User.getAttributes());

            Member member = memberRepository.findByEmail(kakaoUserInfo.getEmail())
                .orElseGet(() ->
                        memberRepository.save(
                            Member.builder()
                                    .email(kakaoUserInfo.getEmail())
                                    .role(Role.USER)
                                    .nickName(PREFIX)
                                    .gender(Gender.NONE)
                                    .updateAgeCount(0)
                                    .updateGenderCount(0)
                                    .build()
                        )
                );

            SimpleGrantedAuthority authority = new SimpleGrantedAuthority(member.getRole().name());

            return new KakaoMemberDetails(String.valueOf(member.getEmail()), Collections.singletonList(authority), 
            oAuth2User.getAttributes());
        }
    }

    DefaultOAuth2UserService 의 loadUser 메소드는 로그인 성공 후 동작하는 메소드로 실질적으로 사용자 정보를 어떻게 처리할 지 작성해야 하는 구현부이다.

        OAuth2User oAuth2User = super.loadUser(userRequest);

    1. 먼저 request 로부터 사용자 정보가 담긴 객체를 꺼내온다.
 
        KakaoUserInfo kakaoUserInfo = new KakaoUserInfo(oAuth2User.getAttributes());

    2. 사용자 정보가 담긴 객체에서 사용자 정보를 꺼내서 KakaoUserInfo 에 담는다.

### 왜 KakaoUserInfo 가 필요한가?

    - 사용자 정보가 담긴 OAuth2User 객체의 attributes 를 출력해보면, 로그인 유저 정보가 아래와 같이 Map<String, Object> 형태로 저장된 것을 볼 수 있다.

    - 그 중에서도 DB에 저장하기 위해 필요한 이메일 정보는 Map<String(key:kakao_account), Map<String, Object>> 형태로 저장되어 있기 때문에 이 정보들을 편하게 parsing 해오기 위해 KakaoUserInro 클래스를 만들어 준다.

    oauth2User.getAttributes() = {
                setPrivacyInfo=true, 
                id=3292176436,
                connected_at=2024-01-21T09:45:17Z,
                properties={nickname=김동웅}, 
                kakao_account = {
                        profile_nickname_needs_agreement=false,        
                        profile={nickname=김동웅}, 
                        has_email=true, 
                        email_needs_agreement=false, 
                        is_email_valid=true, 
                        is_email_verified=true, 
                        email=kdo0422@nate.com
                    }
    }   

    KakaoUserInfo를 아래처럼 oauth2User.getAttributes() 로 생성하면,

    KakaoUserInfo kakaoUserInfo = new KakaoUserInfo(oAuth2User.getAttributes());

    이제 kakaoUserInfo.getEmail() 을 통해,

    1. Map<String, Map<String, String>> attributes 에 담긴 kakao_account key 의 value 를 가져와서,

    2. kakao_account key 의 value 인 Map<String, String> 에서 email 을 가져오도록 함으로써, 실제 DB에 INSERT 할 회원 정보 중 하나인 email 을 편하게 가져올 수 있다.

    본론으로 돌아와 후처리 메소드인 loadUser() 을 보면,

    - 사용자 정보를 통해 kakaoUserInfo 를 생성해주고, kakaoUserInfo 로 회원 리포지토리에 해당 회원이 존재하는 지 조회하여 없으면 생성 후, 리턴하고 있으면 리턴하는 방식으로 구현

    - repository 에 회원이 없는 경우에는 email 필드만 카카오 로그인 계정으로 등록해주고, 나머지 필드들은 요구사하엥 맞게 기본적인 정보들을 Builder 로 생성하여 repository 에 저장해준다.

        SimpleGrantedAuthority authority = new SimpleGrantedAuthority(member.getRole().name());

    3. OAuth2User interface 를 구현한 kakaoMemberDetails 에 회원의 권한 정보를 담기 위해 저장된 회원으로부터 권한 정보를 꺼내오고, SimpleGrantedAuthority 객체를 만들어 주었다.

        return new kakaoMemberDetails(String.valueOf(member.getEmail()),
        Collections.singletonList(authority),
        oAuth2User.getAttributes());

    4. KakaoMemberDetails 는 권한 정보를 Collection 형태의 필드로 갖기 때문에 3에서 만든 권한을 형태에 맞게 변경하여 KakakoMemberDetails 를 생성 후 리턴한다.    

        ● 어디에 return 되는가?

        -  여기서 return 된 UserDetails 객체는 사용자 인증 정보를 나타내기 위해 Authentication 객체에 담겨지고, 이 Authentication 객체는 사용자의 인증 상태를 나타내며, SecurityContext 에 저장된다.

    마지막으로 구현한 후처리 서비스 클래스를 SecurityConfig 에 다음과 같이 등록해주면 끝이다.

    .oauth2Login(oAuth2Login -> {
        oAuth2Login.userInfoEndPoint(userInfoEndpointConfig ->
                userInfoEndPointConfig.userService(kakaoMemberDetailsService));
    });

### 클래스 간 관계 

    KakaoMemberDetailsService
        ㄴ DefaultOAuth2UserService
            ㄴ loadUser() 호출
                ㄴ KakaoUserInfo 생성 (attributes parsing)
                    ㄴ getEmail()
                ㄴ MemberRepository 로, DB 조회/저장
                    ㄴ Member Entity 셍성 (필요 시)
                ㄴ KakaoMemberDetails 반환 (사용자 정보 포함)

### 전체 흐름

    1. OAuth2 로그인 요청
        - 사용자가 카카오 OAuth2 로 로그인하면, kakaoMemberDetailsService 의 loadUser() 가 호출된다.

    2. 사용자 정보 가져오기
        - DefaultOAuth2UserService 의 super.loadUser() 가 실행되어 카카오 API에서 사용자 속성을 가져온다.
      
    3. 속성 파싱
        - KakaoUserInfo 가 속성을 분석하여 이메일과 같은 필요한 정보를 추출.

    4. DB 조회 및 저장
        - Email 기준으로 MemberRepository 를 통해 사용자를 조회한다.
        사용자가 없으면 새로 생성하여 저장한다.

    5. 사용자 정보 반환
        - kakaoMemberDetails 객체를 생성하여 사용자 이메일, 권한, 속성을 반환한다. 이 객체는 스프링 시큐리티의 인증 시스템에서 사용된다.


