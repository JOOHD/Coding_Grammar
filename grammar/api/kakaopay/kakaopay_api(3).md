## 본문

### 1. kakaopay 사전 설명

    카카오페이 연동은 request를 2번(1. 결재요청, 2. 결재승인 후) 결재내역정보를 받아와야 한다.

    카카오페이와 카카오로그인은 연동하는 방법이 좀 다르다.
    - 카카오 로그인 : ajax를 이용해 클라이언트 js에서 연동
    - 카카오페이 : controller로 값을 보내 클라이언트가 아닌 서버단에서 통신을 해야한다.

    그 이유는 cors 정책 때문이다.

### CORS(Cross-Origin Resource Sharing)

    : browser에서 다른 domain 간 요청을 제한하는 보안 기능.

    ● cors 동작 원리

    1. 프리플라이트 요청(Preflight Request) 

    일부 요청(POST, PUT, DELETE 메서드를 사용하는 경우)은 실제 요청 전에 OPTIONS 메서드를 사용해 서버에 프리플라이트 요청을 보낸다.
    이 요청을 통해 서버가 CORS를 허용할지 확인하는데, 서버가 특정 출저에서 온 요청을 허용할 경우 응답헤더에(Access-Controller-Allow-Origin)을 포함하여 응답한다.

    2. 서버 응답

    서버가 응답할 때, Access-Control-Allow-Origin 헤더로 허용할 출처를 명시하면, 브라우저는 해당 출처에서의 요청을 허용하고 차단을 해제한다.
    만약 이 헤더가 설정되지 않거나 불일치하면 브라우저는 요청을 차단

    외부서버에서 ajax 요청을 했을 때, cors라는 정책 위반 이슈가 발생한다.
    웹페이지에서 ajax를 동일서버에 요청하는 건 괜찮지만 외부서버에 요청하는 순가 동일 origin이 아니기 때문에 이 요청을 거부하는 보안관련 이슈가 생긴다.

### Q. 카카오로그인은 웹페이지(JSP)에서 외부서버로 ajax요청을 해도 왜 cors 이슈(ajax 요청거부)가 발생하지 않는가?

    - 카카오 로그인 API는 CORS 문제 없이 웹페이지에서 AJAX 요청을 통해 사용할 수 있도록 미리 CORS를 허용해 둔다.

    - 사용자의 로그인 정보를 인증하기 위한 API이기 때문에, 이를 쉽게 사용할 수 있도록 카카오 측에서 CORS를 열어놓고 웹페이지의 AJAX 요청을 허용하고 있다.

    - 다만, 돈과 관련된 api는 보안이슈로 인해 cors를 닫아놓기 때문에 우회해서 요청해야 한다.

### Q. CORS 설정을 우회하는 이유

    보안이 강화된 API는 프론트엔드에서 직접 접근이 불가능하므로, 일반적으로 백엔드 서버에서 중계하여 처리하는 방식이 사용된다.
    백엔드에서 외부 API에 요청을 보내고, 그 결과를 받아 웹페이지에 전달하는 방식으로 우회하는 것이다.

![cors_detour](../img/cors_detour.png)

    웹페이지에서 ajax요청을 동일서버에 보내고 그 안에있는 controller/service에서 외부서버에 요청을 보내서 우회하면 cors에 걸리지 않는다.

    - 이럴 때 사용하는 HttpUrlController 객체 / Spring : RestTemplate 객체
    (RestTemplate가 json데이터를 바로 java객체로 변환해주기 떄문에 더 편리, 응답을 바로 object로 맵핑해준다. postForObject)
    
### 2. 카카오 페이 결재 구현 과정

![kakaopay_implements](../img/kakaopay_implements.png)

    ● 결재과정 3가지

    1. 결재요청 준비.
    2. 결재요청
    3. 결재승인

    ● 클라이언트 - 서버 - 카카오(외부서버) 데이터 흐름

    1. 클라이언트에서 서버로 주문관련정보를 POST로 던진다.
    2. 서버는 Controller에서 그 값을 받아 카카오 승인요청 url로 던져준다.
    3. 카카오는 승인과 동시에 다음 결재과정의 url과 tid(결재코드)을 서버로 던져주고 서버는 클라이언트에 url을 던진다.
    4. 클라이언트는 그 url로 qr코드 결재 화면에 접근한다.
    5. 결재처리가 되면 카카오에서 우리 서버 controller의 결재 url로 token 값을 보내준다.
    6. 서버는 그 토큰과 처음에 받았던 tid로 결재승인을 요청하고 결재승인 정보를 받아온다.
    7. 주문성공페이지 url을 클라이언트에 보내고 클라이언트는 그 페이지를 서버에 요청한다.
    8. 서버가 주문완료 페이지를 클라이언트에 보내준다.
    (주문완료 페이지가 없다면 ajax에서 결재내역페이지로 보내도 된다.)

### 3. ajax, VO, Service, Controller

    ● ajax

    // 카카오 결재
    $(function(){
        $("#btn-kakao-pay").click(function(){

            // 필수입력 값을 확인
            var name = $("#form-payment input[name='pay-name']").val();
            var tel = $("#form-payment input[name='pay-tel']").val();
			var email = $("#form-payment input[name='pay-email']").val();

            // 필수입력 값 검증
            if (name == "") {
                $("#form-payment input[name='pay-name']").focus()
            }
            if (tel == "") {
				$("#form-payment input[name='pay-tel']").focus()
			}
			if (email == "") {
				$("#form-payment input[name='pay-email']").focus()
			}

            // 결재 정보를 form에 저장.
            let totalPayPrice = parseInt($("#total-pay-price").text().replace(/,/g,''))
            let totalPrice = parseInt($("#total-price").text().replace(/,/g,''))
            let discountPrice = totalPrice - totalPayPrice
            let usePoint = $("#point-use").val()
            let useUserCouponNo = $(":radio[name='userCoupon']:checked").val()

            // 카카오페이 결재전송
            $.ajax({
                type :'get',
                url : '/order/pay',
                data : {
                    total_amount : totalPayPrice,
                    payUserName : name,
                    sumPrice : totalPrice,
                    discountPrice : discountPrice,
                    totalPrice : totalPrice,
                    tel : tel,
                    email : email.
                    usePoint :usePoint,
                    useCouponNo : useUserCouponNo
                },
                success : function(response) {
                    location.href = response.next_redirect_pc_url
                }
            })
        })
    })

    ● VO (결재요청VO, 결재승인요청VO, 결재내역VO)

    1. 결재요청 VO

    public class ReadyResponse {

        private String tid;                     // 결제 고유 번호
        private String next_redirect_pc_url;    // 결제 완료 시, 이동할 페이지
        private String partner_order_id;        // 주문한 사람 ID
    }

    2. 결재승인요청 VO

    public class ApproveResponse {

        private String aid;                     // 요청 고유번호
        private String tid;                     // 결제 고유번호
        private String cid;                     // 가맹점 코드
        private String sid;                     // 정기 결제용 
        private String partner_order_id;        // 주문 번호 
        private String partner_user_id;         // 주문자명 
        private String payment_method_type;     // 결제수단 (CARD/CASH)
        private String item_name;               // 상품명
        private String item_code;               // 상품 코드
        private int quantity;                   // 상품 수량
        private String created_at;              // 결제 준비 요청 시각
        private String approved_at;             // 결제 승인 시각
        private String payload;                 // HTTP 요청 시, 전달되는 데이터
        private Amount amount;                  // 전체 결제 금액(비과세 + 부가세 + 사용한 포인트 + 할인금액)    
    }

    3. 결재내역 VO

    public class Amount {

        private int total;      // 전체 결제 금액
        private int tax_free;   // 비가세
        private int vat;        // 부가세
        private int point;      // 포인트
        private int discount;   // 할인금액
    }

### Service

    @Slf4j
    @Service
    public class KakaoPayService {

        @Autowired
        private CartMapper cartMapper;

        public ReadyResponse payReady(int totalAmount) {

            User user = (User)SessionUtils.getAttribute("LOGIN_USER");
            List<CartDto> carts = cartMapper.getCartByUserNo(user.getNo());

            String[] artNames = new String[carts.size()];
            for (CartDto cart : carts) {
                for (int i=0; i < carts.size(); i++) {
                    cartNames[i] = cart.getClassTitle();
                }
            }

            // -1 : 첫 번째 항목을 제외한 나머지 항목의 개수를 구하기 위한 것.
            String itemName = cartNames[0] + "그외" + (carts.size()-1);
            log.info("강좌이름들:" + itemName);
            String order_id = user.getNo() + itemName;

            // 카카오가 요구한 결제요청 request 값을 담아준다.
            MultiValueMap<String, String> parameters = new LinkedMultiValueMap<String, String>();
            parameters.add("cid", "TC0ONETIME");
            parameters.add("partner_order_id", order_id);
            parameters.add("partner_user_id", "inflearn");
            parameters.add("item_name", itemName);
            parameters.add("quantity", String.valueOf(carts.size()));
            parameters.add("total_amount", String.valueOf(totalAmount));
            parameters.add("tax_free_amount", "0");
            parameters.add("approval_url", "http://localhost/order/pay/completed"); // 결제승인시 넘어갈 url
            parameters.add("cancel_url", "http://localhost/order/pay/cancel"); // 결제취소시 넘어갈 url
            parameters.add("fail_url", "http://localhost/order/pay/fail"); // 결제 실패시 넘어갈 url

            log.info("구매자 주문 아이디 :" + parameters.get("partner_order_id"));

            HttpEntity<MultiValueMap<String, String>> requestEntity = new HttpEntity<>(parameters, this.getHeaders());
            
            // 외부 url 요청 통로 열기
            RestTemplate template = new RestTemplate();
            String url = "http://kapi.kakao.com/v1/payment/ready";

            // Template으로 값을 보내고 받아온 ReadyResponse값 readyResponse에 저장.
            ReadyResponse readyResponse = template.postForObject(url, requestEntity, ReadyResponse.class);

            log.info("결재준비 응답객체: " + readyResponse);

            // 받아온 값 return
            return readyResponse;
        }

        // 결제 승인요청 메서드
        public ApproveResponse payApprove(String tid, String pgToken) {

            User user = (User)SessionUtils.getAttribute("LOGIN_USER");
            List<CartDto> carts = cartMapper.getCartByUserNo(user.getNo());

            // 주문명 만들기
            String[] cartNames = new String[carts.size()];
            for(CartDto cart: carts) {
                for(int i=0; i< carts.size(); i++) {
                    cartNames[i] = cart.getClassTitle();
                }
            }	
            String itemName = cartNames[0] + " 그외" + (carts.size()-1);
            
            String order_id = user.getNo() + itemName;

            // request값 담기.
            MultiValueMap<String, String> parameters = new LinkedMultiValueMap<String, String>();
            parameters.add("cid", "TC0ONETIME");
            parameters.add("tid", tid);
            parameters.add("partner_order_id", order_id); // 주문명
            parameters.add("partner_user_id", "회사명");
            parameters.add("pg_token", pgToken);
            
            // 하나의 map안에 header와 parameter값을 담아줌.
            HttpEntity<MultiValueMap<String, String>> requestEntity = new HttpEntity<>(parameters, this.getHeaders());
            
            // 외부url 통신
            RestTemplate template = new RestTemplate();
            String url = "https://kapi.kakao.com/v1/payment/approve";
            // 보낼 외부 url, 요청 메시지(header,parameter), 처리후 값을 받아올 클래스. 
            ApproveResponse approveResponse = template.postForObject(url, requestEntity, ApproveResponse.class);
            log.info("결재승인 응답객체: " + approveResponse);
            
            return approveResponse;
        }

        // header() 셋팅
        private Httpheaders getHeaders() {
            HttpHeaders headers = new HttpHeaders();
            headers.set("Authorization", "kakaoAK Admin Key");
            headers.set("Content-type", "application/x-www-form-urlencoded;charset=utf-8");

            return headers;
        }
    }

### payReady 결제준비 메서드 설명

    ● 주요 로직 설명

    - 결제 준비를 위한 요청을 보내고, 결제 준비 응답을 받는 로직이다.
    payReady 메서드는 결제 준비 요청을 카카오페이 API에 보내기 위해 필요한 파라미터들을 설정하고, 이를 전송하는 과정.

    ● 기본 흐름 

    1. 주문명 생성
    
        for (int i = 0; i < carts.size(); i++) {
            cartNames[i] = cart.getClassTitle();
        }
        - 각 장바구니 항목의 getClassTitle() 메서드를 호출하여 상품명을 배열에 저장한다.

        - String itemName = cartName[0] + "그외" + (carts.size()-1);
          - 첫 번째 상품명과 나머지 상품의 개수를 기반으로 주문명을 생성한다.

          ex) itemName = "커피 그외2"(cartName[0] + "그외" + carts.size()-1)

    2. 결제 요청 파라미터 구성:

        - 카카오페이 결제 요청에 필요한 파라미터들을 MultiValueMap<String, String>에 담습니다. 주요 파라미터는 다음과 같습니다:

        - cid: 카카오페이 상점 ID (고정값: "TC0ONETIME")
        - partner_order_id: 주문 고유 ID (사용자 번호와 주문명 결합)
        - partner_user_id: 판매자 ID (예: "inflearn")
        - item_name: 상품명
        - quantity: 구매 수량 (장바구니의 크기)
        - total_amount: 총 결제 금액 (예시에서 전달되는 totalAmount)
        - tax_free_amount: 면세 금액 (예시에서는 0)
        - approval_url, cancel_url, fail_url: 결제 성공/취소/실패 후 이동할 URL들.          

    3. 결제 준비 요청

        - HttpEntity<MultiValueMap<String, String>> requestEntity = new HttpEntity<>(parameters, this.getHeaders());

            - 요청 파라미터와 헤더를 HttpEntity로 감싸서 카카오페이에 보낼 준비를 한다.

        - RestTemplate tempate = new RestTemplate();
        
            - RestTemplate을 사용하여 외부 URL로 POST 요청을 보낸다.

        - ReadyResponse readyResponse = template.postForObject(url, requestEntity, ReadyResponse.class);:

            - 카카오페이 API로 결제 준비 요청을 보내고, 응답을 ReadyResponse 객체로 받는다.

    4. 결제 준비 응답 처리

        - 결제 준비 응답(ReadyResponse)을 반환하고, 로그로 응답 정보를 기록.

    ● 흐름 요약

    1. 사용자가 결제 준비를 시작하면, payReady 메서드가 호출된다.
    2. 사용자의 장바구니와 결제 관련 정보를 기반으로 요청 파라미터가 생성.
    3. 요청 파라미터를 카카오페이 결제 준비 API에 보내고, 결제 준비 결과를 응답으로 받는다.
    4. 결제 준비 응답을 받아 후속 작업을 진행할 수 있다. 

### payApprove 결재승인 메서드 설명

    ● 주요 로직 설명

    - 사용자가 셜제 승인을 요청하면 카카오페이 API에 필요한 값들을 요청 파라미터로 설정하고, RestTemplate 을 사용하여 카카오페이 서버에 HTTP 요청을 보내는 방식이다.

    ● 기본 흐름

    1. 사용자 정보 및 장바구니 정보 가져오기

        - User 객체는 SessionUtils.getAttribute(""LOGIN_USER")를 통해 세션에서 로그인된 사용자 정보를 가져온다.

        - carts 는 cartMapper.getCartByUserNo(user.getNo())를 통해 사용자의 장바구니에서 선택된 상품 목록을 가져온다.

    2. 주문명 생성

        - cartNames 배열을 통해 장바구니에 담긴 상품들의 이름을 가져온다. 이 이름을 기반으로 주문명을 생성하는데, 장바구니가 하나 이상일 경우 "그외" 와 남은 상품 개수를 추가하여 주문명을 만든다.

    3. 결제 요청 파라미터 구성

        - 결제 요청에 필요한 tid, pgToken, order_id 등의 파라미터 값을 MultiValueMap에 담는다. 
        여기서 중요한 부분은 카카오페이에 전달하는 order_id, partner_order_id, pg_token 등이다.

        - order_id는 결제 정보에 대한 고유 식별자로 생성되며, partner_user_id는 판매자(회사) 정보를 나타낸다.

    4. 결재 승인 요청

        - HTTP POST 요청을 카카오페이의 결제 승인 API
        (https://kapi.kakao.com/v1/payment/approve)에 보낸다.

        - header 와 parameter 는 HttpHeaders 오 HttpEntity 를 통해 설정된다. header에는 Authorization이 포함되어 있으며, 이는 카카오페이의 Admin key를 포함한 값이다. 

    5. 카카오페이 결재 승인 응답
     
        - 카카오페이 서버로부터 받은 응답은 ApproveResponse 객체로 매핑되어 반환된다.           

    ● 흐름 요약

    1. 사용자가 셜제 버튼을 누르면, 카카오페이 결제 승인 요청이 이루어진다.
    2. tid, pg_token, order_id 등 필수 데이터를 파라미터로 카카오페이에 요청을 보낸다.
    3. 카카오페이는 결제 승인을 처리하고 응답을 보내준다.
    4. 결제 성공 여부에 따라 후속 작업을 진행한다.
 
 ### OrderController

    - @Autowired 는 많아서 생략.
    
    @Sl4fj  
    @Controller
    // 세션에 저장된 값을 사용할 때, 쓰는 어노테이션, session 에서 없으면, model 까지 흝어서 찾아냄
    @SessionAttribute({"tid", "order"})
    public class OrderController {

        // 카카오페이 결제 요청
        @GetMapping("/order/pay")
        public @ResponseBody ReadyResponse payReady(@ReqeustParam(name = "total_amount") int totalAmount, Order order, Model model) {
            
            log.info("주문정보:" + order);
            log.info("주문가격:" + totalAmount);

            // 카카오 결제 준비하기 - 결제요청 service 실행.
            ReadyResponse readyResponse = kakaopayService.payReady(totalAmount);

            // 요청처리후 받아온 결제고유 번호(tid)를 model에 저장.
            model.addAttribute("tid", readyResponse.getTid());

            log.info("결제고유 번호:" + readyResponse.getTid());

            // Order 정보를 모델에 저장
            model.addAttribute("Order", order);

            return readyResponse; // 클랑이언트에 보냄.(tid, next_redirect_pc_url이 담겨있음.)
        }

        // 결제승인요청
        @GetMapping("/order/pay/completed")
        public String payCompleted(
            @RequestParam("pg_token") String pgToken,                   @ModelAttribute("tid") String,
            @ModelAttribute("order"), Order order, Model model) {

                log.info("결제승인 요청을 인증하는 토큰:" + pgToken);
                log.info("주문정보:" + order);
                log.info("결제고유 번호: " : tid);

                // 카카오 결제 요청하기
                ApproveResponse approveResponse = kakaopayService.payApprove(tid, pgToken);

                // 5. payment 저장
                // orderNo, payMathod, 주문명.
                // 카카오 페이로 넘겨받은 결제정보값을 저장.
                Payment payment = Payment.builder()
                        .paymentClassName(approveResponse.getItem_name())
                        .payMathod(approveResponse.getPayment_method_type())
                        .payCode(tid)
                        .build();

                orderService.saveOrder(order, payment);

                return "redirect:/orders";
        }

        // 결제 취소시 실행 url
        @GetMapping("/order/pay/cancel")
        public String payCancel() {
            return "redirect:/carts";
        }
        
        // 결제 실패시 실행 url    	
        @GetMapping("/order/pay/fail")
        public String payFail() {
            return "redirect:/carts";
        }    
    }
