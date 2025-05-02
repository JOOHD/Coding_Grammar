## QueryHelper 

    ★ 왜 QueryHelper가 필요했을까?

    복잡한 동적 쿼리를 작성하다 보면 코드가 길어지고, 중복이 발생하고, 가독성이 떨어지는 문제가 발생한다.

    ex) 
        조건이 5~6개 이상 들어가는 동적 검색 기능을 작성하게 되면...

        BooleanBuilder builder = new BooleanBuilder();

        if (조건1) {
            builder.and(조건1);
        }
        if (조건2) {
            builder.and(조건2);
        }
        if (조건3) {
            builder.or(조건3);
        }

        이런 식으로 계속 반복되는 패턴이 생기고, 추가 조건이 생기면 계속 수정해야 한다.
        게다가 다른 서비스에서도 유사한 조건을 재사용하고 싶을 때 재사용이 어렵고 중복 로직이 생기기 쉽다.

    ★ 어떤 기능이 문제였나?

    - 동적 검색에서 BooleanBuilder가 반복되며 가독성이 떨어지고 유지보수가  어려워진다.
    - 각 서비스 레이어나 리포지토리에 조건을 직접 작성하다 보니 쿼리 조건을 재사용하기가 힘들다.
    - 조건이 많아지면 누락이나 버그 발생 가능성도 증가한다.

    ★ QueryHelper 

    - QueryHelper 클래스는 이런 중복과 가독성 문제를 해결하기 위해 만들었다.
    쿼리 조건을 공통 유틸성 클래스로 분리해서

        - 유지보수를 쉽게 하고
        - 동적 조건 쿼리를 효율적으로 만들기 위함이다.   

### DB data

| id  | 상품명           | 카테고리ID | 가격   | 찜수 | 등록일               | 재고 |
|-----|------------------|------------|-------:|-----:|---------------------|-----:|
| 1   | 아메리카노       | 1          |  3,000 |   50 | 2024-12-01 10:00    |  100 |
| 2   | 카페라떼         | 1          |  4,500 |  100 | 2024-12-05 14:00    |    0 |
| 3   | 녹차프라푸치노   | 2          |  6,000 |   30 | 2025-01-01 09:00    |    5 |
| 4   | 딸기라떼         | 1          |  5,500 |   70 | 2025-02-01 11:00    |   50 |


### QueryHelper code
    @Component
    @RequiredArgsConstructor
    public class ProductQueryHelper {

        /**
         * 상품 검색 메서드
         * 조건에 따라 동적 검색 + 정렬 + 페이징
         */

        private final JPAQueryFactory queryFactory;

        // 상품 목록 조회 (동적 검색 + 정렬 + 페이징)
        public List<Product> searchProducts(ProductSearchCondition condition, Pageable pageable) {
            QProduct product = QProduct.product;

            return queryFactory
                    .selectFrom(product)
                    .where(buildWhere(condition, product)) // 조건에 따라 WHERE 절 생성
                    .orderBy(sortOrder(condition.getOrderBy(), product)) // 정렬
                    .offset(pageable.getOffset())  // 몇 번째부터
                    .limit(pageable.getPageSize()) // 몇 개 가져올지
                    .fetch();
        }

        /**
         * 동적 WHERE 절을 생성하는 메서드
         */
        private BooleanBuilder buildWhere(ProductSearchCondition condition, QProduct product) {
            BooleanBuilder builder = new BooleanBuilder();

            // 상품명 키워드 검색 (예: '라떼')
            if (condition.getKeyword() != null && !condition.getKeyword().isEmpty()) {
                builder.and(product.productName.containsIgnoreCase(condition.getKeyword()));
            }

            // 카테고리 ID로 필터링 (예: 1번 카테고리)
            if (condition.getCategoryId() != null) {
                builder.and(product.productManagements.any().category.categoryId.eq(condition.getCategoryId()));
            }

            // 최소 가격 이상 검색
            if (condition.getMinPrice() != null) {
                builder.and(product.price.goe(condition.getMinPrice()));
            }

            // 최대 가격 이하 검색
            if (condition.getMaxPrice() != null) {
                builder.and(product.price.loe(condition.getMaxPrice()));
            }

            return builder;
        }

        /**
         * 동적 정렬 처리
         * latest(최신순), priceAsc(가격 낮은 순), priceDesc(가격 높은 순), popular(인기순)
         */
        private OrderSpecifier<?> sortOrder(String orderBy, QProduct product) {
            if (orderBy == null) {
                return product.createdAt.desc(); // 기본 정렬: 최신순
            }

            switch (orderBy) {
            case "latest";
                return product.createdAt.desc(); // 최신순 (내림)
            case "priceAsc";
                return product.price.asc();  // 높은 가격순 (올림)
            case "priceDesc";
                return product.price.desc(); // 낮은 가격순 (내림)
            case "popular";
                return product.wishListCount.desc(); // 찜 개수 많은 순으로 정렬
            default:
                return product.createdAt.desc(); // 최신순 (기본 값)
            }
        }
    }

### 서비스 클래스에서 호출

    ProductSearchCondition condition = new ProductSearchCondition();
    condition.setKeyword("라떼");    // 상품명에 "라떼"가 들어간 것만 검색
    condition.setCategoryId(1L);     // 카테고리 ID가 1번인 것만 검색
    condition.setMinPrice(4000);     // 가격 4000 이상
    condition.setOrderBy("popular"); // 찜 개수 많은 순으로 정렬

    Pageable pageable = PageRequest.of(0, 10); // 1페이지, 10개씩

    List<Product> productList = productQueryHelper.searchProducts(condtion, pageable);

    1. Keyword = "라떼"
        → 아메리카노(X), 카페라떼(O), 딸기라떼(O)

    2. CategoryId = 1
        → 카페라떼(O), 딸기라떼(O)

    3. MinPrice = 4000
        → 카페라떼(O, 4500), 딸기라떼(O, 5500)

    4. 정렬 popular(찜 순)
        → 카페라떼(찜수 100) → 딸기라떼(찜수 70)

    최종 결과
    → 카페라떼, 딸기라떼 순으로 리턴!

### QueryHelper 사용 전/후 비교

    사용 전
    - 조건ㅇ르 서비스/리포지토리 레이어에 전부 작성
    - BooleanBuilder 가 여러 번 중복됨
    - 가독성 저하, 유지보수 불편
  
    사용 후
    - QueryHelper 가 조건을 만들어주니까 깔끔
    - 조건 추가가 편리하고 코드 재사용 가능
    - 가독성 개선 + 유지보수 편리

    해결 방법 정리
    - BooleanBuilder 로직 분리 : 공통 클래스(QueryHelper)로 분리하여 중복 제거 및 유지보수 개선
    - 재사용성 강화 : 공통 메서드로 만들어 놓고 여러 쿼리에서 재사용 가능
    - 가독성 향상 : 서비스/리포지토리 레이어는 핵심 로직만 담고, 조건은 헬퍼에서 처리

### 정리
    "QueryHelper 는 동적 검색 조건을 관리하고 반복을 줄이기 위해 만들어졌으며, 유지보수성과 재사용성을 높이기 위한 핵심 유틸 클래스 목적으로 개발자가 만든 클래스이다."    

