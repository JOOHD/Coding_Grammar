## 본문

### Stream() 

    한 번에 한 개씩 만들어지는 연속적인 데이터 항목의 모임이다.
    이론적으로 프로그램은 입력 스트림에서 하나씩 읽어들이고, 출력 스트림에 하나씩 기록한다.
    한 프로그램의 출력 스트림이 다른 프로그램의 입력 스트림이 될 수도 있다.

### 람다 표현식

    자바에서는 인터페이스를 선언하거나 매개변수로 주어야 할 때 1회용 구현체인 익명 클래스를 사용할 수 있다.

    ex)

    public class Test {

        public static void main(String[] args) {

            List<Integer> list = Arrays.asList(2, 1, 3, 5, 4);
            printList(list);

            // 오름차순 정렬
            List<Integer> list2 = list.stream()
                    .sorted(Comparator.naturalOrder())
                    .collect(Collectors.toList());
            printList(list2);

            // 내림차순 정렬
            List<Integer> list3 = list.stream()
                .sorted(Comparator.reverseOrder())
                .collect(Collectors.toList());
            printList(list3);
        }

        private static void printList(List<Integer> list) {
            // 1회용 구현체 new Consumer를 매개변수로 활용
            list.forEach(new Consumer<Integer>() {

                @Override
                public void accept(Integer num) {
                    System.out.println(num + " ");
                }
            });
            System.out.println();
        }
    }

    // 람다식 이용
    private static void printList(List<Integer> list) {
        list.forEach(num -> System.out.println(num + " "));
        System.out.println();
    }

    람다식을 활용하여 new Consumer<Integer>() {...}가 단 한 줄로 줄어들었으며, 개발자의 의도가 드러나는 문장을 사용했다.

    forEach의 매개변수에 사용되는 Consumer 타입과 Consumer::accept가 겉으로 드러나지 않는다.

    람다식은 과도한 정보를 제하고 num을 출력하는 로직에 더 집중할 수 있는 구조이다.

### 람다 표현식?

    람다식은 익명 클래스를 간략하게 표현하는 방법이다.
    식 전체가 인스턴스로 취급된다.

    ● 람다 표현식 구조

    (paramters) -> expression
    (paramters) -> { statements };

    람다 표현식엔 interface/override 할 메소드의 이름이 표기되지 않는다.
    1. 매개변수 목록을 왼쪽에 표기한다.
    2. 매개변수는 타입을 생략할 수 있다.
    3. 매개변수가 1개이면 괄호를 생략할 수 있다.
    4. 매개변수가 0개이면 () ->으로 매개변수가 없음을 나타내어야 한다.
    5. 매개변수 목록 작성이 끝나면 화살표 (->)를 작성한다.
    6. 화살표 우측에는 람다 바디를 작성한다.
    7. 한 줄(세미콜론 한 번)으로 구현할 수 있는 경우 expression이다.
    8. expression은 중괄호, return, 세미콜론을 생략하여야 한다.
 
    ● 연산
    
        Function<T, R>
        - T 타입을 매개변수로 받고 R 타입을 반환한다.

        BigFunction<T, U, R>
        - T, U를 매개변수로 받고 R을 반환한다.
        - T, U, R 모두 다른 타입을 유연하게 사용할 수 있다.
    
    ex)
        1. 매개변수 1개, 타입 생략, 표현식 사용(규칙 2, 3, 7, 8)
        // 숫자를 받아서 제곱값을 반환하는 람다식
        Function<Intger, Integer> square = x -> x * x;
        System.out.println(square.apply(5)); // 출력 : 25

        2. 매개변수 0개, 화살표 -> 사용, 표현식 사용 (규칙 4, 7, 8)
        // 현재 시간을 반환하는 람다식
        Supplier<Long> currentTime = () -> System.currentTimeMillis();
        System.out.println(currentTime.get()); // 현재 시간 밀리초 값 출력

        3. 매개변수 2개, 타입 생략, 여러 줄의 람다 바디 (규칙 1, 2, 6)
        // 두 숫자를 받아서 더한 후 출력하는 람다식
        BiConsumer<Integer, Integer> addAndPrint = (a, b) -> {
            int sum = a + b;
            System.out.println("합계 : " + sum);
        }
        addAndPrint.accept(3, 7); // 출력 : 합계 : 10

### 스트림 개념

    컬렉션과 스트림의 차이를 예시로 나타내기 위해 사용할 데이터를 만들었다.
    
    ex)
        - City
        @Getter
        @AllArgsConstructor 
        public class City {
            private String name;
            private double area;     // 면적
            private int population;  // 인구
            private String areaCode; // 지역 번호
        }

        - cities
        List<City> cities = Array.asList(
            new City("Seoul", 605.2, 9720846, "02"),
            new City("Incheon", 1063.3, 2947217, "032"),
            new City("Busan", 770.1, 3404423, "051"),
            new City("Gwangju", 501.1, 1455048, "062"),
            new City("Daegu", 883.5, 2427954, "053"),
            new City("Ulsan", 1062, 1142190, "052")=
        )

        cities 데이터를 가공해달라는 요구사항이 들어왔다.
        - 면적(area)이 800km^2 이상인 광역시의 목록을 추출한다.
        - 면적을 기준으로 오름차순으로 정렬
        - 위의 기준을 충족한 광역시의 이름 목록만 반환받고자 한다.

        ● collection() 처리

            List<City> largeCiteis = new ArrayList<>();

            for (City city : cities) {
                if (city.getArea() > 800) {
                    largeCities.add(City);
                }
            }

            Collections.sort(largeCities, new Comparator<>(){
                @Override
                public int compare(City o1, City o2) {
                    return (int)(o1.getArea() - o2.getArea());
                }
            });

            List<String> largeCityNames = new ArrayList<>();

            for (CIty city : largeCiteis) {
                largeCityNames.add(city.getName());
            }

        ● stream() 처리

            List<String> streamNameList = cities.stream()
                .filter(city -> city.getArea() > 800) 
                .sorted(Comparator.comparing(City::getArea ))
                .map(City::getName)
                .collect(Collectors.toList());

### stream() 처리 장점

    - 선언형 처리                 

    데이터를 처리하는 로직을 직접 작성하지 않고, filter()와 같이 연산을 정의한 메소드를 활용한다.
    처리 방법 보다는 각 단계에서 수행할 로직의 목료(필터리 > 정렬 > 매핑 > 취합)를 명시한다.

    1. 각 연산을 체이닝으로 잇고 결과를 다음 로직으로 전달
    2. 모든 과정이 내부에서 반복
    3. 컬렉션은 반복자가 겉으로 들어나지만, 스트림은 드러나지 않는다.
    
    컬렉션은 데이터를 저장하는 자료구조 구현에 초점이 맞추어져 있고,
    스트림은 데이터를 연산하는 데에 집중한 API이기 때문이다.

### stream() 연산

    스트림은 연산이라고 불리우는 부품을 모아 하나의 흐름으로 완성시켜 사용한다.
    이를 스트림 파이프라인이라고 하며 그 흐름에는 3아래 가지가 반드시 포함되어야 한다.

    - 스트림 생성
      - 생성된 스트림은 한 번 사용하면 사라지기 때문에 소비한다는 개념으로 사용.

    - 중간 연산
      - Stream을 반환한다.
      - 그 덕분에 중간 연산 메소드는 체이닝으로 다음 중간 연산 메소드를 이어줄 수 있다.
      - 체이닝하여 연결하는 각 연산은 스트림 파이프라인에 저장된다.
      - 가장 중요한 특징은 중간 연산을 아무리 많이 연결한들 파이프라인이 실질적으로 수행되기 전까진 아무 연산도 하지 않는다. (= Lazy Evalution : 스트림 연산은 최종 연산에 호출되기 전까지 미뤄진다.)
       
    - 최종 연산
      - 중간 연산을 이어가며 만든 스트림 파이프라인에서 최종 결과를 도출하는 과정이다. 
      - 최종 연산이 끝나면 컬렉션 (List, Map)이나 자료형 (Integer), void를 반환.

### stream() 주요 메서드

    ex)
        @Getter
        @AllArgsConstructor
        @EqualsAndHashCode    // equals, hashcode 자동 생성
        public class City {
            private String name;
            private double area;
            private int population;
            private String areaCode;
        }

        List<City> cities = Arrays.asList(
                new City("Seoul", 605.2, 9720846, "02"),
                new City("Incheon", 1063.3, 2947217, "032"),
                new City("Ulsan", 1062, 1142190, "052"),
                new City("Daegu", 883.5, 2427954, "053"),
                new City("Gwangju", 501.1, 1455048, "062"),
                new City("Busan", 770.1, 3404423, "051")
        );

        - 데이터 선별(조건에 따라 데이터를 선별하는 중간 연산이다.)

        cities.add(new City("Seoul", 0, 0, "02"));

        List<City> streamNameList = cities.stream()
                .filter(city -> city.getArea() > 800) // 데이터 필터링
                .distinct()                           // 중복 제거  
                .collect(Collectors.toList());

    ● 전체 조건 (Java9)
    데이터는 반드시 사용할 기준을 조건으로 정렬되어 있는 상태여야 한다.

    - sorted(Comparator.comparing()) : 스트림 요소들을 특정 필드를 기준으로 정렬하는 기능.    

    ex)
    1.
        // 예시 데이터를 area 오름차순(default)으로 재가공
        List<City> orderByAreaList = cities.stream()
                .sorted(Comparator.comparing(City::getArea))
                .collect(Collectors.toList());

    2.
        takeWhile : 스트림을 순회하다 조건이 false가 되는 순간 남은 데이터를 버린다.
        orderByAreaList.stream()
                .takeWhile(city -> city.getArea() < 700)
                .map(City::getName)
                .forEach(Sytem.out::println);

                // Gwangju, Seoul

    3.
        dropWhile : 스트림을 순회하다 조건이 true가 되는 순간 순회한 데이터를 모두 버린다.
        orderByAreaList.stream()
                .dropWhile(city -> city.getArea() > 700)
                .map(City::getName)
                .forEach(System.out::println);

                // Busan, Incheon, Daegu, Ulsan

    4.
        map : 스트림 원소를 반환한다.
        List<String> filteringList = cities.steam()
                .filter(city -> city.getArea() < 800)
                .sorted(Comparator.comparing(City::getArea))
                .map(city -> city.getName()) // City 클래스에서 도시명만 사용하도록 한다.
                .collect(Collectors.toList());

                // Ulsan, Incheon

    5.
        cities에서 지역 번호에 사용된 숫자에서 중복을 제거한 목록을 스트림으로 얻어내고 싶다면 어떻게 해야 할까?

        List<String>  areaCodes = cities.stream()
                .map(City::getAreaCode)           
                .collect(Collectors.toList());

                // ["02", "032", "052", "053", "062", "051"]

                1. areaCodes 문자 단위로 분해
                    0, 2
                    0, 3, 2
                    0, 5, 2
                    0, 5, 3
                    0, 6, 2
                    0, 5, 1

                2. areaCodes 단일 스트림으로 변환            
                    0, 2, 0, 3, 2, 0, 5, 2, 0, 5, 3, 0, 6, 2, 0, 5, 1

        areaCodes를 스트림으로 추출하면 위의 1번처럼 문자열 하나가 데이터 하나로 변환된다.
        문자 단위로 분해하여 2와 같은 단일 문자열 스트림을 반환해야 한다.
        이럴 때 사용하는 것이 Stream::flatMap 이다.
        

    6. 
        areaCodes.stream()
                .map(areaCode -> areaCode.split("")) // areaCode를 문자 단위로 분해한 배열로 변환
                .flatMap(Arrays::steam) // 변환된 스트림을 단일 스트림으로 변환
                .distinct() // 중복 제거
                .forEach(System.out::println);

                // 0, 2, 3, 5, 6, 1

### 최종 연산

    최종 연산은 자료형, 컬렉션, void를 반환한다.

    Collection 반환
    1. List
    
        List<String> streamNameList = cities.stream()
            .filter(city -> city.getArea() > 800)
            .sorted(comparator.comparing(City::getArea))
            .map(City::getName)
            .collect(Collectors.toList());

    2. map

        Map<Stirng, List<City>> cityMap = cities.stream()
            .filter(city -> city.getArea() > 800)
            .collect(Collectors.groupingBy(CIty::getName));

    3. set

        Set<City> citySet = cities.stream()
            .filter(city -> city.getArea() > 800)            
            .collect(Collectors.toSet());

    4. void

        cities.stream().map(City::getName).forEach(Sytem.out::println);
        // Seoul, Incheon, Ulsan, Daegu, Gwangju, Busan
