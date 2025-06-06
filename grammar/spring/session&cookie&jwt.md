## Cookie & Session

### Cookie

● 클라이언트에 저장되는 데이터
- 클라이언트가 로그인을 하면 서버가 응답 시 쿠키에 사용자 정보를 담아서 반환해준다.
- 클라이언트는 사용자 정보를 저장하고 서버에 요청할 때마다 HTTP HEADER에 해당 쿠키를 실어서 보낸다.

● 세션 ID 저장
- 쿠키는 주로 서버에서 생성된 세션 ID를 저장하는 데 사용.
- 서버는 쿠키를 통해 세션 ID를 클라이언트에게 전달하고, 클라이언트는 이후 요청할 때마다 이 세션ID를 포함한 쿠키를 서버에 다시 전송.

● 주의 사항
- HTTP'S'가 아니라면 쿠키가 암호화되지 않으므로 노출될 가능성이 있다. 따라서 회원관련 정보는 담지 않는 것이 좋다.
- HTTP Only 설정 시 브라우저를 통한 요청이 아닌 Javascript 등을 통해 cookie를 담아 요청하려면 HTTP ONLY가 아니므로 쿠키를 전부 거절한다.   
	
	쿠키는 서버에서 생성되어 클라이언트(ex, web browser)에게 전송된다. 
	이후 클라이언트는 해당 쿠키를 보관하고 있다가, 같은 서버로 요청을 보낼 때마다 그 쿠키를 서버로 다시 전송한다.

### cookie flow
	
	1. 서버가 쿠키 생성
		- 클라이언트가 서버에 요청을 보낼 떄, 서버는 필요에 따라 쿠키를 생성하여 클라이언트에게 응답한다.
		- 이때 Set-Cookie 헤더를 사용하여 쿠키를 클라이언트에게 전달한다.
			ex) 
				Set-Cookie : sessionId=abc123; Paht=/; HttpOnly; Secure

	2. 클라이언트가 쿠키 저장
		- 클라이언트(웹 브라우저)는 서버로부터 받은 쿠키를 저장. 이 쿠키는 해당 서버와의 통신에 사용할 수 있는 데이터로 유지.

	3.클라이언트가 다시 서버로 쿠키 전송
		- 클라이언트가 같은 서버로 다시 요청을 보낼 때, 저장된 쿠키를 함께 전송한다.
			이때, Cookie 헤더를 사용하여 쿠키를 포함시킨다.
			ex)
				Cookie : sessionId=abc123

	3. 서버가 쿠키를 사용
		- 서버는 클라이언트가 보낸 쿠키를 확인하고, 이를 통해 세션을 관리하거나 사용자 인증 등의 작업을 수행한다.

	● 예시 시나리오
		1. 사용자가 웹사이트에 로그인하면, 서버는 로그인 상태를 유지하기 위해 쿠키를 발급한다.
		2. 이 쿠키에는 사용자의 세션 ID 같은 정보가 포함될 수 있다.
		3. 사용자가 로그인한 후, 다시 같은 웹사이트에 접속할 때 브라우저는 해당 쿠키를 서버로 전송하여 사용자가 이미 로그인한 상태라는 것을 서버가 알 수 있게 한다.

	● 쿠키 주요 기능

	4. 로그인 유지
		- 사용자가 로그인한 후, 서버가 생성한 쿠키를 클라이언트에 전달.	
			이 쿠키는 다음 요청에서 로그인 상태를 유지하기 위해 사용된다.

			ex)	
				sessionId & accessToken 등의 쿠키를 통해 서버는 사용자가 인증된 상태임을 알 수 있다.
				
	5. 유저 식별(PK)
		- 쿠키에는 보통 사용자에 대한 고유 식별자(sessionId)가 포함되어 있다.
		- 이 세션 ID는 서버에서 사용자를 구분하는 primary key 같은 역할을 한다.
			클라이언트는 이 세션 ID를 포함한 쿠키를 서버로 다시 전송하고, 서버는 해당 ID를 보고 구별한다.

	6. 세션 관리
		- 서버는 각 사용자를 위한 세션을 생성하고, 세션 정보는 서버의 메모리나 DB에 저장된다.
			이때 사용자의 세션을 식별하기 위해 세션 ID를 쿠키로 관리한다.
		- 서버는 쿠키에 포함된 세션 ID를 이용해 해당 사용자의 세션 데이터를 찾아 요청을 처리.

	● 예시 시나리오 (유저 식별)
	
	7. 로그인 시,
		- 사용자가 로그인하면, 서버는 세션을 생성하고 sessionId=qwer123 같은 고유한 셋녀id를 포함한 쿠키를 클라이언트에게 전송한다.

	8. 다음 요청 시,
		- 사용자가 이후 서버로 요청할 때, 클라이언트는 Cookie: sessionId=qwer123 헤더를 서버로 전송
		- 서버는 이 qwer123 세션 ID를 기반으로, 해당 사용자가 어떤 사용자인지 식별하고 해당 유저와 관련된 데이터를 조회한다.  

### Session

cookie의 단점을 보완하기 위한 방식으로 클라이언트가 사용자 정보를 보관하지 않고 서버가 보관하도록 한다.
이후 클라이언트는 SessionId를 가지고 서버에 요청하게 된다.

	● 서버에 저장되는 데이터
- 세션은 서버에서 관리되는 데이터로, 각 사용자가 서버에 접속할 떄마다 서버는 해당 사용자만의 세션을 생성한다. 
- 이 세션에는 사용자의 상태나 정보가 저장된다.

● 고유 식별자 (SessionId)
- 서버는 사용자마다 고유한 세션 ID를 부여하여, 이 ID를 통해 사용자를 식별한다. 
- SessionId는 서버에서만 사용자가 누구인지 확인하는 키가 된다.

● 상태 유지
- 세션을 사용하면 사용자가 여러 페이지를 이동하거나 요청을 보내더라도, 서버는 사용자를 식별하고 이전에 발생한 상태(로그인 여부, 장바구니 상태 등)을 유지할 수 있다.

### session flow

1. 클라이언트 -> 서버로 최초 Request를 보낸다.

2. 서버는 세션 저장소에 세션ID와 작은 저장소 쌍을 같이 생성한다.

3. 서버는 세션ID를 Response 메시지 Header에 넣고 반환.

4. 클라이언트의 웹 브라우저에 세션ID가 저장된다.

5. 클라이언트가 서버에 Username & Password으로 로그인 요청을 보낸다.
이때 세션ID를 헤더에 담아서 보낸다.

6. 서버는 DB로 가서 해당 회원이 존재하는지 확인한다.  

7. 해당 회원값이 유효하면 세션 저장소의 저장소에 회원객체를 저장한다.

8. 로그인 성공 시, 메인페이지.html 파일을 반환한다. 

9. 클라이언트가 사용자 정보 조회 요청을 보낸다. 이떄도 세션ID를 가지고 온다.

10. 서버는 세션 저장소에서 해당 세션ID가 존재하는지 확인하고 존재하면 ->
로그인이 된 사람임을 알게된다.

11. DB로 접근해서 사용자 정보를 조회한다.

12. 사용자 정보를 클라이언트에게 반환해준다.

### session 단점

복수의 서버를 운영하는 경우 성능에 좋지 않다. 각 서버마다 Session 정보를 메모리에서 갖고 있어야 하므로 메모리 낭비가 발생한다.

ex)
100명을 처리할 수 있는 서버가 있을 때 300명의 요청이 온다면 200명이 대기해야 할 것이다. 이를 해결하기 위해 서버 2대를 증설해서 3대를 유지하는 경우 트래픽이 올 때 적절히 여유로운 서버로 요청을 보내는 로드 밸런싱이 필요하다.

세션의 경우 서버의 인메모리에 존재하므로 여기서 문제가 발생한다.

- 클라이언트A -> 서버A, B, C 중 B로 로그인을 했다.

- B에 사람이 몰린 상황에서 A가 정보 조회를 요청했고, 로드 밸런싱이 C서버로 보냈다.

- C서버는 A에 대한 정보가 없으므로 처음 온줄 알고 다시 로그인을 하도록 하게 된다.

이를 해결하기 위해 세션 저장소 간 정보를 동기화 or 요청마다 처리 서버를 고정하는 방법이 있지만 둘 다 좋지 않다.

이런 문제를 해결하기 위해 여러 서버가 공유하는 하나의 동일한 세션 저장소를 갖도록 하는 방법이 두 가지 있다.  

하드디스크를 사용 or 리모트 인메모리 저장소(REDIS)
그러나 하드디슼를 세션저장소로 사용하면 가격은 저렴하지만 I/O 작업으로 인해 느리다.

리모트 인메모리 저장소는 속도가 빠르나, 가격이 비싸다.
이렇게 점점 더 트래픽이 올라가는 현 상황에서 세션 외 해결방법을 찾을 수 밖에 없고 세션 외 해결방법이 있다.

● 해결 방법
1. (HTTP Basic)
- HTTP Header에 Authorization 키의 값에 인증정보를 담는다(id, password)
- 이렇게 하면 요청 때마다 id/pw를 갖고 요청하므로 세션, 쿠키를 만들 필요가 없어진다.
- 단, HTTPS가 아닌 경우 HTTP 메시지가 암호화되지 않으므로 탈취 당하기 쉽다.

2. (HTTP Bearer)
- HTTP Header에 Authorization 키의 값에 토큰을 담는다.

- HTTP'S'가 아닌 경우 토큰 방식도 노출되면 해당 토큰을 가지고 요청할 수 있으므로 위험하다.

하지만 토큰은 다시 로그인 할 때마다 새로 발급해주고, AccessToken의 유효시간을 짧게 가지므로 ID, PW가 노출되는 것보단 안전하다.

- 이 토큰을 만드는 방법 중 하나가 JWT 방식이다.

### session & cookie 관계

1. 세션 생성
- 사용자가 웹 서버에 처음 접속하면 서버는 세션을 생성하고, 고유한 세션 id를 부여한다.

2. 쿠키를 통한 세션 id 저장
- 서버는 생성된 세션 id를 쿠키에 담아 클라이언트에 전송한다.
ex)
Set-Cookie: sessionId=abc123와 같은 형태로 쿠키 생성이 된다.

3. 다음 요청 시 쿠키 전달
- 사용자는 다음 번에 같은 서버에 요청을 보낼 때, 브라우저는 서버가 준 쿠키(sessionId)를 함께 전송한다.
- 이 과정은 cookie: sessionId=abc123 형태로 이루어진다.

4. 세션 정보 조회
- 서버는 클라이언트가 전송한 세션 id를 받아, 해당 id에 대응하는 세션 데이터를 서버에서 조회한다. 이를 통해 사용자가 누구인지, 로그인 상태인지, 어떤 작업을 했는지 등을 확인한다.

### session & cookie 차이점

| Session  | Cookie  | 
| ------------ | ------------|
| 서버에 저장   | 클라이언트(브라우저)에 저장 |
| 고유 식별자인 세션 ID를 통해 사용자 식별 | 세션 ID를 저장하고 서버에 전송 |
| 메모리나 데이터베이스에서 관리  | 브라우저에 파일 형태로 저장됨 |
| 브라우저가 닫히거나 타임아웃 시 만료 | 만료 기간 설정에 따라 유지 또는 삭제 |
| 주로 중요한 사용자 상태 유지 (로그인 등) | 사용자 설정, 장바구니, 트레킹 등에 사용 |

### 예시 흐름 

1. 사용자가 로그인
  - 서버는 세션을 생성하고 sessionId = adc123을 부여한다.
  - 서버는 클라이언트에게 Set-Cookie: sessionId=abc123 쿠키를 전송
  
2. 다음 요청
  - 사용자가 서버에 다시 요청을 보낼 때, 클라이언트는 쿠키 sessionId=abc123을 서버로 보낸다.

3. 서버에서 세션 확인
  - 서버는 이 sessionId = abc123을 기반으로, 해당 사용자의 세션 데이터를 조회하여 로그인 상태나 저장된 데이터를 유지한다.

### JWT

HTTP Bearer 방식으로 <Authorization : token> 쌍으로 서버에 요청하게 된다.

● 구조
- Header : 서명 알고리즘 정보가 있다.
- Payload : Claim이 들어가면 일반적으로 사용자 관련 정보가 들어간다.
- Signature : (Header + Payload + SecretKey)를 암호화한 값이 들어간다.

● 매커니즘
1. 클라이언트가 서버에 로그인을 요청한다.

2. 서버는 해당 로그인을 확인 후,
인코딩{알고리즘 정보} + .인코딩{회원정보} + .인코딩{암호화(알고리즘정보 + 회원정보 + 시크릿 값)} 방법으로 Token을 생성한다. 이후 클라이언트에게 반환한다.

3. 클라이언트는 해당 토큰을 쿠키에 저장하고, 서버에 요청을 보낼 떄 마다 HTTP HEADER에 포함해서 보낸다.

4. 서버는 해당 Token의 Signature값과 인코딩{디코딩(토큰알고리즘 + 토큰회원정보 + 시크릿키)} 값이 같은지 비교해서 토큰의 유효성을 검사.
   
