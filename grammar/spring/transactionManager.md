## 본문

### TransactionManager 란?
    트랜잭션 관리를 담당하는 중요한 컴포넌트이다. 트랜잭션 관리는 데이터베이스 작업을 그룹화하고, 이 그룹화된 작업이 모두 성공하거나 모두 실패하도록 보장 한다. 이를 통해 데이터의 일관성과 무결성을 유지할 수 있다.

    다음은 TransactionManager 를 사용하는 예시이다. Spring에서는 다양한 TransactionManager 구현체를 제공하며, 여기서는 DataSourceTransactionManager 를 예로 들어 설명하겠습니다.

    ● Spring Transaction Manager 예시

    1) 데이터베이스 설정
    @Configuration
    public class DataSourceConfig {

        @Bean
        public DataSource dataSource() {
            HikariDataSource dataSource = new HikariDataSource();
            dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
            dataSource.setusername("username");
            dataSource.setPassword("password");
            return dataSource;
        }

        @Bean
        public PlatformTransactionManager transactionManager(DataSource dataSource) {
            return new DataSourceTransactionManager(dataSource);
        }
    }

    2) 서비스 클래스에서 트랜잭션 관리
    @Service
    public class MemberService {

        private final MemberRepository memberRepository;

        public MemberService(MemberRepository memberRepository) {
            this.memberRepository = memberRepository;
        }

        @Transactional
        public void transferMoney(String fromId, String toId, int money) {
            Member fromMember = memberRepository.findById(fromId);
            Memebr toMemebr = memberRepository.findById(toId);

            fromMember.setMoney(fromMember.getMoney() - money);
            toMember.setMoney(toMember.getMoney() + money);

            memberRepository.save(fromMember);
            memberRepository.save(toMember);
        }
    }

    ● 주요 설명
    - DataSource 설정 : 데이터베이스 연결 설정과 TransactionManager bean을 구성한다. DataSourceTransactionManager 는 JDBC를 사용하여 트랜잭션을 관리 한다.
    - Transaction 관리 : @Transactional 어노테이션을 사용하여 transferMoney 메서드가 트랜잭션으로 관리되도록 한다. 이 메서드 내의 모든 데이터베이스 작업이 하나의 트랜잭션으로 묶인다.
      - fromMember 의 돈을 감소시키고, toMember 의 돈을 증가시킨 후, 두 맴버를 저장한다.
      - 메서드가 성공하면 트랜잭션이 커밋되고, 예외가 발생하면 롤백된다.                
      
    ● 트랜잭션 매니저의 동기화 의미
    트랜잭션 매니저가 커넥션을 동기화한다는 의미는, 트랜잭션 범위 내에서 동일한 데이터베이스 커넥션이 사용되도록 보장한다는 것이다.
    1) 일관성 유지 : 트랜잭션 내의 모든 작업이 동일한 커넥션을 사용하여 일관된 상태를 유지한다.
    2) 성능 향상 : 커넥션을 재사용함으로써 커넥션을 계속해서 열고 닫는 오버헤드를 줄일 수 있다.
    3) 자동 커밋 방지 : 트랜잭션 범위 내에서 자동 커밋을 비활성화하여, 모든 작업이 완료되기 전에는 커밋되지 않도록 한다.

    ● 추가 예시
    - 트랜잭션 관리 없는 JDBC 예시

        ex) 
        public void transferMoneyWithoutTransaction(Stirng fromId, String toId, int money) throws SQLException {

            Connection con = dataSource.getConnection();

            try {
                Member fromMember = memberRepository.findById(con, fromId);
                Member toMember = memberRepository.findById(con, toId);

                memberRepository.update(con, fromId, fromMember.getMoney() - money);
                memberRepository.update(con, toId, toMember.getMoney() + money);
            }
        }

    - JPA를 사용한 트랜잭션 예시
    @Service
    public class MemberService {

        private final MemberRepository memberRepository;

        public MemberService(MemberRepository memberRepository) {
            this.memberRepository = memberRepository;
        }

        @Transactional
        public void transferMoney(String fromId, String toId, int money) {
            Member fromMember = memberRepository.findById(fromId).orElseThrow();
            Member toMember = memberRepository.findById(toId).orElseThrow();

            fromMember.setMoney(fromMember.getMoney() - money);
            toMember.setMoney(toMember.getMoney() + money);
        }
    }   
    - JPA 예시 : @Transactional 애너테이션을 통해 JPA 앤티티 관리와 트랜잭션을 통합 관리 한다. MemberRepository 는 JpaRepository 를 확장하여 기본적인 CRUD 메서드를 제공한다.     