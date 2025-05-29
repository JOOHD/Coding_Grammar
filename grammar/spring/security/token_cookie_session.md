## Access/Refresh token 저장 위치

### 개요

    시큐리티를 다루면서 인증/인가의 중요성이 커지게 되고, 인증/인가 구현 중 여러 구글링 결과 토큰의 저장위치에 대해서 고민이 되는 상황에 정리가 필요하여 이 글을 쓰게 되었다.

### 사용자 인증

    - "시스템이나 애플리케이션에 접근하려는 사용자가 실제로 그 사용자인지 확인하는 과정"

    - 이미 사용자 인증을 마친 회원이 페이지를 이동할 때마다 사용자 인증을 하게 된다면 사용자에게는 굉장히 불편한 서비스가 될 것이다. 이런 문제로 사용자 인증 방식들을 보안성, 사용자 경험, 확장성을 중심으로 비교해 보고 선택함이 중요하다.

    - 사용자 인증은 크게 session, token 인증 방식으로 나눌 수 있다.

### Session 인증 방식 특징

    서버에 상태를 저장하는 상태 유지(stateful)이다.
    사용자가 로그인을 서버로 요청하면, 요청 정보가 DB와 일치하는지 확인하고 사용자의 세션을 생성한다. 이때 생성한 세션 id는 서버의 저장소(메모리, DB)에 저장된다. (이 값을 서버에 저장하기 때문에 stateful 이라 한다.)

    이 후로는 쿠키에 담아 사용자의 후속 요청마다 세션 ID가 담긴 쿠키를 서버에 전송하게 되고, 해당 세션 id가 저장소에 존재하는지 확인하는 식으로 사용자 검증을 한다.

    이러한 방식은 서버에서 인증을 관리하기 때문에, 클라이언트 측에서 정보가 누출될 위험이 적다는 보안적인 장점이 있다.

    하지만 빈번히 발생하는 사용자의 요청마다 DB에 접근한다는 것은 응답 속도 저하를 일으키는 원인이 된다.

![stateful](/grammar/img/stateful.png)    

### Token 인증 방식 사용 이유

    토큰 인증은 세션 인증과 반대로 서버에 상태를 저장하지 않는 상태 비저장(stateless)이 가장 큰 특정이다.

    즉, 서버는 사용자의 정보를 검증하기 위해 "DB를 거치지 않고 오로지 토큰만 사용한 방법으로 사용를 검증한다."

    기존 세션 인증 방식의 가장 큰 단점은 응답 속도 저하로 인한 서버 부하(사용자 경험 저하)와 확장에 어려움이 있다는 점이다. 반면, 토큰 인증 방식은 DB를 거치지 않기 때문에 응답속도 저하 문제를 해결할 수 있다.

    또한, 모바일 환경에서는 IP로 세션을 관리하는 경우도 있는데, 이런 경우는 토큰 방식이 더 선호된다.

    -> 즉, 세션 인증 방식에 비해 응답속도, 확장성 면에서 우수하여 토큰 방식을 사용하는 것이다.

![stateless](/grammar/img/stateless.png)        

### Token 인증 방식 단점

    토큰 탈취의 위험이 단점이다. 세션이 서버에 관리되는 장점과 상반되는 이유. 클라이언트 자체에서 탈취되거나, 악성코드로 탈취될 가능성도 있고, 클라이언트에서 서버로 전송되는 동안에도 탈취될 가능성이 있다.

    -> 이번 글은 두 방식 중 세션의 단점을 보완하기 위해 등장한 토큰 인증 방식을 선택, 토큰 탈취에 대한 방비책에 대해서 작성하겠다.

### Access Token        

    주로 OAuth 2.0 프로토콜에서 사용되는 토큰이다.
    access는 사용자의 인증과 권한을 나타내는 단기 유효 토큰(문자열)로, 클라이언트가 서버에 API 요청을 보낼 때 이 토큰을 헤더에 포함하여 요청을 보내고, access가 유효한지 확인하여 사용자를 검증한다.

    session = 서버가 인증 정보를 관리, 
    access  = 클라이언트가 인증 정보를 직접 관리

    물론, access의 경우에도 서버에 인증 정보를 관리할 수 있지만, 앞서 session 방식의 단점을 그대로 안고 가기 때문에 권장되는 방식이 아니다.

### Access Token & JWT

    access 토큰은 문자열 형식으로 구성되어 있으며, 발급되는 토큰 종류에 따라 구조화된 모양일 수 있고, 해석할 수 없는 단순한 문자열일 수 있다.

    이때 다양한 토큰 형식 중, JWT 형식이 stateless (상태 저장 없음)하다는 장점 덕분에 주로 사용된다. stateless 하다는 특성은 다시 말해, 서버에서 access 토큰 자체를 저장할 필요가 없다. DB 접근 또한 없을 뿐더러, 서버 인스턴스가 여러 개 있을 때에도 쉽게 확장할 수 있다는 것이다.

    이런 stateless 한 특징은 JWT 구조에서 비롯된다.
    JWT = Header + Payload + Signature 로 구성되어 있으며, 한 줄의 문자열에서 .(점)을 이용해 세 부분으로 구분된다.

![jwt_structure](/grammar/img/JWT_structure.png)

### JWT Structure

    Header    : 토큰의 유형과 서명 알고리즘을 포함
    Payload   : 사용자 정보 + claim (id, role, expired)이 포함
                (payload 떄문에 서버가 별도 DB 조회 없이도 사용자 정보를 확인) = 요청 처리 속도 UP + 서버 부하 Down -> Self-contained 
    Signature : Header + Payload 조합한 후, 헤더의 서명 알고리즘과 비밀 키를 사용 -> 클라이언트가 토큰 변조 방지

    -> 위 세 부분은 Base64 Url 인코딩 되어 하나의 문자열로 결합되어 JWT 토큰을 구성하게 된다. 이렇게 구성된 JWT는 필요한 모든 정보와 서명 부분을 토큰 자체에 포함하고 있으므로, 서버에서 별도의 세션 저장소를 유지할 필요가 없어진다.

### Access Token 저장 위치

    JWT를 관리하는 방식으로는 Local, Session, Cookie, Memory 에 저장하는 방식으로 나눌 수 있다.

    1. Local
    - 클라이언트 측에 데이터를 영구적으로 저장하는 방법이다.
    - 브라우저를 닫아도 데이터가 유지되며, 명시적으로 삭제하지 않는 한 계속 존재
    - 그러나 XSS(악성 코드를 브라우저에서 실행 되도록 하는 공격 방식) 공격에 취약하여, 로컬 스토리지에 저장된 엑세스 토큰을 탈취할 수 있는 위험이 있다.

    2. Session    
    - 브라우저가 닫히면 데이터가 삭제되며, 같은 탭에서만 유효하다는 특징이 있다.
    - 로컬 보다는 보안적인 측면에서는 안전하지만, 여기 또한 XSS에 취약하다.

    3. Cookie
    - HTTP의 Stateless 특성을 보완하기 위해 등장한 데이터, 따라서 모든 클라이언트 요청에 자동적으로 포함된다. 이는 access 토큰에 사용될 때에도, 클라이언트에서 쿠키를 별도로 포함시키지 않아도 된다는 뜻이다.

    - 쿠키는 만료 날짜를 지정하는 "영구 쿠키" vs 브라우저가 닫힐 때 삭제되는 "세션 쿠키"로 구분된다.

    - 특히 쿠키는 앞선 두 방식에서의 XSS 공격 취약점을 어느 정도 예방할 수 있다. HttpOnly 플래그를 사용하여 JavaScript에서 쿠키에 접근할 수 없게 함으로써 XSS 공격에 대한 보안을 높이는 방식이다.
    
    - 하지만, XSS 공격은 다양한 방법으로 이루어 지므로 완벽하게 방어가 된다고 볼 수 없기에, 추가적인 보안 조치가 필요하다.
    
    - 또한, Secure 플래그를 사용하면, HTTPS 연결에서만 쿠키가 전송되게 하여 중간자 공격 (MITM)으로 부터 쿠키를 보호할 수 있으며, SameSite 쿠키 속성을 설정하면 CSRF 공격을 방어할 수 있다. 

    4. Memory
    - access 토큰을 클라이언트 서버의 메모리에 저장하는 방식
      ex) JavaScript의 private 변수에 JWT를 저장하는 것이다.

    - 매 요청 마다, API 호출 시, access 토큰에 접근이 쉬어지지만, 브라우저의 메모리는 세션 단위로 관리되기 때문에 페이지를 이동하면 access 토큰이 
    소멸하는 문제 발생

    - 따라서 Memory 저장 방법은 SPA(Single Page Application)에서 주로 사용된디.
      ex) React를 사용하여 클라이언트 서버를 구현했다면, 페이지가 이동하는 것처럼 보여도 실제로는 이동하지 않기 때문에 private 변수가 그대로 유지된다. 

      하지만, 페이지를 새로 고침한다면, private 변수가 소멸되기 때문에 다시 로그인을 해주어햐 되는 상황이 발생한다.

    -> 메모리에 저장 시, 스크립트가 메모리 공간을 직접적으로 제어할 수 없기 때문에 XSS 공격에 의해 쉽게 탈취될 수 없다.

    정리
    - 이 네 가지 방법 중 로컬 스토리지와 세션 스토리지의 경우, XSS 공격에 매우 취약하여 권장하지 않는다.
    - 메모리 & 쿠키 경우에도 완벽하지 않지만, 다른 방식에 비해 비교적 안전한 방식으로 많이 사용된다.
    - access 토큰만 사용하는 경우에서 불편하게 보일 수 있지만, 이후에 다루는 Refresh 토큰까지 고려해 본다면 충분히 안정적으로 사용 가능한 방식이다.

### Access Token 사용자 인증 방식

    이제는 JWT를 통해 어떻게 사용자 인증이 이루어지는지를 알아보자.
    JWT 와 쿠키를 이용한 access 토큰 인증 방식은 간단하게 아래와 같은 플로우로 이루어진다.

![jwt_user_authentication](/grammar/img/jwt_user_authentication.png)

    1. access token 생성
    - 로그인 요청 시, JWT를 생성하여 클라이언트에게 반환하는 과정
    - 로그인 요청 : 사용자가 로그인 정보를 입력하고 서버에 로그인 요청을 보낸다.
    - JWT 발급 : 서버는 사용자의 인증 정보를 DB와 일치하는지 확인한 후, JWT를 생성하여 클라이언트에 반환한다.

    2. access token 사용
    - 서버에서는 JWT가 스스로 정보를 인증하므로 DB에서 사용자 정보와 일치하는지 검증할 필요가 없다.
    - 클라이언트 측에서는 헤더에 JWT를 모든 요청에 포함하여 사용자 인증에 사용한다.

    후속 요청 : 클라이언트가 서버에 요청을 보낼 때, 헤더에 access 토큰을 포함 (쿠키의 경우 자동 포함)
    서버에서 JWT 검증 : 서버는 요청받은 access 토큰에서 사용자의 인증 상태를 확인한다.

    -> 이런 인증 방식의 사용이 무수히 일어나도, 서버에서 사용자 인증을 위해 DB에 접근하는 부분은 JWT를 발급하는 로그인 부분뿐이다. 이러한 점이 JWT의 stateless 한 특징이 가장 도드라지는 부분이다.

### Access Token 로그아웃 방식

    사용자 인증이 이루어지는 부분을 생각해 보면, 단순히 access 토큰을 사용하기 때문에 이 부분만 무효화해 주면 된다는 결론에 이를 수 있다.

    1. 메모리의 경우 Access 토큰 제거
    클라이언트 서버 private 변수에 Access 토큰을 저장한 경우, Access 토큰 자체를 제거해 주면 된다. 간단히 null 처리를 통해 로그아웃을 구현할 수 있따.

    2. HTTP-Only 쿠키의 경우 Access Token 만료시키기
    이 경우에는 JavaScript로 쿠키에 접근하는 방법은 불가능하다. 따라서, 서버에 요청을 보내 로그아웃을 구현한다.
    JWT는 토큰 자체에 만료 시간을 포함할 수 있지만, 이미 발급된 JWT를 서버에서 직접 만료시키는 것은 불가능하기 때문에 쿠키의 만료시간을 강제하여 쿠키를 삭제하는 방법으로 로그아웃한다.

    3. BlackList 추가
    서버에서 토큰을 무효화 시키는 방법이다. 서버에서 발급한 JWT를 블랙리스트에 추가하여 해당 토큰을 유효성을 없엔다. 이 방법은 추가적인 저장소를 필요로 한다는 단점이 있으며, 매번 블랙리스트에 있는 access 토큰인지 검증해야 할 필요가 생기므로 오버헤드가 발생한다. 
    또한 JWT를 블랙리스트에 저장하는 과정에서 stateless 하다는 장점을 잃게 된다.

### Access Token 탈취

    위에 글까지 보면, 간단하게 로그인부터 로그아웃까지 access token 하나만으로 구현이 가능하다. 하지만 access token 하나만을 사용한 상황에서 해커가 access token을 탈취하게 되면 어떻게 될까?

![accessToken_steal](/grammar/img/accessToken_steal.png)

    access 토큰을 사용 못하게 하는 방법은 무효화이다.
    하지만 서버는 해커가 가진 stateless한 access 토큰에 관한 정보가 하나도 없다. -> 서버는 만료 시간이 다 될 때까지 아무런 조치도 취할 수 없다.
    -> 이러한 서버의 하염없는 기다림의 대안은 blackList에 추가하는 방법이다.

    그러나 blacklist에 추가하는 것 역시 이상 패턴이 발견됐을 경우 이야기이지, 이상 패턴이 발견되지 않는다면 피해는 막을 수 없다.

    그러면 다른 방향으로 access 토큰 만료 기간을 짧게 가져가면 피해를 최소화할 수 있지 않을까?
    -> 위에 방식을 적용하면 해커는 토큰을 탈취했기 때문에 만료시간까지 공격할 수 있겠지만, 금방 만료되어 버린 토큰은 사용이 불가능해지므로 피해가 최소화되는 효과를 가져온다. 

    하지만, 이는 사용자의 경우에도 마찬가지이다. 홈페이지를 이용하는데, 10분마다 다시 로그인을 하려고 한다면, 사용자 경험이 저하되어 사용자는 줄어들게 될 것이다.
    -> 이런 토큰 유효기간의 딜레마를 해결하기 위해 access 토큰과 함께 refresh 토큰을 사용한다.

### Refresh Token    

    토큰 하나만으로 Refresh 토큰의 유효성 검증을 진행할 수 있으며, 페이로드 부분에 저장된 최소한의 데이터 (user_id, token_id)만을 이용하여 DB에 접근한 후 access 토큰을 재발급(reIssue)할 수 있다.

    refresh 토큰에서 JWT의 Self-contained (자체 데이터 포함)한 특징을 이용하여 DB의 접근을 없앨 수 있지만, 이렇게 구성하면, refresh 토큰의 목적인 access 토큰의 유효성 연장 이외의 정보들을 포함하게 되며, 보안적으로 취약점이 생길 수 있다.

    따라서, self-cotained 한 특성은 사용하지 않는 것이 권장된다.
    그러면 refresh 토큰은 어디서 관리될 수 있을까?

### Refresh Token 저장 위치

    refresh 토큰 관리도 access 토큰과 같다.

    1. 로컬 스토리지 (Local Storage)    [권장 X]
    2. 세션 스토리지 (Session Storage)  [권장 X]
    3. HTTP-only 쿠키 (Cookies)
    4. 메모리 (Memory)                 [권장 X]
 
    서버에서 refresh 토큰을 관리하는 방법은 주로 DB를 이용한 방법을 사용하며, 응답 속도를 빠르게 하기 위하여 Redis를 이용한 관리 방법이 많이 사용된다.

    5. 서버 DB에서 관리 (Redis, RDBMS)

    로컬 스토리지와 세션 스토리지의 경우는 앞서 XSS 공격의 취약점이 크기 때문에 access 토큰에서도 권장하지 않는 방식이라 하였다.
    메모리의 경우, 페이지 이동과 새로고침 시, 소멸하는 문제가 있어 refresh 토큰에는 적합하지 않다.

    그렇다면 쿠키보다는 서버가 보안적으로 우수한 방식인 것을 알 수 있고, 서버보다는 쿠키가 stateless 하다는 토큰 인증 방식의 특징을 잘 살린 것을 알 수 있다.

### Access Token & Refresh Token 작동 방식

    앞서 access 토큰은 클라이언트에 저장되어 있고, refresh 토큰은 클라이언트 측(쿠키)에서 관리하는 경우와 서버에서 관리하는 경우로 나누었다.

    따라서, 요청 시 클라아이언트가 access 토큰 하나만 보내는 경우와 access 토큰과 refresh 토큰을 같이 보내는 경우로 나누어 작동 방식을 알아보자.

![accessToken_RefreshToken](/grammar/img/accessToken_RefreshToken.png)

    1. Access Token, Refresh Token 생성

    로그인 요청 : 
    사용자가 로그인 정보를 입력하고, 서버에 로그인 요청을 보낸다.

    토큰 발급 : 
    서버는 사용자의 인증 정보를 DB와 일치하는지 확인한 후, access 토큰과 refresh 토큰을 함께 발급한다. 

    access 토큰은 짧은 만료 시간을 가지며, refresh 토큰은 상대적으로 긴 만료 시간을 가진다. 이후 클라이언트나 서버에 알맞은 위치로 저장한다.

    2. Access Token 사용

    API 요청 : 
    클라이언트는 API 요청 시, access 토큰을 헤더에 포함하여 요청을 보낸댜.

    서버에서 JWT 검증 : 
    서버는 요청받은 access 토큰의 만료 기간을 확인한다.
    access 토큰이 유효한 동안에는 정상적으로 요청이 처리된다.

    3. Access Token 만료

    API 요청 : access 토큰이 만료되었다면, 클라이언트는 더 이상 api 요청을 수행할 수 없다. 이후 저장 위치에 따라 아래와 같이 진행한다.

    - 클라이언트에 저장한 경우 : 
    클라이언트는 서버로부터 access 토큰 만료 요청을 받아, refresh 토큰을 요청에 포함하여 재요청한다. 
    쿠키의 경우에는 자동으로 refresh 토큰이 요청에 포함되어 있으므로 이 과정이 생략된다. 이후, refresh 토큰을 이용하여 재발급을 요청한다.

    - 서버에 저장한 경우 : 
    서버는 access 토큰 만료를 확인하였으나, 서버 저장소에 저장된 refresh 토큰을 이용하여 재발급을 요청한다.

    - refresh 토큰 유효성 검증 : 
    refresh 토큰의 서명 부분을 이용하여 유효성을 검증한다. 만약 만료되었다면 재로그인을 요청한다.

    - 새로운 access 토큰 발급 : 
    Refresh 토큰이 유효하다면, refresh 토큰의 페이로드 부분을 이용해 새로운 access 토큰을 발급한다.

    refresh 토큰을 서버에 저장한 경우와 클라이언트에 저장한 방식의 가장 큰 차이점은 서버가 refresh 토큰을 획득하는 방법이다. 서버에 저장되어 있는 경우에는, 서버 DB에서 조회를 한 번 더 해야 하는 과정이 이루어진다.

    이를 통해 보안성은 당연히 높겠지만, 토큰 인증 방식의 사용 목적인 응답속도 면에서는 쿠키 방식 보다 떨어지지 않을까?

### Access Token 과 Refresh Token 로그아웃 방식

    1. 메모리, 토큰 Null 처리하기
    2. HTTP-Only 쿠키, Set-Cookie 헤더를 통해 쿠키를 만료시키기
    3. 서버, DB에서 토큰 제거하기
    4. BlackList에 추가하기

### Refresh Token 탈취

    refresh 토큰이 탈취당했다면, 어떤 상황이 발생할까?
    refresh 토큰은 access 토큰을 재발급받을 수 있기 때문에, 해커는 access 토큰을 재발급 받아 다양한 공격에 사용된다.

    사용자가 로그아웃 요청을 해서 refresh 토큰을 제거하였어도, 해커가 refresh 토큰을 이미 탈취한 상태고 서버 측에서 무효화 로직이 없다면, 공격을 막을 수 없을 것이다.

    그러면, refresh 토큰을 무효화할 수 있을까?
    이 역시 앞서 다룬 내용과 마찬가지로 BlackList에 등록하는 방법을 사용할 수 있다. 이 경우 서버에서 refresh 토큰을 관리하고 있는 경우에만 추적하여 사용이 가능하고, access 토큰 처럼 stateless 할 경우 이상현상을 감지하여 BlackList에 추가하는 방법을 사용한다.

    그렇다면, 어떻게 완벽하게 보완할 수 있을까?

    서비스에서 완벽한 보안을 보장하는 것은 매우 어려운 과제이다. 보안성을 높이다 보면 편의성이 떨어지고, 편의성을 높이다 보면 보안성이 떨어진다. 따라서, 이 둘의 적절한 균형을 찾고 보안적인 솔루션을 더하여 피해를 최소화하는 방법을 도입하는 것이다. 이 것이 Refresh Token의 도입 목적이기도 하다.  

### RTR (Refresh Token Rotation)

    refresh 토큰이 탈취당했을 경우, refresh 토큰의 만료 기간을 짧게 가져가면 피해를 최소화할 수 있지않을까?

    이에, Refresh-Refresh-Token을 도입할 수는 없으니, Refresh Token Rotation 기법을 이용하여 피해를 최소화 한다.

    Refresh Token Rotation 은 Access 토큰이 만료되어 Refresh 토큰을 요청할 때, 새로운 Refresh 토큰을 함께 발급받는 방법이다.

    이를 통해 피해를 최소화 할 수 있따.

![RTR](/grammar/img/RTR.png)

    먼저, 사용자가 Refresh Token을 이용하여 새로운 Refresh Token(RT2)를 발급받는다. 그러면 서버는 기존 Refresh Token(RT1)을 블랙리스트 처리하여 Replay Attack을 방지한다. 

    이후, 해커가 기존에 탈취한 Refresh Token(RT1)을 이용하여 재발급을 요청한다면 Replay Attack으로 간주하여 사용자와 연결된 모든 Refresh Token을 무효화시킨다.

    사용자의 Refresh Token을 모두 무효화시키는 이유는 해커가 먼저 재발급 요청을 했을 때를 방지하기 위함이다. 오른쪽 그림에서, 해커가 먼저 Refresh Token을 재발급하였어도, 사용자의 Refresh Token 재발급 요청에 의해 해커의 Refresh Token을 무효화시킬 수 있다.

    즉, RTR(Refresh Token Rotation)은 해커든 사용자든 Refresh Token에 대한 Replay Attack이 감지되면, 토큰을 모두 무효화시키는 방법으로 보안을 강화한다.

### Access Token & Refresh Token 저장 위치 조합

    1. Access (메모리) + Refresh (HttpOnly cookie) + Refresh 유효성 (서버 관리)
    access 토큰은 메모리에 저장하여 접근성을 높이고, refresh 토큰은 HttpOnly 쿠키에 저장하여 stateless 하다는 특성을 살린다.
    이 경우, 페이지가 새로고침 되면 access 토큰이 소멸되기 때문에, refresh 토큰 만으로 access 토큰을 재발급받는 로직을 구현한다.

    또한, refresh 토큰을 HttpOnly 쿠키와 서버에 모두 저장한다.
    이렇게 하는 이유는, 클라이언트 측에만 refresh 토큰이 저장될 경우 비정상적인 활동이 감지되었을 때 토큰을 강제로 만료시킬 방법이 없다. (JWT의 stateless 함을 살리고 싶어서..) 

    이를 통해, access 토큰이 만료되었다면, 우선적으로 HttpOnly의 redis 토큰을 JWT 서명부를 텅해 인증하고 만료기간이 다 되었다면, 로그인 창으로 이동시킨다. 해당 로직을 통해 stateless 함을 잘 활용할 수 있을 것이라고 생각했다. refresh 토큰이 만료되지 않았다면, DB와의 검증을 통해 refresh 토큰이 유효한지 확인한다. refresh 토큰이 BlackList에 추가되어 있거나, isAcive 한 지 확인하는 과정을 거쳐 최종적으로 RTR 기법을 통해 두 토큰을 다시 저장한다.

![access_refresh_save1](/grammar/img/access_refresh_save1.png)

    2. Access Token (HttpOnly cookie) + Refresh Token (HttpOnly cookie)

    두 토큰 모두 클라이언트에 저장하여 stateless 한 특성을 잘 살리고, HttpOnly 쿠키에 저장하여 보안성 또한 높인 방식이다.
    보안적으로 하나의 쿠키에 두 토큰을 모두 저장하는 것이 아닌, 각각의 토큰별로 쿠키를 만들어 사용한다. 쿠키의 특성에 따라 XSS 공격에 대한 저항력이 높고, 모든 요청에 두 토큰이 항상 포함되기 때문에 클라이언트 측에서 토큰을 헤더에 포함하는 구현이 불필요하다.

    반대로, 모든 요청에 두 토큰이 항상 포함되기 때문에 요청 시, 크기가 커지며, 모든 요청에 access 토큰이 포함되어 있기 때문에 CSRF에 취약하다는 단점이 생긴다.

    또한, 쿠키에는 크기 제한이 있어서 access 토큰의 크기를 고려해야 된다.
    또한, 1번 방법과 마찬가지로 refresh 토큰 추적을 위해 서버 측 관리가 필요할 것으로 생각된다.

    3. Access Token (메모리/HttpOnly cookie) + Refresh Token (서버)

    refresh 토큰을 서버에 저장하여, stateless 한 특성을 포기하고 보안성을 높인 방식이다. access 토큰이 만료되면 refresh 토큰을 DB에서 조회해야 하기 때문에 응답 속도 면에서는 떨어지겠지만, refresh 토큰의 탈취 걱정을 할 필요가 없다. 

    하지만 이 역시 많은 사용자가 access 토큰 갱신을 요청하는 상황이 온다면 서버 측에 부하가 올 수 있지 않을까? 

    access 토큰만 사용하는 방식보다는 토큰 인증 방식의 장점을 잘 살릴 수 있다.

### JWT + Token 흐름 요약

    [1] 클라이언트 로그인 요청 (이메일/비밀번호)
            ↓
    [2] 서버: 사용자 인증 (DB 확인)
            ↓
    [3] 서버: JWT 토큰 생성 (Access + Refresh)
            ↓
    [4] 클라이언트에 응답: JWT 토큰 전달 (Cookie 또는 Header)
            ↓
    [5] 이후 요청 시, 클라이언트 → AccessToken 전송 (Header 또는 Cookie)
            ↓
    [6] 서버: 토큰 검증 (SecretKey로 Signature 확인 + 만료 체크)
            ↓
    [7] 토큰 유효하면 요청 처리 / 만료 시 Refresh 사용 또는 재로그인

    Access Token 구조 (JWT)
    eyJhbGciOi...  // header
    .eyJzdWIiOi... // payload
    .ZL4oKSpG...   // signature (→ HMAC_SHA256(Base64(header.payload), secretKey))

    1️⃣ 로그인 요청 (Client → Server)

    POST /login
    Content-Type: application/json

    {
        "email": "user@example.com",
        "password": "password123"
    }

    2️⃣ 서버: 로그인 처리 & JWT 발급

    // 로그인 성공 시
    String accessToken = jwtUtil.createAccessToken(user);
    String refreshToken = jwtUtil.createRefreshToken(user);

    // DB or Redis에 RefreshToken 저장 (선택)
    refreshTokenRepository.save(user.getId(), refreshToken);

    3️⃣ 서버 → 클라이언트 응답

    Header 방식 (보통 AccessToken만)
    HTTP/1.1 200 OK
    Authorization: Bearer {accessToken}
    Set-Cookie: refreshToken=...; HttpOnly; Secure; Path=/auth/refresh

    Cookie 방식 (Access + Refresh 둘 다)
    response.addHeader("Set-Cookie", createHttpOnlyCookie("accessToken", accessToken));
    response.addHeader("Set-Cookie", createHttpOnlyCookie("refreshToken", refreshToken));

    ✅ 클라이언트 요청 시

    GET /api/profile
    Authorization: Bearer {accessToken} or Cookie: accessToken=...; refreshToken=...

    ✅ 서버 측 토큰 검증 (Filter 또는 Interceptor)

    String token = extractTokenFromHeaderOrCookie(request);
    Claims claims = jwtUtil.parseToken(token); // 여기서 Signature 검증 & 만료 체크

    if (claims.getExpiration().before(new Date())) {
        throw new ExpiredJwtException(...);
    }

    ✅ 만료되었을 경우 (Access → 만료 → Refresh 사용)

    // 쿠키에서 RefreshToken 추출
    String refreshToken = request.getCookie("refreshToken");

    // DB에서 해당 refreshToken이 유효한지 검증
    if (jwtUtil.validateToken(refreshToken)) {
        // AccessToken 재발급
        String newAccessToken = jwtUtil.createAccessToken(user);
        response.setHeader("Authorization", "Bearer " + newAccessToken);
    } else {
        // 재로그인 유도
        throw new TokenInvalidException("Refresh Token expired");
    }

    ✅ 실무에서 서버 & 클라이언트 주고받는 JSON 구조 예
    📤 서버 → 클라이언트 응답

    {
        "userId": 123,
        "nickname": "joon",
        "accessToken": "eyJhbGciOi...",
        "refreshToken": "eyJhbGciOi..."
    }

    📥 클라이언트 → 서버 요청 (헤더 방식)

    Authorization: Bearer eyJhbGciOi...

    ✅ 쿠키 저장 예시 (보안 설정)
    Set-Cookie: refreshToken=ey...; HttpOnly; Secure; SameSite=Strict; Path=/auth
    Set-Cookie: accessToken=ey...; HttpOnly; Secure; SameSite=Strict; Path=/

    ✅ 요약 정리
    항목	            내용
    🔐 secretKey	   서버만 알고 있는 키로, Signature 생성/검증에 사용
    📦 JWT 구조	        Header.Payload.Signature (Base64 + HMAC)
    🧾 로그인 시	     Access + Refresh Token 생성 후 응답에 포함
    💾 저장 위치	    클라이언트(Cookie), 서버(DB/Redis - Refresh만)
    🔍 인증 시	        Access Token → Signature + 만료 검증
    🔁 갱신	            Refresh Token으로 Access 재발급 (조건부)
    🧷 Secure flag	   HTTPS에서만 쿠키 전송 (중간자 공격 방지)
    ⚠️ HttpOnly	       자바스크립트에서 쿠키 접근 차단 (XSS 방지)





