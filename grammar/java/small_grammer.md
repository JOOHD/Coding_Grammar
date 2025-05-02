## 자투리 문법 모음

### this vs this()

    public class Book {
        private String title;
        private int price;
    
        1)	
        public Book(String title, int price) {
            this.title = title;
            this.price = price;
        }
    
        2)
        public Book(String title) {
            this(title, 0);
        }
    }
    1번과 2번은 같은 코드이고, this() 메소드로 this.title = ~, this.price = ~ 와 같은 코드의 중복을 줄이고 좀 더 간결하게 구성할 수 있다.

### Arrays.asList() vs ArrayList

	● Arrays.asList()  
	- 배열을 List로 변환하는 정적 메서드이다. (크기 : 배열은 고정/ 리스트는 동적)
	- 반환되는 List는 고정 크기이며, 원소의 추가나 제거가 불가. 즉 읽기 전용 리스트(수정은 가능)
		ex)
			List<String> list = Arrays.asList("apple", "banana", "cherry");
			list.set(1, "blueberry"); // 가능
			list.add("orange");      // UnsupportedOperationException 발생

	- 내부적으로 배열을 기반으로 하고 있어서 원소 추가/삭제가 불가, 크기가 변경되지 않는다.
	
    ※ 추가 (Array.asList() & toFile = 객체 변환)

    ●Path -> File
    toFile()   =  toFile() 메서드는 Path 객체를 File 객체로 변환하는 데 사용됩니다.
        ex) Fiile targetDir = Paths.get("C:\\upload:, getFolderYesterDay()).toFile();

    listFiles() =  해당 디렉터리에 있는 파일 및 디렉터리의 리스트를 가져올 때 사용됩니다.

    File -> Path
    file.toPath() = File 객체를 Path 객체로 변환 
        ㄴ<Path>file equals <Path>checkFile
        ex) for(<File> file : <File> targetFile) {
            
            List<Path>checkFilePath . forEach(<Path>checkFile -> {
                if(file.toPath().equals(checkFile))
                    removeFileList.remove(file);
            });

	● ArrayList
	- 동적인 크기를 가진 리스트이다. 배열과 달리 원소를 추가하거나 삭제할 수 있고, 크기도 자동으로 조정.
	- Java의 표준 라이브러리에서 제공하는 ArrayList 클래스는 배열 기반이지만 동적 크기 조절이 가능, 일반적으로 리스트  자료구조로 사용된다.

### ArrayList() vs ArrayList<>() 

	● ArrayList() 
	- Java7 이전에는 제네릭 타입을 명시적으로 적어줘야 했다.
		그래서  ArrayList<String>() 이런 식으로 각 타입을 명시
	
	● ArrayList<>()
	- Java7 이후, 타입 추로 기능이 도입되면서 우변의 타입 매개변수는 생략 가능

### @ResponseBody vs @RequestBody

	@RequestBody 
    클라이언트가 서버로 JSON 등의 데이터를 전송할 때, 이 데이터를 자바 객체로 변환(역직렬화)하여 받습니다. 즉, 클라이언트에서 서버로 데이터를 전송할 때 사용됩니다.

	@ResponseBody
    서버가 클라이언트에게 JSON 등의 데이터를 응답할 때, 자바 객체를 JSON 형식으로 변환(직렬화)하여 보냅니다. 즉, 서버에서 클라이언트로 데이터를 응답할 때 사용됩니다.

	정리
	@RequestBody: 클라이언트 → 서버 (JSON → 자바 객체)
	@ResponseBody: 서버 → 클라이언트 (자바 객체 → JSON)

	HTML Form vs REST API
	둘의 차이점은 Form 전송은 (@ModelAttribute User user) 어노테이션을 사용하고,
	REST API는 (@RequestBody User user) 어노테이션을 사용하고 REST API 는 JSON/XML 형태이기 때문에,
 	서버에서 데이터를 받을 때, Java 객체로 변환 시켜주어야 한다. 그래서 @RequestBody 를 사용하는 것이고,
	다시 Java 객체를 클라이언트에 JSON/XML 형태로 보내야 하기 때문에, @ResponseBody 어노테이션을 사용하	는 것이다

### 기본 생성자 vs 파라미터가 있는 생성자

    기본 생성자 : 객체를 생성하는 역할을 한다.
    파라미터가 있는 생성자 : 객체 생성과 동시에 값들을 설정한다. 즉, 한 번에 모든 필드를 초기화.
	(이렇게 하면 객체 생성 후 따로 setter를 호출하는 과정을 생략가능.)  

### Enum

    특정한 값 중에서 선택하는 상수의 집합을 나타내는 자료형입니다. 
    ex) 
        RUNTIME_EXCEPTION(HttpStatus.BAD_REQUEST, "E0001"),
        ACCESS_DENIED_EXCEPTION(HttpStatus.UNAUTHORIZED, "E0002"),
        INTERNAL_SERVER_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "E0003"),

        SECURITY_01(HttpStatus.UNAUTHORIZED, "S0001", "권한이 없습니다.");      

### Setter method

    set.method() [변경자 메서드]
	-객체 변수의 값을 바꾸는
	-객체 변수의 값을 매개변수로 넘어오는 값으로 변경함
	-객체 변수의 값을 새로운 값으로 바꾸게 하므로 set 메서드라고 부름
	ㄴVO/DTO Class 의 객체 변수들이 private 로 설정되어 있어서, 직접 접근 가능하게 set 메소드 사용.

	요약하자면, 외부에서 전달된 값을 객체의 특정 필드에 저장할 수 있으며, 
	이는 객체의 상태를 초기화하거나 변경할 때 사용됨.
		ㄴ 이와 같은 방식으로 객체의 내부 데이터를 설정함으로써, 
		    데이터베이스로부터 가져온 값을 객체에 저장하고, 이후에 필요한 작업을 수행할 수 있다.

### Slf4j.xml 설정
ㄴ
    이슈 : controller 외 serviceImpl 작업 중 확인하고자 log를 사용하려고 했는데, log가 찍히지 않음
    원인 : Slf4j.xml 에 service class 를 추가해주지 않았다.
    해결
    <!-- Application Loggers -->
        <logger name="com.joo.controller">
            <level value="info" />
        </logger>
        
        <logger name="com.joo.service">
            <level value="info" />
        </logger>

### transcation 설정

    1.root-context.xml 에 DataSourceTransactionManager 클래스 bean 등록,
    2.namespace 탭에서 tx 항목을 체크.
    3.<tx:annotation-driven /> 추가.            