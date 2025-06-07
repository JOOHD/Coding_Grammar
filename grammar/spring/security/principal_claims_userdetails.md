## principal vs claims vs UserDetails

### 개념 비교

| 용어  | 의미  | 사용 위치  | 예시| 관계 |
| --------------- | ----------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------ |
| **Principal**   | 인증된 사용자를 나타내는 객체| `Authentication.getPrincipal()`| `username`, `CustomUserDetails`, 혹은 JWT에서 추출한 사용자 정보  | `UserDetails` 또는 `claims` 일부로 구성됨  |
| **Claims**  | JWT 내부 payload에 들어가는 key-value 데이터  | JWT 파싱 시 사용 (`JWTUtil.parseClaims`)| `{ "sub": "joo", "role": "USER", "exp": 1717000000 }` | JWT에서 `principal`을 구성할 수 있음|
| **UserDetails** | Spring Security 인증 시 사용하는 사용자 정보 객체 | 인증 과정에서 DB에서 로드됨 (`UserDetailsService.loadUserByUsername`) | 사용자 ID, 패스워드, 권한 등 포함 | `principal`로 사용됨 (`Authentication.getPrincipal()`의 타입) |

### 관계 흐름 (JWT 기반 로그인)

1. 클라이언트 → JWT Access Token 전송
2. JWTFilter (OncePerRequestFilter) →
   JWTUtil.parseClaims(token) → claims 추출

3. claims에서 username/role 꺼내서
   → CustomUserDetails(userId, authorities) 생성
   → UsernamePasswordAuthenticationToken(principal=UserDetails, authorities)

4. Authentication 저장: SecurityContextHolder.getContext().setAuthentication(authentication)

5. 이후 @AuthenticationPrincipal 또는 SecurityContextHolder에서 principal 접근

### 요약

| 상황                                | principal이 무엇인가?                                             |
| --------------------------------- | ------------------------------------------------------------ |
| DB 기반 인증                          | `UserDetails` 객체                                             |
| JWT 기반 인증 (직접 구현)                 | `claims`에서 추출한 사용자 ID 등 정보 (ex. `String`, `Map`, 혹은 커스텀 DTO) |
| JWT 기반 인증 (UserDetailsService 연동) | claims에서 ID를 꺼낸 후 → DB 조회 → `UserDetails` 객체로 principal 구성   |

### tip

- claims : JWT 내부 데이터
- UserDetails : Spring Security 가 요구하는 사용자 정보 객체
- principal : 인증된 사용자 자체를 가리키는 값, 위 둘 중 하나일 수 있다.

## Spring Security + JWT 기반 인증 흐름에서의 작동

1. JWT 기반 인증 (직접 Claims 사용)

    - 흐름
    [1] 클라이언트 → JWT 토큰 전송 (Authorization: Bearer ...)
    [2] 필터(JWTFilter) → JWTUtil로 claims 추출
    [3] claims에서 사용자 ID와 역할 꺼냄
    [4] principal = claims.get("sub") (ex. "user@example.com")
    [5] Authentication 객체 생성 (principal = String, authorities = role)
    [6] SecurityContextHolder에 저장

    - 예시 코드
    String token = resolveToken(request);
    Claims claims = jwtUtil.parseClaims(token);
    String email = claims.getSubject(); // principal
    String role = claims.get("role", String.class);
    List<GrantedAuthority> authorities = List.of(new SimpleGrantedAuthority(role));
    Authentication auth = new UsernamePasswordAuthenticationToken(email, null, authorities);
    SecurityContextHolder.getContext().setAuthentication(auth);

2. JWT + UserDetailsService 기반 인증

    - 흐름
    [1] JWT 토큰 수신
    [2] JWTUtil로 claims 추출 → 사용자 ID(email 등)
    [3] UserDetailsService.loadUserByUsername(email) 호출
    [4] principal = CustomUserDetails 객체
    [5] Authentication 객체 생성 (principal = UserDetails, authorities 포함)
    [6] SecurityContextHolder에 저장

    - Spring Security 구조에 완벽히 부합
    
3. form 로그인 / session 기반 인증 (참고)
    -> JWT 없이 기본 로그인으로 인증하는 경우

    - 흐름
    [1] 사용자 로그인 폼 → username/password 전송
    [2] AuthenticationManager가 UserDetailsService.loadUserByUsername 호출
    [3] 사용자 검증 후 UserDetails 생성
    [4] Authentication 객체에 principal = UserDetails 저장
    [5] 세션에 Authentication 저장됨 (기본 Spring Security 동작)

### 요약 비교

| 흐름                | principal 값  | DB 접근 여부 | 특징            |
| ----------------- | ------------ | -------- | ------------- |
| JWT → Claims 기반   | String / Map | ❌ 없음     | 가볍지만 커스텀 필요   |
| JWT + UserDetails | UserDetails  | ✅ 있음     | Spring 구조에 충실 |
| Form 로그인          | UserDetails  | ✅ 있음     | 세션 기반, 일반 로그인 |
