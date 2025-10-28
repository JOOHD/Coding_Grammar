## Hibernate

### Hibernate 란?

JPA 의 구현체 중 하나이자, **ORM(Object Relational Mapping)** 을 실질적으로 수행하는 엔진.

### 역할 요약

- 자바 객체(entity) <-> DB 테이블 간 자동 매핑
- SQL 을 직접 작성하지 않아도, 객체 변경을 감지해서 자동으로 INSERT, UPDATE, DELETE 실행
- 트랜잭션 단위로 "영속성 컨텍스트"를 관리 (1차 캐시, Dirty Checking 등)

- JPA (interface), Hibernate (implements)
### 내부 핵심 동작 구조

Hibernate 는 @Entity, @Table, @Column 등의 JPA 어노테이션을 해석해서,
자바 객체를 테이블로, 필드를 컬럼으로 변환(mapping)

@Entity
@Table(name = "products_table")
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long productId;

    private String productName;
    private BigDecimal price;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}

-> 이걸 기반으로 Hibernate 가 DDL 자동 생성 (schema-generation) 하거나 DML 자동 실행 (insert/update)을 수행

### 하는 일

| 역할                    | 설명                                                                      |
| --------------------- | ----------------------------------------------------------------------- |
| 매핑(Mapping)           | Entity ↔ Table, Field ↔ Column 자동 매핑                                    |
| SQL 자동 생성             | `persist`, `merge`, `remove` 시 자동으로 `INSERT`, `UPDATE`, `DELETE` SQL 실행 |
| 트랜잭션 관리               | 같은 트랜잭션 안에서 동일한 엔티티를 캐싱하고 상태 추적                                         |
| 변경 감지(Dirty Checking) | Entity 값이 바뀌면 SQL을 자동으로 생성해 UPDATE 실행                                   |
| Flush & Commit 관리     | SQL 실행 시점을 조절하여 최적화                                                     |
ㄴ

### 영속성 컨텍스트 (Persistence Context)

hibernate 핵심, 트랜잭션 동안 엔티티를 관리하고 캐싱하는 메모리 공간.

- 주요 기능
- 1차 캐시: 같은 엔티티를 다시 조회해도 DB를 재조회하지 않음
- Dirty Checking: Entity 필드 값이 변경되면 자동으로 UPDATE SQL 준비.
- 쓰기지연: 즉시 SQL 실행 X -> flush 시점에 모아서 실행

@Transactional
public void updateProduct(Long id, BigDecimal price) {
    Product p = productRepository.findById(id).orElseThrow();
    p.setPrice(price); // UPDATE SQL 안 써도 됨
} // flush + commit 시점에 Hibernate가 UPDATE SQL 실행

### flush / commit / dirty checking 관계

| 단계        | 설명      | Hibernate 동작               |
| --------- | ------- | -------------------------- |
| persist() | 엔티티 등록  | INSERT 준비만 함               |
| setXXX()  | 필드 값 변경 | Dirty Checking으로 UPDATE 준비 |
| flush()   | DB 반영   | SQL 실제 실행 (commit 전 자동 호출) |
| commit()  | 트랜잭션 종료 | flush → commit 순서로 실행됨     |

- 즉, Hibernate 는 트랜잭션이 끝나기 전까지 실제 SQL을 DB에 바로 날리지 않는다.

### Hibernate 가 중요한 이유

- 단순한 ORM 툴이 아니라, "DB와 자바 객체 사이의 상태를 지능적으로 동기화하는 엔진"
- 덕분에 개발자는 SQL 대신 객체 상태 관리에 집중할 수 있지만, 아래 매커니즘을 이해하지 않으면 예기치 않은 문제가 터짐:
  - flush 타이밍
  - dirty checking 시점
  - lazy loading 으로 인한 N +1
  - DB default 값 무시 문제

### 핵심 요약

| 키워드                       | 설명                   |
| ------------------------- | -------------------- |
| ORM                       | 객체와 테이블 자동 매핑        |
| Persistence Context       | 엔티티 캐싱 및 상태 추적       |
| Dirty Checking            | 변경 자동 감지 후 SQL 생성    |
| Flush                     | SQL 실행 타이밍 제어        |
| N+1 문제                    | 지연 로딩으로 인한 쿼리 폭증     |
| Creation/Update Timestamp | Hibernate가 직접 시간 채워줌 |






