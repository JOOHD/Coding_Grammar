## JWT 인증 흐름 관계

### 1. HTTP request -> JWTFilterV3
    클라이언트에서 오는 요청은 JWTFilterV3 로 전달.
    이 필터는 HTTP 요청에서 JWT 토큰을 추출하고, 그 토큰을 검증하는 역할.

    [HTTP 요청] → [JWTFilterV3]

### 2. JWTFilterV3 -> JWTUtil (토큰 검증)
    JWTFilterV3 는 JWTUtil 을 사용하여 JWT 토큰의 유효성을 검증.
    여기서 토큰을 파싱하고, 유효한 토큰인지 확인.

    [JWTFilterV3] → [JWTUtil] → [토큰 검증]

### 3. JWTFilterV3 -> CustomUserDetails
    만약 JWT 토큰이 유효하면, JWTFilterV3 는 CustomUserDetails 객체를 생성한다. 이 떄, CustomUserDetails 는 인증된 사용자 정보를 담는 객체이다.
    CustomUserDetails 는 MemberService 와의 연결을 통해 사용자 정보를 가져온다.

    [JWTFilterV3] → [CustomUserDetails] (생성)

### 4. CustomUserDetails -> MemberService (사용자 정보 조회)    
    CustomUserDetails 는 사용자 정보 ex) Member 를 필요로 한다.
    그래서 MemberService 를 통해 DB에서 사용자 정보를 조회,
    CustomUserDetails 는 MemberService 를 통해 사용자 인증 정보를 처리.

    [CustomUserDetails] → [MemberService] → [Member 정보 조회]

### 5. CustomUserDetails → 인증된 사용자 정보 반환
    MemberService에서 받은 정보를 바탕으로, CustomUserDetails는 Spring Security의 UserDetails 형식에 맞게 사용자 정보를 반환합니다.   

    [MemberService] → [CustomUserDetails] → [UserDetails 반환]

### 6. JWTFilterV3 → SecurityContextHolder (인증 정보 설정)
    최종적으로 JWTFilterV3는 CustomUserDetails에서 받은 인증 정보를 SecurityContextHolder에 설정합니다. 이 정보를 기반으로 Spring Security는 이후 요청에 대해 인증된 사용자로 처리합니다.   

    [CustomUserDetails] → [JWTFilterV3] → [SecurityContextHolder] (인증 정보 설정) 

### 사용자 인증 전체 흐름

    1. 사용자 로그인 요청 : id(email), password
    2. CustomUserDetailsService : 사용자 정보 조회 (loadUserByUsername method)
    3. CustomUserDetails : 사용자 세부 정보 (UserDetails interface implements)
    4. JWTFilter : JWT 토큰 인증 (parse(추출), validation(검증), authentication(인증))
                    사용자 정보 SecurityContextHolder 에 저장
    5. SecurityContextHolder : 인증 정보 보관

### 전체 흐름

    [HTTP 요청] → [JWTFilterV3]
                        ↓
             [JWTUtil] → [토큰 검증]
                        ↓
             [CustomUserDetails] → [MemberService] → [Member 정보 조회]
                        ↓
             [CustomUserDetails] → [UserDetails 반환]
                        ↓
             [JWTFilterV3] → [SecurityContextHolder] (인증 정보 설정)

### 관계 요약

    - JWTFilterV3 는 HTTP 요청에서 JWT 토큰을 추출하고, 검증 역할.
    - JWTFilterV3 는 유효한 토큰을 기반으로 CustomUserDetails 를 생성하며, 이 과정에서 MemberService 를 통해 사용자의 정보(Member) 조회.
    - 최종적으로 CustomUserDetails 는 SpringSecurity 의 ContextHolder 에 인증 정보를 설정하여, 이후의 요청에서 인증된 사용자로 처리되도록 한다.