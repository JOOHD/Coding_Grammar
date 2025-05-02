## N + 1 문제

    ★ N + 1 문제란?

        - "N + 1 문제란, 메인 쿼리 1번 + 추가 쿼리가 N번 발생해서 총 N + 1번의 쿼리가 실행되는 문제입니다."

    ★ N + 1 문제는 언제 발생할까?

    **지연 로딩(LAZY LOADING)**을 사용하는 환경에서 연관된 엔티티를 반복해서 조회할 때 발생합니다.

    대부분 ORM(JPA, Hibernate 등)을 사용해서 객체 지향적으로 데이터를 다룰 때, 
    연관 관계가 설정된 엔티티를 조회하고 그 안에 있는 다른 엔티티에 접근하게 되면, 
    프록시 객체로 인해 DB에 추가 쿼리가 발생합니다.
    

    ex)
        주문(Order) 엔티티와 주문상품(OrderItem) 엔티티가 있다고 해보자.
        Order : OrderItem은 1:N 관계야 (하나의 주문은 여러 상품을 가짐).

        // 예시 엔티티
        @Entity
        public class Order {
            @Id
            private Long id;

            @OneToMany(mappedBy = "order")
            private List<OrderItem> orderItems;
        }

        @Entity
        public class OrderItem {
            @Id
            private Long id;

            @ManyToOne
            private Order order;
        }

        // 쿼리 예제
        List<Order> orders = orderRepository.findAll();
        for (Order order : orders) {
            List<OrderItem> items = order.getOrderItems();
        }

    ★ 어떤 일이 벌어지냐면?

        1. findAll() 실행 -> Order 10개 가져옴
            → SELECT * FROM orders; (1번 쿼리)

        2. 각 Order마다 getOrderItems() 호출할 때마다
            → SELECT * FROM order_items WHERE order_id = ?;
            → 10번 추가 쿼리 실행 (N개)

        그래서 총 1 + N = 11번 쿼리가 나가게 된다. 

    ★ N + 1 문제 생기는 이유

        - 연관 앤티티는 기본적으로 LAZY(지연로딩)    
        - order.getOrderItems() 호출하는 순간마다
            -> DB에 쿼리 날려서 가져오기 때문

    ★ 해결 방법

            방법	                    설명
        fetch join	       처음 쿼리에서 연관 엔티티도 한 번에 가져옴
        @EntityGraph	   특정 관계를 EAGER하게 가져오도록 최적화
        배치 사이즈 조정	IN 쿼리로 묶어서 최소한으로 쿼리 줄이기    



### fetchJoin() 으로 N + 1 문제 해결        

    queryFactory : 쿼리문을 자바 코드로 작성할 수 있게 도와주는 쿼리 빌더지

    // 쿼리 dsl
    List<Order> orders = queryFactory
        .selectFrom(order)
        .join(order.orderItems, orderItem).fetchJoin()
        .fetch();!

    - 기본 조회 
    Order 10개 조회   => 1번
    OrderItem N개 조회 => N번

    - fetch join
    Order + OrderItem 한 번에 조회 => 1번으로 끝!

    - N+1 디버깅 팁
    → hibernate.show_sql=true 켜서 쿼리 로그 보면 쿼리 수를 확인할 수 있음.
    → 예상보다 쿼리가 많이 나가면 대부분 N+1 문제야!

### fetch join 전/후 비교    

    @Entity
    public class Order {
        @Id @GeneratedValue
        private Long id;

        private String customerName;

        @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
        private List<OrderItem> orderItems = new ArrayList<>();
    }

    @Entity
    public class OrderItem {
        @Id @GeneratedValue
        private Long id;

        private String itemName;

        private int price;

        @ManyToOne(fetch = FetchType.LAZY)
        private Order order;
    }

### 1. fetch join 안 한 경우 (N + 1 발생)    

    List<Order> orders = queryFactory
        .selectFrom(order)
        .fetch();

    for (Order order : orders) {
        log.info("Order: " + order.getCustomerName());

        // 지연로딩이라 여기서 쿼리 추가 발생
        for (OrderItem item : order.getOrderItems()) {
            log.info(" -> Item: " + item.getItemName());
        }
    }

    ★ 위에 코드가 실행이 되면, 실행되는 쿼리문

| 구분         | 설명                                              |
|--------------|---------------------------------------------------|
| `o`          | `orders` 테이블의 alias → 직관적으로 `o` 사용    |
| `oi`         | `order_items` 테이블 alias → 직관적으로 `oi` 사용 |
| `AS order_id`| 컬럼 alias 추가해서 명확하게                      |
| `WHERE 조건` | 각 주문에 해당하는 주문 아이템 조회 조건           |
    
    SELECT
        o.id AS order_id,
        o.customer_name
    FROM orders o;

    SELECT
        oi.id AS order_item_id,
        oi.item_name,
        oi.price,
        oi.order_id
    FROM order_items oi
    WHERE oi.order_id = 1;

    SELECT
        oi.id AS order_item_id,
        oi.item_name,
        oi.price,
        oi.order_id
    FROM order_items oi
    WHERE oi.order_id = 2;

    SELECT
        oi.id AS order_item_id,
        oi.item_name,
        oi.price,
        oi.order_id
    FROM order_items oi
    WHERE oi.order_id = 3; 


### 2. fetch join 한 경우 (N + 1 해결)

    List<Order> orders = queryFactory
        .selectFrom(order)
        .join(order.orderItems, orderItem).fetchJoin() // 추가
        .fetch();

    for (Order order : orders) {
        log.info("Order: " + order.getCustomerName());

        for (OrderItem item : order.getOrderItems()) {
            log.info(" -> Item: " + item.getItemName());
        }
    }

    ★ 위에 코드가 실행이 되면, 실행되는 쿼리문

    SELECT
        o1_0.id, o1_0.customer_name,
        oi1_0.id, oi1_0.item_name, oi1_0.price, oi1_0.order_id 
    FROM order o1_0
    JOIN order_item oi1_0 on o1_0.id=oi1_0.order_id

    → Order + OrderItem 한 번에 조인해서 가져옴!
    → 딱 1번 쿼리로 끝!
    → N+1 문제 해결

    ❌ fetch join 없음	Order 조회 + 각 OrderItem 별도 조회 (N+1 문제)
    ✅ fetch join 있음	한 번에 join 해서 가져와서 쿼리 1번으로 해결

### 추가 설명

    1. fetch join은 select 절에 반드시 주의
    → 중복 데이터 나올 수 있으니까 distinct도 자주 붙임!

    queryFactory
        .selectFrom(order)
        .distinct()
        .join(order.orderItems, orderItem).fetchJoin()
        .fetch();