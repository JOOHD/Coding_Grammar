##  New Shop 프로젝트: JWT + OAuth2 인증/인가 흐름 정리

### 전체 시퀀스 

[사용자]
   ↓ (1) OAuth2 로그인 요청
[Frontend] → /oauth2/authorization/google
   ↓ (2) 구글 인증 성공 → 사용자 정보 반환
[SecurityConfig → OAuth2LoginFilter]
   ↓ (3) 사용자 정보 로딩
[CustomOAuth2UserService] → (4) DB 저장/조회 및 CustomOAuth2User 반환
   ↓
[OAuth2AuthenticationSuccessHandler]
   ↓ (5) AccessToken + RefreshToken 생성 (JWTUtil)
   ↓ (6) RefreshToken 저장 (RedisService)
   ↓ (7) AccessToken(HttpOnly 쿠키) 응답에 포함

=== 이후 요청 흐름 ===

[사용자 → 클라이언트]
   ↓ (1) JWT AccessToken 포함 요청
[JwtAuthenticationFilter]
   ↓ (2) JWT 유효성 검사 (JWTUtil)
   ↓ (3) 토큰에서 사용자 정보 추출 (TokenResolver)
   ↓ (4) SecurityContextHolder 에 인증 객체 저장
   ↓ (5) 인증된 상태로 컨트롤러 로직 실행

[만료 시 → 재발급 요청 /api/token/refresh]
   ↓
[TokenController → JWTUtil + RedisService] → AccessToken 재발급

### Google OAuth2 인증 & 사용자 정보 요청 흐름 (Authorization Server + Resource Server)

[1] 사용자가 로그인 버튼 클릭
     ↓
[2] 클라이언트 → Authorization Server (Google) 요청
    https://accounts.google.com/o/oauth2/v2/auth
    ?client_id=...
    &redirect_uri=...
    &response_type=code
    &scope=email profile

     ↓
[3] 사용자 Google 계정 로그인 (Authentication)
     ↓
[4] 로그인 성공 → Authorization Server → 인가 코드 발급 (code)
     ↓
     redirect_uri?code=AUTH_CODE

[5] 클라이언트 서버 → Authorization Server 로 토큰 요청 (인가 코드 사용)
    POST https://oauth2.googleapis.com/token
    {
        code: AUTH_CODE,
        client_id,
        client_secret,
        redirect_uri,
        grant_type: authorization_code
    }

     ↓
[6] Authorization Server → Access Token (+ Refresh Token) 발급
    {
      access_token: "...",
      refresh_token: "...",
      expires_in: 3600,
      token_type: "Bearer"
    }

[7] 클라이언트 → Resource Server(Google API)에 사용자 정보 요청
    GET https://www.googleapis.com/oauth2/v3/userinfo
    Authorization: Bearer {access_token}

     ↓
[8] Resource Server → 사용자 정보 응답
    {
        "sub": "1234567890",
        "name": "John Doe",
        "email": "john@example.com",
        ...
    }


### 클래스별 역할 요약

| 클래스                              | 역할 (한 문장 설명)                                                                           |
|-----------------------------------|------------------------------------------------------------------------------------------|
| `SecurityConfig`                  | 전체 Spring Security 인증/인가 필터 설정 및 JWT/OAuth2 필터 연결 담당                             |
| `JwtAuthenticationFilter`         | 모든 요청에서 JWT AccessToken 을 검증하고 인증 객체를 SecurityContext 에 등록함                    |
| `JWTUtil`                         | JWT 토큰 생성, 파싱, 검증, 만료 확인 등의 핵심 유틸리티 제공                                           |
| `TokenResolver`                   | JWT Claims 로부터 사용자 이메일/권한 등을 추출하는 헬퍼 클래스                                     |
| `TokenController`                 | Refresh Token 기반 Access Token 재발급 API 제공                                        |
| `RedisService`                    | Redis 저장소를 통해 Refresh Token 및 블랙리스트 관리 수행                                 |
| `OAuth2AuthenticationSuccessHandler` | OAuth2 로그인 성공 시 JWT 생성, Redis 저장, 응답 쿠키 설정 처리                          |
| `CustomOAuth2UserService`         | 구글로부터 받은 사용자 정보를 DB에 저장하거나 기존 사용자 반환                                 |
| `CustomOAuth2User`                | OAuth2User 를 상속한 사용자 인증 객체로, SecurityContext 에 등록되는 유저 모델              |
| `CustomUserDetails`              | JWT 기반 인증 시 사용하는 UserDetails 구현체                                           |
| `AuthenticationEntryPointV2`     | 인증 실패 시 (401) 에러 응답 포맷 및 메시지 정의                                        |
| `JWTFilterV3`                     | OncePerRequestFilter 로 AccessToken 이 있는 경우 검증하여 인증 처리 (JwtAuthenticationFilter 와 유사 기능) |


## 🧩 클래스별 역할 요약 (기능과 관계 중심 정리)

1. SecurityConfig

전체 Spring Security 설정 클래스. JWT 필터와 OAuth2 로그인 필터를 등록하며 인증 흐름의 진입점 역할 수행.

2. JwtAuthenticationFilter

JWT가 있는 요청에서 토큰을 검증하고 사용자 인증 객체를 생성해 SecurityContext에 등록함.

3. JWTUtil

JWT 생성/파싱/만료 검증 등의 기능 제공. AccessToken/RefreshToken 모두 이 클래스를 통해 생성함.

4. TokenResolver

JWT Claims로부터 사용자 이메일/권한을 추출해 인증 객체 생성 시 활용됨.

5. TokenController

클라이언트가 RefreshToken으로 AccessToken을 재발급받는 엔드포인트 제공.

6. RedisService

RefreshToken 저장소 및 블랙리스트 관리 수행. 로그아웃 및 재발급 관련 로직에서 활용됨.

7. CustomAuthenticationSuccessHandler

OAuth2 로그인 성공 시 JWT 토큰을 발급하고, RefreshToken은 Redis에 저장 후 응답 쿠키에 AccessToken 세팅.

8. CustomOAuth2UserService

구글 OAuth2 사용자 정보 로딩, DB 사용자 등록 또는 조회, 인증 객체 생성 역할 수행.

9. CustomOAuth2User

OAuth2User 구현체로 SecurityContext에 등록될 사용자 정보 표현.

10. CustomUserDetails

JWT 기반 인증 시 사용할 UserDetails 구현체. 인증 필터에서 이 객체로 사용자 정보 구성.

11. AuthenticationEntryPointV2

인증 실패 시 JSON 기반 에러 응답 반환. 예: AccessToken 유효하지 않을 때 401 응답 생성.

12. JWTFilterV3

OncePerRequestFilter 기반 필터로 JwtAuthenticationFilter와 유사. 요청당 한 번 JWT 검사 수행.


### 클래스 관계 다이어그램 (간략 관계 + 기능 중심)

[SecurityConfig]
  ├── SecurityFilterChain 구성
  │     ├─ JwtAuthenticationFilter / JWTFilterV3 등록
  │     │     ├─ JWTUtil: 토큰 생성, 검증, 만료시간 확인 등 JWT 관련 핵심 로직 처리
  │     │     └─ TokenResolver: HTTP 요청에서 토큰 추출 및 파싱 지원
  │     ├─ OAuth2 로그인 설정
  │     │     ├─ CustomOAuth2UserService: OAuth2 공급자(구글 등)로부터 받은 사용자 정보 처리 및 사용자 매핑
  │     │     └─ CustomAuthenticationSuccessHandler: OAuth2 로그인 성공 후 JWT 토큰 생성 및 Redis에 RefreshToken 저장, 쿠키 설정 담당
  │     └─ AuthenticationEntryPointV2: 인증 실패 시 401 Unauthorized 응답 처리
  │
[JwtAuthenticationFilter or JWTFilterV3]
  ├── doFilterInternal 메서드 내에서
  │     ├─ TokenResolver로부터 AccessToken 추출
  │     ├─ JWTUtil로 토큰 유효성 검사 (서명, 만료 시간, 구조 등)
  │     ├─ RedisService 또는 RedisTemplate로 블랙리스트 여부 체크
  │     ├─ 블랙리스트에 없으면
  │     │     └─ JWTUtil 또는 CustomUserDetailsService를 통해 사용자 정보 로드 및 인증 객체 생성
  │     │     └─ SecurityContext에 인증 정보 저장 (UsernamePasswordAuthenticationToken 등)
  │     └─ 블랙리스트에 있거나 토큰이 유효하지 않으면 인증 거부 처리
  │
[TokenController]
  ├── RefreshToken 검증 (RedisService 사용)
  ├── RefreshToken 만료 전 재발급 요청 처리
  ├── JWTUtil로 AccessToken 재생성
  └── 클라이언트에게 새로운 AccessToken 반환
  │
[CustomOAuth2UserService]
  ├── OAuth2UserRequest 기반으로 OAuth2 공급자에서 사용자 정보 요청
  ├── 사용자 정보를 커스텀 DTO 또는 엔티티(CustomOAuth2User)로 매핑
  └── 신규 회원이면 DB에 저장하거나 기존 회원과 매핑 처리
  │
[CustomAuthenticationSuccessHandler]
  ├── OAuth2 로그인 성공 시 호출
  ├── JWTUtil로 AccessToken, RefreshToken 생성
  ├── RedisService로 RefreshToken Redis 저장 (키: "RT:{userId}")
  ├── HTTP 응답 쿠키에 AccessToken, RefreshToken 세팅 (HttpOnly, Secure 설정)
  └── 로그인 후 리다이렉트 또는 응답 전송 처리
  │
[AuthenticationEntryPointV2]
  ├── 인증 예외 발생 시 인터셉트
  ├── 401 Unauthorized 상태 코드 설정
  ├── JSON 형태로 에러 메시지 응답 전송
  └── 클라이언트가 인증 실패를 인지하도록 처리
  │
[CustomOAuth2User] / [CustomUserDetails]
  ├── Spring Security의 UserDetails 또는 OAuth2User 인터페이스 구현체
  ├── 인증 후 SecurityContextHolder에 등록되어 보안 컨텍스트 유지
  ├── 사용자 권한(ROLE) 및 식별자(userId, email 등) 포함
  └── 인증 이후 서비스 계층에서 사용자 정보 접근용으로 활용
