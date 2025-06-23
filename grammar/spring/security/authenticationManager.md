##  AuthenticationConfiguration vs AuthenticationManager

1. AuthenticationManager
- 스프링 시큐리티에서 인증 처리를 담당하는 핵심 인터페이스
- 로그인 시, 유저가 입력한 id/email , pw 검증해주는 역할
- 보통 내부적으로 여러 AuthenticationProvider 를 가지고 실제 인증 로직 위임

2. AuthenticationConfiguration
- 스프링 시큐리티가 내부에서 생성해 놓은 AuthenticationManager 빈을 제공하는 설정 객체입니다.
- 직접 new AuthenticationManager()를 만들지 않고,
스프링 컨텍스트에서 관리하는 인증 관련 설정들을 바탕으로 만들어진 AuthenticationManager를 얻는 방법입니다.

-> 스프링 부트 2.0 이상부터는 AuthenticationManager 가 자동으로 생성되지만,
우리가 커스텀 필터에 주입하려면 직접 빈으로 가져와야 하므로
AuthenticationConfiguration.getAuthenticationManager() 를 사용한다.

### interface & implements

- 인터페이스 : org.springframework.security.authentication.AuthenticationManager

- 기본 구현체 : ProviderManager

### ProviderManager (implements)

public class ProviderManager implements AuthenticationManager {
    private List<AuthenticationProvider> providers;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        for (AuthenticationProvider provider : providers) {
            if (provider.supports(authentication.getClass())) {
                return provider.authenticate(authentication);
            }
        }
        throw new ProviderNotFoundException("No suitable AuthenticationProvider found");
    }
}

- authenticate() : 호출 시, 내부에 등록된 여러 AuthenticationProvider 중에서 suppors() 조건을 만족하는 provider에게 인증 위임

- 대표적인 provider 
  - DaoAuthenticationProvider: UserDetailsService + 비밀번호 검증
  - JwtAuthenticationProvider: JWT 인증 전용 (커스텀 구현 가능)
  - OAuth2LoginAuthenticationProvider: 소셜 로그인 인증

### Spring Boot 사용 시?  

@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration configuration) throws Exception {
    return configuration.getAuthenticationManager();
}

- AuthenticationConfigure 이 내부적으로 ProviderManager 를 만들어서 리턴
- 내부적으로 DaoAuthenticationProvider 를 포함하고 있고, 등록한 UserDetailsService & PasswordEncoder 를 기반으로 작동한다.

### 흐름 요약

AuthenticationManager (인터페이스)
     ↳ ProviderManager (기본 구현체)
            ↳ DaoAuthenticationProvider
                 ↳ UserDetailsService + PasswordEncoder 조합으로 사용자 인증

### 실무 활용 포인트
- 기본적으로 UsernamePasswordAuthenticationToken 기반 인증은
→ DaoAuthenticationProvider가 처리

- JWT, OAuth2 등은 따로 커스텀 AuthenticationProvider를 만들어
→ ProviderManager에 등록해서 처리 가능                 