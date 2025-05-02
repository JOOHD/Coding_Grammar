## 본문

### 1. 왜 NoArgsConstructor(AccessLevel.PROTECTED)를 사용하는가?

    Lombok 라이어브러리에서 제공하는 어노테이션으로, 매개변수가 없는 기본 생성자를 자동으로 생성해주지만, 접근 수준을 protected로 제한하는 역할이다.

    이 어노테이션은 주로 객체 생성의 제어와 안전한 생성 패턴을 위해 사용된다.

    ● 개별 요소 설명

    1. @NoArgsConstructor
       - 매개변수가 없는 기본 생성자를 자동으로 만들어준다.
       - 기본 생성자는 클래스가 인스턴스화 될 때 아무런 인자 없이 호출되는 생성자이다.
       - 이 생성자가 자동으로 생성되므로, 개발자가 직접 기본 생성자를 작성할 필요가 없다.

    2. access = AccessLevel.PROTECTED
       - 기본 생성자의 접근 수준을 protected로 설정한다. 이는 Lombok의 AccessLevel 열거형에 정의된 값 중 하나이다.
       - protected 접근 수준을 같은 패키지 내의 클래스나 상속 관계에 있는 클래스만 이 생성자를 호출할 수 있음을 의미.
       즉, 외부에서 이 생성자로 객체를 직접 생성하는 것을 제한한다. 

    Entity나 DTO를 사용할 때, @NoArgsConstructor(AccessLevel.PROTECTED) 어노테이션을 많이 사용하는 편이다.

    기본 생성자의 접근 제어를 PROTECTED로 설정해놓게 되면 무분별한 객체 생성에 대해 한번 더 체크할 수 있는 수단이 되기 때문이다.

    ex) 
        User라는 Class는 name, age, email 정보를 모두 가지고있어야만 되는 상황일 경우에 기본 생성자를 막는것은 이를 도와주는 좋은 수단이다.
    
    만약 기본 생성자의 권한이 public이라면 아래 상황이 발생한다.

    @Getter
    @Setter
    @NoArgsConstructor
    public class User {
        private String name;
        private Long age;
        private String email;
    }

    public static void main(String[] args) {
        User user = new User();
        user.setName("testname");
        user.setEmail("test@test.com");

        // age가 설정되지 않았으므로 user는 완전하지 않은 객체
    }

    User의 멤버변수들을 설정할 방법이 없으니 Setter를 만들어서 값을 설정하지만 실수로 setAge()를 누락할 셩우 객체는 불완전한 상태가 된다.

    하지만 아래와 같이 변경하게 되면 IDE 단계에서 누락을 방지할 수 있게 되어 훨씬 setAge()를 누락할 경우 객체는 불완전한 상태가 된다.

    public class User {
        private String name;
        private Long age;
        private String email;

        public User(Long age, String email) {
            // 파라미터가 두 개인 경우 name은 default 설정
            this.name = "blank name';
            this.age = age;
            this.email = email;
        }
    }

    public static void main(String[] args) {
        User user = new User(15, "test@email.com");

        // 기본 생성자가 없고 객체가 지정한 생성자를 사용해야하기 떄문에 
        // 무조건 완전한 상태의 객체가 생성되게 된다.
    }

### 2. @NoArgsConstructor(AccessLevel.PROTECTED) & @Builder 사용

    @Builder
    @NoArgsConstructor(access = AccessLevel.PROTECTED)
    public class User {
        private String name;
        private Long age;
        private String email;
    }

    위 처럼 @NoArgsConstructor(AccessLevel.PROTECTED) & @Builder 사용하면, 바로 에러가 발생한다.

    ● error message

        error : contructor User in class User cannot be applied to given types; @Builder
        [
            required : no arguments
            found : String, Long, String
            reason : actual and foraml argument lists differ in length
        ]

    @Builder는 Class(Type)이 Target일 경우에 생성자 유무에 따라 아래와 같이 동작한다.

    - 생성자가 없는 경우 : 모든 맴버변수를 파라미터로 받는 기본 생성자 생성.
    - 생성자가 있을 경우 : 따로 생성자 생성X
  
    위 과정 이후에 모든 멤버 변수를 설정할 수 있는 Builder class를 생성한다.
    그런데 만약 @NoArgsConstructor(AccessLevel.PROTECTED)라는 생성자가 있는 상태에서 @Builder를 사용하면, 아래와 같이 컴파일 된다.

    public class User {
        private String name;
        private Long age;
        private String email;

        // @NoArgsConstructor(AccessLevel.PROTECTED)로 생성된 생성자
        protected User() {}

        public static USER.UserBuilder builder() {
            return new User.UserBuilder();
        }

        public static class UserBuilder {
            private String name;
            private Long age;
            private String email;

            UserBuilder() {
            }

            public User.UserBuilder name(String name) {
                this.name = name;
                return this;
            }

            public User.UserBuilder age(Long age) {
                this.age = age;
                return this;
            }

            public User.UserBuilder email(String email) {
                this.email = email;
                return this;
            }

            public User build() {
                /// 일치하는 생성자가 없다.
                return new User(this.name, this.age, this.email); 
            }
        }
    }

    User에는 기본 protected 생성자가 이미 존재해서 따로 생성자를 만들지 않았지만, build()를 보면 모든 파라미터를 받는 생성자로 객체를 build하려는 과정에서 알맞는 생성자를 찾을 수 없게 되었다.

### 3. @NoargsConstructor(AccessLevel.PROTECTED)와 @Builder를 함께      사용할 수 없을까?    

    불가능하다고 하더라도 @NoargsConstructor(AccessLevel.PROTECTED)와 @Builder는 의미없는 객체를 생성하기 위한 좋은 방법이자 제약조건이다.

    함께 사용하기 위한 방법은 위 예제들을 정확히 이해했다면 어렵지 않다.

    ● 해결 방법

    1. @AllArgsConstructor

    @Builder
    @AllArgsConstructor 
    public class User {
        private String name;
        private Long age;
        private String email;
    }

    모든 멤버 변수를 받는 생성자가 없는것이 이유이기 떄문에 모든 멤버변수를 받는 생성자를 만들어주면 된다.

    2. 생성자에 설정하는 @Builder
    
    @NoArgsConstructor(access = AccessLevel.PROTECTED)
    public class User {
        private String name;
        private Long age;
        private String email;

        @Builder
        public User(Long age, String email) {
            this.name = "test_name";
            this.age = age;
            this.email = email;
        }

        @Builder
        public User(String name, Long age, String email) {
            this.name = name;
            this.age = age;
            this.email = email;
        }
    }

    생성자별로 설정되는 멤버변수 내용을 정의하고 생성자에 @Builder를 설정하게되면 해당 생성자를 사용하는 Builder가 생성되어 의미있는 객체만 생성할 수 있게 된다.

    위 코드를 Java 코드만으로 변환하여 작성하게 되면 아래와 같다.

    public class User {
        private String name;
        private Long age;
        private String email;

        public User(Long age, String email) {
            this.name = "test_name";
            this.age = age;
            this.email = email;
        }

        public User(String name, Long age, String email) {
            this.name = name;
            this.age = age;
            this.email = email;
        }

        public static User.UserBuilder builder() {
            return new User.UserBuilder();
        }

        protected User() {
        }

        public static class UserBuilder {
            private String name;
            private Long age;
            private String email;

            UserBuilder() {
            }

            public User.UserBuilder age(Long age) {
                this.age = age;
                return this;
            }

            public User.UserBuilder email(String email) {
                this.email = email;
                return this;
            }

            public User.UserBuilder name(String name) {
                this.name = name;
                return this;
            }

            public User build() {
                return new User(this.name, this.age, this.email);
            }
        }
    }

