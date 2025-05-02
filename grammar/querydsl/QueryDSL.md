## QueryDSL 정의 & 메서드 & 예제

### Q클래스
    - QueryDSL 이 Product Entity 기반으로 자동 생성해준 클래스
    - QProduct 클래스는 타입 안전한 쿼리를 작성 도와준다.
    - 클래스 내부를 보면, 필드들이 전부 QueryDSL 의 Path 타입으로 되어있다.

    ex)
        public final StringPath productName = createString("productName");
        public final NumberPath<Integer> price = createNumber("price", Integer.class);

        즉, productName 이나 price 같은 필드에 대해 QueryDSL 이 제공하는 메서드(eq, contains, goe 등) 쿼리 작성할 수 있게 해준다.

### Q클래스 용도
    1. 타입 안전한 쿼리 작성
        QProduct.product.category.eq(...) 처럼 IDE 자동완성과 컴파일 타임 오류 확인 가능
    2. 직관적인 join/filter 가능
        복잡한 join 이나 where 조건을 객체처럼 다룰 수 있음
    3. JPA Criteria 보다 간결하고 유지보수 편함
        코드 가독성도 좋고 실수도 줄어듦

### Q클래스와 관련된 주의사항
    - 앤티티 변경 시, 반드시 재생성 해야 함
    - 보통 build/generated/querydsl 경로에 위치
    - build.gradle 에 sourceSets 로 등록해 IDE에서 인식되도록 해야 함.                

### QeuryDSL 

    예시로 Product Entity 클래스에 QueryDSL 적용하게 되면, @Entity 를 기반으로 타입 세이프한 쿼리 작성을 돕는 메타 모델 클래스이다.
    
    ex)

        QProduct product = QProduct.product;

        queryFactory.selectFrom(product)
                    .where(product.name.eq("아메리카노"))
                    .fetch();

        - QProduct.product: Product Entity에 대한 QueryDSL 객체
        - product.name.eq(...): 타입 안정성 보장! 컴파일 타임에 검증 가능

### QueryDSL 사용이유

    1. 문자열 오타 방지 ("category" → product.category)
    2. 컴파일 타임 타입 체크 가능
    3. IDE 자동 완성으로 실수 줄임 
 
### QueryDSL 메서드

    1. eq(), ne() (같다 / 같지 않다)
    
        // 가격이 10000원인 상품
        product.price.eq(10000);

        // 상품 타입이 FOOD가 아닌 것
        product.productType.ne(ProductType.FOOD);

    2. gt(), goe(), lt(), loe() (크다 / 크거나 같다 / 작다 / 작거나 같다)

        // 가격이 10000원보다 큰 상품
        product.price.gt(10000);

        // 가격이 5000원 이상인 상품
        product.price.goe(5000);

        // 가격이 20000원 이하인 상품
        product.price.loe(20000);

    3. in(), notIn() (포함 / 포함하지 않음)     

        // 특정 카테고리에 속하는 상품
        product.category.categoryId.in(1L, 2L, 3L);

        // 추천 상품이 아닌 것들
        product.isRecommend.notIn(true);

    4. like(), contains(), startsWith(), endsWith() (문자열 검색)

        // 상품명에 "커피"가 포함된 상품
        product.productName.contains("커피");

        // 상품명이 "아메리카노"로 시작하는 상품
        product.productName.startsWith("아메리카노");

        // 상품명이 "에디션"으로 끝나는 상품
        product.productName.endsWith("에디션");

        // 상품명에 특정 키워드가 LIKE 검색 (SQL LIKE '%키워드%')
        product.productName.like("%한정판%");    

    5. any() / all() (컬렉션 조건)
    
        // 어떤 productManagement의 category가 1이면 true
        product.productManagements.any().category.categoryId.eq(1);

        // 모든 productManagement의 category가 1이면 true (자주 안 씀)
        product.productManagements.all().category.categoryId.eq(1);    

    6. exists() (서브쿼리 존재 여부)

        // 서브쿼리로 존재 여부 체크
        JPAQuery<ProductReview> subQuery = new JPAQuery<>(entityManager);
        QProductReivew review = QProductReview.productReview;

        BooleanExpress hasReview = subQuery
            .from(review)
            .where(review.product.eq(product))
            .exists();
    
    7. BooleanBuilder 동적 조건 조합
        
        // BooleanBuilder 를 이용한 다중 필터 검색 (동적 쿼리)
        BooleanBuilder builder = new BooleanBuilder();

        // 조건: 최소 가격 이상
        if (cond.getMinPrice() != null) {
            builder.and(product.price.goe(cond.getMinPrice()));
        }

        // 조건: 최대 가격 이하
        if (cond.getMaxPrice() != null) {
            builder.and(product.price.loe(cond.getMaxPrice()));
        }

        // 조건: 상품명 검색
        if (StringUtils.hasText(cond.getProductName())) {
            builder.and(product.productName.contains(cond.getProductName()));
        }

        // 조건: 추천 상품 여부
        if (cond.getIsRecommend() != null) {
            builder.and(product.isRecommend.eq(cond.getIsRecommend()));
        }

        // 조건: 특정 카테고리
        if (cond.getCategoryId() != null) {
            builder.and(product.category.categoryId.eq(cond.getCategoryId()));
        }

        return queryFactory
                .selectFrom(product)
                .where(builder)   // 동적 조건 조합 적용!
                .fetch();

        ★ 핵심
        - BooleanBuilder는 and() / or()로 조건을 조합할 수 있어.
        - if문을 통해 조건이 null인지 체크해서 필터링 여부 결정!
        - 동적 검색 API 만들 때 자주 씀!
            → 검색 조건이 10개가 돼도 깔끔하게 관리 가능!


    8. orderBy() 정렬

        query.select(product)
            .from(product)
            .orderBy(
                product.price.asc(),                // 가격 오름차순
                product.wishListCount.desc()        // 위시리스트 내림차순
            );     

    9. groupBy() / having() (그룹화/집계 조건)        

        List<Tuple> results = queryFactory
            .select(
                product.category.categoryId,
                product.price.avg()
            )
            .from(product)
            .groupBy(product.category.categoryId)
            .having(product.price.avg().gt(10000)) // 평균 가격 10,000원 초과 카테고리만
            .fetch();

        categoryId	productName	price
        1	아메리카노	9,000
        1	카페라떼	11,000
        2	에스프레소	8,000
        2	콜드브루	7,000
        3	말차라떼	15,000
        3	바닐라라떼	14,000    

        ● groupBy + avg() 계산
        categoryId 1번: (9000 + 11000) / 2 = 10,000
        categoryId 2번: (8000 + 7000) / 2 = 7,500
        categoryId 3번: (15000 + 14000) / 2 = 14,500

        ● having 조건
        avg(price) > 10000
        → 조건 충족하는 카테고리만 리턴!

        categoryId = 3, price =	14,500

        ● 요약
        - groupBy()는 SQL 그대로 그룹을 묶고
        - having()은 그룹핑된 집합 조건 필터링
        - QueryHelper는 개발자가 만든 커스텀 헬퍼 클래스
            - 동적 조건 관리
            - 정렬/페이징 관리 등을 위해 자주 만들어 사용함!


    10. etch(), fetchOne(), fetchFirst() 조회

        // 리스트 조회
        List<Product> products = query.selectFrom(product)
            .fetch();

        // 단일 조회
        Product productOne = query.selectFrom(product)
            .where(product.productId.eq(1L))
            .fetchOne();

        // 첫 번째만 조회 (LIMIT 1)
        Product firstProduct = query.selectFrom(product)
            .fetchFirst();

    11. join(), leftJoin(), fetchJoin()

        // INNER JOIN
        query.selectFrom(product)
            .join(product.category, category)
            .where(category.categoryId.eq(1L))
            .fetch();

        // LEFT JOIN
        query.selectFrom(product)
            .leftJoin(product.category, category)
            .fetch();

        // FETCH JOIN (엔티티 관계 같이 조회)
        query.selectFrom(product)
            .join(product.category, category).fetchJoin()
            .fetch();      

        - 성능 개선할 때 JPQL fetch join or QueryDSL fetchJoin() 꼭 사용함!          
        - fetchJoin()을 사용하면 지연 로딩 프록시 없이 즉시 객체에 값이 들어있음!
          - fetchJoin()은 한 번에 조인해서 가져오는 기능!
        - N+1 문제를 해결하는 대표적인 방식이 fetchJoin()
          - 조회 시 쿼리 1번, 연관 객체 당 추가로 쿼리 N번 → 총 1 + N번 실행됨
    
        ★ 실무 팁
        - 무분별한 fetchJoin() 사용은 지양!
            - 조인한 결과가 데이터가 많을 경우 → 데이터 중복 / 메모리 부하
            - 상황에 맞게 @BatchSize, EntityGraph도 고민해보기!

    12. case() (조건에 따라 다른 값을 반환)    

        // 상품이 할인 중인지, 정상가인지, 추천 상품인지 등을 조건에 따라 문자열 반환하고 싶을 때

        public List<Tuple> getProductWithDiscountStatus() {
            return queryFactory
                .select(
                    product.productId,
                    product.productName,
                    new CaseBuilder()
                        .when(product.isDiscount.eq(true))
                            .then("할인중")
                        .when(product.discountRate.goe(50))
                            .then("반값세일")
                        .otherwise("정상가")
                )
                .from(product)
                .fetch();
        }

        - CaseBuilder()는 SQL CASE WHEN THEN ELSE와 동일한 역할
        - 다양한 상태값이나 라벨링을 쉽게 표현 가능
        (할인 상태, 배송 상태, 회원 등급 등)

    ★ 요약
    - BooleanBuilder	동적 조건 조합 (and(), or())	다중 필터 검색, 동적 where절
    - fetchJoin()	조인 후 함께 가져오기	N+1 문제 해결, 성능 최적화
    - CaseBuilder()	조건에 따라 값 반환	상태값 표시, 라벨링, 비즈니스 로직 가시화


### Q클래스 생성 조건

    1. @Entity 가 선언된 클래스가 있어야 된다.
    2. annotationProcessor 가 querydsl-apt 를 실행하면서 빌드 시점에 자동으로 Q클래스를 만들어준다.
    3. /src/main/generated/QProduct.java

    만약 Q클래스가 생성되지 않는다면, 몇가지 예시를 보여주겠다.

    4. Cannot resolve symbol 'QProduct'	-> Q 클래스가 아예 없음	
        해결 : Gradle 설정 or Entity 문제

    5. Q 클래스는 있는데 인식 안 됨	-> 인텔리제이 설정 문제	
        해결 : Mark Directory as ➜ Generated Sources Root

    3.querydsl-apt가 실행 안 됨	-> Gradle 세팅 오류	
        해결 : annotationProcessor 확인       

### Gradle 설정

    // querydsl
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jakarta"    

### QueryDSL 적용1

    // 1. QProduct
    public class QProduct extends EntityPathBase<Product> {

        public static final QProduct product = new QProduct("product");

        public final StringPath name = createString("name");
        public final NumberPath<Integer> price = createNumber("price", Integer.class);
        public final StringPath category = createString("category");
    }

    // 2. DB data
    1	아메리카노	3000	음료
    2	카페라떼	4000	음료
    3	딸기케이크	5500	디저트
    4	바닐라라떼	4500	음료

    // 3. 데이터 조회
    QProduct product = QProduct.product;

    List<Product> products = queryFactory
        .selectFrom(product)
        .fetch();

    // 4. 결과
    [
        Product(id=1, name=아메리카노, price=3000, category=음료),
        Product(id=2, name=카페라떼, price=4000, category=음료),
        Product(id=3, name=딸기케이크, price=5500, category=디저트),
        Product(id=4, name=바닐라라떼, price=4500, category=음료)
    ]

### QueryDSL 적용2 

    @Entity
	public class Product {

    		@Id
    		@GeneratedValue(strategy = GenerationType.IDENTITY)
    		private Long productId;
    		private String productName;
    		private Integer price;
    		private Integer discountRate;
    		private Boolean isDiscount;
    		private Boolean isRecommend;
    		private LocalDateTime createdAt;
    		private Long wishListCount;

    		// ... 생략 ...
	}
	
   	 // QProduct는 이렇게 자동 생성됩니다 
   	 public class QProduct extends EntityPathBase<Product> {
    		public static final QProduct product = new QProduct("product");
    		public final StringPath productName = createString("productName");
    		public final NumberPath<Integer> price = createNumber("price", Integer.class);
    		public final BooleanPath isDiscount = createBoolean("isDiscount");
    		public final DateTimePath<LocalDateTime> createdAt = createDateTime("createdAt", LocalDateTime.class);
    		public final NumberPath<Long> wishListCount = createNumber("wishListCount", Long.class);
    		// ... 생략 ...
   	 }

	-> EntityPathBase<T>
		- Q 클래스들이 EntityPathBase<T> 를 상속 받는다.
		- @Entity 클래스를 만들고, QueryDSL 을 빌드하면 자동으로 생성되는 게 QProduct 같은 Q 클래스이다.
		- EntityPathBase 는 QueryDSL 에서 사용하는 경로 표현식의 기본 클래스
		- 쉽게 말하면, QProduct 는 EntityPathBase<Product> 를 상속하면서, Product 엔티티에 대한 "타입 안전한 경로(path)" 를 제공한다.
		ex) 
			public class QProduct extends EntityPathBase<Product>
	
	-> OrderSpecifier<?>
		- QeuryDSL 에서 order by 구문을 작성할 때 사용하는 객체.
			return product.price.asc(); // -> 이게 바로 OrderSpecifier<?> 를 반환.
			-> product.price.asc() 를 하면 OrderSpecifier 를 생성하고, 이를 통해 정렬 조건을 명시. 

	// QueryDSL 적용
	QProduct product = QProduct.product;

	// 1. 정렬 기준으로 가져온다. (인기순)
	OrderSpecifier<?> orderSpecifier = ProductQueryHelper.getOrderSpecifier(OrderBy.POPULAR, product);
	// -> product.wishListCount.desc()

	// 2. 필터링 조건을 만든다. (추천 상품 + 카테고리ID 1 + 키워드 '티셔츠')
	BooleanBuilder filterBuilder = ProductQueryHelper.createdFilterBuilder(
		Condition.RECOMMEND, // 추천 상품 필터
		1L, 				   // 카테고리 필터 (카테고리 id가 1)
	        "티셔츠", 			   // 검색 키워드
		product			   
	);	
	
	// 3. QueryDSL 쿼리 생성 (JPAQueryFactory 사용)
	List<Product> products = queryFactory
			.selectFrom(product)
			.where(filterBuilder)
			.orderBy(orderSpecifier)
			.fetch();

	1.정렬 기준

		case POPULAR:
    			return product.wishListCount.desc();
		-> wishListCount 기준으로 내림차순 정렬!
			청바지(100) → 티셔츠(50) → 운동화(10)

	2. 필터 조건

		- 추천 상품만 가져와야 한다
			product.isRecommend.isTrue()
		- 카테고리 ID가 1이거나 부모 카테고리 ID가 1인 것
			product.productManagements.any().category.categoryId.eq(1)
		- 키워드가 티셔츠에 포함
			product.productName.containsIgnoreCase("티셔츠")
		-> AND 조건으로 묶임!

### 흐름 정리

    1. @Entity Product
            ↓ (빌드 시)
    2. querydsl-apt가 자동 생성
            ↓
    3. QProduct 클래스 생성
            ↓
    4. QueryFactory에서 QProduct로 타입 안전한 쿼리 작성
            ↓
    5. JPQL처럼 문자열 아님 → 컴파일 타임에 검증됨
            ↓
    6. SQL 변환 후 실행 & 결과 반환


