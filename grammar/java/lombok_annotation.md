## 본문

### lombok annotation 

    어노테이션 기반으로 코드 자동완성 기능을 제공하는 라이브러리.

    Spring, Spring Boot 로 Web 개발을 하다보면 반복되는 코드가 자주 등장하며 가독성을 떨어트린다.

    예를 들어보면 Getter, Setter, ToString, Constructor(생성자)가 대표적인 예제일 것이다. 

### @NoArgsConstructor

    : 파라미터가 없는 디폴트 생성자 생성.

    ● 기본 생성자가 필요한 이유
        - 직렬화/역직렬화 과정에서 기본 생성자가 필요하다.
        - JSON 변환 라이브러리(Jackson, Gson)가 DTO를 매핑할 때 기본 생성자가 없으면 객체를 만들 수 없다.
        - Spring에서 DTO를 자동으로 매핑할 때도 필요할 수 있음.

    ● 왜 기본 생성자에 @Builder를 붙이지 않았을까?    
        - @Builder 는 필드를 받아서 객체를 생성하는 패턴이므로, 기본 생성자와는 맞지 않다.
        - 기본 생성자는 아무 필드도 초기화하지 않기 때문에 @Builder 를 붙일 필요가 없다.

    ● 주의점

        - 필드들이 final로 생성되어 있는 경우에는 필드를 초기화할 수 없기 때문에, 생성자를 만들 수 없어 에러가 발생한다.
        - 이 때, @NoArgsConstructor(force = true)옵션을 이용해 final 필드를 O, false, null 등으로 강제 초기화시켜 생성자를 만들 수 있다.

        - @NonNull 같이 필드에 제약조건이 설정되어 있는 경우, 생성자 내 null-check 로직이 생성되지 않는다.
        - 후에 초기화를 진행하기 전까지 null-check 로직이 발생하지 않는 점을 염두해야 한다.

    ex)

        @NoArgsConstructor
        public class User {
            private String name;
            private int age;

            // Lombok이 자동으로 아래와 같은 기본 생성자를 생성해줍니다.
            // public User() {
            //     this.name = null;
            //     this.age = 0;
            // }
        }

### @RequiredArgsConstructor

    : final & @NonNull 마크된 필드만을 파라미터로 받는 생성자를 자동 생성.
    (일반 필드는 포함되지 않으므로, 해당 필드들은 기본값으로 초기화됩니다.)

    ● 주의점
    
    - 파라미터의 순서는 클래스에 있는 필드 순서에 맞춰 생성자가 생성.
    - @NonNull 필드들은 null-check 가 추가적으로 생성되며, @NonNull이 마크되어 있어도 파라미터에서 null 값이 들어온다면 생성자에서 NullPointerException이 발생한다.

    ex)

        @RequiredArgsConstructor
        public class Member {
            private final String nickname;
            private final String password;
            private Gender gender;  // 일반 필드

            // Lombok에 의해 자동 생성되는 생성자
            // public Member(String nickname, String password) {
            //     this.nickname = nickname;
            //     this.password = password;
            // }
        }

    - 이 경우, nickname, password 필드는 final로 선언되어 있어, 생성자에서 반드시 초기화해야 한다.

    - 하지만 gender 필드는 final이 아니기 때문에 생성자에서 초기화하지 않고 기본값인 null로 초기화된다.
  
    ex)
        Member member = new Member("joo", "1234");  // gender는 null로 초기화

### @AllArgsConstructor

    ● 개념
      - 모든 필드를 포함하는 생성자를 자동으로 생성하는 어노테이션이다.
      - 즉, 모든 필드를 초기화하는 생성자를 자동으로 만들어주는 역할을 한다.
    
    ● 목적
      - DTO or Entity 에서 객체를 빠르게 생성하기 위해 사용.
      - new Object(값1, 값2, ...., 값N); 처럼 객체를 한 번에 생성할 수 있도록 도와줌.
    
    ● @Builder 와 공생할 수 없는 이유

    @AllArgsConstructor 와 @Builder 를 함께 사용하면 충돌이 발생할 가능성이 크다.

    1. Builder 의 작동 방식
        - @Builder 는 빌더 패턴을 적용한 객체 생성을 지원함
        - 내부적으로 빌더 클래스를 자동 생성하고, Object.builder().objectId(1L).name("홍길도").build(); 형태로 객체를 생성.

    2. 충돌이 발생하는 이유
        - @AllArgsConstructor 가 존재하면 Lombok 이 모든 필드를 받는 생성자를 생성한다.
        - @Builder 도 생성자를 기반으로 빌더 메서드를 만드는데, @AllArgsConstructor 가 있으면 생성자와 빌더 생성 규칙이 충돌할 수 있다.
        - 그래서 @AllArgsConstructor 를 사용하면 빌더가 정상적으로 동작하지 않을 가능성이 높다.

    해결 방법 -> @AllArgsConstructor 를 제거하고, 대신 생성자에 @Builder 를 적용하는 것이 일반적이다.       

    ● 단점 

    - 생성자의 파라미터 순서를 반드시 맞춰야 한다.
    - 필드가 많아질수록 실수할 가능성이 높다  