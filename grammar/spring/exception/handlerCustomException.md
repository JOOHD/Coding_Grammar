## 본문

### @ControllerAdvice + Custom Exception + Validation + BindingReslt 예외처리

    @Slf4j
    @RestControllerAdvice
    public class ExceptionHandler {

        // 컨트롤러 DTO validation handler
        @org.springframework.web.bind.annotation.ExceptionHandler(MethodArgumentNotValidException.class)
        public ResponseEntity<MethodInvalidResponse> methodValidException(final MethodArgumentNoValidExcetpion e) 
        {
            BindingResult bindingReSULT = e.getBindingResult();
            return ResponseEntity
                    .status(HttpStatus.BAD_REQUEST) 
                    .body(MethodInvalidResponse.builder()
                            .errorCode(bindingResult.getFieldErrors().get(0).getCode())
                            .errorMessage(bindingResult.getFieldErrors().get(0).getDefaultMessage())
                            .build());
        }

        // 컨트롤러 @PathVariable TypeMismatch handler
        @org.springframework.web.bind.annotation.ExceptionHandler(MethodArgumentTypeMismatchException.class)
        public ResponseEntity<ExceptionResponse> methodArgumentTypeMismatchException() {
            return ResponseEntity
                    .status(HttpStatus.BAD_REQUEST)
                    .body(ExceptionResponse.builder()
                            .errorCode(METHOD_ARGUMENT_TYPE_MISMATCH)
                            .errorMessage(METHOD_ARGUMENT_TYPE_MISMATCH.getMessage())
                            .build());
        }

        // CustomException handler
        @org.springframework.web.bind.annotation.ExceptionHandler(CustomException.class)
        public ResponseEntity<ExceptionResponse> customRequestException(final CustomException c) {
            log.error("Api Exception => {}, {}", c.getErrorCode(), c.getErrorMessage());
            return ResponseEntity
                    .status(c.getErrorStatus())
                    .body(ExceptionResponse.builder()
                            .errorCode(c.getErrorCode())
                            .errorMessage(c.getErrorMessage())
                            .build()
                    );
        }

        @Getter
        @Builder
        public static class MethodInvalidResponse {

            private String errorCode;
            private String errorMessage;
        }

        @Getter
        @Builder
        public static class ExceptionResponse {

            private ErrorCode errorCode;
            private String errorMessage;
        }
    }

    @Getter
    public class CustomException extends RuntimeException {

        private final ErrorCode errorCode;
        private final String errorMessage;
        private final HttpStatus errorStatus;

        public CustomException(ErrorCode errorCode) {
            super(errorCode.getMessage());
            this.errorCode = errorCode;
            this.errorMessage = errorCode.getMessage();
            this.errorStatus = errorCode.getHttpStatus();
        }
    }

### ExceptionHandler, CustomException, ErrorCode class 설명

    1. ExceptionHandler
    
    역할 :

    전역 예외 처리 역할을 담당하며, 스프링의 @RestControllerAdvice 와 @ExceptionHandler 를 활용해 발생한 예외를 포착하고 적절한 응답을 클라이언트로 반환.

    특징 :
    - 다양한 예외를 처리할 수 있돌고 예외 타입ㄹ려로 처리 메서드를 정의.
    - HTTP 상태 코드와 에러 메시지를 포함한 응답 객체 (ResponseEntity<DTO>)를 클라이언트로 반환.

    2. CustomException

    역할 :

    사용자 정의 예외를 정의하는 클래스이다.
    ErrorCode를 사용하여, RuntimeException을 상속받는 CustomException을 생성해준다. 해당 클래스는 다양한 예외 클래스에 상속시켜 status, message를 사용할 수 있다.

    특징 :

    - 예외 발생 시, 에러 코드를 받아 특정 비즈니스 로직과 관련된 에러를 구분.
    - 에러 코드와 함께 사용자 저으이 메시지를 설정.
    - 런타임 예외를 상속하여 Unchecked 예외로 동작.

    1. ErrorCode

    역할 : 프런트앤드로 전달되는 데이터, 애플리케이션 전반에서 사용할 에러 코드를 enum으로 관리

    특징 :
    - 예외를 일관되게 정의하기 위해 사용.
    - 예러 상태 코드, 에러 메시지 등을 포함할 수 있다.
    - 특정 에러 상황을 코드로 구분하여 처리하기 쉽다.

### 클래스 간 연관성

    ● CustomException <-> ErrorCode

    CustomException은 발생한 예외에 대해 ErrorCode를 참조하여 상태코드와 메시지를 설정

    이를 통해 비즈니스 로직의 예외를 구체적으로 표현.

    ● ExceptionHandler <-> CustomException

    ExceptionHandler는 컨트롤러에서 발생한 CustomException을 포착하여 처리.
    CustomException에 설정된 ErrorCode를 이용해 클라이언트에 적절한 응답을 반환.
    
### ExceptionHandler class method 설명

    1. methodInvalidException
 
    처리예외 : MethodArgumentNotValidException (DTO Validation 실패)

    로직 :
    - BindingResult 를 통해 Validation 에러 메시지를 추출.
    - HTTP 상태 코드 400 Bad Request 와 함께 커스텀 응답 객체를 반환.

    2. methodArgumentTypeMismatchException
 
    처리예외 : MethodArgumentTypeMismatchException (잘못된 URL 파라미터 타입)

    로직 :
    - HTTP 상태 코드 400 Bad Request와 함께 타입 불일치 관련 메시지를 반환.
  
    3. customRequestException
 
    처리예외 : CustomException (사용자 정의 예외)

    로직 :
    - CustomException에서 설정된 에러 코드와 메시지를 응답 객체에 담아 반환합니다.
    - 런타임 로직의 특수한 에러를 처리.

