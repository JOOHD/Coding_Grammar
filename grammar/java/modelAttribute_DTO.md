## 본문

### @ModelAttribute vs DTO 변환

### @ModelAttribute를 사용한 데이터 전송
    ● client -> server 데이터 전송 시, 메서드 파라미터 DTO 객체(@ModelAttribute)를 이용한 데이터 전송
    
    ● MemberVO 
    @Data
    public class MemberVO {

        // 회원 id
        private String memberId;

        // 회원 비밀번호
        private String memberPw;

        // 회원 이름
        private String memberName;

        // 회원 이메일
        private String memberMail;

        // 회원 우편번호
        private String memberAddr1;

        // 회원 주소
        private String memberAddr2;

        // 회원 상세주소
        private String memberAddr3;

        // 관리자 구분(0:일반사용자, 1:관리자)
        private int adminCk;

        // 등록일자
        @DateTimeFormat(pattern = "yyyy-MM-dd")
        private int regDate;

        // 회원 돈
        private int money;

        // 회원 포인트
        private int point;
    }

    ● MemberController
    @ApiOperation(value = "로그인")
	@PostMapping(value="login.do")
	public String loignPOST(HttpServletRequest request, MemberVO member, RedirectAttributes rttr) throws Exception {
	
		HttpSession session = request.getSession();
		String rawPw = "";
		String encodePw = "";
		
		MemberVO lvo = memberservice.memberLogin(member);
		
		if(lvo != null) {	// 일치하는 아이디 존재시, (로그인 성공)
		
			rawPw = member.getMemberPw(); // 사용자가 제출한 비밀번호
			encodePw = lvo.getMemberPw(); // DB에 저장한 인코딩된 비밀번호
			
			if(true == pwEncoder.matches(rawPw, encodePw)) { // 비밀번호 일치여부 판단
				
				lvo.setMemberPw("");	  			// 인코딩된 비밀번호 정보 지움
				session.setAttribute("member",lvo); // session에 사용자의 정보 저장
				return "redirect:/main";			// 메인 페이지 이동
				
			} else {
				
				rttr.addFlashAttribute("result", 0);
				return "redirect:/member/login"; 	// 로그인 페이지로 이동
			}
		
				
		} else {		
		
			rttr.addFlashAttribute("result", 0); 
			return "redirect:/member/login"; 		// 로그인 페이지로 이동
		}
		
	}

    ● 정리

    - 설명  
    @ModelAttribute 는 Spring MVC에서 HTTP 요청의 파라미터를 자바 객체로 자동 변환하여 컨트롤러 메서드에 전달하는 데 사용된다. 주로 HTML Form 데이터와 같은 간단한 데이터를 서버 측에서 처리할 떄 유용.

    - 장점  
    1. 자동 바인딩 : HTML Form 데이터와 자바 객체의 속성이 자동으로 바인딩
    2. 간단함 : 설정이 간단하며, 별도의 변환 로직이 필요 없다.

    - 단점
    1. 제한된 제어 : 복잡한 변환 로직이 필요한 경우 @ModelAttribute 만으로는 제어가 어렵다.
    2. Form & DTO 의 역할 혼합 : Form 데이터와 DTO 의 역할이 혼합될 수 있어 코드의 명확성이 떨어질 수 있다.
    
    
### DTO 변환을 사용한 데이터 전송(Form 객체와 DTO 객체 구분)
    ● 클라이언트에서 받은 데이터를 바로 서비스 계층이나 데이터베이스에 전달하기 전에 검증, 변환 필요. 
    Form 객체는 클라이언트와 서버 간의 데이터 전송을 위한 구조를 나타내며, DTO 객체는 서비스 계층과 데이터베이스 간의 데이터 전송을 위한 구조를 나타낸다. 
    
    ● AdminOptionCategoryForm
    public class AdminOptionCategoryForm {

        @ApiModel(value = "AdminOptionCategoryFormRequest")
        @Getter
        public static class Request {
            @Schema(description = "옵션 카테고리명", example = "샷추가")
            @Pattern(regexp = "^[가-힣a-zA-Z]+$", message = "옵션 카테고리명은 한글, 영어 대소문자만 입력해야 합니다.")
            @Size(min = 1, max = 20, message = "옵션 카테고리명은 1~20 자리로 입력해야 합니다.")
            @NotBlank
            private String name;
        }

        @ApiModel(value ="AdminOptionCategoryFormResponse" )
        @Getter
        @Builder
        public static class Response {

            private Integer optionCategoryId;

            private String name;

            public static AdminOptionCategoryForm.Response
            from(AdminOptionCategoryDto.Response adminOptionCategoryDto) {
                return Response.builder()
                        .optionCategoryId(adminOptionCategoryDto.getOptionCategoryId())
                        .name(adminOptionCategoryDto.getName())
                        .build();
            }
        }
    }

    ● AdminOptionCategoryDto
    public class AdminOptionCategoryDto {

        @Getter
        @Builder
        public static class Request {

            String name;

            public static AdminOptionCategoryDto.Request from(AdminOptionCategoryForm.Request request) {
                return Request.builder()
                        .name(request.getName())
                        .build();
            }
        }

        @Getter
        @Builder
        public static class Response {

            private Integer optionCategoryId;

            private String name;

            public static AdminOptionCategoryDto.Response from(OptionCategory optionCategory){
                return Response.builder()
                        .optionCategoryId(optionCategory.getId())
                        .name(optionCategory.getName())
                        .build();
            }

        }
    }

    ● AdminOptionCategoryController
    @Tag(name = "admin-option-category-controller", description = "관리자 옵션 카테고리 CRUD API")
    @Controller
    @RequiredArgsConstructor
    @PreAuthorize("hasRole('ROLE_ADMIN')")
    @RequestMapping("/admin/option-category")
    public class AdminOptionCategoryController {

        private final AdminOptionCategoryService adminOptionCategoryService;

        // jiyeon-23.09.22
        @ApiOperation(value = "옵션 카테고리 등록", notes = "관리자가 옵션 카테고리를 등록합니다.")
        @PostMapping
        public ResponseEntity<Void> OptionCategoryAdd(
                @Valid @RequestBody AdminOptionCategoryForm.Request optionCategoryFormRequest) {
            adminOptionCategoryService.addOptionCategory(optionCategoryFormRequest);
            return ResponseEntity.status(CREATED).build();
        }

        // jiyeon-23.09.22
        @ApiOperation(value = "옵션 카테고리 수정", notes = "관리자가 옵션 카테고리를 수정합니다.")
        @PatchMapping("/{optionCategoryId}")
        public ResponseEntity<Void> optionCategoryModify(
                @PathVariable Integer optionCategoryId,
                @Valid @RequestBody AdminOptionCategoryForm.Request optionCategoryForm) {
            adminOptionCategoryService.modifyOptionCategory(optionCategoryId, optionCategoryForm);
            return ResponseEntity.status(NO_CONTENT).build();
        }

        // jiyeon-23.09.09
        @ApiOperation(value = "옵션 카테고리 삭제", notes = "관리자가 옵션 카테고리를 삭제합니다.")
        @DeleteMapping("/{optionCategoryId}")
        public ResponseEntity<Void> OptionCategoryRemove(@PathVariable Integer optionCategoryId) {
            adminOptionCategoryService.removeOptionCategory(optionCategoryId);
            return ResponseEntity.status(NO_CONTENT).build();
        }

        // jiyeon-23.09.09
        @ApiOperation(value = "옵션 카테고리 전체 목록 조회", notes = "관리자가 옵션 카테고리 목록을 조회합니다.")
        @GetMapping
        public ResponseEntity<List<AdminOptionCategoryForm.Response>> OptionCategoryList() {
            List<AdminOptionCategoryDto.Response> optionCategoryDtoList =
                    adminOptionCategoryService.findOptionCategoryList();
            List<AdminOptionCategoryForm.Response> adminOptionCategoryFormList =
                    optionCategoryDtoList.stream()
                            .map(AdminOptionCategoryForm.Response::from)
                            .collect(Collectors.toList());
            return ResponseEntity.ok(adminOptionCategoryFormList);
        }

        // jiyeon-23.09.09
        @ApiOperation(value = "옵션 카테고리 단건 조회", notes = "관리자가 하나의 옵션 카테고리에 대한 정보를 조회합니다.")
        @GetMapping("/{optionCategoryId}")
        public ResponseEntity<AdminOptionCategoryForm.Response> optionCategoryListById(
                @PathVariable Integer optionCategoryId) {
            AdminOptionCategoryDto.Response optionCategoryDto =
                    adminOptionCategoryService.findOptionCategoryListById(optionCategoryId);
            AdminOptionCategoryForm.Response optionCategoryForm =
                    AdminOptionCategoryForm.Response.from(optionCategoryDto);
            return ResponseEntity.ok(optionCategoryForm);
        }
    }

    ● 정리

    - 설명 
    DTO 변환 방법은 클라이언트의 요청 데이터를 특정 DTO 객체로 변환한 후, 이 DTO를 컨트롤러 메서드의 파라미터로 전달한다. 변환 로직을 명시적으로 작성하여 데이터를 전송을 처리.

    - 장점
    1. 명확한 변환 : 변환 로직을 명시적으로 구현하므로 데이터 변환 과정을 명확히 제어할 수 있다.
    2. 유연성 : 복잡한 변환 로직이 필요한 경우에도 유연하게 처리.
    3. 객체의 역할 분리 : Form 객체와 DTO 객체의 역할을 명확히 분리하여 코드의 가독성을 높일 수 있다.

    - 단점
    1. 추가 작업 필요 : DTO 변환을 위해 별도의 변환 메서드나 유틸리티 클래스가 필요.
    2. 추가 설정 : 변환 과정에서의 설정이 필요하며, 직접적으로 변환 로직을 구현.

    ● 추가 설명
    DTO 변환에서는 HttpServletRequest, HttpServletResponse 를 사용하지 않고, DTO, Form 클래스의 메서드로 사용한다.

    ● 기존 HttpServletRequest/Response 를 파라미터로 사용했을 경우
    ex)
        public String loignPOST(HttpServletRequest request, MemberVO member, RedirectAttributes rttr) throws Exception {
        
            HttpSession session = request.getSession();

        }

    ● Request/Response method 를 사용했을 때
    ex)
        @Getter
        @Builder
        public static class Request {

            String name;

            public static AdminOptionCategoryDto.Request from(AdminOptionCategoryForm.Request request) {
                return Request.builder()
                        .name(request.getName())
                        .build();
            }
        }

        @Getter
        @Builder
        public static class Response {

            private Integer optionCategoryId;

            private String name;

            public static AdminOptionCategoryDto.Response from(OptionCategory optionCategory){
                return Response.builder()
                        .optionCategoryId(optionCategory.getId())
                        .name(optionCategory.getName())
                        .build();
            }

        }

    ● 정리
    Spring MVC에서는 클라이언트의 요청을 처리하고, 응답을 생성할 때 주로 메서드 파라미터로 Java 객체를 사용한다. 이 방식은 코드의 가독성과 유지보두성을 높이고, 요청과 응답을 보다 직관적, 선언적으로 처리.

        1. 가독성
        - 메서드 파라미터로 Java 객체를 사용하면 코드가 더 읽기 쉽고 명확해진다. HTTP 요청 파라미터를 객체로 바인딩하면 각 파라미터의 목적과 의미가 명확해진다.
        
        2. 자동 바인딩
        - Spring MVC는 요청 파라미터를 자동으로 Java 객체에 바인딩 해준다. 이를 통해 개발자는 일일이 요청 파라미터를 추출하고 설정할 필요 없이, 객체를 직접 사용할 수 있다.

        3. 데이터 검증 및 변환
        - Java 객체를 사용하면 데이터 검증과 변환을 보다 쉽게 처리할 수 있따. 예를 들어, @Valid 와 같은 어노테이션을 사용해 요청 데이터를 검증할 수 있다.

        4. 재사용성
        - DTO 나 Form 객체는 재사용할 수 있다. 이를 통해 여러 컨트롤러에서 동일한 객체를 사용해 일관성을 유지.     

    ● HttpServletRequest/Response 를 사용할 때
    1. 세션 관리, 쿠키 처리 등 HTTP 프로토콜의 특정 기능을 사용할 떄.
    2. HTTP 헤더를 직접 조작하거나 응답 스트림을 직접 제어해야 할 떄.
    3. 파일 업로드/다운로드와 같이 스트림 처리가 필요한 경우.

    ● 메서드 파라미터로 Java 객체를 사용하는 이유
    - HttpServletRequest 사용
        - HttpServletRequest 를 사용하면 요청 파라미터를 수동으로 추출하고 설정해야 한다.

        ex) request.getParameter("name") 와 같이 요청 파라미터를 직접 추출해야 한다.

    - Java 객체 사용
        - Spring MVC 는 요청 파라미터를 자동으로 Java 객체에 바인딩.
        - @ModelAttribute 또는 기본 바인딩 메커니즘을 통해 요청 데이터를 객체에 바인딩 할 수 있따.

    ● 결론
    메서드 파라미터로 Java 객체를 사용하는 것은 더 선언적이고, 유지보수성이 높으며, 코드의 가독성을 향상시킨다. 또한 데이터 검증과 변환을 쉽게 처리할 수 있게 한다. Spring MVC 는 이러한 자동 바인딩을 지원함으로써 개발자가 보다 효율적으로 요청과 응답을 처리할 수 있도록 돕는다.    

### DTO -> Form/ Form -> DTO 객체 변환
    
    ● Form -> DTO 변환
    1. DTO 객체는 데이터베이스 or 다른 서비스에서 가져온 데이터를 포함하는 객체
    2. Form 객체는 클라이언트에게 데이터를 제공할 때 사용되는 객체로, 보통 클라이언트가 필요한 형태로 데이터를 변환하고 제공하기 위해 사용.
    
    ● 코드
    - AdminOptionCategoryForm.Request (Form 객체) -> AdminOptionCategoryDto.Request (DTO 객체)
    @Getter
    @Builder
    public static class Request {

        String name;

        public static AdminOptionCategoryDto.Request from(AdminOptionCategoryForm.Request request) {
            return Request.builder()
                    .name(request.getName())
                    .build();
        }
    }

    - OptionCategory (Entity 객체) -> AdminOptionCategoryDto.Response (DTO 객체)
    @Getter
    @Builder
    public static class Response {

        private Integer optionCategoryId;

        private String name;

        public static AdminOptionCategoryDto.Response from(OptionCategory optionCategory){
            return Response.builder()
                    .optionCategoryId(optionCategory.getId())
                    .name(optionCategory.getName())
                    .build();
        }

    }   

    ● DTO -> Form 변환    

    ● 코드
    - AdminOptionCategoryDto.Response (DTO 객체) -> AdminOptionCategoryForm.Response (Form 객체)   
    @ApiModel(value ="AdminOptionCategoryFormResponse" )
    @Getter
    @Builder
    public static class Response {

        private Integer optionCategoryId;
        private String name;

        public static AdminOptionCategoryForm.Response from(AdminOptionCategoryDto.Response adminOptionCategoryDto) {
            return Response.builder()
                    .optionCategoryId(adminOptionCategoryDto.getOptionCategoryId())
                    .name(adminOptionCategoryDto.getName())
                    .build();
        }
    }
    - Response.builder()를 사용하여 AdminOptionCategoryForm.Response의 빌더를 생성하고, DTO에서 받은 값을 설정합니다.

    - 이 메서드는 DTO 객체를 Form 객체로 변환할 때 사용되고, Form 객체는 보통 사용자에게 표시하거나 클라이언트에 응답을 반환할 때 사용된다. DTO 객체는 데이터베이스와의 상화작용 또는 다른 시스템과의 데이터 전송에 주로 사용.