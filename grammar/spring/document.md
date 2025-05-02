## 본문

### Document 객체의 주요 기능과 메서드
    document 객체를 통해 다양한 방식으로 HTML 요소를 선택하고 조작 가능하다.
    아래에서는 자주 사용되는 메서드와 그 사용 예시를 소개한다.

    1. 요소 선택 메서드
        1) document.getElementById(id)
            설명 : 주어진 'id' 속성을 가진 단일 요소를 반환.
            ex)
                <!-- HTML -->
                <div id="myDiv">Hello World!<div>

                // Javascript
                const myDiv = document.getElementById("myDiv");
                console.log(myDiv.textContent); // "Hello world!"w

        2) document.getElementByClassName(className)
            설명 : 주어진 class를 가진 모든 요소를 HTMLCollection으로 반환.
            ex) 
                <!-- HTML -->        
                <p class="text">Paragraph 1</p>
                <p class="text">Paragraph 2</p>

                // Javascript
                const texts = document.getElementByClassName("text");
                console.log(texts.lenth); // 2
                console.log(texts[0].textContent); // "Paragraph 1"

        3) document.querySelector(selector)        
            설명 : 주어진 CSS 선택자와 일치하는 첫 번째 요소를 반환.
            ex) 
                <!-- HTML -->
                <div class="container">
                    <p class="text">Hello</p>
                </div>

                // Javascript
                const test = document.querySelector('.container .text');
                console.log(text.textContent); // "Hello"

        4) document.querySelectorAll(selector)        
            설명 : 주어진 CSS 선택자와 일치하는 모든 요소를 NodeList로 반환.
            ex)
                <!-- HTML -->
                <ul>
                    <li class="item">A</li>
                    <li class="item">B</li>
                    <li class="item">C</li>
                </ul>

                // JavaScript
                const items = document.querySelectorAll('.item');
                items.forEach((item) => {
                    console.log(item.textContent);
                });
                // 출력:
                // "A"
                // "B"
                // "C"

    2. 요소 생성 및 조작
        1) document.createElement(tagName)
            설명 : 주어진 태그 이름의 새로운 요소를 생성한다.
            ex)
                const newDiv = document.createElement('div');
                newDiv.textContent = 'New Div';
                document.body.appendChild(newDiv);

        2) element.appendChild(childElement)        
            설명 : 특정 요소에 자식 요소를 추가
            ex) 
                const parent = document.getElementById('parentDiv')
                const child = document.createElement('p')
                child.textContent = 'I am a child paragraph.';
                parent.appendChild(child);

        3) element.innerHTML
            설명 : 요소의 내부 HTML 콘텐츠를 설정하거나 가져온다.
            ex)
                const container = document.querySelector('.container');
                container.innerHTML = '
                    <h2>Title</h2>
                    <p>This is a paragraph inside the container.</p>
                    `;

        4) element.setAttribute(name, value)                    
            설명 : 요소에 새로운 속성을 추가하거나 기존 속성의 값을 변경.
            ex)
                const link = document.createElement('a');
                link.setAttribute('href', 'http://www.example.com');
                link.textContent = 'Go to Example.com';
                document.body.appendChild(link);

### document 객체의 구조 이해
    document 객체는 트리 구조로 되어 있으며, HTML 문서의 각 요소들은 이 트리의 노드로 표현된다. 이 구조를 통해 부모-자식 관계를 이해하고 탐색할 수 있다.                

    <!DOCTYPE html>
    <html>
    <head>
        <title>Document Example</title>
    </head>
    <body>
        <div id="main">
            <h1>Hello, World!</h1>
            <p>This is a paragraph.</p>
        </div>
    </body>

    ● 해당 문서의 DOM 트리 구조
    - document
    
      - html

        - head

          - title

        - body

          - div #main

            - h1

            - p
            
    ● DOM 트리 탐색 예시
    // body 요소 접근
    const body = document.body;

    // #main div 접근
    const mainDiv = document.getElementById('main');

    // mainDiv의 첫 번째 자식 요소(h1) 접근
    const heading = mainDiv.firstElementChild;
    console.log(heading.texgContent); // "Hello, World!"

    // mainDiv의 두 번째 자식 요소(p) 접근
    const paragraph = heading.nextElementSibling;
    console.log(paragraph.textContent); // "This is a paragraph."

### 실제 활용 예시
    동적으로 콘텐츠 추가하기
    <!-- HTML -->    
    <div id="userList"></div>

    // javascript
    const users = ['Alice', 'Bob', 'Charlie'];
    const userListDiv = document.getElementById('userList');

    user.forEach(user => {
        const userItem = document.createElement('p');
        userItem.textContent = user;
        userListDiv.appendChild(userItem);
    });

    ● 결과
    <div id="userList">
        <p>Alice</p>
        <p>Bob</p>
        <p>Charlie</p>
    </div>

### form 제출 시 데이터 검증하기
    <!-- HTML -->    
    <form id="myForm">
        <input type="text" id="username" placeholder="Username" required />
        <input type="password" id="password" placeholder="Password" required />
        <button type="submit">Submit</button>
    </form>
    <p id="errorMsg" style="color: red;"></p>

    // Javascript
    const form = document.getElementById('myForm');
    const errorMsg = document.getElementById('errorMsg');

    form.addEventListener('submit', (event) => {
        const username = document.getElementById('username').value;
        const password = document.getElementById('password').value;

        if (username === '' || password === '') {
            event.preventDefault(); // 폼 제출 막기
            errorMsg.textContent = 'All fields are required.';
        }
    });

### 요약
    ● document 객체는 현재 로드된 전체 HTML 문서를 표현하며, 이를 통해 웹 페이지의 모든 요소와 콘텐츠에 접근하고 조작할 수 있따.  
    
    ● 다양한 메서드와 속성을 통해 특정 요소를 선택, 생성, 수정, 삭제할 수 있으며, 이벤트를 처리하고, 스타일을 변경하는 등의 작업을 수행할 수 있다.

    ● DOM 트리 구조를 이해하면 요소 간의 관계를 파악하고 효율적인 요소를 탐색하고 조작할 수 있다.

    ● 실제 웹 개발에서 document 객체를 활용하여 동적인 사용자 인터페이스를 만들고, 사용자 상호작용에 반응하는 웹 페이지를 구축할 수 있다.