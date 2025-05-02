## 본문

### SpringBoot - paging(@PageableDefault 적용)

### part1.Pageable - 정의 
    Spring에서 제공하는 Pagination을 위한 interface 이다.

![pageable_structure](/grammer/img/pageable_structure.png)

    위에 구조로 실제 구현체인 PageRequest를 사용한다.

    ● PageRequest 생성자
        
        public class PageRequest extends AbstractPageRequest {

            private static final long serablVersionUID = -45415099...L;

            private final Sort sort;

            protected PageRequest(int page, int size, Sort sort) {

                super(page, size);

                Assert.notNull(sort, "Sort must not be null");

                this.sort = sort;
            }
        }

    - size : 한 페이지당 담길 데이터의 양
    - page : size를 기준으로 몇번째 페이지인지
    - sort : 무엇을 기준으로 정렬할 것인지? 
        ex) createdAt, DESC, description

    - 이것을 기준으로 요청 URL에 변수를 넘기면, 페이지네이션이 동작한다.
        ex) GET/chats/:id?member=1&page=0&size=10&sort=description, DESC
        (없으면 기본값으로 동작한다.)

    ● AbstractPageRequest

        public abstract class AbstarctPageRequest implements Pageable, Serializable {

            private static final long serialVersionUID = 12328255786...L;

            private final int page;
            private final int size;

            public AbstractPageRequest(int page, int size) {

                if (page < 0) {
                    throw new IllegalArgumentException("page index must not be less than zero");
                }

                if (size < 1) {
                    throw new IllegalArgumentException("page size must not be less than one");
                }

                this.page = page;
                this.size = size;
            }
        }

        - 추가적으로 size, page 는 추상 클래스인 AbstractPageRequest에 있습니다. 페이지 넘버와 페이지 사이즈 오류처리를 확인.

### part1.Pageable - 적용
    Pageable이 편리한 점은 Spring Data JPA를 사용시 직ㅈ버 넘겨주면 편리하게 페이지네이션을 할 수 있다는 점 입니다.

    ● Controller
    @GetMapping("/chats/{id}")
    public ResponseEntity<BaseResponse>  findMessageByChatRoomId(@PathVariable("id") Long chatRoo,Id,
     @RequestParam(value = "member", required = true)
     Long memberId, Pageable pageable) {

        List<MessageResponseDto> messages = chatService.findMessage(chatRoomId, memberId, pageable);

        return new ResponseEntity<>(new BaseResponse(message), HttpStatus.OK);
    }

    ● Service
    public List<MessageResponseDto> findMessages(Long chatRoomId, Long requestUserId, Pageable pageable) throws BaseException {

        List<Message> messages = messageRepository.fidnAllByChatRoom_IdOrderByCreatedAt(chatRoomId, pageable);

        return message.stream()
                .map(message -> new MessageResponseDto(message, requestUserId))
                .collect(Collectors.toList());
    }

    ● Repository
    public interface MessageRepository extends JpaRepository<Message, Long> {
        List<Message> findAllByChatRoom_IdOrderByCreatedAt(Long charRoomId, Pageable pageable)
    }

    ● 결과

![pageable_result](/grammer/img/pageable_result.png)

### part1.Pageable - 테스트
    public ResponseEntity<BaseResponse> findMessageByChatRoomId(
        @PathVariable("id") Long chatRoomId,
        @RequestParam(value = "member", required = true) Long memberId,
        Pageable pageable) {

            log.inf("size = {}, page = {}, sorted = {}, pageable.getPageSize(), pageable.getPageNumber(), pageable.getSort());

            List<MessageResponseDto> message = chatService.findMessages(chatRoomId, memberId, pageable);

            return new ResponseEntity<>(new BaseResponse(message), HttpStatus.OK);
        }

    - 페이지 당 데이터 객수를 2개로 설정한 pagenation에서 0번째 페이지를 가져와 보자

![pageable_test](/grammer/img/pageable_test.png)

    - 로그에서도 정상적으로 Pageable 에 값이 넘어온것을 확인 가능하다.
        ex) log.info : size = 2, page = 0, sorted = UNSORTED  

### part2.Pageable - Pageable & search 
    part2에서는 세 부분으로 나누어 구현해보겠다.
    1. 검색
    2. 페이징 처리
    3. 검색 페이지에서 페이징

### part2.Pageable - search
    ● Repository
    public interface BoardRepository extends JpaRepository<Board, Long> {

        List<Board> findByTitleContaining(String keyword);
    } 

    - JpaRepository 에서 메서드명의 By이후는 SQL의 where 조건 절에 대응되는 것이다. 이렇게 Containing를 붙여주면 Like 검색이 가능해진다.           

    ● Service
    @Trancsactional
    public List<Board> search(Stirng keyword) {

        List<Board> boardList = boardRepository.findByTitleContaining(keyword);

        return boardList;
    }

    ● Controller
    @GetMapping("/board/search")
    public String search(String keyword, Model model) {

        List<Board> searchList = boardService.search(keyword);

        model.addAttribute("searchList", searchList);

        return "search/searchPage";
    }

    - 이 번 코드는 검색을 하게 되면 URL을 /board/search로 이동시킨 후 검색 리스트들을 뿌려주게 하려고 한다.
  
    ● searchPage.mustache
    
        - mustache
        Mustache는 간단하면서도 강력한 템플릿 엔진으로, 다양한 언어와 프레임워크에서 사용됩니다. 특히, 로직 없는 템플릿 엔진을 지향하며, 데이터와 템플릿을 분리하여 보다 유지보수가 용이한 코드를 작성할 수 있게 도와줍니다.

    {{>layout/header}}

    <table class="table">
        <thead class="thead-light">
        <tr>
            <th scope="col">#</th>
            <th scope="col">제목</th>
            <th scope="col">작성자</th>
            <th scope="col">작성시간</th>
        </tr>
        </thead>
        {{#searchList}}
            <tbody>
            <tr>
                <th scope="row">{{id}}</th>
                <td><a href="/{{id}}">{{title}}</a></td>
                <td>{{author}}</td>
                <td>{{createdTime}}</td>
            </tr>
            </tbody>
        {{/searchList}}
    </table>

    {{>search/searchForm}}

    {{>layout/footer}}    

    ● searchForm.mustache
    
        <form action="/board/search" method="GET">
            <div class="btn-group" role="group" aria-label="Basic example">
                <input name="keyword" type="text" placeholder="검색어를 입력해주세요">
                <button class="btn btn-secondary">검색</button>
            </div>
        </form>

    - 다음음 검색창이다. form data를 해당 주소로 GET하는 걸 볼 수 있다.
    input에 데이터를 입력하게 되면 Controller가 해당 데이터를 받아 지지고 볶은 다음에 searchPage에 보낸다. 그럼 searchPage가 데이터를 받아 화면을 구성한다.

![pageable_search_test](/grammer/img/pageable_search_test.png)

### part2.Pageable - paging
    ● Service
    @Transactional
    public Page<Board> getBoardList(Pageable pageable) {

        return boardRepository.findAll(pageable);
    }

    - 서비스에서 구현한 페이징 기능이다. 
        Pageable은 JpaRepository.findAll() 파라미터로 pageable만 넣어주면 된다.
    - 주의할 점은 리스트 타입을 Page로 해야 한다.

    ● Controler
    @GetMapping("/")
    public String index(Model model, @PageableDefault(size = 10, sort = "id", direction = Sort.Direction.DESC) Pageable pageable) {

        model.addAttribute("boardList", boardService.getBoardList(pageable));

        return "index";
    }   

    - @PageableDefault 어노테이션을 사용하면 정렬은 물론 페이징 처리, 페이지 사이즈까지 한 줄로 구현 가능하다.
    
    ● 결과

![paging_result](/grammer/img/paging_result.png)    

    - 페이지 사이즈가 10이고, id를 기준으로 내림차순 정렬되었다.

### part2.Pageable - 검색 페이지에서 페이징
    ● Repository
    public interface BoardRepository extends JpaRepository<Board, Long> {

        List<Board> findByTitleContaining(String keyword, Pageable pageable);
    }

    ● Service
    @Transactional
    public List<Board> search(String keyword, Pageable pageable) {

        List<Board> boardList = boardRepository.findByTitleContaining(keyword, pageable);

        return boardList;
    }

    - 그런데 위에 두 코드를 비교해보면 리스트 타입이 다르다.
      Repository에서는 List 타입이고, Service는 Page로 받는다.
      다른 이유는 JpaRepository가 제공하는 기본 메서드인 findAll()에 Pageable을 사용할 경우에는 리스트 타입을 Page로 해줘야 한다. 

    ● Controller
    @GetMapping("/board/search")  
    public String search(String keyword, @PageableDefault(size = 10, sort = "id", direction = Sort.Direction.DESC) Pageable pageable, Model model) {

        List<Board> searchList = boardService.search(keyword, pageable);

        model.addAttribute("searchList", searchList);

        return "search/searchList";
    }

### part3.Pageable - Mybatis에서 Pageable 적용    
    Spring Data에서 제공하는 Pageable을 사용하면 페이징 작업을 좀 더 간편하게 개발할 수 있다. JPA에서만 사용할 수 있는 것으로 생각했던 나였는데 구글링 중 MyBatis에서도 사용할 수 있는 기능이라는 것을 알게되었다. 이제 Mybatis에 적용한 pageable을 확인해 보자.

    ● 라이브러리 추가
    
        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-commons</artifactId>
        </dependency>

    ● RequestList class
    @Builder
    @Data
    public class RequestList<T> {

        private T data;
        private Pageable pageable;
    }        

    ● application.properties 변경
    # xml 파일의 parameter type, result type에 패키지명 생략 가능하도록 alias 설정
    mybatis.type-aliass-package-com.company.helloBoard.*.*.model, com.company.helloBoard.common.data

    ● Controller
    @RestController
    @RequestMapping(value = "/boards")
    public class BoardController {

        @Autowired
        BoardService boardService;

        @GetMapping("")
        public ResponseEntity<?> getListBoard(Board board, @PageableDefault(size = 10) Pageable pageable) {
            return ResponseEntity.OK(boardService.getListBoard(board, pageable));
        }
    }

    ● Service
    public interface BoardService {

        public Pageable<String, Object> getListBoard(Board board, Pageable pageable);
    }

    ● ServiceImpl
    public class BoardServiceImpl implements BoardService {

        @Autowired
        BoardMapper boardMapper; 

        @Override
        public Page<Map<String, Object>> getListBoard(Board board, Pageable pageable) {

            // 빌더 패턴으로 data, pageable 파라미터에 데이터 주입
            RequestList<?> requestList = RequestList.builder()
                            .data(board)
                            .pageable(pageable)
                            .build();

            List<Map<String, Objrect>> content = boardMapper.getListBoard(requestList);
            int total = boardMapper.getListsBoardCount(board);

            return new PageImpl<>(content, pageable, total);
        }
    }

    - 위에서 만든 ReqeustList class의 파일이 실제로 서비스단에서 쓰이는 부분이다. 빌더 패턴으로 data, pageable 파라미터에 데이터를 주입하나 requestList 변수에는 아래와 같이 값이 세팅된다.

![pageable_requestList](/grammer/img/pageable_requestList.png)

    - PageRequest 객체는 Pageable interface를 상속받고 있는데, 이 정보에는 정렬 정보, 페이지 offset, page와 같은 정보가 담겨있따. 또한 리턴 타입을 Page<T>으로 설정했는데 이는 일반적인 게시판 형태의 페이징에서 사용된다.
    
    ● Mapper
    @Mapper
    public interface BoardMapper {

        List<Map<String, Object>> getListBoard(RequestList<?> requestList);

        int getListBoardCount(Board board);
    }

    ● XML
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
	    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
    <mapper namespace="com.company.helloBoard.domain.board.mapper.BoardMapper">

        <select id="getListBoard parameterType="RequestList" resultType="Board">
            SELECT BOARD_ID
                    , WRITER
                    , TITLE
                    , CONTETN
                    , REG_DATE
                    , UPDATE_DATE
                    , DELETE_DATE
            FROM BOARD
            <where>
                <if test="data.title != null and data.title != ''">
                    AND TITLE LIKE '%' || #{data.title} || '%'
                </if>
                <if test="data.writer != null and data.writer != ''">
                    AND WRITER LIKE '%' || #{data.writer} || '%' 
                </if>       
            </where>
        OFFSET #{pageable.offset} ROWS FETCH NEXT #{pageable.pageSize} ROW ONLY
        </select>

        <select id="getListBoardCount" parameterType = "Board" resultType = "int">
            SELECT COUNT(*) AS CNT
            FROM BOARD
            <where>
                <if test="title != null and title != ''">
                    AND TITLE LIKE '%' || #{title} || '%'
                </if>
                 <if test="writer != null and writer != ''">
			        AND WRITER LIKE '%' || #{writer} || '%'
		        </if>
		    </where>    
        </select>     
    </mapper>

    - xml 파일에서 parameterType = "RequestList"으로 변경했다.
    - 서비스단에서 data에는 Board 정보를 담았고,
    - pageable에는 Pageable 정보를 담았기 때문에 data.필드, pageable.필드 같은 방법으로 접근이 가능하다.
    
    별다른 설정 없이 Pageable이 제공하는 offset, pageSize를 이용하여 간단하게 페이징을 구현해보았다.

    ● page.js
    const PAGE = {
        paging: function(totalPageCount, pageNo, totalElementCount, fn) {

            if (totalElementCount == 0) {
                document.querySelector("#pagingArea").innerHTML = "";
                return false;
            }

            let pageBlock = 10;
            let blockNo = PAGE.toInt(pageNo / pageBlock) + 1;
            let startPageNo = (blockNo - 1) * pageBlock;
            let endPageNo = blockNo * pageBlock - 1;

            if (endPageNo > totalPageCount - 1) {
                endPageNo = totalPageCount - 1;
            }

            let prevBlockPageNo = (blockNo - 1) * pageBlock - 1;
            let nextBlockPageNo = blockNo * pageBlock;

            let strHTML = "";

            // <, << 활성화/비활성화 처리
            if (prevBlockPageNo >= 0) {
                // <, << 활성화
                strHTML += "<li><a href='javascript:" + fn + "(" + 0 + ");' ><span class='icon page_prev_on'></span></a></li>";
                strHTML += "<li><a href='javascript:" + fn + "(" + prevBlockPageNo + ");' ><span class='icon page_left_on'></span></a></li>";
            } else {
                // <, << 비활성화
                strHTML += "<li><a><span class='icon page_prev_off'></span></a></li>";
                strHTML += "<li><a><span class='icon page_left_off'></span></a></li>";
            }

            // 페이징 번호 생성
            for (let i = startPageNo; i <= endPageNo; i++) {
                if (i == pageNo) {
                    strHTML += "<li class='active'><a>" + (i + 1) + "</a></li>";
                } else {
                    strHTML += "<li><a href='javascript:" + fn + "(" + i + ");' >" + (i + 1) + "</a></li>";
                }
            }

            // >, >> 활성화/비활성화 처리
            if (nextBlockPageNo < totalPageCount) {
                // >, >> 활성화
                strHTML += "<li><a href='javascript:" + fn + "(" + nextBlockPageNo + ");' ><span class='icon page_right_on'></span></a></li>";
                strHTML += "<li><a href='javascript:" + fn + "(" + (totalPageCount - 1) + ");' ><span class='icon page_next_on'></span></a></li>";
            } else {
                // >, >> 비활성화
                strHTML += "<li><a><span class='icon page_right_off'></span></a></li>";
                strHTML += "<li><a><span class='icon page_next_off'></span></a></li>";
            }

            let element = document.querySelector("#pagingArea");
            element.innerHTML = strHTML;
        },

        toInt: function(value) {
            if (value != null) {
                return parseInt(value, 10);
            }
        },

        pageRowNumber: function(pageNo, pageSize, index, totalCount) {
            debugger;
            if (totalCount) {
                return totalCount - ((pageNo) * pageSize + index);
            } else {
                return (pageNo) * pageSize + (index + 1);
            }
        }
    }

    ● list.html    
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">

    <head>
        <meta charset="UTF-8">
        <title>게시판</title>
        
        <!-- ✅ 추가 -->
        <link type="text/css" href="/css/style.css" rel="stylesheet">
        <script type="text/javascript" src="/js/com-ajax.js"></script>
        <script type="text/javascript" src="/js/com-page.js"></script>
    </head>
    <script type="text/javascript">

        // 페이지 온로드시 게시글 목록 조회
        window.onload = function () {
            getList();
        }

        // 게시글 목록 조회
        function getList(pageNo) {
            
            // ✅ 페이지번호 추가 
            pageNo = pageNo || 0;
            
            const title = document.querySelector("#title").value;
            const writer = document.querySelector("#writer").value;
            
            // ✅ 파라미터 전달시에도 추가
            const data = "?title=" + title + "&writer=" + writer + "&page=" + pageNo;
            
            AJAX.ajaxCall("GET", "/boards", data, afterGetList);
        }
        
        // 조회 후 처리
        function afterGetList(response) {
            
            // ✅ 페이징 처리
            PAGE.paging(response.totalPages, response.number, response.totalElements, "getList");
            
            // 결과 테이블 생성
            resultTable(response);
        }

        // 동적으로 테이블 생성
        function resultTable(response) {
            document.querySelector("#fieldListBody").innerHTML = "";
            
            if (response.size > 0) {
                const content = response.content;
                
                // ✅ 반복문 변경 (Pageable 결과값을 기준으로 값 가져오기 위함)
                for (var i = 0; i < content.length; i++) {
                    let element = document.querySelector("#fieldListBody");
                    
                    let result = content[i];
                    let template = `
                        <td><p>${PAGE.pageRowNumber(response.number, response.size, i, response.totalElements)}</p></td>
                        <td><p>${result.title}</p></td>
                        <td><p>${result.writer}</p></td>
                        <td><p>${result.regDate}</p></td>
                    `;
                    element.insertAdjacentHTML('beforeend', template);
                }
            }
        }
        
        // 초기화
        function resetList() {
            document.querySelector("#title").value = "";
            document.querySelector("#writer").value = "";
            document.querySelector("#fieldListBody").innerHTML = "";
            
            getList();
        }
        
    </script>
    <body>
        <div>
            <h2>게시판 목록</h2>
            <table>
                <tr>
                    <th>제목</th>
                    <td><input type="text" id="title"></td>
                </tr>
                <tr>
                    <th>작성자</th>
                    <td><input type="text" id="writer"></td>
                </tr>
                <tr>
                    <td><button onclick="getList()">조회</button></td>
                    <td><button onclick="resetList()">초기화</button></td>
                </tr>
            </table>
            <!-- ✅ 페이지 요소 추가 -->
            <input type="hidden" name="page" id="page" value="0" />
        </div>
        <div>
            <table>
                <colgroup>
                    <!-- ✅ 페이지 번호 추가 -->
                    <col width="150px" />
                    <col width="150px" />
                    <col width="150px" />
                    <col width="250px" />
                </colgroup>
                <thead>
                    <tr>
                        <!-- ✅ 페이지 번호 추가 -->
                        <th>No.</th>
                        <th>제목</th>
                        <th>작성자</th>
                        <th>작성시간</th>
                    </tr>
                </thead>
                <tbody id="fieldListBody">
                </tbody>
            </table>
            <!-- ✅ 페이징 표시되는 부분 추가 -->
            <ul id = "pagingArea" class="pagination"></ul>
        </div>
    </body>
    </html>

    - Pageable로 가져온 페이징 정보들을 기반으로 com-page.js 파일에서 pagingArea 변수에 동적으로 페이징 정보를 추가했다.

    여기까지 추가한 결과는 아래와 같다. 만약 1,2,... 10까지의 블록을 조절하고 싶다면 pageBlock 변수에 값을 변경하면 된다.

![pageable_mybatis_result](/grammer/img/pageable_mybatis_result.png)    