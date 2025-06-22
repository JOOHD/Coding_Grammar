## SameSite, Secure

### 1. 브라우저 내부 쿠키 저장 구조 
브라우저는 도메인/경로별로 쿠키를 저장한다. 아래는 Chrome 기준의 쿠키 저장 구조 예시이다.

| Domain      | Path  | Cookies                                         |
|-------------|-------|------------------------------------------------|
| localhost   | /     | accessToken=abc...; HttpOnly; Secure; SameSite=None |
| localhost   | /     | refreshToken=xyz...; HttpOnly; Secure; SameSite=None |
| example.com | /user | sessionId=123...; SameSite=Strict               |


- Domain : 쿠키가 유효한 도메인
- Path : 쿠키가 유효한 URL 경로
- Cookie : key=value 구조 + 옵션 (HttpOnly, Secure 등)

브라우저 개발자 도구에서 : 
Application tap -> Storage -> Cookie 항목에서 실제로 확인 가능

### 2. Set-Cookie Header 구조 (Server -> Browser)
서버가 쿠키를 전송할 때는 HTTP 응답 헤더에 다음과 같이 포함된다.

Set-Cookie: refreshToken=Bearer+xyz.abc.def;
Max-Age=1209600;// Expires 쿠키 만료 시간 (없으면 세션 쿠키)
Path=/; // 쿠키가 유효한 경로
HttpOnly;   // js 접근 제한 (보안 강화)
Secure; // HTTPS에서만 전송
SameSite=None   // 크로스 도메인 요청에 대한 쿠키 제한 정책

### 3. 브라우저 저장 예시 (JSON-like 구조)
{
"domain": "localhost",
"path": "/",
"name": "refreshToken",
"value": "Bearer+xyz.abc.def",
"httpOnly": true,
"secure": true,
"sameSite": "None",
"expirationDate": 1718000000
}

요약 
1. 브라우저는 도메인 + 경로 기준으로 쿠키를 그룹화하여 저장한다.
2. 쿠키는 Set-Cookie 헤더 형식으로 내려오고, 브라우저는 해당 값을 내부 저장소에 구조화하여 보관한다.
3. 보안 설정 (HttpOnly, Secure, SameSite)에 따라 전송 여부와 접근 가능 여부가 결정된다.


### 핵심 개념 정리

HttpOnly : JS에서 쿠키를 읽지 못하도록 막는 것이고,
SameSite : 브라우저가 요청에 쿠키를 포함할지 결정하는 정책입니다.
Secure   : HTTPS 보장 하에만 쿠키 전송 -> MITM 공격 방지
None : CORS, OAuth2, 프론트/백 분리 시 쿠키 전송 허용을 위한 필수 설정

| 옵션   | 의미  | 목적| 크로스 사이트 요청 시 쿠키 전송  |
| -------- | ------------------------------- | --------------------------------- | --------------------------- |
| `Strict` | **같은 사이트에서만** 쿠키 전송 | 보안 극대화 (CSRF 방지)  | ❌ 차단됨   |
| `Lax`| \*\*안전한 메서드(GET, 리디렉션)\*\*에만 허용 | 일반적인 보안 유지 + 기본 사용| ✅ 일부 허용 (GET 요청 등)  |
| `None`   | **어떤 출처든 허용**   | OAuth2, 프론트/백 분리 등 크로스 요청 필요 시 사용 | ✅ 전송됨 (단, `Secure=true` 필수) |

### SameSite 옵션 - 크로스 사이트 요청에 대한 쿠키 전송 

| 옵션   | 의미  | 목적| 크로스 사이트 요청 시 쿠키 전송  |
| -------- | ------------------------------- | --------------------------------- | --------------------------- |
| `Strict` | **같은 사이트에서만** 쿠키 전송 | 보안 극대화 (CSRF 방지)  | ❌ 차단됨   |
| `Lax`| \*\*안전한 메서드(GET, 리디렉션)\*\*에만 허용 | 일반적인 보안 유지 + 기본 사용| ✅ 일부 허용 (GET 요청 등)  |
| `None`   | **어떤 출처든 허용**   | OAuth2, 프론트/백 분리 등 크로스 요청 필요 시 사용 | ✅ 전송됨 (단, `Secure=true` 필수) |

SameSite = None 사용 시에는 반드시 Secure=true 와 함께 사용해야ㅐ 브라우저가 쿠키를 저장함, 그렇지 않으면 브라우저가 쿠키를 버림.

### Secure 옵션 - HTTPS에서만 쿠키 전송 허용

Secure=true : 쿠키는 HTTPS 환경에서만 전송됨. HTTP 요청에는 쿠키 포함되지 않음.
Secure=false(기본값) : HTTP에서도 쿠키 전송 허용

Secure=true는 브라우저 보안 정책의 강제 조건으로, SameSite=None과 함께 쓰는 경우 필수.

### 상황별 요약

| 시나리오| SameSite | Secure   | 결과|
| ----------------------------------- | -------- | ------------ | ------------- |
| **동일 출처 요청 (SPA 내부)**   | Strict   | false/true   | ✅ 전송됨 |
| **프론트: 3000, 백: 8080** (크로스 도메인 요청) | Strict   | true | ❌ 전송 안됨   |
| OAuth2 로그인 후 리다이렉트  | Strict   | true | ❌ 쿠키 누락됨  |
| 프론트 ↔ 백 완전 분리 (CORS 환경) | None | ✅ 필수 (HTTPS) | ✅ 전송됨 |
| 로컬 개발환경 (HTTP) + SameSite=None  | None | ❌ (HTTP라서)   | ❌ 쿠키 저장/전송 실패 |

목적 요약
SameSite : 쿠키 전송을 요청 출처 기준으로 제한해서 CSRF 등 보안 위협 방지
Secure : HTTPS 보장 하에만 쿠키 전송 -> MITM 공격 방지
None : CORS, OAuth2, Front/Back 분리 시, 쿠키 전송 허용을 위한 필수 설정 

### 브라우저 보안 정책 질문

1. SameSite=Strict 쿠키는 CORS 환경 (ex 3000 -> 8080 port) 에서 전송되지 않는가?
- 그렇다 전송되지 않는다. SameSite=Strict 는 정말 엄격하게 '같은 출처에서만' 쿠키를 전송한다.
  - localhost:3000 (front) -> localhost:8080 (back) 요청은 다른 출처 (다른 port)
  - port 가 다른 경우 브라우저는 쿠키를 전송하지 않는다.

-> SameSite=Strick 는 크로스 도메인 요청을 전부 차단한다.

2. OAuth2 로그인 후, redirect는 왜 SameSite=Strict에서 쿠키를 전송 할 수 없나? 
- OAuth2 redirection 은 외부 -> 내 애플리케이션으로의 "크로스 사이트 요청"으로 간주되기 때문이다.
  - ex) accounts.google.com -> localhost:8080/oauth2/callback
  - 리다이렉션은 자동으로 열리는 외부 요청이며, 이 요청은 브라우저가 보기엔 "타 사이트에서 오는 요청" 이다.
-> 그래서 SameSite=Strick일 경우, 이런 리다이렉션 요청에서는 쿠키가 브라우저에서 포함되지 않는다.

3. "브라우저는 다른 port로부터의 요청에서는 쿠키를 절대 전송하지 않는다."는 JS 접근 차단 때문인가?
- 아니다. JS 접근 차단은 HttpOnly 설정 때문이고, 이 말은 브라우저의 보안 정책 때문이다.
  - "브라우저가 다른 출처에서 쿠키를 내보내지 않는다."는 SameSite 정책에 의한 것이다.
  - HttpOnly는 JS에서 쿠키를 읽지 못하도록 막는 것이고, SameSite는 브라우저가 요청에 쿠키를 포함할지 결정하는 정책이다.

4. OAuth2 redirect 요청이 외부 요청으로 처리되어 쿠키 저장에 실패하는 이유는?
- OAuth2 는 보통 구글 로그인 -> 내 서비스로 redirect 흐름이다.
- 이 리다이렉트 요청은 브라우저가 외부 사이트에서 들어온 요청이라고 인식한다.
- SameSite=Strict 이거나 Secure=false 이면 쿠키가 저장되거나 전송되지 않는다.

5. SameSite 옵션의 의미 (Strick, Lax, None)  
| 옵션   | 목적  | 크로스 사이트 요청에서 쿠키 전송 여부 |
| -------- | --------------------------------------- | --------------------- |
| `Strict` | **최고 보안**. 오직 동일 출처 요청에만 쿠키 전송. | ❌ 차단  |
| `Lax`| 안전한 요청(GET, link, redirect)에는 쿠키 전송 허용. | 🔸 **GET 리다이렉션만 허용**  |
| `None`   | 크로스 도메인 요청에서도 허용| ✅ 허용 (단, `Secure` 필수) |

-> SameSite=None 을 쓰면 무조건 Secure=true도 설정해야 브라우저가 허용한다. 그렇지 않으면 아예 쿠키를 버린다.

6. SameSite=None 은 수동 설정이고, Strict 는 자동인가?
- 맞다. SameSite 를 명시하지 않으면 브라우저는 보통 기본값으로 Lax or Strict 를 적용한다. (브라우저에 따라 다르다.)

// 자동 (명시 안함 -> 브라우저 기본값)
Cookie cookie = new Cookie("key", "value");
response.addCookie(cookie);

// 수동 (SameSite=None + Secure 직접 설정)
String cookieString = String.format(
"refreshToken=%s; Max-Age=1209600; Path=/; HttpOnly; Secure; SameSite=None", 
URLEncoder.encode(value, StandardCharsets.UTF_8)
);
response.addHeader("Set-Cookie", cookieString);

7. 로컬환경에서 Secure=true 가 적용되지 않는 이유는?
- Secure=true 는 HTTPS(SSL)에서만 작동한다.
  - localhost 에서 http://localhost:8080 처럼 HTTPS 가 아닌 요청은 Secure 옵션을 적용해도 브라우저가 쿠키를 저장하지 않는다.
  - 따라서 로컬에서 테스트하련 :
- SameSite=None -> 쿠키 저장 안 됨 (Secure 필요)
- Secure -> HTTPS 가 아니면 무효
-> 반면, SameSite-Lax 는 HTTPS가 아니어도 작동하기 때문에, 로컬에서 테스트할 수 있따.

정리
Secure + SameSite=None = 반드시 HTTPS 환경 필요
- 이 조합은 크로스 도메인 허용 + 보안 쿠키 설정을 위해 사용된다.
- 하지만, Secure가 적용되려면 HTTPS(로컬은 X)여야 하므로, 로컬에서는 안 된다.
| 설정  | 로컬에서 동작 여부 | 설명|
| --------------------------- | ---------- | ------------------------- |
| SameSite=Strict | ✅  | 같은 출처만 허용 (보안 높음) |
| SameSite=Lax| ✅  | GET 요청 redirect는 쿠키 전송 가능 |
| SameSite=None + Secure=true | ❌  | **HTTPS 필요** – 로컬에선 안 됨   |
| Secure=true | ❌  | **HTTPS에서만 쿠키 저장** 가능 |

### CSRF & CORS 개념

| 구분       | CSRF (Cross-Site Request Forgery)        | CORS (Cross-Origin Resource Sharing)       |
| -------- | ---------------------------------------- | ------------------------------------------ |
| 목적       | 공격자가 사용자의 인증된 상태를 이용해 악의적 요청을 보내는 공격 방어  | 브라우저가 다른 도메인의 자원 접근을 제한하는 보안 정책 해제         |
| 대상       | 인증된 사용자 세션 (주로 쿠키 인증)                    | 브라우저의 보안 정책 (출처가 다른 도메인 요청)                |
| 동작 원리    | 서버가 요청이 올바른 출처에서 왔는지 토큰으로 검증             | 서버가 허용한 출처(origin)에만 리소스 공유 허용             |
| 개발 시 조치  | REST API는 보통 stateless JWT를 써서 CSRF 비활성화 | API 서버에 CORS 설정으로 허용 도메인 및 메서드 지정          |
| 스프링 설정 예 | `http.csrf().disable()` 또는 특정 경로 제외 설정   | `http.cors().configurationSource(...)`로 설정 |

- 요약
CSRF: 사용자의 의도하지 않은 요청을 막는 방어막 (주로 세션/쿠키 기반 인증 시 필요)

CORS: 브라우저가 다른 도메인 요청을 제한하는 보안 정책 제어 (주로 API 호출 시 필수)

### CSRF & CORS 역할 및 실제 시나리오

- CORS :
프론트엔드가 다른 도메인 (ex. localhost:3000)에서 백엔드 API 호출 시, 브라우저 차원에서 거부됨
  -> 백엔드에서 허용 도메인/헤더/메서드 설정 필요

- CSRF :
세션 인증 기반 폼 로그인에서, 악성 사이트가 쿠키를 이용해 백엔드에 악의적 요청 보낼 수 있음
  -> CSRF 토큰 확인으로 방어
  -> REST API + JWT 방식에서는 보통 비활성화