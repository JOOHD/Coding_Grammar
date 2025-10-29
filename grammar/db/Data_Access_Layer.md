## DAL

### JPA (Java Persistence API)

하나의 요청(Request) → 트랜잭션 시작 → 영속성 컨텍스트 생성 → 엔티티 조회/변경 → 커밋 시점에 flush → DB 반영 → 트랜잭션 종료

| 기능                         | 설명                                 |
| -------------------------- | ---------------------------------- |
| Entity                     | DB 테이블과 매핑되는 클래스                   |
| EntityManager              | JPA의 핵심. 영속성 컨텍스트를 관리              |
| Persistence Context        | 엔티티의 생명주기를 관리하는 1차 캐시 같은 공간        |
| JPQL                       | 객체 기반 쿼리 언어 (SQL이 아니라 엔티티 필드 기준으로) |
| flush()                    | 영속성 컨텍스트 → DB로 변경사항 반영             |
| detach(), clear(), close() | 영속성 컨텍스트 제어 메서드                    |

- @Entity, @Id, @GeneratedValue
- @OneToMany, @ManyToOne, @JoinColumn
- JPQL, Criteria API, Native Query
- 영속성 컨텍스트를 기준으로 작동한다는 점 (조회한 엔티티는 자동 dirty checking 됨)

### Hibernate (JPA 구현체)

- Hibernate Session, Lazy Loading, Proxy 등이 Hibernate에서 제공
  
| 개념                     | 설명                                         |
| ---------------------- | ------------------------------------------ |
| Session                | JPA의 EntityManager와 유사한 Hibernate 고유 인터페이스 |
| 1차 캐시                  | 동일 트랜잭션 내 동일 엔티티 재조회 시 DB 재조회 안 함          |
| Dirty Checking         | 영속 상태 엔티티의 변경 감지 → flush 시 update SQL 생성   |
| Proxy (Lazy Loading)   | 실제 엔티티 대신 프록시 객체로 지연로딩 구현                  |
| FetchType.LAZY / EAGER | 연관관계 로딩 전략 지정                              |
| N+1 Problem            | Lazy 로딩으로 인한 과도한 쿼리 발생 이슈                  |
| FlushMode              | 언제 flush를 수행할지 설정 (AUTO, COMMIT 등)         |

### Hibernate Session (and EntityManager)

역할
- 실제 DB Connection을 관리하고, 엔티티의 상태 변화를 추적.
- EntityManager 는 내부적으로 Hibernate Session을 감싸고 있다.

주요 동작
- session.save(), session.update() -> insert/update SQL 생성
- flush() 시저ㅣㅁ에 SQL이 DB로 전달
- Commit 시 flush 자동 호출

알야야 할 기증
- 영속성 컨텍스트 1차 캐시
- flush 시점 조정 
- SessionFactory -> Session 생성 (Spring 에서는 @PersistenceContext 자동 관리)

### Proxy (지연 로딩)

정의
- Hibernate 가 실제 엔티티를 즉시 로딩하지 않고, 대신 가짜(Proxy) 객체를 먼저 리턴
- 아직 DB에 쿼리를 안 날리고, 필드 접근 시점에 쿼리를 실행함.

동작 방식
Member member = em.find(Member.class, 1L);
Team team = member.getTeam(); // Lazy 로딩 시 아직 DB 조회 안 함
team.getName(); // 실제로 이때 DB 조회 발생

핵심 개념
- Proxy는 라이브러리로 생성된 가짜 클래스
- Hibernate Session 이 살아있지 않으면, LazyInitializtionException 발생
- EAGER 로딩은 지연로딩 대신 즉시 조인(Fetch Join)

실무에서의 핵심 포인트
- @Transactional 범위를 벗어나면 Lazy 로딩 불가
- DTO로 변환 시점 조심
- fetch join, entity graph로 최적화

### Transaction

정의
- 하나의 작업 단위를 atomic 으로 보장하는 메커니즘
- DB 작업 중 실패 시, 전체 롤백

JPA 와 트랜잭션 관계
- JPA 의 모든 영속성 컨텍스트 동작은 트랜잭션 안에서만 의미 있음
- @Transactional 로 감싸지 않으면 flush 나 commit 이 동작하지 않음

알아야 할 기능
- @Transactional(propagation = Propagation.REQUIRED)
- readOnly 트랜잭션 (flush 생략 가능)
- 트랜잭션 전파
- 트랜잭션 격리

### Spring Data JPA

정의
- JPA를 더 쉽게 쓰기 위한 Spring Wrapper
- Repository 인터페이스 자동 구현, CRUD 메서드 자동 생성

알아야 할 기능
- JpaRepository, CrudRepository
- 메서드 이름으로 쿼리 생성 (findByEmail, findBySatatusAndType)
- @Query / @Modifing + @Transacational + update/delete
- Pageable, Sort 기능

전체 동작 흐름 예시
@Transactional
public void updateMember(Long id, String name) {
    // 트랜잭션 시작(@Transactional)
    // 영속성 컨텍스트 생성
    Member member = memberRepository.findById(id).orElseThrow(); // Hibernate Session에서 조회
    member.setName(name); // Dirty Checking 발생
    // 메서드 종료 시점 -> commit -> flush() -> update SQL 실행
}

### 이 5단계에 모든 개념이 들어있음
| 단계          | 관련 개념                                  |
| ----------- | -------------------------------------- |
| 트랜잭션 시작     | Spring TransactionManager              |
| 영속성 컨텍스트 생성 | JPA EntityManager                      |
| 조회          | Hibernate Session, Proxy, Lazy Loading |
| 변경 감지       | Dirty Checking                         |
| 커밋          | flush(), SQL 생성, DB 반영                 |

### “영속성 컨텍스트 → 조회(Proxy) → Dirty Checking → Flush → Commit”

영속성 컨텍스트: "엔티티의 생명주기를 관리하는 공간"

정의 
- DB에서 가져온 엔티티를 "메모리에 저장해두고 관리하는 공간"
- JPA가 DB와 직접 붙지 않고, 이 컨텍스트를 거쳐서 DB를 다룸.

비유
- DB는 진열장, 영속성 컨텍스트는 장바구니
- 장바구니 안에서 물건(엔티티)을 수정하면, 계산할 때(DB flush 시점)에만 실제로 반영

주요 특징
- 같은 트랜잭션 내에서 같은 엔티티는 한 번만 조회(DB 재조회 X)
- 엔티티의 변경 상태를 추적 (Dirty Checking)
- 트랜잭션이 commit 될 때, 변경사항을 DB에 반영 (flush)

### 조회(find / getReference): Proxy 객체와 Lazy Loading

- find() vs getReference()

| 메서드              | 반환       | 설명               |
| ---------------- | -------- | ---------------- |
| `find()`         | 실제 엔티티   | 즉시 DB 쿼리 수행      |
| `getReference()` | Proxy 객체 | 실제 데이터는 필요할 때 로딩 |

Proxy
- Hibernate 가 진짜 엔티티 대신 "가짜 객체(프록시)"를 만들어서 리턴
- 아직 DB에 쿼리를 안 날리고, 필드 접근 시점에 쿼리를 실행
- 프록시는 실제로는 "DB 조회를 늦추는 지연 로딩용 가짜 클래스"

Proxy를 쓰는 이유 
- N + 1을 해결하기 위해서가 아님 
  - 불필요한 즉시 조회를 방지해서 성능 최적화 위함

Order order = orderRepository.findByID(1L);
Member membere = order.getMember(); // Lazy 로딩 : 아직 쿼리 안 날림

- 즉, N + 1문제는 Proxy 를 잘못 사용할 때 생기는 부작용이다.
- Proxy는 성능 최적화를 위한 Lazy Loading 도구이다.

### Dirty Checking: “변경 자동 감지”

- JPA 가 영속성 컨텍스트 안에서 엔티티의 필드 값을 계속 추적
- setter 로 값이 변경되면 Hibernate 가 원본 스냅샷과 비교해서 변경을 감지함
- 트랜잭션이 커밋되기 전에 flush가 일어나면 Hibernate가 자동으로 UPDATE SQL 생성함

Member member = em.find(Member.class, 1L);
member.setName("JOO"); // setter만 호출했는데
// commit 시점에 UPDATE member SET name = 'JOO' where id = 1;

- 즉, save() 나 update() 를 직접 호출할 필요가 없다.
    단, 이 엔티티가 영속 상태일 때만 자동 감지

### Flush: "DB 반영 타이밍 제어"

- flush는 "영속성 컨텍스트의 변경 내용을 DB에 반영"하는 단계
- 트랜잭션 커밋 전에 자동으로 호출됨

동작 순서:
1. 변경 감지 (Dirty Checking)
2. SQL 생성
3. SQL 실행 (DB 반영)

-> flush 가 일어나도 트랜잭션이 commit되지 않으면 실제 반영은 안 됨 (rollback 가능)

Commit: “트랜잭션 확정”
- flush 후 commit이 되면 DB 변경이 확정됨.
- commit이 실패하면 rollback으로 되돌림.

### 전체 흐름 정리 (한 사이클)

@Transactional
public void updateMember(Long id, String name) {

    // 영속성 컨텍스트 생성
    Member member = em.find(Member.class, id); // -> DB 조회 or Proxy 반환

    // 영속 상태로 관리됨 (컨텍스트에 저장)
    member.setName();                          // -> Dirty checking

    // 트랜잭션 커밋 시, flush 발생
    // -> Hibernate 가 UPDATE SQL 생성 및 실행

    // commit 후, DB 반영 완료
}

### 한 줄 요약 정리 (기억용)

| 단계          | 역할        | 핵심 키워드                 |
| ----------- | --------- | ---------------------- |
| 1. 영속성 컨텍스트 | 엔티티 관리 공간 | 1차 캐시, 생명주기            |
| 2. 조회       | 엔티티 로딩    | Proxy, Lazy Loading    |
| 3. 변경       | 엔티티 수정 추적 | Dirty Checking         |
| 4. flush    | 변경 SQL 생성 | flushMode, commit 시 자동 |
| 5. commit   | DB 반영 확정  | TransactionManager     |

### 핵심 이미지로 기억하기 (말로 설명)

“영속성 컨텍스트는 DB 앞단의 메모리 상자”
1️⃣ find() 하면 상자에 엔티티가 들어오고
2️⃣ setName() 하면 상자 안의 값이 바뀌고
3️⃣ flush() 하면 상자 내용이 DB로 흘러가고
4️⃣ commit() 하면 진짜 DB에 확정된다.