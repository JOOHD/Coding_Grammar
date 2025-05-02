## 본문

### Setter method

    객체의 필드 값을 수정할 수 있게 해주는 메서드이다.
    보통 set 이라는 접두사와 함께 필드명을 이어 붙여서 메서드 이름을 만든다.
    setter 메서드는 캡슐화와 데이터 보호의 핵심 요소로, 객체 내부 데이터에 대해 제어된 접근을 제공하는데, 이를 통해 외부 코드에서 직접 필드에 접근하지 않고도 객체의 상태를 변경할 수 있다.

    public class Person {

        private String name;  // private로 직접 접근 제한
        private int age;

        // Setter method for name
        public void setName(String name) {
            this.name = name;  // 외부에서 이름을 변경할 수 있는 방법을 제공
        }

        // Setter method for age
        public void setAge(int age) {
            if (age > 0) {   // 나이 값에 대한 간단한 검증 로직 포함 가능
                this.age = age;
            }
        }
    }

### Setter 메서드를 다시 정리하는 이유는?    

    내가 클라이언트에서 서버에 데이터를 넘길 때, form 태그를 이용한 데이터 전송 방식을 자주 사용하고 선호 한다. 그런데 간혹, controller, service 클래스에서 object.set(); 방식이 나온다.

    그래서 왜 이미 넘어온 데이터를 받아서 DB에 넘기면 되는데, 중간에 set 메서드가 나오는 것일까라는 의문이 들었다.

### 예시 코드

    1. 상태 변경 및 추적
    
    - 주문 상태를 관리하는 경우, 주문이 생성되고 나서 서비스 로직에서 특정 조건에 따라 상태를 변경하는 경우.

    public class Order {
        private Long id;
        private String status; // 주문 상태 : "결제 대기", "결제 완료", "배송 준비 중"
        
        public void setStatus(String status) {
            this.status = status;
        }

        public String getStatus() {
            return status;
        }
    }

    @Service
    public class OrderService {
        public void completePayment(Order order) {
            // 결제 완료 처리 후 상태 변경
            order.setStatus("결제 완료");
        }
    }

    - 여기서 completePayment 메서드는 결제 로직이 끝난 후 주문의 상태를 "결제 완료"로 변경한다.
  
    2. 다양한 비즈니스 규칙 반영
    
    public class Prodcut {
        private double price;
        private double discountPrice;

        public Product(double price) {
            this.price = price;
        }

        public void setDiscountPrice(double discountPrice) {
            this.discountPrice = discountPrice;
        }

        public double getDiscountPrice() {
            return discountPrice;
        }
    }

    @Service
    public class ProductService {
        public void applyDiscount(Product product, double discountPercentage) {
            double discountPrice = product.getPrice() * (1 - discountPercentage / 100);
            product.setDiscountPrice(discountPrice); // 할인 적용 
        }
    }

    - 이 경우, applyDiscount 메서드는 비즈니스 로직에 따라 제품의 할인된 가격을 계산하고 설정 한다.
  
    3. 동적 데이터 업데이트

    처음 객체가 생성될 때, 데이터가 모두 정해지지 않았고, 이후 정보가 추가될 수 있는 경우이다.

    public class UserPrice {
        private String name;
        private String address;

        public UserProfile(String name) {
            this.name = name;
        }

        public void setAddress(String address) {
            this.address = address;
        }

        public void getAddress() {
            return address;
        }
    }

    @Service
    public class UserService {
        public void updateAddress(UserProfile userProfile, String newAddress) {
            // 사용자 주소 정보가 확보되면 설정
            userProfile.setAddress(newAddress);
        }
    }

### 예시 코드 (getter/setter)    

    @Data
    @NoArgsConstructor
    public class ProductDTO {
        private double price;

        public ProductDTO(double price) {
            this.price = price; // 생성자를 통해 가격 초기화
        }
    }

    ● Setter
    - setPrice(double price) : ProductDTO의 price를 설정하는 메서드이다.
    - ProductDTO 객체의 상태를 변경할 수 있게 해준다. 
    - DTO 클래스에서 setter 메서드를 사용하면 가격을 외부에서 쉽게 수정할 수 있다.

    @RestController
    @RequiredArgsConsstructor
    @RequestMapping("/products")
    public class ProductController {
        private ProductDTO productDTO;

        @PostMapping
        public String crerateProduct(@RequestBody ProductDTO productDTO) {
            if (isValidPrice(productDTO.getPrice())) { // DTO에서 가격을 가져옴
                productDTO = productDTO; // DTO를 인스턴스 변수에 저장.
                return "Product created with price: $" + productDTO.getPrice(); // 가격 반환
            } else {
                return "Invalid initial price.";
            }
        }

        @PutMapping
        public String updateProductPrice(@RequestBody double newPrice) {
            if (isValidPrice(newPrice)) { // 기존 DTO의 price를 새로운 
            가격으로 업데이트
                if (productDTO != null) {
                    productDTO.setPrice(newPrice); // 가격 업데이트를 위해 새로운 DTO 생성
                    return "Update price: $" + productDTO.getPrice(); // 업데이트된 가격 반환
                } else {
                    return "No product found to update.";
            } else {
                return "Invalid new price.";
            }
        } 

        // 가격 유효성 검증 로직
        private boolean isValidPrice(double price) {
            return price >= 0;
        }
    }

    ● 설명

    - Getter
        - productDTO.getPrice() : ProductDTO의 price를 읽기 위해 호출된다.
        - 가격 유효성 검증과 결과 반환에 사용된다.

    - Setter
        - productDTO.setPrice(newPrice) : 새로운 가격으로 price를 업데이트 하기 위해 사용된다.
        - DTO 객체의 상태를 변경할 수 있도록 한다.         

### 예시 코드 (project) 

    ● ItemDTO class

    @Builder
    @AllArgsConstructor
    @NoArgsConstructor
    @Setter
    @Getter
    @Entity
    public class Item {

        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private int id;

        private String name;

        private String text; // 물건에 대한 상세설명

        private int price; // 가격

        private int count; // 판매 개수

        private int stock; // 재고

        private int isSoldout; // 상품 상태 (0 : 판매중 / 1 : 품절)

        @ManyToOne(fetch = FetchType.EAGER)
        @JoinColumn(name = "seller_id")
        private User seller; // 판매자 아이디

        @OneToMany(mappedBy = "item")
        private List<CartItem> cartItems = new ArrayList<>();
        
        private String imgName; // 이미지 파일명
        private String imgPath; // 이미지 조회 경로

        @DateTimeFormat(pattern = "yyyy-mm-dd")
        private LocalDate createDate; // 상품 등록 날짜

        @PrePersist // DB에 INSERT 되기 직전에 실행. 즉 DB에 값을 넣으면 자동으로 실행됨
        public void createDate() {
            this.createDate = LocalDate.now();
        }
    }

    ● ItemController class

    // 상품 수정 페이지 (admin)
    @GetMapping("/item/modify/{id})
    public String itemModifyForm(@PathVariable("id") Integer id, Model model, @AuthenticationPrincipal PrincipalDetails principalDetails) {
        if (principalDetails.getUser().getRole().equals("ROLE_SELLER")) {
            // 판매자
            User user = itemService.itemView(id).getSeller();
            // 상품을 올린 판매자 id와 현재 로그인 중인 판매자의 id가 같아야 한다.
            if (user.getId() == principalDetails.getUser().getId()) {
                model.addAttribute("item", itemService.itemView(id));
                model.addAttribute("user", principalDetails.getUser());

                return "/seller/itemModify";
            } else {
                return "redirect:/main";
            } 
        } else {
            // 일반 회원이면 거절 -> main
            return "redirect:/main";
        }
    }
    
    ● ItemService class

    // 상품 수정
    @Transactional
    public void itemModify(Item item, Integer id, MultipartFile imgFile) throws Exception {
        String projectPath = System.getProperty("user.dir") + "/src/main/resources/static/files";
        UUID uuid = UUID.randomUUID();
        String fileName = uuid + "_" + imgFile.getOriginalFilename();
        File saveFile = new File(projectPath, fileName);
        imgFile.transferTo(saveFile);

        Item update = itemRepository.findItemById(id);
            update.setName(item.getName());
            update.setText(item.getText());
            update.setPrice(item.getPrice());
            update.setStock(item.getStock());
            update.setIsSoldout(item.getIsSoldout());
            update.setImgName(fileName);
            update.setImgPath("/files/"+fileName);
        itemRepository.save(update);
    }

    그런데 위에 코드는 너무 하드 코딩이다. 리펙토링을 하여 간략하게 최소화 해보자.

### ver.1 - itemModify refactory (construct this pattern)  

    update.set(~); 이 부분을 dto 클래스에 매개변수가 있는 생성자에 재설정

    // Item 클래스에 추가
    public void update(Item item, String fileName) {
        this.name = item.getName();
        this.text = item.getText();
        this.price = item.getPrice();
        this.stock = item.getStock();
        this.isSoldout = item.getIsSoldout();
        this.imgName = fileName;
        this.imgPath = "/files/" + fileName;
    }

    // itemModify 메서드 리팩토링
    @Transactional
    public void itemModify(Item item, Integer id, MultipartFile imgFile) throws Exception {
        String projectPath = System.getProperty("user.dir") + "/src/main/resources/static/files";
        UUID uuid = UUID.randomUUID();
        String fileName = uuid + "_" + imgFile.getOriginalFilename();
        File saveFile = new File(projectPath, fileName);
        imgFile.transferTo(saveFile);

        Item update = itemRepository.findItemById(id);
        update.update(item, fileName); // 업데이트 메서드 호출
        itemRepository.save(update);
    }

### ver.2 - itemModify refactory (builder pattern)    

    // Item 클래스에 빌더 추가
    @Getter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public class Item {
        private String name;
        private String text;
        private Double price;
        private Integer stock;
        private Boolean isSoldout;
        private String imgName;
        private String imgPath;

        // 빌더를 통해서 객체를 업데이트하는 메서드
        public static Item from(Item item, String fileName) {
            return Item.builder()
                .name(item.getName())
                .text(item.getText())
                .price(item.getPrice())
                .stock(item.getStock())
                .isSoldout(item.getIsSoldout())
                .imgName(fileName)
                .imgPath("/files/" + fileName)
                .build();
        }
    }

    // itemModify 메서드 수정
    @Transactional
    public void itemModify(Item item, Integer id, MultipartFile imgFile) throws Exception {
        String projectPath = System.getProperty("user.dir") + "/src/main/resources/static/files";
        UUID uuid = UUID.randomUUID();
        String fileName = uuid + "_" + imgFile.getOriginalFilename();
        File saveFile = new File(projectPath, fileName);
        imgFile.transferTo(saveFile);

        Item updateItem = Item.from(item, fileName); // Builder를 통해 새로운 Item 객체 생성.
        itemRepository.save(updatedItem); // 새롭게 생성한 객체를 저장.
    }

### set() vs this(Construct) VS Builder pattern

    setter method 패턴 지양.

    1.
    Form 태그를 통한 데이터 전송: 주로 JSON 형식으로 데이터를 전송하기 때문에 직접적인 setter 메서드 호출이 필요하지 않음.

    2.
    DTO에서의 초기화: 생성자를 통해 필드를 초기화하거나, 빌더 패턴을 통해 객체를 생성하는 것이 더 일반적.

    3.
    비즈니스 로직: 서비스 클래스에서는 업데이트와 같은 비즈니스 로직을 처리할 때 필요한 경우에 한해 setter 메서드를 사용하는 것이 일반적입니다.