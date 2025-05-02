## 본문

### @RequestParam

    클라이언트에서 전달한 쿼리 파라미터 값을 서버에서 받아와 메서드의 파라미터에 매핑하는 역할.

    주로 HTML Form 태크 or URL 쿼리스트링에서 전달된 데이터를 서버에서 처리할 때 사용한다.

    쿼리 파라미터(?key=value) 형식으로 전송된 값을 받아올 때 사용

    ex) 
        GET https://example.com/api/products?category=books&page=2

        @GetMappig("/api/products")
        public ResponseEntity<List<product>> getProduct(
            @RequestParam String category,
            @RequestParam(defaultValue = "1") int page)
        {
            // category = "books", page = 2
            return ResponseEntity.ok(productService.getProductsByCategory(category,page));
        }


    ex)
        @GetMapping("/order/pay")
        public @ResponseBody ReadyResponse payReady(
               @RequestParam(name = "total_amount") int totalAmount,,,)
        {
            ReadyResponse readyResponse = kakaopayService.payReady(totalAmount);
        }

        클라이언트가 보낸 요청 파라미터를 컨트롤러 메서드의 파라미터로 매핑하는 역할을 한다.

        ex)
            GET/order/pay?total_amount=50000

            이 요청에서 total_amount=50000 이라는 쿼리 파라미터가 컨트롤러 메서드의 @RequestParam(name = "total_amount") int totalAmount 부분에 매핑이 된다.

            그래서 total_amount 값에 50000 값이 할당이 된다.

        ex)
            public String search(@RequestParam("keyword") String keyword) {}

            클라이언트가 /search?keyword=java 로 요청을 보내면, @RequestParam("keyword")는 URL 의 keyword 값인 "java"를 String keyword 변수에 매핑한다.


### @PathVariable

    URL 경로의 일부를 변수로 받아와 메서드의 파라미터에 매핑할 때 사용된다.

    경로의 특정 부분을 변수로 사용.

    데이터를 리소스의 식별자처럼 사용할 때 적합.

    ex)
        GET https://exmple.com/api/products/123

        @GetMapping("/api/products/{id}")
        public ResponseEntity<Product> getProductById(@PathVariable Long id)
        {
            // id = 123
            return ResponseEntity.ok(productService.getProductById(id));
        }

    ex)
        @GetMapping("/order/{orderId}")
        public String getOrder(@PathVariable("orderId") Long orderId) {}

        클라이언트가 /order/123 라는 URL로 요청을 보내면 orderId 변수에 123이 할당된다.

### @RequestParam vs @PathVariable

| Feature                  | `@RequestParam`                        | `@PathVariable`                       |
|--------------------------|-----------------------------------------|---------------------------------------|
| **데이터 위치**            | URL 쿼리 파라미터                     | URL 경로                              |
| **형식**                 | `?key=value` 형태                      | `/path/{variable}` 형태               |
| **사용 목적**             | 필터링, 검색, 정렬, 선택적 데이터 전달 | 리소스 식별, 고정된 URL 패턴          |
| **선택적 파라미터**       | 기본값 설정 가능                       | 기본값 설정 불가                      |
| **URL의 의미**            | 파라미터는 부가적 정보                 | 경로는 리소스 식별에 사용             |
| **중복 허용**             | 동일한 키로 여러 값 전달 가능          | 경로 변수는 중복 불가                 |    


### 함께 사용하는 경우

    @GetMapping("/api/products/{id}")
    public ResponseEntity<Product> getFilterProduct
    (
        @PathVariable Long id,
        @RequestParam String color,
        @RequestParam String size
    )
    {
        // id = 123, color = red, size = "medium"
        return ResponseEntity.ok(productService.getFilteredProduct(id, color, size));
    }

### 선택 기준

    @RequestParam 

    - 필터링, 검색, 정렬과 같이 동적으로 값이 바뀌는 추가 데이터를 전달하는

    ex)
        /products?category=book&page=2

    @PathVariable

    - 리소스를 식별하거나 URL에 고정된 값이 필요하다면

    ex)
        /users/{id}, /orders/{orderId}
