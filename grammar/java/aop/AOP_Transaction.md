## 본문

### 트랜잭션 

    Transaction : DB의 상태를 변화시키는 하나의 논리적 기능을 수행하기 위한 작업 단위, 일련의 연산들.

    스프링에서는 JDBC, JPA 등 여러 데이터 접근 기술을 편리하게 사용하기 위해 트랜잭션 추상화(PlatformTransactionManager)을 제공, 또 여러 데이터 접근 기술들(JDBC, Mybatis, JAP등)에 대한 트랜잭션 매니저를 제공.

![platformTransaction](/grammer/img/platformTransaction.png)

    ● PlatformTransactionManager

    public interface platformTransaction extends TransactionManager {

        TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;

        void commit(TransactionStatus status) throws TransactionException;
 
        void rollback(TransactionStatus status) throws TransactionException;
    }

    + Spring Boot는 어떤 데이터 접근 기술을 사용하는지 자동으로 인식해서 적절한 트랜잭션 매니저를 선택해서 스프링 빈으로 등록까지 해준다. 
    따라서 개발자는 트랜잭션 매니저를 선택하고 등록하는 과정도 생략 가능.

    "transactionManager는 DataSource를 주입받아 등록되는데 그렇다면 DB Server에 대한 정보인 DataSource는 어떻게 설정할까?"

        // 두 가지 방법이 있따.
        1. applicationproperties에 설정정보를 작성함으로써 스프링부트가 설정정보를 보고 자동으로 빈으로 등록하도록 유도

        2. 직접 수동으로 빈 등록

    또 트랜잭션 매니저는 트랜잭션 매니저를 직접 작성애서 사용하는 방법과 간접적으로 어노테이션을 사용하는 방법이 있다.

    ● 트랜잭션 직접 작성

        @Service
        public class MyService {

            @Autowired
            PlatformTransactionManager transactionManager;

            TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        }       

        - 스프링 부트가 자동으로 등록한 트랜잭션 매니저를 사용 by @Autowired
        - status는 commit, rollback 에서 사용된다.

    ● 트랜잭션 시작 - commit/rollback

        TransactionStatus status = transactionManager.getTransaction(new DefaulTransactionDefinition());

        public void logic()
        {
            try {
                // 비즈니스 로직
                biziLogic(fromId, toId, money);
                transactionManager.commit(status); // 성공 시, 커밋
            } catch (Exception e) {
                transactionManager.rollback(status); // 실패 시, 롤백
                throw new IllegalStatueException(e);
            }
        }    

    ● 간접적으로 어노테이션을 통해 사용

    위 방법에서는 PlatformTransactionManager를 직접 사용하여 트랜잭션을 관리하고 있따. 하지만 @Transactional 어노테이션을 사용하면 더 간결해진다.  

        @Service
        public class MyService {
            
            @Transactional
            public void logic()
            { 
                bizlogic();
                ...
            }
        }  

![transactional_flow](/grammer/img/transactional_flow.png)

    메소드나 클래스에 @Transactional을 걸어주면

    1. AOP 프록시에서 스프링 부트를 통해 자동으로 빈으로 등록된 트랜잭션 매니저를 사용해 등록된 DataSource를 통해 커넥션을 생성해서 트랜잭션 동기화 매니저에 보관하고 트랜잭션을 시작한다.

    2. 트랜잭션 처리로직에서 실제 비즈니스로직을 호출하여 트랜잭션을 실행하는데 이 때 repository에서는 동기화 매니저에 등록된 커넥션을 얻어 비즈니스 로직 실행한다.

    3. AOP 프록시에서 트랜잭션에 성공하면 commit, 실패하면 rollback

    스프링의 트랜잭션 AOP는 @Transactional 어노테이션을 인식해서 트랜잭션을 처리하는 프록시를 적용해준다.

    @Transactional 어노테이션을 사용해 트랜잭션을 처리하는 방법을 도입하면 코드가 매우 깔끔해진다.

### 트랜잭션 주의점 & 옵션

    메소드 or 클래스에 @Transactional을 붙이면, AOP 대상이 된다. 따라서 실제 객체 대상에 트랜잭션을 처리해주는 프록시 객체가 스프링 빈에 등록이 되과, 해당 클래스를 주입을 받을 때에도 실제 객체 대신 프록시 객체가 주입된다.

    ● 트랜잭션 적용 위치

    스프링은 항상 더 구체적이고, 자세한 것이 높은 우선 순위를 가진다.
    
    ex)
        class -> @Transactional (readOnly = true)
        method -> @Transactional (readOnly = false)

        class < method -> 클래스 true여도 method에 영향을 주진 못함.

    ● 트랜잭션 AOP @Transactional 사용 시, 주의사항 
    
    1. 프록시 내부 호출

    결론적으로 동일 클래스 내부에 존재하는 메소드1, 메소드2가 있다고 가정하면,
    @Transactional이 적용되지 않은 메소드1에서 내부 호출로 @Transactional이 적용된 메소드2를 호출을 하게 되면 메소드2에는 @Transactional이 적용되지 않는다.

![trasactional_cause1](/grammer/img/trasactional_cause1.png)    

    ● Why?

    @Transactional이 적용된 메소드는 AOP 프록시 가짜 객체에서 @Transactional이 붙은 해당 메소드를 실행하고, 실제 객체를 호출되어 동작한다.

![trasactional_cause2](/grammer/img/trasactional_cause2.png)   

    만약 @Transactional 이 적용되지 않은 메소드를 호출하면 트랜잭션이 적용되어 있지 않기 때문에 가짜 객체를 통해 실제 객체가 호출되는 것이 아니라, 곧바로 실제 객체가 호출되어 실제 객체의 메소드를 실행하게 된다.

    따라서 @Transactional이 붙어있다 하더라도 실제 객체에서는 이를 인식하지 못하기 때문에 트랜잭션이 적용되지 않는다.

    ● 해결

    클래스를 분리해주면 된다.

![transactional_cause3](/grammer/img/trasactional_cause3.png)    

    트랜잭션이 적용되지 않은 메소드1과 트랜잭션이 적용된 메소드2를 한 클래스 내에 설정하는 것이 아닌 트랜잭션이 적용된 메소드를 따로 분리함으로써 해결할 수 있습니다.

    
