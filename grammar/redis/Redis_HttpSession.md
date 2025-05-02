## HttpSession vs Redis

    세션 관리 방식을 어떻게 할지 결정할 때, 자주 비교되는 대상.
    

### 1. HttpSession (기본 세션 방식)

    서버 내부 메모리에 세션 데이터를 저장함, Java EE or Spring Boot 가 기본 제공.

    ● 특징
    - 세션 데이터는 WAS 메모리에 저장된 (Map 형태)
    - 클라이언트는 JESSIOND 쿠키로 세션 식별
    - WAS 가 꺼지거나 재시작되면 세션 정보 사라짐
    

    ● 장점
    - 구현 간단 (request.getSession()만 써도 됨)
    - 별도 인프라 필요 없음
    - 로컬 개발 및 단일 서버에 적합

    ● 단점
    - 서버 재시작 시, 세션 날라감
    - 서버 간 세션 공유 안 됨 -> 로드 밸런싱 불가
    - 세션이 많아지면 WAS 메모리 과부화


### 프로젝트 로직 (Order 주문, HttpSession 사용)    

    ● OrderController

    @PostMapping("/create")
    public ResponseEntity<String> createOrder(@RequestBody Map<String, Object> payload) {
        List<Integer> cartIdsInteger = (List<Integer> payload.get("cartIds"));
        List<Long> cartIds = cartIdsInteger.stream()
                                .map(Long::valueOf)
                                .collection(Collectors.toList());
        Orders temporaryOrder = orderService.createOrder(cartIds);

        // 세션에 임시 주문 정보를 저장
        httpSession.setAttribute("temporaryOrder", temporaryOrder);
        httpSession.setAttribute("cartIds", cartIds);

        Object cartIdsAttribute = httpSession.getAttribute("cartIds");

        return ResponseEntity.ok("주문 임시 저장 완료");                        
    }

    ● 세션에 임시 주문 정보를 저장
    - httpSession 사용 시점, 사용자가 장바구니(cartIds)를 선택해서 주문서를 생성
    - 지정된 값 : 
    1) temporaryOrder : Orders 객체 -> 주문서에 표시될 임시 주문 정보
    2) cartIds : 선택한 장바구니 아이템 ID 목록
 
    ● completeOrder() 메서드
    Orders temporaryOrder = (Orders) httpSession.getAttribute("temporaryOrder");
    - 사용된 시점 : 사용자가 주문서에서 배송지, 수령자면 등 최종 정보를 입력하고 주문을 확정할 때
    - 가져오는 값 :
    1) 세션에서 이전 단계에세 저장했던 temporaryOrder 를 꺼내와서 사용

### HttpSession 적용된 프로젝트 흐름 정리

    (1) /api/v1/order/create
        └─> 선택한 cartIds를 기반으로 Orders 임시 생성
        └─> HttpSession에 Orders, cartIds 저장

    (2) /api/v1/order/done
        └─> 사용자가 입력한 배송 정보, 수령인 등 OrderDto 전달
        └─> HttpSession에서 임시 주문 정보 꺼냄
        └─> 주문 최종 확정 및 저장

### HttpSession 의 사용 목적 분석

    목적                    설명
    1. 임시 저장소로 활용    사용자의 장바구니(cartIds)로 만든 Orders 객체를 세션에 잠깐 저장함
    2. 주문 프로세스 분리    주문 생성과 주문 확정을 두 단계로 분리해서 처리 (create -> done)
    3. 데이터 유지          화면 전환 혹은 중간 입력 없이도 사용자 브라우저 안에서 정보 유지 가능
    4. 유지 기반 분기 기능   세션은 유저별로 분리되므로, 여러 사용자가 동시에 접근해소 격리 가능
 
### 결론

    1. 주문 단계가 두 개로 나뉘는 구조
    2. 첫 단계 결과(Orders)를 임시 저장해두고,
    3. 다음 단계에서 꺼내 쓰기 위해 사용하는 구조
    
    이걸 Redis 기반 세션으로 바꾸고 싶다면 spring-session-data-redis를 도입해서 세션 데이터를 Redis에 저장하도록 설정하면 돼. 그러면 서비스가 여러 대로 확장되거나 서버 재시작이 있어도 세션 유지가 가능해져.

### 2. Redis 세션 저장소 (분산 캐시 기반)

    세션 정보를 Redis 같은 외부 저장소에 저장, Spring Session + Redis 조합이 대표적

    ● 특징
    - 세션이 Redis에 저장됨
    - 서버 재시작해도 Redis에 정보 살아있음
    - 서버 간 세션 공유 가능 (scale-out 대응응)

    ● 장점
    - 서버 무관 세션 유지 -> 클러스터 환경 가능
    - 세션 스토리지와 로직 분리됨 (SRP 지킴)
    - 성능 빠름 (Redis는 메모리 기반)

    ● 단점
    - Redis 설치/운영 필요
    - 초기 설정 조금 복잡 (spring-session, Redis 설정 등)
    - Redis 장애 시 전체 서비스에 영향을 줄 수 있음 (=>Sentinel 등으로 대비 가능능)
  
    1. Gradle 의존성 추가
    // build.gradle
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-data-redis'
        implementation 'org.springframework.session:spring-session-data-redis'
    }

    2. application.yml 설정
    spring:
        session:
            store-type: redis # 세션 저장소를 Redis로 설정
            timeout: 1800s    # 세션 만료 시간 (30분)
        redis:
            host: localhost   # Redis 주소 (Docker 사용 시 컨테이너 이름으로도 가능)
            port: 6379

### 프로젝트 사용 로직 (Order 주문, Redis 사용)

    간략한 설명은 Redis를 사용해서 "임시 주문 정보를 저장"하고, 나중에 이를 기반으로 "최종 주문 확정"을 수행하는 구조이다. Redis는 빠른 속도를 이용해 일시적인 데이터를 저장할 때 적합한 코드이다.

    ● 흐름 요약
    1. POST /api/v1/order/create -> 사용자가 장바구니에서 주문 생성 클릭
        - OrderDto 를 기반으로 임시 Orders 객체 생성
        - Redis에 임시 주문 정보(TemporaryOrderRedis) 저장
        
    2. POST /api/v1/order/done -> 사용자가 이름/주소/결제 수단 입력 후 결제 완료 클릭
        - Redis에서 임시 주문 정보 꺼냄
        - 꺼낸 정보를 바탕으로 실제 Orders 엔티티를 생성하고 DB에 저장

    ● OrderService

    1. Redis 저장 로직 -> saveTemporaryOrder(OrderDto orderDto)
    TemporaryOrderRedis tempOrder = TemporaryOrderRedis.builder()
        .id("tempOrder:" + orderDto.getMemberId()) // Redis 키
        .memberId(orderDto.getMemberId())
        .username(orderDto.getOrdererName())
        .cartIds(cartIds)
        .productNames(productNames)
        .totalPrice(totalPrice)
        .phoneNumber(orderDto.getPhoneNumber())
        .build();    

    redisOrderRepository.save(tempOrder);

    - TemporaryOrderRedis 라는 DTO 객체를 Redis에 저장
    - Redis Key는 tempOrder:{memberId} 형태로 설정 ex) tempOrder:10
    - 내부에 담긴 값 (회원id, 주문자 이름, 장바구니 List<id>, 상품명 리스트, 총 가격, 전화번호)
    -> 임시 저장이기 때문에 DB에 저장하는 게 아니라, Redis에 빠르게 저장.

    2. Redis에서 읽어서 실제 주문으로 저장, confirmOrder(OrderDto orderDto)
    temporaryOrderRedis temporaryOrderRedis = redisOrderRepository.findById(redisKey)
    - Redis에서 임시로 저장해둔 주문 정보를 불러움
    - 가져온 데이터를 기반으로 실제 Orders 객체를 빌드

    Orders newOrder = Orders.builder()
        .member(memberRepository.findById(orderDto.getMemberId()...))
        ...
        .productName(String.join(",", temporaryOrderRedis.getProductName()))
        .totalPrice(BigDecimal.valueOf(temporaryOrderRedis.getTotalPrice()))
        .build();
    - Redis에 있던 productNames, totalPrice, phoneNumber 등의 정보를 그대로 사용해서 주문을 확정

### 왜 Redis를 사용하는가?

    이유                   설명
    1. 속도                세션/임시 데이터를 DB에 넣는 것보다 빠르다
    2. TTL 설정 가능       일정 시간이 지나면 자동으로 삭제 가능(위에 코드에는 TTL 없음)
    3. 세션 대체           사용자가 화면에 머물고 있는 동안 임시 저장용으로 적합
    4. 중간 저장           실제 DB에 저장하기 전, 사용자가 "결제" 버튼을 누를 떄까지 보류 가능

    추가 개선 필요
    ※ TTL : 일정 시간이 지나면 자동 삭제
    @RedisHash(value = "tempOrder", timeToLive = 1800) // 30분
    public class TemporaryOrderRedis {...}

### 언제 어떤 걸 써야 할까?

    상황                                  추천 방식
    단일 서버, 빠른 개발 필요               HttpSession
    서버 여러 대, 클라우드 환경             Redis
    세션 유지가 중요 (로그인 유지)          Redis
    Kubernetes, ECS 등 컨테이너 기반 환경   Redis