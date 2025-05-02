## 본문

### part1 - paging(MyBatis 적용)

### 1.페이징(Paging)이란?
    - 페이징은 사용자에게 데이터를 제공할 때, 전체 데이터 중의 일부를 보여주는 방식이다.
  
### 2.페이징 파라미터 관리용 클래스 추가하기
    page : 현재 페이지 번호를 의미, 페이지 정보 계산에 사용된다.
    recordSize : 페이지당 출력할 데이터 개수를 의미하며, page와 함께 페이지 정보 계산에 사용된다.
    pageSize : 화면 하단에 출력할 페이지의 크기를 의미하며, 5로 지정하면 1~5까지, 10으로 지정하면 1~10까지의 페이지가 보이게 된다.
    keyword : 검색 키워드를 의미하며, MyBatis의 동적(Dynamic) SQL 처리에 사용된다.
    searchType : 검색 유형을 의미하며, keyword와 함께 검색처리에 사용된다.

    - 페이징과 검색만 해도, view에서 받을 파라미터가 5개나 됩니다.
    이처럼 적지 않은 수의 파라미터는 클래스로 관리하는 게 효율적입니다.

    ● SearchDto
    @Data
    public class SearchDto {

        private int page;             // 현재 페이지 번호
        private int recordSize;       // 페이지당 출력할 데이터 개수
        private int pageSize;         // 화면 하단에 출력할 페이지 사이즈
        private String keyword;       // 검색 키워드
        private String searchType;    // 검색 유형

        public SearchDto() {
            this.page = 1;
            this.recordSize = 10;
            this.pageSize = 10;
        }

        public int getOffset() {
            return (page - 1) * recordSize;
        }

    }
    - 생성자 : 객체가 생성되는 시점에 현재 페이지 번호는 1로, 페이지당 출력할 데이터 개수와 하단에 출력할 페이지 개수를 10으로 초기화합니다.
    - getOffset() : MariaDB에서 LIMIT 구문의 시작 부분에 사용되는 메서드입니다. SQL 쿼리를 작성한 후 해당 메서드가 어떻게 사용되는지 알아보자.

### 추가 : PagingVO (페이징 로직을 컨트롤러, 서비스가 아닌 VO에서 처리)
    public class PagingVO {

        // 현재 페이지, 시작 페이지, 끝 페이지, 게시글 총 갯수, 페이지당 글 갯수, 마지막 페이지, SQL쿼리에 쓸 start, end
        private int nowPage, startPage, endPage, total, cntPerPage, lastPage, start, end;
	    private int cntPage = 5;

        public PagingVO() {            
        }

        public PagingVO(int total, int nowPage, int cntPerPage) {
            setNowPage(nowPage);
            setCntPerPage(cntPerPage);
            setTotal(total);
            calcLastPage(getTotal(), getCntPerPage());
            calcStartEndPage(getNowPage(), cntPage);
            calcStartEnd(getNowPage(), getCntPerPage());
        }
        // 제일 마지막 페이지 계산
        public void calcLastPage(int total, int cntPerPage) {
            setLastPage((int) Math.ceil((double)total / (double)cntPerPage));
        }
        // 시작, 끝 페이지 계산
        public void calcStartEndPage(int nowPage, int cntPage) {
            setEndPage(((int)Math.ceil((double)nowPage / (double)cntPage)) * cntPage);
            if (getLastPage() < getEndPage()) {
                setEndPage(getLastPage());
            }
            setStartPage(getEndPage() - cntPage + 1);
            if (getStartPage() < 1) {
                setStartPage(1);
            }
        }
        // DB 쿼리에서 사용할 start, end값 계산
        public void calcStartEnd(int nowPage, int cntPerPage) {
            setEnd(nowPage * cntPerPage);
            setStart(getEnd() - cntPerPage + 1);
        }
    }

### 3.Mapper와 XML Mapper 수정하기
    ● Mapper 
    // 게시글 리스트 조회
    List<PostResponse> findAll(SearchDto params);

    // 게시글 수 카운팅
    int count(SearchDto params);

    ● Mapper.xml
    <!-- 게시글 리스트 조회 -->
    <select id="findAll" parameterType="com.study.common.dto.SearchDto" 
    resultType="com.study.domain.post.PostResponse">
        SELECT
            <include refid="pstColumns" />
        FROM
            tb_post
        WHERE
            delete_yn = 0
        ORDER BY
            id DESC
        LIMIT #{offset}, #{recordSize}
    </select>

    <!-- 게시글 수 카운팅 -->
    <select id="count" parameterType="com.study.common.dto.SearchDto" resultType="int">
        SELECT 
            COUNT(8)
        FROM
            tb_post
        WHERE
            delete_yn = 0
    </select>

    ● LIMIT
    LIMIT 구문은 SELECT 쿼리와 함께 사용되며, 반환되는 데이터의 개수를 지정할 수 있다.
    - offset : MyBaits에서 쿼리의 parameterType이 클래스의 객체인 경우, XML Mapper의 #{parameter}는 맴버 변수의 getter에 해당됩니다.
    쉽게 말해, get() 메서드를 이용해서 쿼리에 파라미터를 바인딩하는 개념입니다.
    
    findAll 쿼리에서 offset은 SearchDto 클래스의 getOffset() 메서드가 리턴하는, (page - 1) * recordSize를 계산한 값입니다.

        ex) page=3, recordSize를 10으로 가정해보면 (3-1) * 10 = 20입니다. 
        즉, 현재 페이지 번호가 3이라면 "LIMIT 20,10"으로 쿼리가 실행되며, 결론적으로 offset은 조회할 데이터의 시작 위치(몇 번째 데이터부터 조회할 것인지)를 의미.
    
    - recordSize : 시작 위치(offset)를 기준으로 조회할 데이터의 개수를 의미합니다.
  
### 4.Service 수정하기
    public List<PostResponse> findAllPost(final SearchDto params) {
        return postMapper.findAll(params);
    }

### 5.Controller 수정하기
    // 게시글 리스트 페이지
    @GetMapping("/post/list.do")
    public String openPostList(@ModelAttribute("params") final SearchDto params, Model model) {
        List<PostResponse> posts = postService.findAllPost(params);
        model.addAttribute("posts", posts);
        return "post/list";
    }

### 6.자가 복사 이용
    1. 게시글(tb_post) 테이블 초기화 하기
    페이징 테스트를 위해서는 대량의 데이터가 필요하다. 이럴 때 자가 복사를 이용하면 되는데요, 
    DBMS 툴에서 게시글 테이블을 초기화하는 다음의 명령어를 실행.
    (TRUNCATE는 테이블의 모든 데이터를 DELETE하고, AUTO_INCREMENT를 1로 초기화한다.)    
        ex) TRUNCATE tb_post;

    2. 게시글 1,000개 등록하기
    PostServiceTest에 다음의 테스트 코드를 작성한 후 실행해 주면된다. 
    PostMapperTest에서 호출하는 PostMapper의 findAll()의 구조가 변경되어서 오류가 발생할 테니, PostMapperTest를 전체 주석 처리한 후 실행.
        
        @Test
        void saveByForeach() {
            for (int i = 1; i <= 1000; i++) {
                PostRequest params = new PostRequest();
                params.setTitle(i + "번 게시글 제목");
                params.setContnt(i + "번 게시글 내용");
                params.setWriter("테스터" + i);
                params.setNoticeYn(false);
                postService.savePost(params);
            }
        }
        - 결과 : insert_id = 1000

    3. 테이블 구조 확인하기
    이제 테이블에 대량의 데이터를 INSERT 해주면 된다. 자가 복사를 이용할 때는 테이블 구조를 확인해서 필수로 값이 입력되어야 하는 칼럼을 우선적으로 체크해야 한다. 
    tb_post 테이블은 전체 칼럼 중 id와 날짜를 제외한 모든 칼럼에 필수로 값이 입력되어야 한다.

![paging_table_structure](/grammer/img/paging_table_structure.png)   

    4.자가 복사 쿼리 실행하기
    DBMS 툴에서 다음의 SQL 쿼리를 실행하면 된다. 쿼리를 실행할 때마다(테이블 전체 데이터 * 2)만큼의 데이터가 INSERT 된다.

        ex) INSERT INTO tb_post 
            (title, content, writer, view_cnt, notice_yn, delete_yn)
            (SELECT title, content, writer, view_cnt, notice_yn, delete_yn FROM tb_post WHERE delete_yn = 0);

    다음은 자가 복사 쿼리를 5번 실행한 결과입니다.

![paging_selfCopy](/grammer/img/paging_selfCopy.png)

### 7.리스트 페이지 테스트 
    1. 쿼리 스트링 파라미터가 없는 경우
    리스트 페이지에 접속해 보면 가장 마지막으로 등록된 데이터를 기준으로 10건이 출력된다.

![no_query_parameter](/grammer/img/no_query_parameter.png)

    여기서 포인트는 findAll 쿼리의 LIMIT 구문이다. LIMIT의 offset은 page(현재 페이지 번호)와 recordSize(페이지당 출력할 데이터 개수)를 기준으로 계산되는데, 따로 전달한 파라미터가 없기 때문에 리스트 페이지에 접근하는 시점에 SearchDto 클래스의 생성자에 의해 page는 1로, recordSize는 10으로 객체가 생성된다.

![param_result](/grammer/img/param_result.png)

    실행된 쿼리의 LIMIT 구문은 getOffset() 과 getRecordSize()의 결과값인 (0, 10)으로 처리된다.

![findAll_query](/grammer/img/findAll_query.png)

    2. 쿼리 스트링 파라미터가 있는 경우
    이번에는 URL에 강제로 파라미터(page=5, recordeSize=10)를 연결한 후 리스트 페이지로 접근한 결과이다.

![query_parameter](/grammer/img/query_parameter.png)

    page와recordSize를 파라미터로 전달했기 때문에 LIMIT 구문의 offset과 recordSize에 변화가 생긴다.

![param_result2](/grammer/img/param_result2.png)  
![findAll_query2](/grammer/img/findAll_query2.png)

### 8.Pagination 처리용 클래스 추가하기
    브라우저에서 URL에 강제로 파라미터를 연결했을 때 리스트 데이터가 정상적으로 출력되는 걸 확인했으니, 지금부터는 실전이다.
    웹에서 화면 하단에 페이지 번호를 출력하는 기능을 "Pagination" 이라고 한다. 앞에서 했듯이, SearchDto의 맴버 변수를 이용해서 페이지 정보를 계산한다.

        ex) pagination
        @Getter
        public class Pagination {
            private int totalRecordCount;   // 전체 데이터 수
            private int totalPageCount;     // 전체 페이지 수
            private int startPage;          // 첫 페이지 번호
            private int endPage;            // 끝 페이지 번호
            private int limitStart;         // LIMIT 시작 위치
            private boolean existPrevPage;  // 이전 페이지 존재 여부
            private boolean existNextPage;  // 다음 페이지 존재 여부

            public Pagination(int totalRecordCount, SearchDto params) {
                if (totalRecordCount > 0) {
                    this.totalRecordCount = totalRecordCount;
                    calculation(params);
                }
            }

            private void calculation(SearchDto params) {

                // 전체 페이지 수 계산
                totalPageCount = ((totalRecordCount - 1) / params.getRecordSize()) + 1;

                // 현재 페이지 번호가 전체 페이지 수보다 큰 경우, 현재 페이지 번호에 전체 페이지 수 저장
                if (params.getPage() > totalPageCount) {
                    params.setPage(totalPageCount);
                }

                // 첫 페이지 번호 계산
                setarPage = ((params.getPage() - 1) / params.getPageSize()) * params.getPageSize() + 1;

                // 끝 페이지 번호 계산
                endPage = startPage + params.getPageSize() -1;

                // 끝 페이지가 전체 페이지 수보다 큰 경우, 끝 페이지 전체 페이지 수 저장
                if (endPage > totalPageCount) {
                    endPage = totalPageCount;
                }

                // LIMIT 시작 위치 계산
                limitStart = (params.getPage() - 1) * params.getRecordSize();
                // 이전 페이지 존재 여부 확인
                existPrePage = startPage != 1;

                // 다음 페이지 존재 여부 확인
                existNextPage = (encPage * params.getRecordSize()) < totalRecordCount;
            }
        }

        ● 클래스 구성요소 설명
        - totalRecordCount 
        COUNT(*) 쿼리의 실행 결과, 즉 전체 게시글 개수를 의미한다. 당장은 테이블에서 삭제되지 않은 데이터를 기준으로 카운팅 하지만, 
        검색 기능이 적용된 후에는 검색 조건의 유무에 따라 필터링해서 카운팅 한다.

        - totalPageCount 
        페이지 하단에 출력할 전체 페이지 개수를 의미한다. 테이블에 1,000개의 데이터(record)가 있고, 
        recordSize(페이지당 출력할 데이터 개수)가 10개라고 가정했을 때, 전체 페이지 개수는 (1,000/10)의 결과인 100개가 된다.

        - startPage
        현재 페이지네이션의 첫 페이지를 의미한다. pageSize(화면 하단에 출력할 페이지 개수)가 10이고, page(현재 페이지 번호)가 5라고 가정했을 때 1을 의미한다.

        - endPage
        현재 페이지네이션의 끝 페이지를 의미한다.pageSize(화면 하단에 출력할 페이지 개수)가 10이고, page(현재 페이지 번호)가 5라고 가정했을 때 10을 의미한다.
        다른 예로 page가 15일 때 startPage는 11이 되고, page가 29일 때 sartPage는 21이 된다.

        - limimStart
        현재 findAll 쿼리의 LIMIT 구문에 사용되는 offset과 동일한 기능을 하는 변수이다. SearchDto의 offset을 대신해서 LIMIT 구문의 첫 번째 인자로 사용된다.

        - existPrevPage
        이전 페이지의 존재 여부를 확인한다.
        현재 위치한 페이지에서 startPage(첫 페이지 번호)가 1이 아니라는 건, 이전 페이지가 존재한다는 걸 의미한다.

        - existNextPage
        다음 페이지의 존재 여부를 확인한다.
        recordSize(페이지당 출력할 데이터 개수)가 10개, endPage(끝 페이지 번호)가 10이라고 가정했을 때 (10 * 10) = 100이라는 결과가 나온다.
        만약 totalRecordCount(전체 데이터 개수)가 105개라면, 다음 페이지가 존재한다는 걸 의미한다.

### 9.XML Mapper 수정하기
    <!-- 게시글 리스트 조회 -->
    <select id="findAll" parameterType="com.study.common.dto.SearchDto" resultType="com.study.domain.post.PostResponse">
        SELECT 
            <include refid="postColumns" />
        FROM
            tb_post
        WHERE
            delete_yn = 0
        ORDER BY 
            id DESC
        LIMIT #{pagination.limitStart}, #{recordSize}
    </select>

### 10.SearchDto 수정
    findAll 쿼리에서는 pagination 객체의 limitStart를 사용하고 있으나, 
    SearchDto에는 Pagination 타입의 멤버가 없으니, 
    SearchDto가 Pagination 타입의 멤버를 갖도록 선언해 주어야 한다.

        ex) SearchDto
        @Data
        public class SearchDto {

            private int page;
            private int recordSize;
            private int pageSize;
            private String keyword;
            private String searchType;
            private Pagination pagination // 기존 게시글에 pagination(하단 번호) 추가

            public SearchDto() {
                this.page = 1;
                this.recordSize = 10;
                this.pageSize = 10;
            }
        }

### 11.Paging 전용 Response 클래스 추가하기
    화면에 페이지 번호를 그리는 작업은 view 단에서 이루어져야 한다. 이때 list.html은 리스트 데이터와 Pagination 객체 모두를 필요로 한다.

    그런데, 현재 PostService의 findAllPost()의 리턴 타입은 List<PostResponse>이다. 
    Pagination 객체를 생성해서 페이지 정보를 계산하는 것까지는 문제가 되지 않지만, 
    현재의 리턴 타입으로는 리스트 데이터와 Pagination 객체 모두를 컨트롤러로 반환할 수가 없다.

    이를 해결하기 위해, key-value 구조로 이루어진 Map에 리스트 데이터와 Pagination 객체를 담아 리턴해 주어도 되지만, 
    좀 더 직관적인 처리를 위해 페이징 전용 응답 클래스를 생성해 보자.

        ex) PagingResponse<T> {

            private List<T> list = new ArrayList<>();
            private Pagination pagination;

            public PagingResponse(List<> list, Pagination pagination) {
                this.list.addAll(list);
                this.pagination = pagination;
            }
        }

        - list : java의 제네릭을 활용, T는 Type을 의미하며, 어떤 타입의 객체던 데이터로 받겠다는 의미입니다.

### 12.Service 수정
    이제 PostService의 findAllPost()에 페이지 정보를 계산하는 로직이 추가되어야 한다. 
    계산된 페이지 정보를 기준으로 findAll 쿼리를 실행하고, 
    PagingResponse 클래스를 이용해서 리스트 데이터와 계산된 페이지 정보를 함께 리턴해 주어야 한다.

    /**
     *  게시글 리스트 조회
     *  @param params - search conditions 
     *  @return list & pagination information
     */      
    public PagingResponse<PostResponse> findAllPost(final SearchDto params) {
        
        // 조건에 해당하는 데이터가 없는 경우, 응답 데이터에 비어있는 리스트와 null을 담아 반환
        int count = postMapper.count(params);
        if (count < 1) {
            return new PagingResponse<>(Collections.emptyList(), null);
        }

        // Pagination 객체를 생성해서 페이지 정보 계산 후 SearchDto 타입의 객체인 params에 계산된 페이지 정보 저장.
        Pagination pagination = new Pagination(count, params);
        params.setPagination(pagination);

        // 계산된 페이지 정보의 일부(limitStart, recordSize)를 기준으로 리스트 데이터 조회 후 응답 데이터 반환
        List<PostResponse> list = postMapper.findAll(params);
        return new PagingResponse<>(list, pagination);
    }

### 13.Controller 수정
    마지막으로 PostController의 openPostList()r가 view로 전달하는 데이터만 변경해 주면 된다.

        ex) Controller
        // 게시글 리스트 페이지
        @GetMapping("/post/list.do")
        public String openPostList(@ModelAttribute ("param") final SearchDto params, Model model) {
            PagingResponse<PostReposne> response = postService.findAllPost(params);
            model.addAttribute("response", repseon);
            return "post/list";
        }

### 14.HTML 수정하기
    이제, Pagination 객체를 이용해서 화면 하단에 페이지 번호를 그리고, 번호를 클릭했을 때 번호에 해당하는 페이지로 이동하는 기능만 구현하면 된다.

    Thymeleaf를 이용해서 HTML을 그려도 되지만, 좀 더 깨끗한 소스 코드를 작성하기 위해 JS를 이용해서 리스트 데이터와 페이지 번호를 그려보자.

    우선 list.html을 다음과 같이 변경해주자. Diffchecker를 이용하면, 기존 코드와 변경된 코드를 쉽게 비교해 볼 수 있다.        
        
    ● list.html(Thymeleaf를 이용한 paging)

    <!DOCTYPE html>
    <html lang="ko" xmlns:th="http://www.thymeleaf.org" xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout" layout:decorate="layout/basic">
        <th:block layout:fragment="title">
            <title>리스트 페이지</title>
        </th:block>

        <th:block layout:fragment="content">
            <script th:inline="javascript">
                /*<![CDATA[*/

                window.onload = () => {
                    // 페이지가 로드되었을 때, 딱 한 번만 함수를 실행
                    findAllPost();
                }


                // 게시글 리스트 조회
                function findAllPost() {

                    // 1. PagingResponse의 멤버인 List<T> 타입의 list를 의미
                    const list = [[ ${response.list} ]];

                    // 2. 리스트가 비어있는 경우, 행에 "검색 결과가 없다"는 메시지를 출력하고, 페이지 번호(페이지네이션) HTML을 제거(초기화)한 후 로직을 종료
                    if ( !list.length ) {
                        document.getElementById('list').innerHTML = '<td colspan="6"><div className="no_data_msg">검색된 결과가 없습니다.</div></td>';
                        drawPage();
                    }

                    // 3. PagingResponse의 멤버인 pagination을 의미
                    const pagination = [[ ${response.pagination} ]];

                    // 4. @ModelAttribute를 이용해서 뷰(HTML)로 전달한 SearchDto 타입의 객체인 params를 의미
                    const params = [[ ${params} ]];

                    // 5. 리스트에 출력되는 게시글 번호를 처리하기 위해 사용되는 변수 (리스트에서 번호는 페이지 정보를 이용해서 계산해야 함)
                    let num = pagination.totalRecordCount - ((params.page - 1) * params.recordSize);

                    // 6. 리스트 데이터 렌더링
                    drawList(list, num);

                    // 7. 페이지 번호 렌더링
                    drawPage(pagination, params);
                }


                // 리스트 HTML draw
                function drawList(list, num) {

                    // 1.렌더링 할 HTML을 저장할 변수
                    let html = '';

                    /*
                       2.기존에 타임리프(Thymeleaf)를 이용해서 리스트 데이터를 그리던 것과 유사한 로직
                         기존에는 게시글 번호를 (전체 데이터 수 - loop의 인덱스 번호)로 처리했으나, 
                         현재는 (전체 데이터 수 - ((현재 페이지 번호 - 1) * 페이지당 출력할 데이터 개수))로 정밀히 계산
                    */
                    list.forEach(row => {
                        html += `
                            <tr>
                                <td><input type="checkbox" /></td>
                                <td>${row.noticeYn === false ? num-- : '공지'}</td>
                                <td class="tl"><a href="/post/view.do?id=${row.id}">${row.title}</a></td>
                                <td>${row.writer}</td>
                                <td>${dayjs(row.createdDate).format('YYYY-MM-DD HH:mm')}</td>
                                <td>${row.viewCnt}</td>
                            </tr>
                        `;
                    })

                    // 3. id가 "list"인 요소를 찾아 HTML을 렌더링
                    document.getElementById('list').innerHTML = html;
                }


                // 페이지 HTML draw
                function drawPage(pagination, params) {

                    // 1. 필수 파라미터가 없는 경우, 페이지 번호(페이지네이션) HTML을 제거(초기화)한 후 로직 종료
                    if ( !pagination || !params ) {
                        document.querySelector('.paging').innerHTML = '';
                        throw new Error('Missing required parameters...');
                    }

                    // 2. 렌더링 할 HTML을 저장할 변수
                    let html = '';

                    // 3. 이전 페이지가 있는 경우, 즉 시작 페이지(startPage)가 1이 아닌 경우 첫 페이지 버튼과 이전 페이지 버튼을 HTML에 추가
                    if (pagination.existPrevPage) {
                        html += `
                            <a href="javascript:void(0);" onclick="movePage(1)" class="page_bt first">첫 페이지</a>
                            <a href="javascript:void(0);" onclick="movePage(${pagination.startPage - 1})" class="page_bt prev">이전 페이지</a>
                        `;
                    }

                    /*
                    * 4. 시작 페이지(startPage)와 끝 페이지(endPage) 사이의 페이지 번호(i)를 넘버링 하는 로직
                    *    페이지 번호(i)와 현재 페이지 번호(params.page)가 동일한 경우, 페이지 번호(i)를 활성화(on) 처리
                    */
                    html += '<p>';
                    for (let i = pagination.startPage; i <= pagination.endPage; i++) {
                        html += (i !== params.page)
                            ? `<a href="javascript:void(0);" onclick="movePage(${i});">${i}</a>`
                            : `<span class="on">${i}</span>`
                    }
                    html += '</p>';

                    // 5. 현재 위치한 페이지 뒤에 데이터가 더 있는 경우, 다음 페이지 버튼과 끝 페이지 버튼을 HTML에 추가
                    if (pagination.existNextPage) {
                        html += `
                            <a href="javascript:void(0);" onclick="movePage(${pagination.endPage + 1});" class="page_bt next">다음 페이지</a>
                            <a href="javascript:void(0);" onclick="movePage(${pagination.totalPageCount});" class="page_bt last">마지막 페이지</a>
                        `;
                    }

                    // 6. class가 "paging"인 요소를 찾아 HTML을 렌더링
                    document.querySelector('.paging').innerHTML = html;
                }


                // 페이지 이동
                function movePage(page) {

                    // 1. drawPage( )의 각 버튼에 선언된 onclick 이벤트를 통해 전달받는 page(페이지 번호)를 기준으로 객체 생성
                    const queryParams = {
                        page: (page) ? page : 1,
                        recordSize: 10,
                        pageSize: 10
                    }

                    /*
                    * 2. location.pathname : 리스트 페이지의 URI("/post/list.do")를 의미
                    *    new URLSearchParams(queryParams).toString() : queryParams의 모든 프로퍼티(key-value)를 쿼리 스트링으로 변환
                    *    URI + 쿼리 스트링에 해당하는 주소로 이동
                    *    (해당 함수가 리턴해주는 값을 브라우저 콘솔(console)에 찍어보시면 쉽게 이해하실 수 있습니다.)
                    */
                    location.href = location.pathname + '?' + new URLSearchParams(queryParams).toString();
                }

            /*]]>*/
            </script>

            <div class="page_tits">
                <h3>게시판 관리</h3>
                <p class="path"><strong>현재 위치 :</strong> <span>게시판 관리</span> <span>리스트형</span> <span>리스트</span></p>
            </div>

            <div class="content">
                <section>
                    <!--/* 검색 */-->
                    <div class="search_box">
                        <form id="searchForm" onsubmit="return false;" autocomplete="off">
                            <div class="sch_group fl">
                                <select title="검색 유형 선택">
                                    <option value="">전체 검색</option>
                                    <option value="">제목</option>
                                    <option value="">내용</option>
                                </select>
                                <input type="text" placeholder="키워드를 입력해 주세요." title="키워드 입력"/>
                                <button type="button" class="bt_search"><i class="fas fa-search"></i><span class="skip_info">검색</span></button>
                            </div>
                        </form>
                    </div>

                    <!--/* 리스트 */-->
                    <table class="tb tb_col">
                        <colgroup>
                            <col style="width:50px;"/><col style="width:7.5%;"/><col style="width:auto;"/><col style="width:10%;"/><col style="width:15%;"/><col style="width:7.5%;"/>
                        </colgroup>
                        <thead>
                            <tr>
                                <th scope="col"><input type="checkbox"/></th>
                                <th scope="col">번호</th>
                                <th scope="col">제목</th>
                                <th scope="col">작성자</th>
                                <th scope="col">등록일</th>
                                <th scope="col">조회</th>
                            </tr>
                        </thead>

                        <!--/* 리스트 데이터 렌더링 영역 */-->
                        <tbody id="list">

                        </tbody>
                    </table>

                    <!--/* 페이지네이션 렌더링 영역 */-->
                    <div class="paging">

                    </div>

                    <!--/* 버튼 */-->
                    <p class="btn_set tr">
                        <a th:href="@{/post/write.do}" class="btns btn_st3 btn_mid">글쓰기</a>
                    </p>
                </section>
            </div> <!--/* .content */-->
        </th:block>
    </html>
    - <tbody id="list"> : 게시글 리스트 데이터 HTML을 렌더링 할 영역입니다.
    기존에는 타임리프의 th:each를 이용해서 HTML을 그렸었고, 이번에는 자바스크립드를 이용해서 HTML을 그립니다.

### 16.Paging Test
    1. 게시글 리스트 페이지에서 5페이지를 클릭한 결과.
    - URL에 쿼리 스트링이 정상적으로 연결됨을 확인하면
      - localhost:8080/post/list.do?page=5&recordSize=10&pageSize=10 
      - 
![pagingTest](/grammer/img/pagingTest.png)

    2. 5페이지에서 다음 페이지(>)버튼을 세 번 클릭한 결과.
    
![pagingTest2](/grammer/img/pagingTest2.png)

    3. 마지막 페이지에서 이전 페이지(<) 버튼을 세 번 클릭한 결과.
    
![pagingTest3](/grammer/img/pagingTest3.png)    

### 마치며
    여기까지가 기본 적인 페이징 기능 구현입니다. 
    아직 개선해야 할 부분이 몇 가지 남아 있습니다.

    1.검색 유형(searchType)과 검색 키워드(keyword)를 이용한 검색 처리.
    2.SearchDto를 이용해서 이전 페이지 정보 유지하기.
        ex)
            25페이지에 있는 50번 게시글을 "수정/삭제" 하거나, 상세 또는 수정 페이지에서 "뒤로" 버튼을 클릭하면 다시 25페이지로 이동하는 기능.
            즉, 이전에 머무르던 페이지 정보가 유지되어야 한다.

### par2 - searchType & keyword 검색 처리.
    part1에서는 페이징과 검색 기능에 필요한 공통 클래스들을 정의하고,, 게시글 리스트 페이지에 페이징을 적용해보았다.
    
    이번에는 기존의 페이징에 게시글 검색 기능을 추가하고, 특정 페이지에서 어떠한 액션이 발생했을 때 이전 페이지 정보와 검색 조건을 유지시키는 기능을 구현해보겠습니다.

### 1.검색 처리 (Dynamic SQL)
    페이징과 검색 기능은 하나의 세트로 생각된다. 
    페이징은 전체 데이터를 카운팅 한 기준으로 1 ~ N개까지의 페이지를 보여주는 반면, 
    검색 기능은 SQL 쿼리에서 LIKE 검색 결과에 해당되는, 즉 필터링된 데이터를 카운팅 한 기준으로 1 ~ N개까지의 페이지를 보여준다.

    검색 기능은 이전 글에서 생성한 SearchDto 클래스의 검색 keyword, searchType, MyBatis의 Dynamic SQL 기능을 이용해 처리 한다.

### 2.HTML(검색 영역) 수정하기
    part1에서는 sever단 로직을 미리 구성해 두고 client을 처리하는 순서로 진행을 했는데, 이와 반대로 검색 기능은 화면 처리를 우선으로 해보겠습니다.

    ● list.html
    <!-- 검색 -->
    <div class="seach_box">
        <form id="searchForm" onsubmit="return false;" autocomplete="off">
            <div class="sch_group fi">
                <select id="seachType" name="searchType" title="검색 유형 선택">
                    <option value="">전체 검색</option>
                    <option value="title">제목</option>
                    <option value="content">내용</option>
                    <option value="writer">작성자</option>
                </select>
                <input type="text" 
                       id="keyword" 
                       name="keyowrd" 
                       placeholder="키워드를 입력해 주세요." 
                       title="키워드 입력" />
                <button type="button" 
                        class="bt_search" 
                        onclick="movePage(1);">
                        <i class="fas fa-search"></i>
                        <span class="skip_info">검색</span>
                </button>
            </div>            
        </form>                
    </div>

    - searchForm : 검색 form을 의미한다. form의 searchType, keyword를 SQL 쿼리의 검색 조건으로 이용해ㅓ 게시글을 검색한다.
    - movePage() : 검색 버튼에 연결된 클릭 이벤트입니다. movePage()는 페이지를 이동하는 기능을 하는데요, 
                   여기서 포인트는 검색 버튼을 클릭했을 때 movePage() 인자로 '1'을 전달한다는 것이며, 
                   검색 처리에서 현재 페이지 번호(page)는 항상 '1'로 유지되어야 한다.
  
    ● 페이지 번호가 1로 유지되어야 하는 이유1
    - A가 15페이지에서 게시글을 흝어보고 있따.
    - A는 B가 작성한 게시글을 보기 위해 '작성자'를 B로 해서 검색을 했다.
    - A는 B의 게시글을 조회하는 데 성공했으나 이상한 점이 있다. B가 가장 최근에 등록한 글부터 보고 싶은데, 현재 머물러 있는 페이지 번호가 15페이지인 것이다.
    
    포인트는 여기에 있다. 페이지 번호를 '1'이 아닌 기존에 머물러 있던 '15'로 전달한다면, 
    검색 조건에 해당하는 데이터가 있다고 하더라도, 중간에 등록된 글부터 보이거나, 조회 자체가 되지 않는 상황이 벌어질 수 있다.

    ● 페이지 번호가 1로 유지되어야 하는 이유2
    - 페이지당 출력할 데이터 개수가 10개라고 가정했을 때 
    - A가 작성한 게시글은 총 20개이고 '작성자'를 A로 검색하면, A의 게시글은 1페이지와 2페이지에서는 볼 수 있으나
    -  B가 기존에 머물러 있던, 3페이지를 기준으로 검색이 되어버리니 A의 게시글이 검색되지 않는 상황이 발생한 것이다.

### 3.movePage() 함수 수정하기
    SQL 쿼리의 검색 조건으로 사용하기 위해, 검색 영역의 searchType, keyword를 파라미터로 전달해 주어야 한다.

    ● 페이지 이동

        function movePage(page) {

            // 1. 검색 form
            const form = document.getElementById('searchForm');

            // 2. drawPage()의 각 버튼에 선언된 onclick 이벤트를 통해 전달받는 page(페이지 번호)를 기준으로 객체 생성
            const queryParams = {
                page : (page) ? page : 1,
                recordSize : 10,
                pageSize : 10,
                searchType : form.searchType.value,
                keyword : form.keyword.value
            }

            /*
                location.pathname : 리스트 페이지의 URI("/post/list.do")를 의미, 
                new URLSearchParams(queryParams).toString() : queryParams의 모든 프로퍼티 (key-value)를 
                쿼리 스트링으로 변환 URI + 쿼리 스트링에 해당하는 주소로 이동 (해당 함수가 리턴해주는 값을 브라우저 console에 찍어보자.)
             */

            location.href = location.pathname + '?' + new URLSearchParams(queryParams).toString(); 
        }   

        - 이제, list 페이지에서 searchType, keyword 를 세팅하고 검색해 보면, searchType, keyword가 파라미터로 함께 전송된다.
      
![param_result3](/grammer/img/param_result3.png)      

        - 쿼리 스트링 파라미터 연결 시,
          - locathost:8080/post/list.do?page=1&recordSize=10&pagSize=10&searchType=writer&keyword=테스터1000

### 4.MyBatis 동적 SQL
    마이바티스는 동적 SQL을 처리할 수 있는 몇 가지의 태그와 표현식을 제공한다. 
    일반적으로 조건문과 반복문이 주로 사용된다. 
    이 중에서도 조건문의 if, choose와 반복문의 foreach가 가장 많이 사용된다.

    반복문의 foreach는 대표적으로 SQL 쿼리에서 WHERE 조건의 IN() 구문에 주로 사용되는데, 
    이때 파라미터의 타입은 collection 타입이어야 한다.(List, Array, Map 등)      

        ex) foreach문
            <foreach collection="list" item="item" index="index" open="(" seperator="," cloas=")">#{item}</foreach>

### 5.XML Mapper 수정
    1. postMapper.xml
     <!-- 게시글 검색 -->
    <sql id="search">
        <!-- 검색 키워드가 있을 때 -->
        <if test="keyword != null and keyword != ''">
            <choose>
                <!-- 검색 유형이 있을 때 -->
                <when test="searchType != null and searchType != ''">
                    <choose>
                        <when test="'title'.equals( searchType )">
                            AND title LIKE CONCAT('%', #{keyword}, '%')
                        </when>
                        <when test="'content'.equals( searchType )">
                            AND content LIKE CONCAT('%', #{keyword}, '%')
                        </when>
                        <when test="'writer'.equals( searchType )">
                            AND writer LIKE CONCAT('%', #{keyword}, '%')
                        </when>
                    </choose>
                </when>
                
                <!-- 전체 검색일 때 -->
                <otherwise>
                    AND (
                           title LIKE CONCAT('%', #{keyword}, '%')
                        OR content LIKE CONCAT('%', #{keyword}, '%')
                        OR writer LIKE CONCAT('%', #{keyword}, '%')
                    )
                </otherwise>
            </choose>
        </if>
    </sql>

    ● 코드 설명
    keyword가 파라미터로 넘어온 경우에만 실행되는 검색 쿼리입니다. 
    searchType이 선택된 경우에는 각각의 <where> 조건에 해당되는 LIKE 쿼리가 실행되고, 
    전체 검색인 경우에는 <otherwise> 안에 선언한 LIKE 쿼리가 실행.

    2. 검색용 SQL 조각 include
    findAll 쿼리와 count 쿼리에서 search SQL을 include
    <!-- 게시글 리스트 조회 -->
    <select id="findAll" parameterType="com.study.common.dto.SearchDto"  resultType="com.study.domain.post.PostResponse">
        SELECT
            <include refid="postColumns" />
        FROM 
            tb_post
        WHERE
            delete_yn = 0
            <include refid="search" />
        ORDER BY
            id DESC
        LIMIT #{pagination.limitStar}, #{recordSize}
    </select>

    <!-- 게시글 수 카운팅 -->
    <select id="count" parameterType="com.study.common.dto.searchDto" resultTpe="int">
        SELECT 
            COUNT(*)
        FROM
            tb_post
        WHERE
            delete_yn = 0
            <include refid="search" />
    </select>

### 6.검색 기능 테스트
    전체 검색과 제목을 기준으로 검색 기능을 테스트한 결과이다.

![findAll_keyword_1000](/grammer/img/findAll_keyword_1000.png)

![count_query_test](/grammer/img/count_query_test.png)

![findAll_query_test](/grammer/img/findAll_query_test.png)

### 7.keryword 검색 조건 유지하기
    모든 경우에서 문제없이 검색 기능이 작동하고 있다. 하지만 검색 버튼을 클릭하면 검색 조건이 풀려버린다. 
    이를 해결하기 위해 쿼리 스트링 파라미터를 세팅해 주는 기능이 필요하다.

    ● list.html
    - setQueryStringParams() 함수를 추가
    - onload() 함수에서 리스트 데이터를 조회하기 전 해당 함수를 호출.

        // 페이지가 로드되었을 때, 딱 한 번만 함수를 실행
        window.onload = () => {

            setQueryStringParams();

            findAllPost();
        }

        // 쿼리 스트링 파라미터 셋팅
        function setQueryStringParams() {

            if (!location.search) {
                return false;
            }

            const form = document.getElemnentById('searchForm');

            new URLSearchParams(location.search).forEach((value, key) =>
            {
                if (form[key]) {
                    form[key].value = value;
                }
            })
        }

        ● 로직 해석
        - JS에서 location 객체의 search를 이용하면 쿼리 스트링 파라미터를 조회할 수 있다.

        - form은 리스트 페이지의 searchForm을 의미하며, 
        new URLSearchParams()함수의 인자로 현재 페이지의 쿼리 스트링을 전달해서 쿼리 스트링 문자열에 포함된 각 파라미터(key = value)를 객체화한 후, 
        searchType, keyword의 값을 searchForm에 세팅한다.
    
### 8.이전 페이지 정보 유지
    마지막으로 게시글을 '수정/삭제' 하거나, 상세 또는 수정 페이지에서 '뒤로' 버튼을 클릭했을 때 이전 페이지 정보가 유지되도록 해주면 페이징과 검색 처리는 모두 끝이 난다.

    1. 리스트 페이지 drawList() 함수 수정
    리스트 페이지에서 상세 페에지로 이동할 때 쿼리 스트링 파라미터를 전달하도록 list.html의 drawList()를 다음과 같이 변경해 주자.

    ● drawList() 수정
    // 리스트 HTML draw
    function drawList(list, num) {
    
        // 1. 렌더링 할 HTML을 저장할 변수
        let html = '';
    
        /*
            1. 기존에 타임리프(Thymeleaf)를 이용해서 리스트 데이터를 그리던 것과 유사한 로직
            기존에는 게시글 번호를 (전체 데이터 수 - loop의 인덱스 번호)로 처리했으나, 
            현재는 (전체 데이터 수 - ((현재 페이지 번호 - 1) * 페이지당 출력할 데이터 개수))로 정밀히 계산
         */
        list.forEach(row => {
            html += `
                <tr>
                    <td><input type="checkbox" /></td>
                    <td>${row.noticeYn === false ? num-- : '공지'}</td>
                    <td class="tl"><a href="javascript:void(0);" onclick="goViewPage(${row.id});">${row.title}</a></td>
                    <td>${row.writer}</td>
                    <td>${dayjs(row.createdDate).format('YYYY-MM-DD HH:mm')}</td>
                    <td>${row.viewCnt}</td>
                </tr>
            `;
        })
    
        // 3. id가 "list"인 요소를 찾아 HTML을 렌더링
        document.getElementById('list').innerHTML = html;
    }

    ● 코드 해석
    - <td class="tl"><a href="javascript:void(0);" onclick="goViewPage(${row.id});">${row.title}</a></td>
      - 기존에는 제목을 클릭했을 떼, <a> 태그의 href 속성을 이용해서 게시글  상세 페이지로 이동하는 구조였다. 
      변경된 코드에서는 href 속성을 무효화시키고, onclick 이벤트로 goViewPage() 함수를 호출한다.

    2. 리스트 페이지에 goViewPage() 함수 선언하기

        // 게시글 상세 페이지로 이동
        function goViewPage(id) {
            const queryString = (location.search) ? location.search + '&id=${id}` : `?id=${id}`;
            location.href = '/post/view.do' + queryStirng;
        }

        ● 코드 해석
        location.search를 이용해서 게시글 번호(id)와 쿼리 스트링 파라밑를 상세 페이지로 함께 전달한다.

        해당 함수에서 queryString에 삼항 연산자가 사용되었는데, 처음 리스트로 접근 했을때는 쿼리 스트링이 비어있는 상태가 되며, 
        이는 현재 페이지가 1페이지임을 의미한다. 이떄 location.search는 빈 문자열('')을 return하기 때문에 게시글 번호(id)만 쿼리 스트링으로 전달하고, 
        이외의 경우에는 페이지 정보, 검색 조건, 게시글 번호(id)를 함께 전달한다.

    3. 상세 페이지 버튼 수정
    이제, 상세 페이지에서 '수정' 버튼과 '뒤로' 버튼 클릭 시, 이전 페이지 정보가 유지되도록 구현.

    ● view.html
    <p class="btn_set">
        <button type="button" onclick="goWritePage();" class="btns btn_bdr4 btn_mid">수정</button>
        <button type="button" onclick="deletePost();" class="btns btn_bdr1 btn_mid">삭제</button>
        <button type="button" onclick="goListPage();" class="btns btn_bdr3 btn_mid">뒤로</button>
    </p>

    4. 상세 페이지 goWritePage() 함수 선언
    
        // 게시글 수정 페이지로 이동
        function goWritePage() {
            location.href = '/post/write.do' + location.seacrh;
        }

        ● 코드 해석
        앞에서 했듯이 location.search 는 URL의 쿼리 스트링을 return해준다.
        리스트 페이지에서 전달받은 쿼리 스트링을 있는 그대로 수정 페이지로 전달해 주면 되기 때문에, 
        URI 경로(Path)만 view.do에서 write.do로 변경되고, 나머지 쿼리 스트링은 그대로 가지고 간다.

    5. 상세 페이지에 goListPage() 함수 선언   
    
        // 게시글 리스트 페이지로 이동
        function goListPage() {
            const queryStirng = new URLSearchParams(location.search);
            queryString.delete('id');
            location.href = '/post/list.do' + '?' + queryString.toStirng();
        }

        ● 코드 해석
        new URLSearchParams()를 이용해서 쿼리 스트링을 객체화한다. 
        리스트 페이지는 게시글 번호(id)를 필요로 하지 않기 때문에, 
        delete() 함수를 이용해서 게시글 번호(id)를 삭제한 후 나머지 쿼리 스트링(이전 페이지 정보)을 리스트 페이지로 전달.

    6. 상세 페에지 deletePost() 함수 수정

        // 게시글 삭제
        function deletePost() {

            const id = [{$post.id}];

            if (!confirm(id + '번 게시글을 삭제할까요?')) {
                return false;
            }

            let inputHtml = '';

            new URLSearchParams(location.search).forEach((value, key)) => {
                inputHtml += '<input type="hidden" name="${key}" value="${value}" />';
            }

            const formHtml = `
            <form id="deleteForm" action="/post/delete.do" method="post">
                ${inputHtml}
            </form>
        
            const doc = new DOMParser().parseFromString(formHtml, 'text/html');
            const form = doc.body.firstChild;
            document.body.append(form);
            document.getElementById('deleteForm').submit();

        }          

        ● 코드 해석
        기존에는 deleteForm을 그릴 때 게시글 번호만 hidden으로 넘겼는데, 지금은 전달받은 쿼리 스트링 파라미터를 전부inputHtml에 담아서 form에 추가한다.

![deleteForm_structure](/grammer/img/deleteForm_structure.png)  

    7. PostController의 deletePost() 메서드 수정
        게시글을 삭제하면 게시글 번호(id)와 이전 페이지 정보(쿼리 스트링)가 함께 전송되기 때문에, 
        게시글이 삭제된 후에 이전 페이지 정보가 유지될 수 있도록 컨트롤러 구조를 변경.

        // 게시글 삭제
        @PostMapping("/post/delete.do")
        public String deletePost(@RequestParam final Long id, final SearchDto queryParams, Model model) {

            postService.deletePost(id);
            MessageDto message = new MessageDto("게시글 삭제가 완료되었습니다.", "/post/list.do", RequestMethod.GET, queryParamsToMap(queryParams));

            return showMessageAndRedirect(message, model);
    }

    8. PostController에 queryParamsToMap() 메서드 추가.
    
        // 쿼리 스트링 파라미터를 Map에 담아 반환.
        private Map<Stirng, Object> queryParamsToMap(final SearchDto queryParams) {

            Map<String, Object> data = new HashMap<>();
            data.put("page", queryParams.getPage());
            data.put("recordSize", queryParams.getRecordSize());
            data.put("pageSize", queryParams.getPageSize());
            data.put("keyword", queryParams.getKeyword());
            data.put("searchType", queryParams.getSearchType());

            return data;
        }

        ● 코드 해석
        해당 메서드는 deletePost()에서 수집한 이전 페이지 정보(queryParams의 모든 멤버)를 Map에 담아 return 해주는 역할을 한다.
        MessageDto의 생성자는 마지막 파라미터로 Map을 전달받아 객체를 생성하기 때문에 컨트롤러에 해당 메서드를 추가해 주었다.

        참고로, 해당 기능은 컨트롤러에서 alert 메시지 처리하기 기능의 연장선이다.

    9. 이전 페이지 정보 유지 테스트
 
![deleteForm_test](/grammer/img/deleteForm_test.png)

    - 15페이지 최상단의 180번 게시글 상세 페이지로 이동
    
![deleteForm_test2](/grammer/img/deleteForm_test2.png)

    - 상세 페이지에서 '뒤로' 버튼을 클릭하면, 상세 페이지로 이동하기 전의 검색 조건과 페이지 번호가 유지된다.

![deleteForm_test3](/grammer/img/deleteForm_test3.png)

    - 다음은 전체 검색으로 '테스트77'을 검색한 후 마지막(36) 페이지로 이동.
    
![deleteForm_test4](/grammer/img/deleteForm_test4.png)    

    - 36페이지의 2번 게시글 상세 페이지 삭제
     
![deleteForm_test5](/grammer/img/deleteForm_test5.png)

    - 2번 게시글이 삭제된 상태로, 이전 페이지 정보가 정상적으로 유지된다.
  
![deleteForm_test6](/grammer/img/deleteForm_test6.png)  
