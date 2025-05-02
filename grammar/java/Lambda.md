## 본문

### 개요

    Java에는 두 번의 큰 변화가 있었는데, 그 중 한번은 JDK 1.6부터 추가된 Generics 기능, 한 번은 JDK 1.8 (SE8) 부터 추가된 람다식이다.

    - 특히 람다식 도입으로 자바는 객체 지향 언어이자 동시에 함수형 언어가 되었다.

    객체 지향 언어가 함수형 언어의 기능까지 갖추기는 어렵다고들 한다.
    이 부분에 대해서 어떻게 람다식은 함수형의 언어 기능까지 갖추게 되었는지 알아보자.

### Lambda

    메소드를 하나의 식으로 표현하는 것을 말한다. 

    람다식을 사용하지 않는, 일반식

    public class MethodClass {

        int method() {
            return (int) (Math.random() * 5);
        }
    }   

    public class ArrayFileExample {

        public static void main(String[] args) {

            MethodClass methodClass = new MethodClass();
            int arr[] = new int[5];

            for (int i : arr) {
                arr[i] = methodClass.method();
                System.out.println("arr[i] = " + arr[i]);
            }
        }
    }

    위 예시를 보면 Method를 사용하기 위해 객체를 생성하고 메인 함수에서 객체의 메소드를 불러와 사용했따.

    하지만 람다식은 위 과정에서 객체를 생성하는 과정을 생략하고, 람다식 자체만으로 이 메소드의 역할을 대신할 수 있다.

    즉, 람다식은 객체의 선언, 생성을 생략해주는 "익명 객체 역할"을 한다.

    Q : 메소드와 함수의 차이가 무엇일까?
    A : 기능적으로 동일한 의미를 가진다.
        차이가 있다면, 함수는 클래스에 독립, 메소드는 클래스에 종속.
        객체지향 언어에서는 객체의 행위, 동작을 의미하는 의미로 '메소드'라는 용어로 사용한다.

    람다식은 함수형 언어를 접목한 기능이라고 한다. 
    이제부터 람다식에 관하여 설명할 때는 '메소드' 대신 '함수'라는 용어를 사용하겠다.

### 작성법

    람다식은 기존 메소드에서 메소드 이름과 반환타입을 제거하고, 매개변수 선언부와 중괄호 {} 사이에 '->' 를 추가해주면 된다.

    // 기존 메소드 사용 방식
    반환타입 메소드이름(매개변수) {
        문장
    }

    // 람다식 사용 방식
    (매개변수) -> { 문장 }

    ex)
        int max(int a, int b) {
            return a > b ? a : b;
        }

        (int a, int b) -> { return a > b ? a : b; }

    ex)
        int max(int a, int b) {
            return a > b ? a : b;
        }       

        (a,b) -> { return a > b ? a : b }

    ex)
        int printVar(String name, int i) {
            System.out.println(name + "=" + i);
        }    

        (String name, int i) -> { System.out.println(name + "=" + i); }

    ex)
        int square(int x) {
            return x * x;
        }

        (int x) -> { return x * x; }
        (x) -> { x * x}
        x -> x * x

    ex)
        int roll(){
            return (int) (Math.random()*6);
        }

        () -> { return (int)(Math.random() * 6);}
        () -> (int)(Math.random() * 6)    

    (인자 리스트) -> (바디)

    - 인자 리스트
        - 인자가 없을 때 : ()
        - 인자가 한개일 때 : (one) or one 으로 괄호 생략 가능.
        - 인자가 여러개일 때 : (one, two) 괄호 생략 불가능.
        - 인자의 타입은 생략 가능, 컴파일러가 추론 하지만 명시할수도 있다.

    - 바디
        - 화살표 오른쪽에 함수 본문을 정의.
        - 여러 줄인 경우 {} 를 사용하여 묶는다.
        - 한 줄인 경우 생략 가능하며, return 도 생략 가능하다.
    
### 람다로 복잡한 Enum 리펙토링

    public enum Operation {
        PLUS("+") {
            public double apply(double x, double y) {
                return x + y;
            }
        },

        MINUS("-") {
            public double apply(double x, double y) {
                return x - y;
            }
        }
    }

    public enum Operation {
        PLUS("+", (x, y) -> x + y),
        MINUS("-", (x, y) -> x - y);
    }

### 템플릿 콜백 패턴과 같은 람다

    public class 군인 {

        public void 공격(무기 weapon) {
            weapon.공격();
        }
    }    

    "무기" 인터페이스의 구현체의 인스턴스를 인수로 받아왔으니, 
    군인이 공격할 때마다 무기 인터페이스 구현체가 구현한 "공격"
    메서드를 실행 가능하다.

    public class Main {

        public static void main(String[] args) {
            군인 soldier = new 군인();
            soldier.공격(new 총());
            soldier.공격(new 돈까스_망치());
        }
    }

    -> 총() : 빵야!!!, 돈까스_망치() : 와장창

        이러한 패턴을 활용하면 우리는 "총을 든 군인", "망치를 든 군인" 클래스를 따로 구현할 필요가 없다.

        이번에는 람다를 활용해 똑같이 구현해보자.

    public class Main {

        public static void main(String[] args) {
            soldier.공격(() -> System.out.println("빵야"));
            soldier.공격(() -> System.out.println("와장창"));
        }
    }

        이건 템플릿 콜백 패턴이다. 이런 식으로 메서드 자체를 전달할 수가 있게 된 것이다.. 람다를 활용했더니, 이제 "무기" 인터페이스를 상속한 "총", "망치" 클래스를 만들 필요가 없어진 것이다.


### 람다식 = 익명 클래스 객체

    기존의 객체지향적 패러다임을 가진 자바의 세계에선 아예 불가능했던 메서드의 전달을 가능하게 해준 것이 람다이다.

    (int a, int b) -> a > b ? a : b;

    // 위와 같은 람다식은 실제로는 아래와 같이 생겼다고 이해하면 된다.

    new Objcet() {
         
        int max(int a, int b) {
            return a > b ? a : b;
        }
    }

    - 익명 클래스의 인스턴스를 만든다.
    - 내부적으론 메서드를 가지게 한다.
    - 람다식은 위와 같은 익명 클래스의 인스턴스이다.
  
### stream()를 적용한 람다

### 1. List -> set으로 변환
    public class Main {

        public static void main(String[] args) {
            // List 컬렉션 및 제너릭스를 활용하여 배열 생성
            List<String> list = new ArrayList<>();

            // 배열에 데이터 추가
            list.add("서울");
            list.add("부산");
            list.add("속초");
            list.add("서울");

            System.out.println(list);

            // 스트림 활용 과정 (스트림을 활용해 list에서 set으로 데이터를 반환)
            List<String> result = list.stream() // 1. 스트림 생성
                    .limit(2)   // 2. 중간 연산 (0,1,2 제외 나머지 데이터 삭제)
                    .collect(Collectors.toList()); // 3. 최종 연산 (결과값을 리스트화)
            
            System.out.println(result);
            System.out.println("list -> transformation -> set");

            Set<String> set = list.stream() // 1. Set 스트림 생성 (새로운 List 스트림 생성)
                    .filter("서울"::equals) // 2. 중간 연산 ("서올" 데이터만 필터림)
                    .collect(Collectors.toSet()); // 3.최종 연산 (필터링한 데이터를 set으로 변환)
                
            set.forEach(System.out::println); // 람다식을 활용하여 결과 출력 (set의 데이터를 출력);
        }
    }

    list = [서울, 부산, 속초, 서울] // 리스트 데이터 초기화
    result = [서울, 부산]  // List 스트림을 활용한 연산 결과 반환
    list -> transformation -> set  
    set.forEach = 서울  // Set 스트림을 활용한 연산 결과 반환

### 2. 배열 -> 스트림 변환
    public class Main {

        public static void main(String[] args) {

            // 배열 데이터 생성
            String[] arr = {"자세히 보아야 이쁘다", "오래 보아야 사랑스럽다.", "너도 그렇다.", "<풀꽃> - 나태주"};

            // 배열 스트림 생성하여 스트림으로 변환
            Stream<String> stringStream = Arrays.stream(arr);

            // 람다식을 활용하여 스트림 데이터 출력
            stringStream.forEach(poem -> System.out.println(poem));
            System.out.println();
        }
    }

    자세히 보아야 예쁘다
    오래 보아야 사랑스럽다
    너도 그렇다
    <풀꽃> - 나태주

### 3. Map 활용

    // Sale class (객체의 기본 속성을 정의)
    class Sale {
        String fruitName;
        int price;
        float discount;
        public Sale(String fruitName, int price, float discount) {
            this.fruitName = fruitName;
            this.price = price;
            this.discount = discount;
        }
    }

    // 메인 클래스에서의 활용
    public class Main {

        public static void main(String[] args) {

            // List 컬렉션 클래스 및 제네릭스를 활용한 배열 선언
            List<Sale> saleList = Array.asList(
                new Sale("Apple", 5000, 0.05f),
                new Sale("Grape", 3000, 0.1f),
                new Sale("Orange", 4000, 0.2f),
                new Sale("Tangerine", 2000, 0)
            );

            // 스트림 생성
            Stream<Sale> saleStream = saleList.stream();

            // 스트림 매핑작업을 통해 각 과일의 실 구매가를 계산 및 출력
            saleStream.map(sale -> Pair.of(sale.fruitName, 
                                   sale.price * (1 - sale.discount)))
                  .forEach(pair -> System.out.println(pair.getLeft() + " 실 구매가: " + pair.getRight() + "원 "));
        }
    }

    Apple 실 구매가: 4750.0원 
    Grape 실 구매가: 2700.0원 
    Orange 실 구매가: 3200.0원 
    Tangerine 실 구매가: 2000.0원 