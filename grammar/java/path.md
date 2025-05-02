## 본문

### 소개
    인프런 강의 정리 내용을 Readme.md 파일에 작성하여 시각적으로 필요한 정보를 img 폴더에 저장하여 깃허브 레퍼지토리에 올린 후, 이미지 파일 경로를 정리 파일에 작성하여 경로로 이미지를 출력하는 코드를 구현하였습니다.
    그러나 local에서나 깃허브에서 정리 파일에서 나와야 할 이미지는 출력이 되지 않아 전전긍긍하던 중, GPT에게 여러 반복된 질문을 통해 에러를 바로 잡을 수 있게 되었습니다. 

    해결된 과정을 보여 드리겠습니다.

    ● 프로젝트 폴더 구조
    Infleran_MVC1,2
        ㄴ servlet
            ㄴ servlet_img
                ㄴ servlet.png
            ㄴ servlet_sum    
                ㄴ servlet_HTTP.md    

    위에 구조가 프로젝트의 폴더 상태입니다.
    제가 이미지 출력을 위한 구조는 HTTP.md 파일 안에 ex) ![강의 정리](./servlet/servlet_img/servlet.png) 경로를 설정하여 servlet_img - servlet.png 이미지 파일을 참조해서 이미지를 출력을 목표로 하였습니다.

### 문제점
    그런데 원하는 이미지는 출력 되지 않았고, 문제점을 분석하기 시작했습니다. 분석한 결과 
    1. 경로 문제
       - GitHub에서는 파일 경로가 상대 경로로 지정.
       - 한글 경로가 인코딩되어 나타나는 경우, 경로가 잘못 인식.
    2. 경로를 다시 설정
       - 이미지가 실제로 저장소 내에 있는지 확인.
       - Markdown 파일과 이미지 파일의 정확한 상태 경로를 다시 확인.
    
### 해결과정
    - 2번 문제를 인지하고 'servlet_정리'라고 작성한 폴더 이름을 'servlet_sum'으로 바꾸어 주었지만, 이미지는 출력되지 않았습니다.
     
    - 가장 확률이 높을 것 같던, 1번 경로 문제는 여러 구글링 결과,
    servlet/serlvet_sum/ -> ./servlet/servlet_sum 형태로 바꾸라는 피드백이 가장 많아서 피드백을 적용하여 코드를 수정하였지만 해결되지 않았습니다.

### 해결
    마지막으로 본 피드백이 로컬과 깃허브의 경로는 다를 것 이다. 라는 글을 보고 GPT에게 두 폴더의 경로를 보여 주었고 이미지 폴더 경로를 참조해서 sum 파일에 해당 이미지를 출력하려고 한다고 질문을 주었더니,
    img  : Inflearn_MVC1,2\servlet\servlet_img
    sum  : Inflearn_MVC1,2\servlet\servlet_sum

    이 경로 구조를 기준으로 Markdown 파일에서 이미지 파일을 참조하는 경로는 다음과 같아야 합니다.
    ![Serlvet_Response](../servlet_img/servlet.png)
    위에 경로대로 수정하였고 결과는 이미지가 출력이 되었습니다.

### 결과
    ● Markdown 파일에서 경로 지정
    - 상대 경로 Servlet_HTTP.md 파일에서 servlet.png 파일을 참조하려면, 상위 디렉토리로 이동한 후 servlet_img 디렉토리로 이동해야 한다.
    
        ex) ![Servlet_Response](../servlet_img/servlet.png)
        
        이 경로는 Servlet_HTTP.md 파일에서 시작하여, 상위 디렉토리로 이동한 다음 servlet_img 디렉토리의 servlet.png 파일을 가리킵니다.
        
        따라서, (../servlet_img/servlet.png)는 servlet/servlet_img/servlet.png와 동일한 파일을 가리키고 있습니다.

    - Serlvet_HTTP.md 파일에서 현재 디렉토리(.servlet/servlet_img/servlet.png)를 기준으로 경로를 지정하면 이미지 파일을 찾을 수 없다.

    - Markdown 파일과 이미지 파일이 프로젝트 내의 서로 다른 위치에 있을 때 이를 올바르게 참조해야 한다. 

    - 상대 경로와 현재 경로의 차이점을 이해하는 것이 중요.

### 정리
    이러한 경험을 토대로 내가 아직 './'(현재 경로), '../(상위 경로)'에 대해 아직 이해하지 못하고 있구나 라고 생각되었고 다시 경로에 대해 공부하게 된 계기가 되었습니다. 

    밑에 글은 상대, 현재, 절대, 상위 경로에 대한 정리 글 입니다.

### 상위 경로 vs 현재 경로
    1. 현재 디렉토리 (./) : 현재 디렉토리를 기준으로 경로를 지정.
    2. 상위 디렉토리 (../) : 현재 디렉토리의 상위 디렉토리로 이동하여 경로를 지정.
        ex) servlet_HTTP.md 에서 ![강의 정리](./servlet/servlet_img/servlet.png) 경로의 이미지를 참조오고 싶으면, servlet_img로 이동(../) 후, 해당 폴더의 serlvet.png 파일에 도달하는 흐름으로 가야 한다.

### 상위 경로 vs 상대 경로
    상대 경로와 상위 경로는 같은 개념이 아니다. 둘은 관련이 있지만, 서로 다른 의미를 가지고 있으므로 확실하게 알아 보자.

    상대 경로 : 현재 작업 디렉토리를 기준으로 파일이나 디렉토리의 위치를 나타내는 경로
        ex) 
        - /file.txt : 현재 디렉토리에 있는 file.txt 파일
        - ../file.txt : 현재 디렉토리의 상위 디렉토리에 있는 file.txt 파일
        - folder/file.txt : 현재 디렉토리의 하위 디렉토리 folder 에 있는 file.txgt 파일

    상위 경로 : 현재 디렉토리의 바로 한 단계 상위에 있는 디렉토리를 나타내는 경로
        ex)
        - ../ : 현재 디렉토리의 상위 디렉토리
        - ../../ : 현재 디렉토리의 상위 디렉토리의 상위 디렉토리

    ● 예시
    ex)
    - 현재 디렉토리가 /home/user/project 라고 가정.
    - 상대 경로로 ../images/pic.png 를 사용하면
      - '..' 는 상위 디렉토리 (/home/user)를 나타낸다.
      - images/pic.png 는 사우이 디렉토리 내의 images 디렉토리에 있는 pic.png 파일을 나타낸다.
    - 따라서 ../images/pic.png 는 /home/user/images/pic.png 를 가리키는 상대 경로이다.

        ex)
        /home/user/project
        ㄴ serlvet
            ㄴ serlvet_img
                ㄴ servlet.png
            ㄴ serlvet_sum
                ㄴ servlet_HTTP.md
        
        - 상대 경로와 상위 경로 사용 예시
        - servlet_HTTP.md 파일에서 servlet.png 이미지를 참조하는 경우
          - 상대 경로 : ../servlet_img/servlet.png
            - '..' 는 servlet_sum 의 상위 디렉토리인 servlet을 나타낸다.
            - servlet_img/servlet.png 는 servlet 디렉토리 내의 servlet_img 디렉토리에 있는 servlet.png 파일을 가리킨다.

### 왜 상대 경로를 사용해야 하는가?
    - 프로젝트 내의 파일 구조를 올바르게 참조
      - 상대 경로는 파일이 위치한 디렉토리를 기준으로 파일을 참조하기 때문에, 현재 파일이 어디에 있는지 경로를 통해 올바른 파일을 참조할 수 있다.
      
    - Markdown 파일의 위치 
      - 'servlet_HTTP.md' 파일이 'servlet_sum' 디렉토리에 있기 때문에, 현재 디렉토리를 기준으로 참조하면 'servlet_img' 디렉토리를 찾을 수 없다. 따라서 상대 경로를 사용하여 상위 디렉토리로 이동한 후 이미지 파일을 참조해야 한다.
      ex) (../servlet_img/servlet.png)

### 브라우저와 GitHub에서의 경로 처리
    - Github는 상대 경로를 기반으로 파일을 참조.
    따라서 Markdown 파일과 이미지 파일이 서로 다른 디렉토리에 있을 경우 상대 경로를 사용하여 참조해야 이미지가 올바르게 표시.
    
    - 브라우저 역시 파일을 로드할 때 상대 경로를 기준으로 참조, 만약 경로가 잘못 지정되어 있으면 파일을 찾지 못하고 404 오류를 반환.

### 요약
    상대 경로: 파일이 서로 다른 디렉토리에 있을 때, 상위 디렉토리로 이동하여 파일을 참조.
    현재 경로: 현재 파일의 디렉토리 기준으로 파일을 참조.    

### 현재 경로 vs 상대 경로 vs 절대 경로
    기준 :/Users/user/Inflearn_MVC1,2/servlet/servlet_img/servlet.png

    현재 경로 : ./servlet.png (현재 파일이 위치한 디렉토리의 파일을 참조)
    상대 경로 : ../servlet_img/servlet.png (현재 디렉토리의 상위 디렉토리로 이동하여 )
    절대 경로 : /Users/user/Inflearn_MVC1,2/servlet/servlet_img/servlet.png
