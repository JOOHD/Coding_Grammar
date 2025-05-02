## 본문

### Spring Security PrincipalDetails

    스프링 시큐리티는 "/login" 주소 요청이 오면 해당 요청을 낚아채서 로그인을 진행시킬 수 있다.

    -> WebSecurityConfigurerAdapter 를 상속받아 configure을 override 한 후, 다음 로직 작성

    @EnableWebSecurity // 해당 파일로 시큐리티를 활성화
    @Configuration // IoC
    