## 본문

### Builder pattern

    1. 생성자 파라미터 생성 
    
    복잡한 객체의 생성 과정과 표현 방법을 분리하여 다양한 구성의 인스턴스를 만드는 생성 패턴이다. 
    생성자에 들어갈 매개 변수를 메서드로 하나하나 받아드리고 마지막에 통합 빌드해서 객체를 생성하는 방식이다.

        일반 객체 생성 (생성자 이용)
        Bag bag = new Bag("name", 1000, "memo");

        빌더 패턴
        Bag bag = Bag.builder()
                        .name("name")
                        .money(1000)
                        .memo("memo")
                        .build();

    이해하기 쉬운 사례로 수제 햄버거를 들 수 있다. 
    수제 햄버거를 주문할때 빵이나 패티 등 속재료들은 주문하는 사람이 마음대로 결정된다.
    어느 사람은 치즈를 빼달라고 할 수 있고, 어느 사람은 토마토를 빼달라고 할 수 있다. 
    이처럼 선택적 속재료들을 보다 유연하게 받아 다양한 타입의 인스턴스를 생성할 수 있어, 클래스의 선택적 매개변수가 많은 상황에서 유용하게 사용된다.

### 핵심 결론

    밑에 글들도 결국에는 생성자 파라미터가 많아지면 가독성이 떨어지는 비효율을 막기 위해 보완된 기능이 빌더 패턴인 것인데, 핵심은

    1. (Builder builder)
    
    Member member = new Member.Builder(1L, "John Doe".....)
    이렇게 파라미터가 많아지는 불편함을,

    public Member(Builder builder)
    처럼 command object와 같은 개념으로 빌더 클래스 객체를 선언해서 묶는다.

    2. static class (inner class)

    - Builder builder 객체에 어떤 값들이 담겨있는지 모를 수 있다.
    - 외부 클래스에 필드 인스턴스가 담겨 있는 것(필요한 값만 뽑아 올 수도 있음.)
    - static class = 빌더 객체가 외부 클래스의 필드 값을 안전하고 일관되게 전달하는 목적.
      
    ex)

        public class Member {
            private final Long id;       // 필수 필드
            private final String name;   // 필수 필드
            private final String address; // 선택 필드

            // Member의 생성자, 빌더 클래스에서 사용됨
            public Member(Builder builder) {
                this.id = builder.id;           // 빌더 객체의 필드 값을 Member로 옮김
                this.name = builder.name;
                this.address = builder.address;
            }

            // 정적 빌더 클래스
            public static class Builder {
                private Long id;          // 필수 필드
                private String name;      // 필수 필드
                private String address;   // 선택 필드

                // 필수 필드를 빌더의 생성자로 받음
                public Builder(Long id, String name) {
                    this.id = id;
                    this.name = name;
                }

                // 선택 필드를 설정하는 메서드
                public Builder address(String address) {
                    this.address = address;
                    return this;  // 빌더 객체를 반환해 체인처럼 이어지게 함
                }

                // 최종적으로 Member 객체를 생성하는 메서드
                public Member build() {
                    return new Member(this);  // Member 생성자를 호출해 객체 생성
                }
            }
        }

    1. 외부/이너 클래스
        - Member = 외부 클래스, Builder = 이너 클래스(static) 
        - Builder 는 Memebr 의 필수 필드와 선택 필드를 저장하고, 최종적으로 build() 메서드를 통해 Member 객체를 생성.
    
    2. Builder 클새스에서 값 설정
        - Builder 클래스에서는 Member 객체의 필드를 설정할 수 있는 메서드들을 제공한다. 그 값들은 Builder 클래스의 인스턴스 변수에 저장.
            ex)
            Member.Builder(1L, "JOO")는 Builder의 id와 name 필드를 설정하고, 이후 address, email 등의 선택적인 필드는 메서드를 통해 설정

    3. Member(Builder builder) 생성자에서 값을 받는 과정
        - Member 클래스는 Builder 객체로부터 필드를 받아 자신의 필드를 초기화ㅏ 한다. 이렇게 하면 Builder에 설정된 값들이 최종적으로 Member 객체에 저장된다.

            ex)
            public Member(Builder builder) {
                this.id = builder.id;
                this.name = builder.name;
                this.address = builder.address;
            }

            - 이 과정에서 Builder 내부의 값들이 Member로 옮겨지며, 외부에서 직접 Member의 필드에 접근할 필요 없이, 안전하게 객체가 생성된다.

    4. 필요한 인스턴스만 가져오는 유연성
        - Builder 패턴의 장점 중 하나는, 필수적인 값들만 생성자에서 받고, 나머지 선택적인 값들은 체이닝 방식으로 필요한 만큼만 설정할 수 잇다.

    5. 결론
        외부 클래스의 필드 인스턴스가 Builder를 통해 필요한 값들만 가져와서 객체를 생성하는 것이 필더 패턴의 핵심이다.        

### 점층적 생성자 패턴

    필수 매개변수와 함께 선택 매개변수를 0개, 1개, 2개... 받는 형태로, 우리가 다양한 매개변수를 입력받아 인스턴스를 생성하고 싶을 때, 사용하던 생성자를 오버로딩 하는 방식이다.

    ● Hambuger class

        class Hanbufer {
            // 필수 매개변수
            private int bun;
            private int patty;

            // 선택 매개변수
            private int cheese;
            private int lettuce;
            private int tomato;
            private int bacon;

            public Hamburger(int bun, int patty, int cheese, int lettuce, int tomato, int bacon) {
                this.bun = bun;
                this.patty = patty;
                this.cheese = cheese;
                this.lettuce = lettuce;
                this.tomato = tomato;
                this.bacon = bacon;
            }

            public Hamburger(int bun, int patty, int cheese) {
                this.bun = bun;
                this.patty = patty;
                this.cheese = cheese;
            }
        }

        ● main class

            public static void main(String[] args) {
                // 모든 재료가 있는 햄버거
                Hamburger hamburger1 = new Hamburger(2, 1, 2, 4, 6, 8);

                // 빵과 패티 치즈만 있는 햄버거
                Hamburger hamburger2 = new Hamburger(2, 1, 1);
            }

    하지만 이러한 방식은 클래스 인스턴스 필드들이 많으면 많을 수록 생성자에 들어갈 인자의 수가 늘어나 몇번째 인자가 어떤 필드였는지 햇갈릴 경우가 있다.

### Java bean pattern

    이러한 단점을 보완하기 위해 setter 메서드를 사용한 자바 빈 패턴이 고안 되었다.

    매개 변수가 없는 생성자로 객체 생성후 Setter 메소드를 이용해 클래스 필드의 초깃값을 설정하는 방식이다.

    ● Hambuger class (Java bean 적용)

        class Hamburger {
        // 필수 매개변수
        private int bun;
        private int patty;

        // 선택 매개변수
        private int cheese;
        private int lettuce;
        private int tomato;
        private int bacon;
        
        public Hamburger() {}

        public void setBun(int bun) {
            this.bun = bun;
        }

        public void setPatty(int patty) {
            this.patty = patty;
        }

        public void setCheese(int cheese) {
            this.cheese = cheese;
        }

        public void setLettuce(int lettuce) {
            this.lettuce = lettuce;
        }

        public void setTomato(int tomato) {
            this.tomato = tomato;
        }

        public void setBacon(int bacon) {
            this.bacon = bacon;
        }
    }

    ● main class

        public static void main(String[] args) {

            // 모든 재료가 있는 햄버거
            Hamburger hamburger1 = new Hamburger();
            hamburger1.setBun(2);
            hamburger1.setPatty(1);
            hamburger1.setCheese(2);
            hamburger1.setLettuce(4);
            hamburger1.setTomato(6);
            hamburger1.setBacon(8);

            // 빵과 패티 치즈만 있는 햄버거
            Hamburger hamburger2 = new Hamburger();
            hamburger2.setBun(2);
            hamburger2.setPatty(1);
            hamburger2.setCheese(2);

        }

    기존 생성자 오버로딩에서 나타났던 가독성 문제점이 사라지고 선택적인 파라미터에 대해 해당되는 Setter 메서드를 호출함으로써 유연적으로 객체 생성이 가능해졌다. 

    하지만 이러한 방식은 객체 생성 시점에 모든 값들을 주입 하지 않아 일관성 문제와 불변성 문제가 나타나게 된다.


    1. 일관성 문제
        필수 매개변수란 객체가 초기화될때 반드시 설정되어야 하는 값이다. 
        
        하지만 개발자가 실수로 set() 메서드를 호출하지 않았다면 이 객체는 일관성이 무너진 상태가 된다.(RuntimeException 발생)

    2. 불변성 문제    
        자바 빈즈 패턴의 Setter 메서드는 객체를 처음 생성할때 필드값을 설정하기 위해 존재하는 메서드이다.

        하지만 객체를 생성했음에도 여전히 외부적으로 Setter 메소드를 노출하고 있으므로, 협업 과정에서 언제 어디서나 누군가 Setter 메서드를 호출해 함부로 객체를 조작할 수 있게 된다.

        마치 완성된 햄버거에 중간에 치즈를 교체한다고 햄버거를 막 분리하는 것과 같은 이치이다.

### Builder pattern

    빌더 패턴은 이러한 문제들을 해결하기 위해 별도의 Builder 클래스를 만들어 메소드를 통해 step - by - step으로 값을 입력받은 후에 최종적으로 build() 메소드로 하나의 인스턴스를 생성하여 리턴하는 패턴이다.

    빌더 패턴 사용법을 잠시 살펴보면, StudentBuilder 빌더 클래스의 메서드를 체이닝(Chaining) 형태로 호출함으로써 자연스럽게 인스턴스를 구성하고 마지막에 build() 메서드를 통해 최종적으로 객체를 생성하도록 되어있음을 볼 수 있다.

    ● main class (builder pattern 적용)

        public static void main(String[] args) {

        // 생성자 방식
        Hamburger hamburger = new Hamburger(2, 3, 0, 3, 0, 0);

        // 빌더 방식
        Hamburger hamburger = new Hamburger.Builder(10)
            .bun(2)
            .patty(3)
            .lettuce(3)
            .build();
        }

    빌더 패턴을 이용하면 더 이상 생성자 오버로딩 열거를 하지 않아도 되며, 데이터의 순서에 상관없이 객체를 만들어내 생성자 인자 순서를 파악할 필요도 없고 잘못된 값을 넣는 실수도 하지 않게 된다.

### 빌더 패턴 va @Data(Command Object)

    빌더 패턴의 목적이 코드 간결화를 위한 기능이라고 알고있다.
    즉, 생성자나 setter 메서드를 사용하여 필드를 초기화, 많은 필드를 가진 객체의 경우 초기화 코드가 길어지며 가독성이 떨어지는 부분을 보완하기 위한 기능이라고 알고있다.

    그런데 의문점이 생성자 or setter 생성은 Lombok에서 제공해주는 @Data 어노테이션도 같은 기능을 가진 것으로 알고 있다.
    (@Data 어노테이션은 생성자를 정의할 필요 없이 간편하게 객체를 생성하고,
    필드를 초기화할 수 있다.)

    빌더 패턴은 정적 데이터만 다루는 줄 알았다. 그래서 생성자 setter 메서드를 생성해주는 기능인 @Data를 사용해서 command object와 함꼐 사용하게 되면 생성자 생성, 필드 초기화, setter 메서드 생성, 동적 데이터 구현이라는 빌더 패턴 보다 더 많은 기능을 가진 @Data를 사용한 command object를 사용하는게 더 효율적이라고 생각했다.

### 빌터 패턴 @Data 공존

    빌더 패턴과 @Data 어노테이션은 함께 공존 가능하다.
    이 둘은 서로 다른 목적을 갖고 있으며, 동시에 사용하면 객체의 편리한 생성과 데이터 접근을 모두 지원할 수 있다.

    ● @Data & 빌더 패턴 함께 사용하는 경우

        @Data
        @Builder
        public class AdminProductDto {
            private Integer id;
            private Integer productCategoryId;
            private String name;
            private String description;
            private Integer price;
            private SoldOutStatus soldOutStatus;
            private String picture;
        }

    하지만 내 생각에는 @Data 와 @Builder 패턴을 같이 사용하는 것은 별로 인 것 같다.
    두 가지 접근 방식이 각기 다른 목적을 가지고 있기 때문이다.

### 빌더 패턴과 @Data 공존 시, 문제점

    1. 중복 기능

    - 빌더 패턴은 객체 생성 시 필드를 명시적으로 지정하여 가독성을 높이고, 필수 필드 강제 등의 기능을 제공.
    
    만약 두 기능을 함께 사용하면, 필드를 설정하는 방식이 두 개가 된다.
    builder() 메서드로 설정할 수 있고, 동시에 setter() 메서드로도 값을 넣을 수 있다. 이렇게 되면 객체 설정 방식이 일관되지 않게 되어, 코드의 명확성이 떨어진다. 

    2. 명확하지 않은 의도

    - 빌더 패턴을 쓰는 이유는 안정적으로 객체를 생성하고, 필수 필드를 강제하거나 유연하게 선택적 필드를 설정하기 위함. 
    @Data로 setter를 열어 두면, 객체를 빌더로 만들다가 갑자기 setter로 필드를 바꾸는 일이 생긴다.

    결론 

    빌더 패턴 : 명시적으로 필드를 설정하는 것에 초점을 맞추고, 필요치 않은 setter는 제공하지 않는 것이 좋다.

    @Data : DTO 나 간단한 Entity 클래스처럼 객체의 필드를 유연하게 변경해야 할 때 사용하는 것이 더 적합.


### 빌더 패턴 동적 데이터 기능

    ● 동적 코드

        public AdminProductDto createProduct(String name, int price, String description) {
            // 동적으로 사용자 입력 데이터를 바탕으로 객체를 생성
            return AdminProductDto.builder()
                .name(name)
                .price(price)
                .description(description)
                .build();
        }

        public void enrollProduct(String name, int price, String description) {
            AdminProductDto product = createProduct(name, price, description);
        }

### static class (정적 내부 클래스)

    ● 개념

    정적 내부 클래스라는 심플 빌더 패턴 방법 중 하나이다.
    생성자가 많을 경우, 변경 불가능한 불변 객체가 필요한 경우 코드의 가독성과 일관성, 불변성을 유지하는 것에 중점을 둔다.
    (우리가 사용하는 빌더 패턴과 차이는 거의 없다.) 
    다만 다른 점은 빌더 클래스가 구현할 클래스의 내부에 static class를 구현한다는 점이다. 

    ● 사용 이유

    빌더 패턴을 가능하게 만들고, 객체 생성을 더 유연하고 안전하게 만들기 위함.

    ● 일반적인 생성자 방식 문제점
    public InquiryDto(Long inquiryId, Long memberId, String inquiryTile) {
        this.inquiryId = inquiryId;
        this.memberId = memberId;
        this.inquiryTitle = inquiryTitle;
    }

    위와 같은 방식으로 객체를 생성하면, setter 를 사용해 객체 상태를 변경할 가능성이 있다.

        InquiryDto dto = new InquiryDto(1L, 2L, "상품문의");
        dto.setInquiryTitle("변경된 문의 제목"); // 필드 변경 가능 -> 불변성 깨짐

        -> 이 문제를 해결하기 위해, 빌더 패턴을 활용하면 setter 없이 필드 변경을 방지

        -> 빌더를 사용하면 setter 없이 객체가 생성된 이후 필드 변경이 불가능 

        ● 빌더 패턴 적용    
        InquiryDto dto = InquiryDto.builder()
            .inquiryId(1L)
            .memberId(2L)
            .inquiryTitle("상품 문의")
            .build();   
        // dto.setInquiryTitle(변경된 제목"); // 불가능 -> 불변성 유지

    ex)
    class Person {
        String name;
        int age;

        // 정적 내부 빌더 클래스
        public static class Builder {
            String name;
            int age

            Builder name(String name) {
                this.name = name;
                return this;
            }

            Builder age(int name) {
                this.age = age;
                return this;
            }
        }

        private Person(Builder builder) {
            this.name = builder.name;
            this.age = builder.age;
        }
    }

    위 코드에서 Person.Builder는 정적으로 선언되어 있기 때문에, Person 객체를 먼저 만들지 않아도 Builder 클래스를 사용할 수 있다.

    일반적으로, 비정적 클래스는 외부 클래스의 인스턴스가 필요하다.
    만약 Builder가 정적이 아니라면, Person 인스턴스를 먼저 만들어야 Builder 인스턴스도 만들 수 있다. 

    그러나 빌더 패턴의 핵심은 객체를 유연하게 생성하는 것이므로, 외부 클래스의 인스턴스 없이도 Builder 객체를 바로 생성할 수 있어야 한다.

### @Builder vs @AllArgsConstructor
	
	● @Builder 선호
	1. 필드를 한 줄씩 명시적으로 할당할 수 있어 가독성이 높다.

	2. 매개변수의 순서와 관계없이 필드를 설정할 수 있다. (반대로 @AllArgsConstructor 는 순서 중요), 메서드 체이닝 방식으로 순서를 자유롭게 지정 가능

	3. 특정 필드만 설정하고, 나머지 필드는 기본값 그대로 사용

    ● Builder 패턴을 사용하기 위해 생성자를 구현해야 한다.
    public InquiryDto(Long inquiryId, Long memberId, Long productId, String name, String email,
                    InquiryType inquiryType, String inquiryTitle, String inquiryContent, String password,
                    LocalDateTime createdAt, Boolean isSecret, Boolean isResponse, List<InquiryReplyDto> replies) {
        this.inquiryId = inquiryId;
        this.memberId = memberId;
        this.productId = productId;
        this.name = name;
        this.email = email;
        this.inquiryType = inquiryType;
        this.inquiryTitle = inquiryTitle;
        this.inquiryContent = inquiryContent;
        this.password = password;
        this.createdAt = createdAt;
        this.isSecret = isSecret;
        this.isResponse = isResponse;
        this.replies = replies;
    }

    @Builder
    Inquiry inquiry = Inquiry.builder()
        .product(product)
        .inquiryType(requestDto.getInquiryType())
        .inquiryTitle(requestDto.getInquiryTitle())
        .inquiryContent(requestDto.getInquiryContent())
        .password(requestDto.getPassword())
        .build();

    @AllArgsConstructor
    Inquiry inquiry = new Inquiry(
        product, 
        requestDto.getInquiryType(), 
        requestDto.getInquiryTitle(), 
        requestDto.getInquiryContent(), 
        requestDto.getPassword()
    );

    ● new (...) vs .builder()
    
    Entity 클래스 생성자 사용 (new (...))
    - 모든 필드 순서대로 입력해야 함
    - 불필요한 값도 넣어야 할 수도 있음
    - 특정 필드는 생략 가능 (null 값이 들어감).

    .builder()
    - 필요한 값만 선택해서 설정 가능
    - 순서 상관 없음
    - 일부 필드를 선택적으로 설정하고 싶을 때 적합.
    - 기본값을 자동으로 설정하는 필드 (예: createdAt = LocalDateTime.now())가 있는 경우 더욱 유용.
