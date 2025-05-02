## 본문

### Spring AOP를 사용한 애플리케이션 클래스

    @Slf4j
    @Aspect
    @Component
    public class LoggerAspect {

        /**
        * @param joinPoint
        * @return
        * @throws Throwable
        */

        @Around("execution(* com.study.domain..*Controller.*(..)) || execution(* com.study.domain..*Service.*(..)) || execution(* com.study.domain..*Mapper.*(..))")
        public Object printLog(ProceedingJoinPoint joinPoint) throws Throwable {

            // joinPoint.getSignature().getDeclaringTypeName(); // 클래스 이름 (예: com.study.domain.user.UserController)
            // joinPoint.getSignature().getName();              // 메서드 이름 (예: getUser)
            String name = joinPoint.getSignature().getDeclaringTypeName();
            String type =
                    StringUtils.contains(name, "Controller") ? "Controller ===> " :
                    StringUtils.contains(name, "Service") ? "Service ===> " :
                    StringUtils.contains(name, "Mapper") ? "Mapper ===> " :
                    "";

            log.debug(type + name + "." + joinPoint.getSignature().getName() + "()");
            return joinPoint.proceed();
        }

    }

    
    위 코드는 Spring AOP(Aspect-Oriented Programming)를 사용하여 애플리케이션의 주요 구성 요소인 'Controller', 'Service', 'Mapper' 계층의 메서드 호출을 로깅하는 기능을 구현한 Aspect 클래스이다.

    이를 통해 애플리케이션의 동작을 모니터링하고 디벙깅하기 쉽도록 돕는다.

### 코드 해석   

    1. 어노테이션 설명
    
    ● @Sl4fj

        - Lombok이 제공하는 어노테이션으로, 로그를 쉽게 작성할 수 있도록 돕는다.
         
        - 이를 통해 'log.info()', 'log.debug()', 'log.error()' 등의 메서드를 사용할 수 있다.

    ● @Aspect

        - 이 클래스가 AOP의 'Aspect'(관점)임을 선언, 'Aspect'는 특정 관심사를 모듈화한 코드로, 주로 로깅, 보안. 트랜잭션 관리 등과 같은 기능을 구현.

    ● @Component

        - Spring의 컴포넌트 스캔에 의해 자동으로 빈으로 등록되도록 하는 어노테이션이다.

        - @Aspect 와 함께 사용되어, 이 Aspect 클래스가 Spring의 관리 하에 동작하게 된다.
    
    2. 메서드 설명
    
    ● @Around

        - AOP의 어드바이스로, 타겟 메서드 실행 전후에 동작하는 코드를 정의할 때 사용된다.

        - execution(* com.study.domain..*Controller.*(..)) || 
          execution(* com.study.domain..*Service.*(..)) || 
          execution(* com.study.domain..*Mapper.*(..))":

          - 이 pointCut은 'com.study.domain' 패키지와 그 하위 패키지 내에 있는 모든 'Controller', 'Service', 'Mapper' 클래스의 모든 메서드를 타겟으로 지정한다.

          - *Controller.*(..), *Service.*(..), *Mapper.*(..)은 각각 컨트롤러, 서비스, 매퍼 클래스의 모든 메서드를 의미합니다.

    3. 로직 설명

    ● String name = joinPoint.getSignature().getDeclaringTypeName();

        - 현재 실행 중인 메서드가 속한 클래스의 이름을 가져온다.
      
        ● joinPoint.getSignature().getName();

            - Signature 객체를 반환한다. 이 객체는 메서드의 이름, 반환 타입, 선언된 클래스등의 메타데이터를 포함하고 있다.

        ● getName()

            - Signature 객체에서 호출할 수 있는 메서드로, 현재 실행 중인 메서드의 이름을 문자열로 반환.

            ● 코드에서의 역할

              - String methodName = joinPoint.getSignature().getName();

                - 위 코드에서 methodName 변수는 현재 실행 중인 메서드의 이름을 가지게 된다. 

                    ex) UserController 클래스의 getUser 메서드가 호출되고 있다면,
                        joinPoint.getSignature().getName() 은 'getUser'를 반환한다.

                    ex) String name = joinPoint.getSignature().getDeclaringTypeName(); // 클래스 이름 (예: com.study.domain.user.UserController)
                    String methodName = joinPoint.getSignature().getName();            // 메서드 이름 (예: getUser)

                    System.out.println("Class: " + name);
                    System.out.println("Method: " + methodName);  

                    만약 UserController 의 getUser() 메서드가 호출되고 있을 때, 위 코드를 실행하면 다음과 같은 출력이 이루어진다.

                    Class : com.study. domain.user.UserController
                    Method : getUser

                    따라서 joinPoint.getSignature().getName() 은 현재 AOP가 적용된 메서드가 무엇인지 확인하는 데 사용된다.

                    이를 통해 어떤 메서드가 실행되었는지 로깅하거나, 특정 메서드에 대한 추가 처리를 수행할 수 있다.  

    ● String type 

        - 클래스 이름에 'Controller', 'Service', 'Mapper' 가 포함되어 있는지를 확인하고, 
            이에 따라 로그 메시지의 앞부분에 해당하는 문자열('Controller ===> ', 'Service ===> ', 'Mapper ===> ')을 설정한다.

        - 이 과정은 StringUtils.contains() 메서드를 사용하여 이루어진다.

    ● log.debug(type + name + "." + joinPoint.getSignature().getName() + "()");

        - 설정한 문자열을 포함하여 최종 로그 메시지를 작성한다. 이 메시지는 현재 실행 중인 메서드의 클래스아 메서드 이름을 포맣.

        - ex) Controller ===> com.study.domain.user.UserController.getUser() 와 같은 메시지가 출력.

    ● return joinPoing.proceed();

        - 이 메서드는 실제 타겟 메서드를 실행한다. 이 메서드 호출 전후에 로직을 추가할 수 있다.

        - proceed() 가 호출되기 전에는 타겟 메서드가 실행되지 않으며, 이 호출 이후에 실행된다.

        - 타겟 메서드가 정상적으로 실행된 후, 그 결과가 그대로 반환된다.

### 전체적인 동작

    ● 이 LoggerAspect 는 Controller, Service, Mapper 계층에서 메서드가 호출될 때마다, 해당 메서드의 이름과 소속된 클래스 정보를 디버그 로그로 출력한다.                

    ● 로깅은 개발이나 디버깅 시, 코드가 어떤 흐름으로 실행되는지 파악하는 데 유용하다.

    ● 로그 메시지는 형식적으로 '타입 ===> 클래스.메서드()' 형태로 출력되며, 이를 통해 로깅 시각화가 쉽게 이루어진다.

    


