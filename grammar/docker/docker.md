## Docker

### Docker 란?

    역할 
    - 컨테이너 생성 및 실행 도구
    
    설명
    - 개발 환경이나 애플리케이션을 컨테이너라는 독립적인 공간에 격리시켜 실행
    - 컨테이너는 실행 가능한 이미지(Docker image)를 기반으로 만들어짐
    - 예전에는 서버마다 환경 설정이 달라서 "내 PC에서는 되는데.." 문제가 많았다.

    핵심 파일 : Dockerfile
    - 애플리케이션을 어떻게 컨테이너로 만들지를 정의하는 파일
    ex) 
        : Spring Boot 애플리케이션용 Dockerfile
        FROM openjdk:17-alpine
        COPY build/libs/app.jar app.jar
        ENTRYPOINT ["java", "-jar", "app.jar"]

    ex)
        명령어 
            docker build -t my-app .
            docker run -p 8080:8080 my -app    

### docker-compose 란?

    역할
    - 환경 변수 정의 파일 (민감 정보나 설정값을 분리)

    설명
    - .env 파일에 정의한 변수들은 docker-compose.yml 또는 애플리케이션 내부에서 사용 가능.
        ex) DB 비밀번호, 포트 번호 등을 하드코딩하지 않고 관리할 수 있다.
    - 여러 개의 컨테이너(DB, 백엔드, 프론트엔드 등)를 한 번에 실행할 수 있돌고 도와주는 도구
    - 하나하나 docker run... 하지 않아도 됨
        ex) Spring Boot + MySQL + Redis 같은 조합을 쉽게 실행 가능

    핵심 파일 : docker-compose.yml

    version: "3"
    services:
    app:
        build:
        context: .
        dockerfile: Dockerfile
        ports:
        - "8080:8080"
        environment:
        - JWT_SECRET=your_secret_key
        depends_on:
        - db

    db:
        image: mysql:8
        environment:
        MYSQL_ROOT_PASSWORD: root
        MYSQL_DATABASE: joocafe
        ports:
        - "3306:3306"

    -> docker-compose up 명령어 한 줄로 전부 실행 가능!

### Deploy(배포) 란?

    역할
    - 완성된 컨테이너들을 실제 서비스 환경(서버)에 올리는 것
    
    설명
    - 개발 환경에서 만든 이미지를 서버에 옮겨서 실행하는 과정정
    - 로컬에서만 실행하는 게 아니라, "클라우드(VM, EC2 등)"에 Docker 컨테이너로 실행행
    - 보통 이렇게 구성
      1. 코드 빌드 ex) ./gradlew build
      2. Docker 이미지 빌드 ex) docker build
      3. 서버에서 이미지 다운로드 및 실행 ex) docker-compose up -d

### 실제 예시

    1. Dockerfile 로 Dockerfile로 컨테이너 이미지를 만든다 (Docker)
    2. docker-compose.yml 로 MySQL 과 함께 실행
    3. 민감 정보나 설정은 .env 파일로 분리한다 ex) JWT_SECRET, DB 비밀번호 설정
    4. 만든 이미지를 서버에 배포한다 (deploy)

    흐름
    - .env에 DB 환경 변수 설정
    - docker-compose.yml에서 .env 값을 참조하여 컨테이너 구성
    - docker-compose up --build 명령으로 모든 서비스 실행
    - 서버에서 동일한 방식으로 배포

### 디렉토리 구조

    project/
    │
    ├── build.gradle
    ├── Dockerfile
    ├── docker-compose.yml
    ├── .env               # 환경 변수 (선택)
    ├── src/
    │   └── main/java/...

