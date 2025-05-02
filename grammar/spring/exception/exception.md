## 본문

### 프로그램 오류

    프로그램이 실행중 어떤 원인에 의해서 오작동을 하거나 비정상적으로 종료되는 경우가 있다.
    이러한 결과를 초래하는 원인을 프로그램 오류라고 한다.

    오류는 발생시점을 기준으로 나누면 "컴파일 에러", "런타임 에러"로 나뉜다.

    ● 컴파일 에러 : 컴파일 시점에 발생하는 에러, Checked 예외
    ● 런타임 에러 : 실행 도중에 발생하는 에러, Unchecked 예외

    이 외에도 논리적 에러가 있는데, 논리적 에러는 컨파일도 잘되고 실행도 잘 되지만 개발자가 의도한 것과 다르게 동작하는 에러를 말한다.
    ex) 창고의 재고가 음수가 되는 등의 에러.

    이렇게 자바에서는 실행 시 발생할 수 있는 프로그램 오류를 "에러" & "예외" 두 가지로 나뉜다. 

    에러는 프로그램 코드로 수습할 수 없는 심각한 오류로, 예를 들면 메모리 부족, 스택 오버 플로우가 있다. 에러는 일단 발생하면 프로그램 실행중에 복구할 수 없기 때문에 비정상적인 종료를 막을 길이 없지만, 예외는 개발자가 이에 대해 적절한 코드를 미리 작성해 놓음으로써 프로그램의 비정상적인 종료를 막을 수 있다.

    자바는 실행 시, 발생할 수 있는 오류를 클래스로 정의하였다.

![exception_tree](/grammer/img/exception_tree.png)    

    예외 클래스의 계층구조를 살펴보면 Exception(예외)와 Error(에러)로 나누어져 있다.
    Exception 클래스는 모든 예외의 조상 클래스이며, Exception 또한 Checked 예외와 Unchecked 예외로 나누어져 있다.

![checked_unchecked](/grammer/img/checked_unchecked.png)    

    - Unchecked 예외는 실행 도중 발생하는 예외로 RuntimeException을 칭하며, 개발자의 실수에 의해서 발생될 수 있는 예외들로 자바의 프로그래밍 요소들과 관계가 깊은 예외이다.

    - 중요한 점은 Unchecked 예외(RuntimeException)는 개발자에 의해 실수로 발생히는 예외이기 때문에 예외에 대한 처리가 강제되지 않는다는 점이다.

    + Unchecked 예외는 throws를 작성하지 않아도 된다.

       ex)

        값의 null인 참조변수의 멤버를 호출하려 했다던지(NullPointException), 정수를 0으로 나누려고 했다던지 하는 경우에 발생하는 예외(ArtihmeticException)가 있다.

    - Checked 예외는 RuntimeException을 제외한 모든 예외로 프로그램의 사용자들의 동작에 의해서 발생하는 예외

       ex)

       입력한 데이터 형식이 잘못된 형식이거나, 실수로 클래스의 이름을 잘못 적었다던지 하는 경우에 발생하는 예외가 바로 Checked 예외이다.

       Checked 예외는 예외에 대한 처리가 강제되기 때문에 개발자는 Checked 예외에 대한 처리를 반드시 해주어야 한다.

    - Exception class : 사용자의 실수와 같이 외적 요인에 의해 발생하는 예외.
    - RuntimeException class : 개발자의 실수로 발생하는 예외.       

### 예외 처리 : try ~ catch(+finally)    

    프로그램 실행도중에 발생하는 예외는 개발자가 이에 대해 미리 처리를 해주어야 한다. 이를 예외 처리라고 한다.

    예외 처리의 목적 : 프로그램의 비정상적인 종료를 막고, 정상적인 실행상태를 유지할 수 있도록 하기 위함.

    예외를 처리하기 위해서는 try ~ catch문을 사용.

    eX)
        try{
            /* 예외가 발생할 수 있는 문장 */
        }
        catch(/* try 문에서 발생한 예외1 */){ /* 예외1에 대한 처리 문장 */ }
        catch(/* try 문에서 발생한 예외2 */){ /* 예외2에 대한 처리 문장 */}
        catch(/* try 문에서 발생한 예외3 */){ /* 예외3에 대한 처리 문장 */}

    ● try block 내에서 예외가 발생한 경우

      - 발생한 예외와 일치하는 catch 블러이 있는지 확인 -> 일치하는 catch 블럭 발생 시, 블럭 내의 문장 수행 -> try catch 문 빠져나옴.

      - 만약 일치하는 catch 블럭이 없으면 해당 예외는 처리되지 못하고 빠져나온다.
  
    ● try block 내에서 예외가 발생하지 않은 경우

      - try ~ catch문을 빠져나옴

    "모든 예외의 조상 예외인 Exception을 catch 블럭에 사용하게 되면 어떤 종류의 예외가 발생하더라도 처리하게 된다.(다형성)"

    ● try ~ catch ~ finally

      - try ~ catch문 뒤에는 finally를 추가하여 try ~ catch문에서 예외의 발생여부와 상관없이 실행되어야 할 코드를 작성할 수 있다.

    ex)
        try
        {
        ...
        }catch(Exception1 e1)
        {// Exception1에 대한 처리 문장}
        finally{
            // 예외의 발생여부와 관계없이 항상 수행되어야 하는 문장
        }

        - 이 경우 예외가 발생하면, try -> catch -> finally 순으로 실행.
        - 예외가 발생하지 않으면, try -> finally 순으로 실행.

### 예외의 printStackTrace(), getMessage() 메소드

    예외가 발생했을 때 생성되는 예외 클래스의 인스턴스에는 발생한 예외에 대한 정보가 담겨져 있다. 이 정보들은 getMessage() 와 printStackTrace() 를 통해 시스템 console에 출력.

    @Test
    void printEx() {
        Controller controller = new Controller();
        try {
            controller.request();
        } catch (Exception e) {
            // void
            e.printStackTrace();

            // String
            String message = e.getMessage();
            System.out.println("message = " + message);
        }
    }

    이처럼 try ~ catch문으로 예외가 발생해도 비정상적으로 종료되지 않도록 해주는 동시에 메소드를 통해 예외의 발생 원인을 알 수 있다.

### 사용자 정의 예외 만들기

    모든 예외의 조상클래스인 Exception을 상속받아 새로운 예외 클래스를 정의할 수 있다.

    class MyException extends Exception {
        MyException(String msg) // String을 매개변수로 받는 생성자 msg : 에러메시지
        super(msg); // 조상인 Exception 클래스의 생성자를 호출
    }

    기존의 예외 클래스는 주로 Exception을 상속받아 Checked 예외로 작성하는 경우가 많았지만, 요즘에는 뒤에서 예외를 처리할 방법중 하나인 예외전환 방법을 사용하기 위해 RuntimeException을 상속받아 작성하는 쪽을 선택하기도 한다.

### 예외 발생시키기 by throw

    키워드 throw를 사용해서 개발자가 고의로 예외를 발생시킬수 있다.

    1. 먼저 연산자 new를 이용해서 발생시키려는 예외 클래스의 객체를 만든 다음
        Exception e = new Exception("고의로 발생시킨 예외").

    2. throw 키워드를 이용해서 예외를 발생시킨다.
        throw e;

    ● 발생한 예외를 처리하는 방법에는 3가지 방법이 있다.

![exception_process](/grammer/img/exception_three.png)       

### 예외처리 방법 1. 예외 복구 try ~ catch문

    ● 예제 코드1

    public void callCatch()
    {
        try {
            repository.call();
        } catch (MyCheckedException e) {
            // 예외 처리 로직
            log.info("예외 처리, message = {}",e.getMessage(),e);
        }
    }
    
    /**
    * 내던져진 예외 던지기
    * 체크 예외는 예외를 잡지 않고 밖으로 던지려면 throws 예외를 메소드에 필수로 선언해야함
    * @throws MyCheckedException
    */
    public void callThrow() throws MyCheckedException {
        repository.call();
    }

    - callCatch() : 넘어온 예외를 처리

        - try-catch문을 이용해 try문의 repository.call()에 의해 발생한 예외를 catch 블럭에서 로그를 남기고 끝냄으로써 다른작업 흐름으로 유도했습니다.

        - 해당 예외를 잡아서 복구(처리)했기때문에 throws 키워드를 이용해 예외를 내던질 필요가 없습니다.

    ● 예제 코드2

    - 발생한 예외를 바로 처리하고, 시스템이 정상적으로 작동하도록 한다.
    - 사용자가 실수로 비밀번호를 입력하지 않거나, 체크박스를 체크하지 않는 상황에서 적합하다.

    public void registerUer(String username, String password, boolean agreeToTerms) {

        try 
        {
            if (password == null || password.isEmpty()) {
                throw new IllegalArgumentException("비밀번호를 입력해주세요.");
            }
            if (!agreeToTerms) {
                throw new IllegalStateException("약관에 동의해야 합니다.");
            }
            System.out.println("회원가입 성공: " + username);
        } catch (IllegalArgmentsException e) {
            System.out.println("입력 오류: " + e.getMessage());
        } catch (IllegalStateException e) {
            System.out.println("상태 오류: " + e.getMessage());
        }
    }

    - IllegalArgumentException은 비밀번호 미입력에 대한 예외를 처리한다.
    - IllegalStateException 은 체크박스 미체크에 대한 예외를 처리한다.
    - 장점 : 예외를 복구하여 프로그램이 종료되지 않고 정상적으로 작동.

### 예외처리 방법 2. 예외 회피 by throws

    ● 예제 코드1

    예외가 발생한 해당 메소드에서 예외를 처리하지 못하는 경우,
    해당 메소드를 호출한 메소드에게 예외를 던짐으로써 해당 메소드에서 예외르 ㄹ처리하지 못하였음을 알릴 수 있다.

    예외 중에서도 Checked 예외 일 떄는 코드를 통한 예외처리가 필수적이기 때문에 해당 메소드에서 처리하지 못하는 겨우 throws 키워드를 이용해 예외를 넘겨주어야 한다.

        "Checked 예외는 반드시 처리되거나 넘겨주거나 둘 중 하나 필수."

    호출한 메소드에게 예외를 넘기기 위해서는 호출된 해당 메소드 뒤에 throws 키워드와 함께 해당 메소드에서 발생한 예외 혹은 해당 메소드가 호출한 메소드를 처리하지 못헤서 넘어온 예외를 작성해 주면 된다.

    public void callCatch()
    {
        try {
            repository.call();
        } catch (MyCheckedException e) {
            // 예외 처리 로직
            log.info("예외 처리, message = {}",e.getMessage(),e);
        }
    }
    
    /**
    * 내던져진 예외 던지기
    * 체크 예외는 예외를 잡지 않고 밖으로 던지려면 throws 예외를 메소드에 필수로 선언해야함
    * @throws MyCheckedException
    */
    public void callThrow() throws MyCheckedException {
        repository.call();
    }

    - callThrow() : 넘어온 예외를 처리하지 못하고 던지기

      - repository.call()에서 발생한 예외를 메소드 뒤에 throws 키워드 = 넘겨받은 예외를 작성해 자신 또한 처리하지 못하였음을 알리고 callThorw()를 호추랗ㄴ 메소드에게 예외를 던진다.

      + try-catch문과 throws 키워드를 모두 이용하여 현재 메소드와 호출한 메소드에서 예외를 나누어서 처리하도록 할 수 있습니다.

        public void excute1() throws Exception1
        {
            try
            {
                repository.call(); // Exception1 , Exception2 발생 
            }
            catch(Exception2 e)
            {
                // repository.call()에서 넘겨받은 Exception2 처리
            }
        }

    ● 예제 코드2

    - 예외를 직접 처리하지 않고 호출한 메서드로 전달.
    - 이 방법은 더 높은 수준의 코드에서 예외를 처리하고자 할 떄, 유용하다.
    
    public void validateUserInput(String password, boolean agreeToTerms) throws IllegalArgumentException 
    {
        if (password == null || password.isEmpty()) {
            throw new IllegalArgumentException("비밀번호를 입력해주세요.");
        }
        if (!agreeToTerms) {
            throw new IllegalStateException("약관에 동의해야 합니다.");
        }
    }

    public void registerUser(String username, String password, boolean agreeToTerms) 
    {
        try {
            validateUserInput(password, agreeToTerms); // 예외 회피
            System.out.println("회원가입 성공: " + username);
        } catch (IllegalArgumentException | IllegalStateException e) {
            System.out.println("입력 오류: " + e.getMessage());
        }
    }

    - validateUserInput 메서드는 예외를 던지고, registerUser에서 이를 처리
    - 장점 : 예외 처리를 분리해 코드의 가독성을 높이고, 예외를 한 곳에서 관리.

### 예외처리 방법 3. 예외 전환하기 

    ● 예제 코드1

    Unchecked 예외 (RuntimeException)은 예외처리가 강제되지 않는다고 했었다.

    - Checked : 예외를 잡아서 처리하지 않으면 반드시 thrwos에 던지는 예외를 선엉해야 함.

    - UnChecked : 예외를 잡아서 처리하지 않아도 throws 생략 가능. (처리 강제성 x)

    Checked 예외의 경우

![checked_Exception_problem](/grammer/img/checked_Exception_problem.png) 

    각 메소드가 예외를 처리하지 못하고 넘겨줄 때 메소드 모두 throws 예외에 의존하고 있다.

    만약 JDBC 기술에서 JPA 기술로 바꾼다면 SQLException에 의존하던 모든 서비스, 컨트롤러를 JPAException에 의존하도록 고쳐야 한다.

        "logic() throws SQLException ▶︎ logic() throws JPAException"

    throw Exception으로 통일하게 되면 다른 체크예외를 체크할 수 있는 기능이 무효화되기 때문에 권장되지 않는다.

    이럴 때 사용하는 방법이 예외 전환이다. (Checked -> UnChecked 전환)        

    static class Service
    {
        NetworkClient networkClient = new ~
        Repository repository = new ~
        
        public vodi logic() {
            repository.call();
            networkClient.call();
        }
    }

    static class NetworkClient
    {
        public void call() {
            // Unchecked exception 내던지기
            throw new RuntimeException("연결 실패"); // extends RuntimeException
        }
    }

    ● 예제 코드2

    - 발생한 예외를 다른 예외로 변환하여 처리.
    - 구체적인 예외를 더 높은 수준의 코드에서 의미 있는 도메인 예외로 변환할 때 사용한다.
  
    class UserRegisterationException extends RuntimeException 
    {
        public UserRegistrationException(String message, Throwable cause) 
        {
            super(message, cause);
        }

        public void validateUserInput(String password, boolean agreeToTerms) throws IllegaArgumentException 
        {
            if (password == null || password.isEmpty()) {
                throw new IllegaArgumentException("비밀번호를 입력해주세요");
            }
            if (!agreeToTerms) {
                throw new IllegalStateException("약관에 동의해야 합니다.");
            }
        }

        public void registerUser(String username, String password, boolean agreeToTerms) 
        {
            try {
                validateUserInput(password, agreeToTerms);
                System.out.println("회원가입 성공: " + username);
            } catch (IllegalArgumentException | IllegalStateException e){
                throw new UserRegistrationException("회원가입 중 오류가 발생했습니다.", e); // 예외 전환
            }
        }
    }

    - IllegalArgumentException과 IllegalStateException을 도메인에 적합한 UserRegistrationException으로 전환합니다.

    - 예외 전환으로 인해 호출자는 더 도메인에 적합한 정보를 얻을 수 있습니다.

    - 장점: 예외의 추상화 수준을 높여 도메인 모델에서 의미를 갖도록 만듭니다.

### throws vs throw

    ● throws 

    역할 : 메서드가 호출될 때 예외를 던질 가능성이 있다고 선언.
    위치 : 메서드 선언부 옆에 붙음
    용도 : 예외를 호출한 메서드로 넘기기 위해 사용, 예외 처리를 현재 메서드에서 하지 않고, 호출한 쪽에서 처리하도록 위임.

    ex)
        public void readFile(String filePath) throws IOException {
            FileReader fileReader = new FileReader(filePath); // IOException 발생 가능
        }

        - throws IOException은 이 메서드가 IOException을 던질 가능성이 있음을 선언.

        - 호출한 쪽에서 try ~ catch 로 처리. 

    ● throw

    역할: 실제로 예외를 던짐.
    위치: 메서드의 본문이나 블록 내에서 사용.
    용도: 특정 조건이 발생하면 직접 예외를 던질 때 사용, 예외 객체를 생성하고 즉시 발생시킴.    

    ex)
        public void validateAge(int age) {
            if (age < 0) {
                throw new IllegalArgumentException("Age cannot be negative"); // 직접 예외 던짐
            }
        }

        - 위 코드는 유효하지 않은 입력에 대해 IllegalArgumentException을 즉시 발생.

    ● throws 와 throw 함께 사용.

    public void readAndValidateFile(String filePath) throws IOException 
    {
        if (filePath == null || filePath.isEmpty()) {
            throw new IllegalArgumentException("File path cannot be null or empty"); // throw로 직접 예외 발생
        }

        FileReader fileReader = new FileReader(filePath); // IOException 발생 가능
    }         

### 정리

    예전에는 Checked 예외를 자주 사용했지만, throws 키워드를 붙여주어야 하는 번거로움 때문에 최근 라이브러리들은 대부분 Unchecked (런타임예외)를 기본으로 제공한다.

    + 하지만 Rutime error는 명시를 해놓지 않으면, 어떤 예외가 발생할 지 코드를 읽는 개발자 입장에서 어려움을 느낄 수 있따.

    그럴 때에 메소드에서 발생할 수 있는 예외를 문서로 명시해 주거나 + 코드에도 throws로 명시해주는 것이 중요하다.

    static class Repository {
        public void call() {
            try {
                runSQL();
                // catch문에서 체크 -> 언체크(런타임) 변경해서 내던지기
            } catch (SQLException e) {
                throw new RuntimeSQLException(e);
            }
        }
    }

    + 예외를 전환할 때에는 반드시 기존예외를 포함해서 스택 트레이스를 남기도록 합시다. 스택 트레이스를 남겨야 변환되기 전의 예외(SQLException)를 확인할 수 있습니다.

## 프로젝트 exception 구현

### 1. @ExceptionHandler 를 활용한 개별 예외 처리
    - 특정 컨트롤러 내에서 발생한 예외를 처리.
    - 컨트롤러 클래스에서 정의도니 예외만 처리

    ex)
        @RestController
        @RequestMapping("/api/orders")
        public class OrderController {

            @GetMapping("/{id}")
            public String getOrder(@PathVariable Long id) {
                if (id <= 0) {
                    throw new IllegalArguemntException("Order ID must bew greater than 0");
                }
                return "Order: " + id;
            }

            @ExceptionHandler(IllegalArgumentException.class)
            public ResponseEnitty<String> handlerIllegalArgumentException(IllegalArgumentException ex) {
                return ResponseEntity.badRequest().body("Error: " + ex.getMessage());
            }
        }

        - 장점 : 특정 컨트롤러에 대해 세밀한 예외 처리가 가능.
        - 단점 : 모든 컨트롤러마다 작성해야 하므로 코드 중복 발생.

### 2. @ControllerAdvice 를 활용한 전역 예외 처리

    - 전역 예외 처리를 제공하며, 프로젝트 전반에 걸쳐 발생하는 예외를 한 곳에서 처리.

    ex)
        @RestControllerAdvice
        public class GlobalExceptionHandler {

            @ExceptionHandler(IllegalArgumentException.class) 
            public ResposneEntity<String> handlerIllegalArgumentException(IllegalArgumentException ex) {
                return ResponseEntity.badRequest().body("Error: " + ex.getMessage());
            }

            @ExceptionHandler(Exception.class)
            public ResponseEntity<String> handleGeneralException(Exception ex) {
                return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                        .body("An unexpected error occurred: " + ex.getMessage());
            }
        }

        - 장점 : 코드 중복 줄임, 공통 로직으로 재사용 가능.
        - 단점 : 세부 컨트롤러 수준에서 예외 처리를 구분하기 어려울 수 있다.
    
### 3. ResponseStatusException

    - Spring Boot에서 자주 사용되는 간단한 예외 처리 방법으로, 특정 HTTP 상태와 메시지를 함께 반환할 수 있다.
    
    ex)
        @GetMapping("/order/{id}")
        public String getOrder(@PathVariable int id) {
            if (id <= 0) {
                throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "invalid Order Id");
            }
            return "ORder ID: " + id;
        }

### 4. Custom Exception 

    정의 :

    스프링에서 제공하는 예외들을 제외한 프로젝트에서 발생하는 특별한 예외를 처리하기 위해 개발자가 커스터마이즈한 방식

    스프링이 제공하는 기본 예외
    ex)
        MethodArgumentNotValidException, 
        ex)
            요청 본문에 포함된 DTO에서 유효성 검사 실패 시 발생 (예: 필수 값 누락, 형식 오류 등).

        MethodArgumentTypeMismatchException
        ex)
            입 불일치 시 발생 (예: 숫자 대신 문자열 전달).

    개발자가 커스터마이징 한 예외
    ex)
        CustomException 
        비즈니스 로직에서 발생하는 예외
        ex)
            MEMBER_NOT_EXISTS("존재하지 않는 회원입니다.", BAD_REQUEST),
            ORDER_NOT_EXISTS("존재하지 않는 주문입니다.", BAD_REQUEST),
    

### 5. @ResponseStatus

    - 커스텀 예외에 HTTP 상태를 직접 설정할 수 있는 간단한 방식.

    ex)
        @ResponseStatus(HttpStatus.NOT_FOUND)
        public class OrderNotFoundException extends RuntimeException {
            public OrderNotFoundException(String message) {
                super(message);
            }
        }

        - 자동으로 HTTP 404 상태와 메시지를 반환합니다.

### 6. Spring Validation & BindingResult        

    - 요청 데이터 검증에 실패했을 때, 자동으로 예외를 발생시키거나 커스텀 처리를 할 수 있다.

    ex)
        @PostMapping("/orders")
        public ResponseEntity<String> createOrder(@Valid @RequestBody OrderRequest request, BindingResult bindingResult) {
            if (bindingReslt.hasErrors()) {
                return ResponseEntity.badRequest().body("Validation failed: " + bindingResult.getFieldError().getDefaultMessage()); 
            }
            return ResponseEntity.ok("Order created");
        }

    ● 일반적으로 @ControllerAdvice + Custom Exception 방식이 가장 널리 사용됩니다.        

### Exception 종류

    1. MethodArgumentNotValidException

    발생 조건 : @Valid 사용하여, DTO 검증 중 유효성 검사 실패.
    사용 위치 : @RequestBody, @ModelAttribute 데이터 바인딩 시.

    2. MethodArgumentTypeMismatchException

    발생 조건 : @PathVariable, @RequestParam에서 전달된 값의 타입이 예상 타입과 일치하지 않을 때.
    사용 위치 : 컨트롤러의 요청 매핑 파라미터.

    3. MissingServletRequestParameterException

    발생 조건 : @RequestParam으로 명시된 필수 파라미터가 누락된 경우.
    사용 위치 : 컨트롤러 메서드에서 요청 매핑 파라미터 처리.

    ex)
        @GetMapping("/search")
        public String search(@RequestParam String query) {
            // query가 누락되면 해당 Exception 발생
            return "검색어: " + query;
        }

    4. HttpMessageNotReadableException

    발생 조건 : 요청 본문(@RequestBody)의 JSON 데이터가 잘못된 형식이거나 역직렬화가 실패할 경우.
    사용 위치 : JSON 요청 처리.

    ex)
        // 잘못된 요청 예시
        {"name" : "joo", "age" : "not-a-number"} 

    5. HttpRequestMethodNotSupportedException

    발생 조건 : 지원되지 않는 HTTP 메서드로 요청이 들어올 경우
    사용 위치 : 잘못된 HTTP 메서드로 컨트롤러 호출.

    ex)
        @PostMapping("/users")      
        public String createUser() {
            return "사용자 생성";
        }
        // GET/users 요청 시 예외 발생.

    6. MissingServletRequestParException

    발생 조건 : Multipart 요청 (@RequestPart)에서 파일이나 파트가 누락된 경우
    사용 위치 : 파일 업로드 처리.

    ex)
        @PostMapping("/upload")
        public String uploadFile(@RequestPart MultipartFile file) {
            // file 이 누락되면 Missing~Exception 발생
            return "업로드 성공";
        }

    위에 예외 이외에도 여러가지의 예외가 더 있다. 이런 예외들은 스프링에서 기본적으로 제공되며, 각각의 예외 상황에 맞게 @ExceptionHandler or @ControllerAdvice 를 사용하여 처리할 수 있다.

    CustomExcepton 은 위와 같은 예외들과는 별도로, 애플리케이션에서 발생할 수 있는 비즈니스 로직 관련 예외를 처리하는 데 사용된다.