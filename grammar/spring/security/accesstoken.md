## accesstoken 흐름 정리

### 1. 클라이언트 요청

    - browser/front 에서 API 요청 시, Authorization 헤더에 Bearer AccessToken 포함
    - RefreshToken 은 보통 쿠키로 전송됨
    
    http
        GET /api/mypage
        Authorization: Bearer <accessToken>
        Cookie: refreshAuthorization=Bearer <refresh_token>

### 2. JWTFilterV3#doFilterInternal 에서 필터링 시작 

    log.info("[JWT Filter] 요청 URI: {}", request.getRequestURI());
    log.info("[JWT Filter] Authorization 헤더: {}", authorization);
    log.info("[JWT Filter] RefreshAuthorization 쿠키: {}", refreshAuthorization);

    - 요청의 헤더와 쿠키를 확인해 로그로 남겨, 토큰이 아예 누락되었는지 / 포맷이 잘못됐는지 등을 1차적으로 확인할 수 있다.

### 3. RefreshToken 먼저 검증

    if (refreshAuthorization == null || !refreshAuthorization.startWith("Bearer ")) {
        log.warn("[JWT Filter] RefreshToken 없음 또는 형식 이상");
        filterChain.doFilter(request, response);
        return;
    }

### 4. RefreshToken 유효성 검증 -> jwtUtil.validateToken(refreshToken)

    if (!jwtUtil.validateToken(refreshToken)) {
        log.warn("[JWT Filter] RefreshToken 유효하지 않음");
        filterChain.doFilter(request, response);
        return;
    }    

    - log.error("validateToken: JWT 유효성 검사 실패. token: {}", token, e);

### 5. AccessToken 존재 + 형식 확인

    if (authorization != null && authorization.startsWith("Bearer ")) {
        ...
    }

    - log.info("[JWT Filter] AccessToken 존재 및 Bearer 확인됨: {}", accessToken); 
  
### 6. AccessToken 만료 여부 확인 → jwtUtil.isExpired(accessToken)

    if (jwtUtil.isExpired(accessToken)) {
        log.warn("[JWT Filter] AccessToken 만료됨");
        filterChain.doFilter(request, response);
        return;
    }

    - log.warn("[JWT Filter] AccessToken 만료됨");

### 7. AccessToken 유효 → 인증 시도: getAuthentication(accessToken)    

    Authentication authToken = getAuthentication(accessToken);

### 8. memberId 파싱 시도 → jwtUtil.getMemberId(token)

    log.debug("parseToken: 토큰 파싱 시도 - {}", token);
    log.info("parseToken: 토큰 파싱 성공 - subject: {}, expiration: {}", claims.getSubject(), claims.getExpiration());

### 9. MemberRepository 에서 사용자 조회

    - 사용자가 있을 경우
    Optional<Member> optionalMember = memberRepository.findById(Long.valueOf(memberId));
    log.info("[getAuthentication] memberId from token: {}", memberId);
    log.info("[getAuthentication] 사용자 조회 성공: {}", member.getEmail());

    - 사용자가 없을 경우
    log.warn("[getAuthentication] 사용자 없음 (memberId: {})", memberId);

### 10. CustomUserDetails 생성 + SecurityContextHolder에 저장    

    log.info("[getAuthentication] CustomUserDetails 생성 완료: {}", customUserDetails.getUsername());
    log.info("[JWT Filter] SecurityContext 에 인증 정보 설정 완료: {}",     authToken.getPrincipal());

### 순서 요약    

| 순서 | 위치 / 클래스                     | 설명                                                                 |
|------|-----------------------------------|----------------------------------------------------------------------|
| 1    | 프론트엔드                        | 이메일 + 비밀번호로 로그인 요청 (`POST /auth/login`)               |
| 2    | `AuthController.login()`          | 요청을 `MemberService.memberLogin()`에 위임                         |
| 3    | `MemberService.memberLogin()`     | 이메일로 회원 조회 → 비밀번호 일치 시 `Member` 객체 반환            |
| 4    | `JWTUtil.createJwt()`             | 로그인 성공 시 accessToken, refreshToken 생성                       |
| 5    | `CookieUtil.createHttpOnlyCookie()` | refreshToken을 HttpOnly 쿠키로 생성                              |
| 6    | `ResponseEntity`                  | accessToken은 응답 헤더에, refreshToken은 쿠키로 클라이언트에 응답 |
| 7    | 프론트엔드                        | accessToken은 localStorage 등 저장, refreshToken은 브라우저 쿠키 저장 |
| 8    | 이후 요청 발생 (`GET /api/xxx`)   | accessToken을 Authorization 헤더에 실어 백엔드로 요청              |
| 9    | `JWTFilterV3.doFilterInternal()`  | 요청 헤더에서 accessToken 추출 (`Authorization: Bearer xxx`)       |
| 10   | `JWTUtil.isTokenValid()`          | accessToken 유효성 검사                                              |
| 11   | `JWTUtil.extractEmail()`          | accessToken에서 이메일(또는 ID 등 claims) 파싱                      |
| 12   | SecurityContext 설정              | 이메일 기반 사용자 정보를 `UserDetails`로 로드 → 인증 완료         |
| 13   | 컨트롤러 도달                     | 이후 컨트롤러에서 인증된 사용자로 요청 처리                         |
 

### 추가 CORS + JWT 흐름

| 단계 | 설명 |
|------|------|
| 1 | 프론트에서 로그인 요청 (예: POST `/login`) |
| 2 | 백엔드가 `accessToken`, `refreshToken` 생성 |
| 3 | `refreshToken`을 **Set-Cookie** 헤더에 담아 응답 |
| 4 | CORS 설정이 없으면 브라우저가 쿠키를 받지 못함 |
| 5 | 다음 요청 시 브라우저가 쿠키 자동 포함 (CORS에서 `allowCredentials: true` 필요) |
| 6 | 백엔드는 쿠키에서 `refreshToken`을 읽고 인증 처리 |