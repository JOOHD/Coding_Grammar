##  DTO → Entity 변환 정적 팩토리 메서드(from)로 분리

### 구조

    - CartItemDto는 사용자 요청을 담음

    - CartItem Entity는 from(dto, product)라는 정적 팩토리 메서드를 통해 생성

    - CartService는 from()을 호출하여 변환

### 1. CartItemDto class (사용자 입력 데이터)    

    public class CartItemDto {
        private Long productId;
        private int quantity;

        public CartItemDto(Long productId, int quantity) {
            this.productId = productId;
            this.quantity = quantity;
        }

        public Long getProductId() {
            return productId;
        }

        public int getQuantity() {
            return quantity;
        }
    }

### 2. CartItem class (Entity + 정적 팩토리 메서드)

    @Entity
    @Getter
    @NoArgsConstructor(access = AccessLevel.PROTECTED)
    public class CartItem {

        @Id @GeneratedValue
        private Long id;

        @ManyToOne(fetch = FetchType.LAZY)
        private Product product;

        private int quantity;

        @Builder
        private CartItem(Product product, int quantity) {
            this.product = product;
            this.quantity = quantity;
        }

        // ✅ 정적 팩토리 메서드 – DTO -> Entity
        public static CartItem from(CartItemDto dto, Product product) {
            return CartItem.builder()
                    .product(product)
                    .quantity(dto.getQuantity())
                    .build();
        }
    }

### 3. CartService class  (Entity 변환)    

    @Service
    @RequiredArgsConstructor
    public class CartService {

        private final ProductRepository productRepository;
        private final CartItemRepository cartItemRepository;

        public CartItem addCartItem(CartItemDto dto) {
            Product product = productRepository.findById(dto.getProductId())
                    .orElseThrow(() -> new IllegalArgumentException("상품을 찾을 수 없습니다."));

            // ✅ DTO -> Entity 변환은 엔티티에게 위임
            CartItem cartItem = CartItem.from(dto, product);

            return cartItemRepository.save(cartItem);
        }

        public CartItem changeQuantity(CartItem oldItem, int newQuantity) {
            CartItem updatedItem = CartItem.builder()
                    .product(oldItem.getProduct())
                    .quantity(newQuantity)
                    .build();

            cartItemRepository.delete(oldItem);
            return cartItemRepository.save(updatedItem);
        }
    }

### 서비스 계층에서 수량을 변경할 때, 처리

    ● from() 정적
    
        설명
        - 보통 Entity 클래스에 public static CartItem from(dto, etc) 형태로 사용

        장점
        - 책임이 명확 (entity 에서 변환), 일관된 생성 방식
        
        단점
        - 매번 객체 새로 생성이므로, 업데이트보단 새 생성에 적합

    ● static 내부 Builder 클래스

        설명
        - DTO/Entity 내부에 Builder 를 정의해서, .builder().field() 식 사용
        
        장점
        - 가독성이 높다. (필드 많을 때, 명확하게 초기화)

        단점
        - 단순 수정일 때는 과할수 있다.

### 핵심 비교: "수량 변경" 상황에서 어떤 게 더 좋을까?       

    1. setQuantity() 사용
    
    ✔️ 장점: 단순, 간편, 성능 부담 없음
    ❌ 단점: 객체 불변성 깨짐

    ex) cartItem.setQuantity(newQuantity);

    추천 조건: 단순한 필드 하나 업데이트, 로직 간결함 유지가 중요할 때

    2. from() 또는 builder()를 사용해 새 객체 생성

    ✔️ 장점: 불변 객체 지향, 변경 이력 추적 용이
    ❌ 단점: DB에서 삭제 후 새로 저장해야 하므로 비용 ↑

    ex)
        CartItem updated = CartItem.builder()
        .product(cartItem.getProduct())
        .quantity(newQuantity)
        .build();

    ex) CartItem updated = CartItem.from(dto, product);

    추천 조건: 엔티티가 변경 불가능해야 하는 도메인 (ex. 주문, 결제 이력), 새로 생성이 더 자연스러울 때

### 추가 팁

    - from()은 표준화된 객체 생성을 원할 때 유용

    - builder()는 선택적 필드가 많거나, DTO/Entity에 필드가 많을 때 유리

    - static 내부 클래스로 직접 빌더를 구현하는 건 롬복의 @Builder를 쓰지 않을 때 주로 사용    