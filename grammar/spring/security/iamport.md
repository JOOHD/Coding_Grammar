## iamport

### REST API KEY & Secret KEY

| 항목                                 | 용도                                                                    |
| ---------------------------------- | --------------------------------------------------------------------- |
| **REST API 키** (`imp_key`)         | Iamport에서 프로젝트를 식별하는 **공개 키**. 프론트엔드나 백엔드에서 사용 가능                     |
| **REST API Secret** (`imp_secret`) | 해당 프로젝트의 **비공개 키**. 민감 정보이므로 **서버에서만 사용**해야 함                         |
| → Access Token 발급에 필요              | `imp_key + imp_secret`으로 Iamport 인증 서버에 토큰을 요청하고, 이 토큰으로 결제 관련 API 호출 |

-> 요약 : iamport 는 자체 인증 시스템을 쓰기 때문에 imp_key + imp_secret 조합으로 Access Token 을 받아야 하며, 이 토큰을 이용해 결제 승인, 취소, 상태 조회 등을 처리한다.

### 구현 예시 

- 상품명 : 아이폰 16
- 금액 : 1,500,000원
- 프론트에서 Iamport.request_pay() 호출
- 결제 완료 시 imp_uid, merchant_uid 전달 -> 백엔드 검증 및 주문 처리

### 1. 프론트엔드 (HTML + JS)

<head>
  <meta charset="UTF-8">
  <title>아이폰 16 결제</title>
  <script src="https://cdn.iamport.kr/js/iamport.payment-1.1.8.js"></script>
</head>
<body>
  <h1>아이폰 16 결제 테스트</h1>
  <button onclick="requestPay()">결제하기</button>

  <script>
    function requestPay() {
        IMP.init("imp_yourcode"); // 본인의 가맹점 식별코드 사용

        IMP.request_pay({
            pg: "kakaopay",
            pay_method: "card",
            merchant_uid: "iphone16_" + new Date().getTime(), // 주문번호
            name: "아이폰 16",
            amount: 1500000,
            buyer_email: "test@naver.com",
            buyer_name: "홍길동",
            buyer_tel: "010-1234-5678",
            buyer_addr: "서울특별시 강남구",
            buyer_postcode: "12345" 
        }, function (rsp) {
            if (rsp.success) {
                // 결제 성공 시, 서버에 검증 요청
                fetch("http://localhost:8080/api/payment/validate", {
                    method: "POST",
                    headers: {"Content-Type": "application/json"},
                    body: JSON.stringify({
                        imp_uid: rsp.imp_uid,
                        merchant_uid: rsp.merchant_uid,
                        amount: rsp.paid_amount
                    })
                })
                .then(res => res.text())
                .then(msg => alert("서버 검증 완료: " + msg));
            } else {
                alert("결제 실패: " + rsp.error_msg);
            }
        });
    }
      </script>
</body>
</html>

### 2. 백엔드 코드

1. IamportPaymentRequestDTO.java

@Data
@NoArgsConstructor
@AllArgsConstructor
public class IamportPaymentRequestDTO {
    private String imp_uid;
    private String merchant_uid;
    private int amount;
}

2. Orders.java (JPA Entity)

@Entity
public class Orders {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String productName;
    private String impUid;
    private String merchantUid;
    private int amount;
}

3. OrderRepository.java

public interface OrderRepository extends JpaRepository<Orders, Long> {}

4. IamportService.java

@Service
@RequiredArgsConstructor
public class IamportService {
    private final RestTemplate restTemplate = new RestTemplate();
    private final OrderRepository orderRepository;

    @Value("${iamport.api-key}")
    private String apikey;

    @Value("${iamport.api-secret}")
    private String apiSecret;

    public String getAccessToken() {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);

        Map<String, String> payload = Map.of(
            "imp_key", apiKey,
            "imp_secret", apiSecret
        );

        HttpEntity<?> entity = new HttpEntity<>(payload, headers);
        ResponseEntity<JsonNode> response = restTemplate.exchange(
            "https://api.iamport.kr/users/getToken",
            HttpMethod.POST,
            entity,
            JsonNode.class
        );

        return response.getBody().get("response").get("access_token").asText();
    }

    public PaymentInfo getPaymentInfo(String impUid) {
        String token = getAccessToken();
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(token);
        HttpEntity<?> entity = new HttpEntity<>(headers);

        ResponseEntity<JsonNode> response = restTemplate.exchange(
            "https://api.iamport.kr/payments/" + impUid,
            HttpMethod.GET,
            entity,
            JsonNode.class
        );

        JsonNod data = response.getBody().get("response");
        return enw PaymentInfo(
                data.get("imp_uid").asText(),
                data.get("merchant_uid").asText(),
                data.get("amount").asInt()
        );
    }

    public void validateAndSave(IamportPaymentRequestDTO dto) {
        PaymentInfo info = getPaymentInfo(dto.getImp_uid());
        if (info.getAmount() != dto.getAmount()) {
            throw new IllegalArgumentException("금액 불일치");
        }

        Orders order = new Orders();
        order.setProductName("아이폰 16");
        order.setImpUid(info.getImp_uid());
        order.setMerchantUid(info.getMerchant_uid());
        order.setAmount(info.getAmount());
        orderRepository.save(order);
    }
}

5. PaymentController.java

@RestController
@RequiredArgsConstructor
@RequestMapping("/api/payments")
public class PaymentController {
    private final IamportService iamportService;

    @PostMapping("/validate")
    public ResponseEntity<String> validate(@RequestBody IamportPaymentRequestDTO dto) {
        iamportService.validateAndSave(dto);
        return ResponseEntity.ok("결제 성공 및 주문 생성 완료");
    }
    
}

### New Shop 프로젝트 기준 OAuth2 & Iamport 연동 플로우

프로젝트 : New Shop
특징 : OAuth2 기반 로그인 (Kakao) -> JWT 인증 -> Redis 세션 -> Iamport 결제 연동

전체 흐름 
[client] -> [OAuth2 로그인 (kakao)]
         -> [Access Token + Refresh Token 발급 (JWT + Redis 저장)]
         -> [Frontend: IMP.request_pay() 결제 요청]
         -> [결제 성공 시, imp_uid, merchant_uid 전달]
         -> [Server: 결제 정보 검증 -> Orders 생성 + 저장]