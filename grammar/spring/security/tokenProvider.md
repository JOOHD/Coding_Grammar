##  Joo_Cafe - TokenProvider class 분석 및 학습

### TokenProvider class code

    @Slf4j
    @Component
    @RequiredArgsConstructor
    public class TokenProvider {
    
        private static final String KEY_ROLE = "role";
        private static final long TOKEN_EXPIRE_TIME = 1000 * 60 * 60 * 24; // 1일 (24시간)
    
        private final AuthService authService;
    
        @Value("${spring.jwt.secret}")
        private String secretKey;
    
        // 토큰 생성(발급)
        public String generateToken(Long userId, String email, Role role) {
            Claims claims = Jwts.claims() // 사용자의 정보를 저장하기 위한 claim
                .setSubject(email)
                .setId(userId + "");
            claims.put(KEY_ROLE, role);
    
            Date now = new Date();
            Date expiredDate = new Date(now.getTime() + TOKEN_EXPIRE_TIME);
    
            return Jwts.builder()
                .setClaims(claims)
                .setIssuedAt(now) // 토큰 생성 시간
                .setExpiration(expiredDate) // 토큰 만료 시간
                .signWith(SignatureAlgorithm.HS512, secretKey) // 사용할 암호화 알고리즘, 시크릿 키
                .compact();
        }
    
        // jwt 에서 인증정보 추출
        public Authentication getAuthentication(String token) {
            UserDetails userDetails = authService.loadUserByUsername(getEmail(token));
            List<GrantedAuthority> authorities = new ArrayList<>(userDetails.getAuthorities());
            return new UsernamePasswordAuthenticationToken(userDetails, "", authorities);
        }
    
        // 토큰에서 사용자 이메일 추출
        public String getEmail(String token) {
            return parseClaims(token).getSubject();
        }
    
        // 토큰에서 사용자 id 추출
        public Long getId(String token) {
            return Long.parseLong(parseClaims(removeBearerFromToken(token)).getId());
        }
    
        // 토큰 유효성검사
        public boolean validateToken(String token) {
            if (!StringUtils.hasText(token)) {
                return false;
            }
    
            Claims claims = parseClaims(token);
            return !claims.getExpiration().before(new Date());
        }
    
        // 토큰에서 클레임 정보 추출
        private Claims parseClaims(String token) {
            try {
                return Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token).getBody();
            } catch (ExpiredJwtException e) {
                log.warn("만료된 JWT 토큰입니다.");
                return e.getClaims();
            }
        }
    
        // 토큰 인증 타입 제거
        private String removeBearerFromToken(String token) {
            return token.substring(JwtAuthenticationFilter.TOKEN_PREFIX.length());
        }
    }

### 코드 분석

    ● 필드 설명
        - KEY_ROLE : JWT에서 사용자의 역할을 저장할 key 값이다.
        - TOKEN_EXPIRE_TIME : 토큰의 유효 기간을 1일(24시간)로 설정합니다.
        - authService : 사용자 정보를 조회하기 위한 서비스입니다.
        - secretKey : JWT 토큰을 암호화할 때 사용하는 비밀 key입니다.

    ● @Value("${spring.jwt.secret}")
        - 외부 설정 파일(application.properties) 같은 파일에서 값을 가져와서, 
            해당 필드에 주입하는 역할이다.

            ex)
                application.properties 예를 들자면,
                spring.jwt.secret = mySecretKey

                ${spring.jwt.secret} = mySecretKey 값을 가져온다.
                secretKey = JWT 토큰을 서명 및 검증에 사용하는 비밀 키.

    ● 메서드 설명
        1. generateToken(Long userId, String email, Role role)
            - JWT 토큰을 생성하는 목적의 메서드.
            - Claim claim : 사용자의 정보(email, role)를 담은 객체, 여기서
                                setSubject(email)로 이메일을, setId(userId)로 사용자 ID를 설정.
            - claims.put(KEY_ROLE, role) : 사용자 역할을 클레임에 추가.
            - signWith(SignatureAlgorithm.HS512, secretKey) 
                HS512 알고리즘과 secretKey를 사용하여 토큰을 서명

                ● Claim 이란?
                    - JWT(Jason Web Token)에서 사용자의 정보나 추가 데이터를 담는 부분.
                        JWT는 크게 세 부분으로 나뉘어 있는데, 각각 Header, Payload, signature로 구성된다.
                        그 중 Payload가 바로 Claim을 담고 있는 부분이다.
                        - Claim은 사용자의 데이터(email, auth, id등)를 토큰에 저장할 수 있게 해준다.
                        - JWT는 Claim을 기반으로 사용자의 정보를 안전하게 교환.
                        - Claim은 주로 두 가지로 나뉜다.
                            1. 등록된 클레임
                                subject, issuer, expireation, time등 사전에 정의된 클레임
                            2. 비공식 클레임
                                개발자가 필요에 따라 추가하는 클레임, role, userId 같은

                    - TokenProvider class에서는 claims에 담긴 정보는
                        1. subject : email(사용자 식별을 위한 주제)
                        2. id : 사용자 ID
                        3. role : 사용자 역할(권한)
                        
                        - 위 정보들은 JWT 토큰에 포함되어, 토큰이 생성되면 클라이언트에 전달된다.
                          서버는 이후 요청에서 이 토큰을 받아 클레임 정보를 추출하고, 해당 사용자가 누구인지, 어떤 권한을 가진지 검증.

                    - Token을 String으로 선언하는 이유
                        - JWT가 기본적으로 Base64로 인코딩된 문자열.

            - Date expireDate : 토큰의 만료 시간을 현재 시간으로부터 24시간 후로 설정.
            - Jwts.builder() : JWT 토큰을 생성, 여기서 사용자 정보, 생성 시간, 만료 시간, 
                                암호화 알고리즘과 비밀 키를 설정하고 토큰을 최종적으로 만든다.

        2. getAuthentication(String token)
            - JWT 토큰에서 인증 정보를 추출하는 메서드.
            - authService.loadUserByUsername(email), UserDetails에서 (Authorities) 추출    
                - 토큰에서 사용자 이메일을 추출하여 authService로 사용자 정보를 로드하고, 이를 바탕으로 
                UsernamePasswordAuthenticationToken 을 생성하여 인증 정보 반환.

        3. getEamil(String token)
            - JWT 토큰에서 사용자의 이메일을 추출하는 메서드.
            - 토큰에서 parseClaims()를 통해 클레임 정보를 추출하고, getId()를 호출하여 사용자의 ID를 반환.
        
        4. validateToken(String token)
            - JWT 토큰의 유효성을 검사하는 메서드.

        5. parseClaims(String token)
            - JWT 토큰을 해석하고, 유효하면 클레임을 반환, 만약 토큰이 만료되었다면
                ExpiredJwtException 을 던지며, 만료된 클레임을 반환.

        6. removeBearerFromToken(String token)
            - 토큰에서 Bearer 라는 인증 타입을 제거하는 메서드아디.
                JWT 토큰 앞에 붙어있는 Bearer 문자열을 제거하고 실제 토큰만 추출한다.
                    - TOKEN_PREFIX = "Bearer"이며, 접두사를 제거한 순수 jwt token만 추출
                    

### 로직 흐름
        
    1. 사용자가 로그인 시, generateToken() 메서드로 JWT 토큰을 생성하여 발급.
        - 사용자 정보는 JWT의 Claims에 (ID, email, role)등의 정보가 포함.

    2. 클라이언트는 이후 요청 시 헤더에 이 JWT 토큰을 포함하여 서버에 요청을 보낸다.
        - 생성된 토큰은 클라이언트에게 전달되어 저장, 이후 API 요청 시, 
            HTTP Header의 Authorization 필드에 Bearer <token> 형식으로 토큰을 포함하여 전송.
                - Authorization : HTTP 요청 헤더의 이름
                - Bearer : 인증 방식을 나타내는 접두어 
                - JWT 토큰 : 서버에서 발급한 JSON Web Token(사용자의 정보가 포함되어 있다.)
                
                ex)
                    Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9(JWT header)
                                          eyJzdWIiOiJqb2huZG9lQGV4YW1wbGUuY29t(payload(사용자 정보))
                                          SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_a(서명(토큰 위조를 방지)

    3. 서버는 요청을 받을 때 getAuthentication()을 통해 토큰을 해석하고, 사용자의 인증 정보를 추출.
        - TokenProvider를 사용하여 토큰을 검증
        - validateToken 메서드를 통해 토큰의 유효성을 검사
        - getAuthentication 메서드를 통해 Authentication 객체를 생성하여 Spring security의 컨텍스트에 설정.

    4. Authentication 객체에 포함된 역할 정보를 바탕으로 사용자의 접근 권한을 결정.

### 정리
    TokenProvider 
        - JWT 토큰의 생성, 검증, 인증 정보 추출 등을 담당하는 커스텀 클래스
    enum class의 관계
        - Role과 같은 enum을 사용하여 사용자 역할을 명확하게 정의하고, 이를 토큰에 포함시켜 인증 및 인가 과정에서 활용.
    Spring과 연동
        - Spring의 컴포넌트 스캔, 의존성 주입, Spring Security와 통합하여 인증 및 인가 로직을 처리.










