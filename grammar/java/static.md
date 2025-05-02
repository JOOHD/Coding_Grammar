## 본문

### Static

    "정적", "고정된"이란 의미를 가지고 있다.
    자바에서는 'satic' 키워드를 사용하여 static 변수와 static 메소드, static 클래스를 만든다.

    Java에서 static 키워드를 사용한다는 것은 메모리에 한번에 할당되어 프로그램이 종료될 때 해케하는 것을 의미한다.
    
    위에 글의 의미하는 것은 JVM의 GC(garbage collector = 동적인 데이터 중에 더 이상 쓰이지 않는 영역을 찾아내어 해제하는 기능)의 관리 영역 밖에 존재하여, "static 영역에 할당된 메모리는 모든 객체가 공유할 수 있다."

    static 키워드는 변수나 필드 앞에 붙여 사용 가능하다.
    즉, static이 붙은 변수는 어디에서나 공유하여 사용하는 변수이고, 

    "static이 붙은 메소드는 어디에서나 공유하여 사용하는 메소드이다."

    ※ 일반적으로 정적 변수는 상수의 값을 갖는 경우가 많아,
    1. public 으로 선언하고,
    2. final로 변경을 불가하게 만든다.

### Static 변수 (쩡적 변수)    

    예를들어 아래와 같은 클래스가 있을 때, 클래스로부터 생성된 각 인스턴스는 name이란 propertie에 "human_being" 이라는 값이 할당되어 각 인스턴스마다 메모리를 할당하여 값을 갖게 된다.

    public class Person {
        private String name = "human_being";
        public void printName() {
            System.out.println(this.name);
        }
    }

    public class PersonTest {
        public static void main(String[] args) {
            Person p = new Person();
            p.printName();
    
            Person p2 = new Person();
            p2.printName();
    
        }
    }

    이렇게 공통적으로 값은 하나를 갖지만 100개의 메모리를 사용하게 되면 메모리 효율이 나빠진다. 

    이럴때 static 키워드를 사용해 공통적으로 같은 메모리에 하나의 값만 할당하여 공유하면 쓸데없는 메모리 낭비를 막을 수 잇다.

    public class Person {
        public static final String name = "human_being";
        public void printName() {
            System.out.println(this.name);
        }
    }

    public class PersonTest {
        public static void main(String[] args) {
            System.out.println("Person.name = " + Person.name);
        }
    }

    ● 일반적으로 정적 변수는 상수의 값을 갖는 경우가 많다.
    1. public으로 선언하고,
    2. 변경이 불가능하도록 final로 막아
    
    - public static final 타입 변수명 = 리터럴; 로 사용하는게 일반적이다.
  
### Static Method

    ● 유틸리티 클래스

    public class MathUtils {

        // 인스턴스를 생성하지 못하도록 private 생성자를 명시 (필요 없음)
        private MathUtils() {
            throw new UnsupportedOperationException("유틸리티 클래스는 인스턴스화할 수 없습니다.");
        }

        // 두 숫자의 합을 구하는 static 메서드
        public static int add(int a, int b) {
            return a + b;
        }

        // 숫자의 절대값을 반환하는 static 메서드
        public static int abs(int number) {
            return (number < 0) ? -number : number;
        }

        // 숫자의 제곱을 계산하는 static 메서드
        public static int square(int number) {
            return number * number;
        }
    }

    ● 사용 예시

    public class Main {
        public static void main(String[] args) {
            // MathUtils 클래스를 인스턴스화 하지 않고, static 메서드를 바로 사용.
            int sum = MathUtils.add(5, 3);       // 8
            int absolute = MathUtils.abs(-10);   // 10
            int square = MathUtils.square(4);    // 16 
        }
    }

    ● 설명

    1. 유틸리티 클래스
        - add(), abs(), square() 메서드는 모두 static으로 선언.
        - 이 클래스는 공통 기능을 제공하지만, 인스턴스를 만들 필요가 없기 때문에 인스턴스 생성 자체를 막기 위해 생성자를 private으로 선언.

    2. 메인 클래스
        - MathUtils 클래스를 인스턴스화 하지 않고, MathUtils.add(), MathUtils.abs(), **MathUtils.square()**처럼 바로 클래스 이름을 통해 메서드를 호출할 수 있습니다

    ● 정리
    - 유틸리티 클래스의 메서드들은 static으로 선언되어, 인스턴스 생성 없이 클래스 레벨에서 호출 가능.

    - 유틸리티 클래스 자체는 필드를 가지지 않으며, 특정 동작을 제공하기 위해 설계된다.  
    
### static method를 정하는 기준

    1. 메소드가 인스턴스 변수를 사용하지 않을 때,
    2. 인스턴스 생성에 의존하지 않을 때,
    3. 어떤 메소드가 여러 클래스에서 공유되고 있으며, 이 메소드를 static 메소드로 추출해낼 수 있을 때.

    즉, 인스턴스 변수와 밀접한 작업을 하는 메소드는 (static이 붙지 않은) 메소드로 사용하고, 인스턴스와 관계없는 (인스턴스 변수나 인스턴스 메소드를 사용하지 않는) 경우 static 메소드를 사용한다.

### 요약

    1. static 변수, static 메소드는 클래스 레벨에서 사용 가능하기 때문에 굳이 인스턴스를 만들 필요가 없습니다.
        - 반면 인스턴스 변수, 인스턴스 메소드는 인스턴스를 생성해야 사용할 수 있습니다.    

        - static 변수/메서드: 클래스 이름으로 직접 호출 가능.
        - 인스턴스 변수/메서드: 객체를 생성한 후에만 호출 가능.

    2. 인스턴스 메소드는 인스턴스 변수와 밀접한 작업을 하는 메소드입니다. 

    3. static 메소드는 static 변수와 밀접한 작업을 하는 메소드로, 인스턴스 변수에 접근할 수 없다.

    4. 물론 인스턴스 변수를 사용하지 않는다고 해서 인스턴스 메소드가 아닌, static 메소드로 정의하는 것이 필수적인 것은 아니나, 일반적으로 변수를 사용하지 않으면 static 메소드로 정의한다. 
    
    - static method -> 인스턴스 변수, 인스턴스 메소드 사용 불가능
    오로지 static 메소드, static 변수에만 접근 가능

    - instance Method -> 인스턴스 변수, 인스턴스 메소드 사용 가능
    다만, 인스턴스 메소드는 static 메소드, static 변수에도 접근할 수 있다.