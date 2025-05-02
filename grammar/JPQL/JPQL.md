## JPQL 

### Java Persistaence Query Languate 개념

    build.gradle

    implementation "com.querydsl:querydsl-jpa"
    annotationProcessor "com.querydsl:querydsl-apt:${querydslVersion}:jpa"

    - JPA 에서 사용하는 객체 지향 쿼리 언어
    - SQL = table 기준으로 쿼리하지만,
    - JPQL = entity 기준으로 쿼리한다.
  
    ex)
        - SQL
        SELECT * FROM addresses WHERE member_id = 1;

        - JPQL
        @Query("SELECT a FROM Addresses a WHERE a.member.id = :memberId")
        List<Addresses> findByMemberId(@Param("memberId") Long memberId);

        - Addresses 는 테이블이 아니고 엔티티 클래스 이름
        - a.member.id 는 엔티티 내부의 객체 관계를 그대로 탐색
        - JPA가 이걸 DB에 맞게 SQL로 번역해준다.
      
### 기능

    1. SELECT (조회)

        @Query("SELECT a FROM Addresses a WHERE a.member.id = :memberId")
        - SELECT a -> Addresses 엔티티 자체를 가져옴
        - member_id 가 아니라, 객체로 탐색해서 a.member.id 임
        
    2. WHERE/AND/OR (조건 탐색)

        @Query("SELECT a FROM Addresses a WHERE a.member.id = :memberId AND a.defaultAddress = true")    

    3. ORDER BY (정렬)    

        @Query("SELECT a FROM Addresses a WHERE a.member.id = :memberId ORDER BY a.createdAt DESC")

    4. Pagination (페이징)

        @Query("SELECT a FROM Addresses a WHERE a.member.id = :memberId")    
        Page<board> findByMemberId(@Param("memberId") Long memberId, Pageable pageable);

        - Page<Entity> 로 결과 반환
        - totalElements, totalPages 등 정보도 자동으로 포함해서 front에 넘기기 편하다.
        
    5. JOIN (엔티티 연관 관계 탐색)

        @Query("SELECT o FROM Orders o JOIN o.member m WHERE m.id = :memberId")

        - Orders 엔티티에서 member라는 객체 필드를 기준으로 조인
        - JOIN FETCH 사용하면 지연 로딩 방지할 수도 있다.

    6. LEFT JOIN / FETCH JOIN   

        LEFT JOIN
        @Query("SELECT o FROM Orders o LEFT JOIN o.member m WHERE m.name = :name")

        FETCH JOIN -> N + 1 문제 해결
        @Query("SELECT o FROM Orders o JOIN FETCH o.member WHERE o.id = :orderId")
        - N+1 문제 해결할 때 정말 자주 씀
    
### 심화 예제

    1. 서브쿼리
        - JPQL 에서는 WHERE 절에만 서브쿼리를 넣을 수 있다.

        @Query("SELECT a FROM Addresses a WHERE a.createdAt = (" +
               "SELECT MAX(sub.createdAt) FROM Addresses sub WHERE sub.member.id = :memberId))
        Optional<Addresses> findLatestAddressByMemberId(@Param("memberId") Long memberId);               

    2. GROUP BY & HAVING
        - 주로 집계 결과를 그룹핑하거나 특정 조건 걸 때!
      
        @Query("SELECT a.member.id, COUNT(a) " +
               "FROM Addresses a " +
               "WHERE a.defaultAddress = true " +
               "GROUP BY a.member.id " +
               "HAVING COUNT(a) > 1")
        List<Object[]> findMembersWithMultipleDefaultAddresses();

        - 기본 주소가 2개 이상인 회원 찾아내는 쿼리!
        - Object[]로 결과 받기 → 1: memberId, 2: 기본 주소 개수

    3. CASE WHEN

        - JPQL로 조건 분기값 구현
      
        @Query("SELECT a.addressId, " +
               "CASE WHEN a.defaultAddress = true THEN '기본' ELSE '일반' END " +
               "FROM Addresses a WHERE a.member.id = :memberId")
        List<Object[]> findAddressStatuses(@Param("memberId") Long memberId);

        - 결과는 List<Object[]>로 받는다

### 정리

    1. 단순 조회는 JPQL + @Query 
        - JPQL = 고정된 쿼리 -> 조건이 다양할 땐 비추
        - QueryDSL 은 코드로 동적 쿼리 생성 -> 유지보수 편함
        - 타입 안정성 -> 컴파일 에러에서 잡힘 (오타로 인한 런타임 에러 X)
        - JPQL은 문자열 기반이라서 컴파일 오류 없음 → 위험
        - QueryDSL은 타입 안전성 보장, IDE 자동완성 지원, 유지보수, 가독성에 더 좋다

    2. 복잡한 쿼리 or 동적 쿼리는 QueryDSL 추천
    3. @Modifying 은 반드시 @Transactional 과 세트로 생각하기
    4. JOIN FETCH 는 N + 1 문제 해결 필수 무기
        (LazyInitializationException 방지)
        