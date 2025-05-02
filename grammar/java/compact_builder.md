## 본문

### compact() vs build()

    compact

    역할 : JWT 토큰을 문자열로 압축하고 직렬화하는 메서드이다.

    동작 : Jwts.builder() 를 통해 구성한 JWT의 Header, Payload, Signature 을 조합하여 최종적으로 token을 생성한다.

    String jwtToken = Jwts.builder()
        .setSubject("user")
        .setExpiration(expirationDate)
        .signWith(secretKey, SignatureAlgorithm.HS256)
        .compact(); // 최종 JWT 문자열 생성

    build

    역할 : 객체 생성을 최종화하는 메서드이다.

    동작 : 객체의 구성 작업이 완료된 뒤, 최종적으로 객체를 반환.

    Object obj = Object.builder()
        .field("value1")
        .field("value2")    
        .build(); // 최종 객체 생성

### token ValidateExpire 메소드의 로직 중, build() 중간에 들어가는 이유

    Jwts.parserBuilder().build() 는 JWT 파싱 설정을 구성하기 위한 빌더 객체를 반환한다. 
    build() 는 그 빌더 객체를 JWT 파서로 최종 변환하는 역할을 하며, 이후 토큰 검증을 위해 마지막이 아닌, 중간에 온다.

    1. compact() - JWT 문자열 생성
    ex)
        public static void main(String[] args) {
            String secretKey = "mySecretKey"; // 토큰 서명에 사용할 키
            long expirationTime = 1000 * 60 * 60; // 1시간 유효기간

            // 토큰 생성
            String jwtToken = Jwts.builder()
                    .setSubject("user123") // 사용자 식별자
                    .setIssuedAt(new Date()) // 발급 시간
                    .setExpiration(new Date(System.currentTimeMillis() + expirationTime)) // 만료 시간
                    .signWith(SignatureAlgorithm.HS256, secretKey) // 서명 알고리즘 및 키
                    .compact(); // 최종 JWT 문자열 반환

            System.out.println("Generated JWT Token: " + jwtToken);
        }

        compact(jwtToken) :eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIiwia....

    2. build() - JWT 파서 생성
    ex)
        public static void main(String[] args) {
            String secretKey = "mySecretKey"; // 서명 검증에 사용할 키
            String jwtToken = "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIiwiaWF0IjoxNjkxNjgyNjAwLCJleHAiOjE2OTE2ODYyMDB9.S3uDs4YRgWn5MZwoEZ-5F1ey_GtpsXN_FmjLdjMsbM0";

            try {
                // 토큰 검증 및 파싱
                Claims claims = Jwts.parserBuilder()
                        .setSigningKey(secretKey) // 서명 키 설정
                        .build()                  // JWT 파서 객체 생성
                        .parseClaimsJws(jwtToken) // 토큰 검증 및 파싱
                        .getBody();               // 페이로드 추출

                // 검증 후 페이로드 데이터 출력
                System.out.println("Subject: " + claims.getSubject());
                System.out.println("Expiration: " + claims.getExpiration());
            } catch (Exception e) {
                System.out.println("Invalid JWT Token: " + e.getMessage());
            }
        }    