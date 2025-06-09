## BlackList

### JWT 블랙리스트

JWT 는 stateless (서버가 상태를 저장하지 않음)한 토큰 기반 인증 방식이다.
따라서 로그아웃을 하더라도 기존 토큰은 만료 시간 전까지 계속 유효하다. 

이 문제를 해결하기 위해 BlackList 를 도입한다.
블랙리스트는 "더 이상 사용해서는 안 되는 토큰 목록"을 Redis 등에 저장하고, 매 요청마다 확인하여 차단하는 방식이다.

### 왜 JWT에 블랙리스트가 필요한가?

| 상황| 설명   |
| ----------------- | -------------------------------------------------------- |
| 🔐 로그아웃 시 | JWT는 유효 기간이 남아 있어도 서버가 무효화할 방법이 없음 → 블랙리스트에 등록하여 강제로 무효화 |
| 🧑‍💻 관리자 강제 로그아웃 | 사용자가 탈퇴하거나 강제로 세션 종료 시, 기존 토큰 차단 필요  |
| 🦠 보안 위협  | 유출된 JWT를 무효화하고 차단 가능   
  |

### 구현 방식 요약

1. JWTUtil 클래스 : getId() 와 getExpiration() 사용

code
public String getId(String token) {
return parseToken(token).getId(); // JWT의 jti (토큰 고유 식별자)
}

public Date getExpiration(String token) {
return parseToken(token).getExpiration(); // 토큰 만료일시 반환
}

2. 로그아웃 시 블랙리스트 등록 ex) CustomLogoutFilter

String jti = jwtUtil.getId(accessToken);
Date expiration = jwtUtil.getExpiration(accessToken);
long expirationMillis = expiration.getTime();
long currentTime = System.currentTimeMillis();
long remainTime = (expirationMillis - currentTime) / 1000;

redisTemplate.opsForValue().set("BLACKLIST:" + jti, "logout", remainTime, TimeUnit.SECONDS);

   - jti (JWT ID) : 토큰 고유 식별값
   - Redis 에 BlackList:{jti} 를 키로 저장
   - 남은 시간 (remainTime)만큼 Redis 에 저장 -> 자동 만료

3. 인증 필터에서 블랙리스트 검사 ex) JWTFilterV3
 
String jti = jwtUtil.getId(token);
String blacklisted = redisTemplate.opsForValue().get("BLACKLIST:" + jti);

if (blacklisted != null) {
  log.warn("블랙리스트에 등록된 토큰입니다.");
  filterChain.doFilter(request, response);
  return;
}

- 블랙리스트에 해당 토큰(jti)이 등록되어 있으면 인증을 중단한다.

4. 사용 예시 흐름 요약 (프로젝트 V3 기준)

[사용자 요청]
    ↓
[JWTFilterV3]
→ AccessToken 유효성 검사
→ Redis에 블랙리스트 확인
→ 있으면 차단
→ 없으면 SecurityContext 에 인증 정보 저장
    ↓
[Controller 진입]

[로그아웃 시]
    ↓
[CustomLogoutFilter]
→ AccessToken에서 jti 추출
→ 만료 시간 계산
→ Redis에 BLACKLIST:{jti} 저장

### 프로젝트 적용

내가 구현한 프로젝트에서는 JWTFilterV3 의 doFilterInternal 에 블랙리스트 체크를 구현했다. 기존 선택지는 TokenController & JWTFilterV3 & CustomLogoutFilter 중에 택 하는 것이였는데, 나는 JWTFilterV3 클래스의 doFilterInternal 메서드와 CustomLogoutFilter 에 blacklist 구현을 목표로 했다. 그 이유에 대해서 알아보자.

1. JWTFilterV3 -> 블랙리스트에 등록된 토큰인지 검사하는 역할
- 클라이언트가 보낸 JWT 토큰이
- Redis 블랙리스트에 존재하는지 확인
- 존재하면 인증 거부 (차단)
  - 즉, "로그아웃된 토큰인지 판단"하는 작업

-> 만약 블랙리스트 체크를 TokenController 같은 특정 API에서만 하면, 로그아웃 요청할 때만 차단하고 그 이후 들어오는 다른 요청은 차단하지 못한다.

2. doFilterInternal 은 실제 인증과 차단 로직을 담당
- 클라이언트가 매번 서버에 요청할 때마다,
- 필터가 토큰 유효성과 블랙리스트 여부를 동시에 검사한다.
- 차단할지 허용할지 결정하는 곳이다.   

3. TokenController 는 토큰을 블랙리스트에 등록하는 역할만 담당
- TokenController (또는 로그아웃 API)는 토큰을 블랙리스트에 넣는 역할만 한다.
- 즉, 사용자가 로그아웃 요청할 때 토큰을 등록하는 주제고, 그 외 요청은 처리하지 않는다.

4. LogoutFilter -> 로그아웃 시, 토큰을 블랙리스트에 등록하는 역할
- 사용자가 로그아웃을 요청 하면,
- 해당 토큰의 고유 식별자 jti 를 뽑아서,
- Redis 블랙리스트에 저장 (만료시간까지 TTL 설정)
  - > 즉, "이 토큰은 더 이상 유효하지 않다" 라고 표시하는 작업

5. 한쪽만 있으면 안 되는 이유
- 로그아웃 시, 블랙리스트 등록을 해도,
- 다른 요청에서 해당 토큰을 체크하지 않으면,
- 여전히 서버가 토큰을 유효하다고 인정해서 접근을 허용하게 된다.
- 반대로, 모든 요청에서 블랙리스트 체크만 해도,
- 로그아웃 시 토큰을 블랙리스트에 등록하지 않으면,
- 로그아웃한 토큰을 구분할 방법이 없으니 무용지물이다.

6. 결론 
| 역할           | 위치                            | 내용                            |
| ------------ | ----------------------------- | ----------------------------- |
| **블랙리스트 등록** | `CustomLogoutFilter` (로그아웃 시) | 로그아웃한 토큰을 Redis에 저장해서 차단 표시   |
| **블랙리스트 검사** | `JWTFilterV3` (모든 요청 시)       | Redis에서 토큰이 블랙리스트에 있는지 검사, 차단 |


### 핵심 요약 정리

| 항목  | 내용|
| --- | ------------------------------------------------------------- |
| 목적  | JWT는 서버에서 상태를 기억하지 않기 때문에, 로그아웃/차단 등 "토큰 무효화" 처리를 위해 블랙리스트 필요 |
| 저장소 | Redis (빠르고 TTL 지원)|
| 키   | `BLACKLIST:{jti}` |
| TTL | 해당 토큰의 남은 유효시간만큼 설정   |
| 위치  | 로그아웃 처리 시 등록, 인증 필터에서 검사  |


