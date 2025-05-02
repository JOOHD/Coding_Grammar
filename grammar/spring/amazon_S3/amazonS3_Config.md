##  Joo_Cafe - S3Config class 분석 및 학습

### S3Config class 

    @Configuration
    public class S3Config {

        @Value("${cloud.aws.credentials.accessKey}")
        private String accessKey;
    
        @Value("${cloud.aws.credentials.secretKey}")
        private String secretKey;
    
        @Value("${cloud.aws.region.static}")
        private String region;
    
        @Bean
        public AmazonS3Client amazonS3Client() {

            // AWS 자격 증명 객체 생성
            BasicAWSCredentials awsCreds = new BasicAWSCredentials(accessKey, secretKey);
            return (AmazonS3Client) AmazonS3ClientBuilder.standard()
                    .withRegion(region)
                    .withCredentials(new AWSStaticCredentialsProvider(awsCreds))
                    .build();
        }
    }

    위 클래스는 Spring Framework의 @Configuration 클래스를 사용하여 AWS S3 클라이언트를 설정하는 클래스
    AWS S3와의 통신을 위한 클라이언트를 생성하고, Spring의 의존성을 주입을 통해 다른 컴포넌트에서 사용할 수 있도록 제공.

### 코드 설명
    ● @Configuration
        - 어노테이션은 이 클래스가 Spring의 설정 클래스를 정의하고 있음을 나타낸다.
          이 클래스는 애플리케이션 컨텍스트가 시작될 때 Bean을 정의하는 데 사용된다.

    ● @Value (필드 주입)
        - Spring의 property file에서 값을 주입받기 위해 사용된다.
          여기서는 AWS S3의 access key, scret key, region static 정보를, 
          property file에서 가져와서 필드에 주입한다. 

    ● @Bean
        - 이 메서드가 Spring의 애플리케이션 컨텍스트에 빈을 등록하는 메서드임을 나타낸다.
          이 메서드는 AmazonS3Client 객체를 반환하며, 이 객체는 AWS S3와의 상호작용에 사용.

    ● amazonS3Client()
        - BasicAWSCredentials 객체를 생성하여 AWS의 access key, secret key를 사용하여 인증을 설정

        - AmazonS3ClientBuilder 를 사용하여 AmazonS3Client 객체를 생성,
          이 객체는 AWS S3와의 통신을 처리.

        - withCredentials(new AWSStaticCredentialsProvider(awsCreds)) 
          위에서 생성한 자격증명을 사용하여 인증을 설정.

        - .build() : AmazonS3Client 객체를 빌드.

    ● 주요 포인트
    1. 자격 증명 : AWS S3에 접근하기 위해 필요한 access key, secret key를 설정한다.
    2. 리전 설정 : AWS S3의 region을 설정하나다 이는 요청을 해당 리전의 S3 버킷으로 라우팅한다.
    3. 빈 등록 : AmazonS3Client 는 애플리케이션 내의 다른 컴포넌트에서 주입받아 사용할 수 있으며,
                S3 버킷에 대한 파일 업로드, 다운로드, 삭제 등의 작업을 수행.
    
    이렇게 설정된 AmazonS3Client는 애플리케이션 내의 다른 컴포넌트에서 주입받아 사용할 수 있으며, 
    S3 버밋에 대한 파일 업로드, 다운로드, 삭제 등의 작업을 수행.

### Access key & Secret key 발급
    ● AWS Management Console을 통해 발급받을 수 있습니다.

    ● AWS에서 제공하는 IAM (Identity and Access Management) 서비스를 사용하여, 
      애플리케이션에서 사용할 IAM 사용자나 **역할(role)**을 생성한 뒤, 
      이 사용자에게 프로그램적으로 AWS 리소스에 접근할 수 있도록 Access Key와 Secret Key를 발급합니다.

    ● 발급된 Access Key는 사용자 ID 역할을 하며, Secret Key는 해당 사용자의 비밀번호처럼 비밀리에 관리되어야 합니다.

    ● 발급 과정
        1. AWS Management Console에 로그인
        2. IAM 서비스로 이동
        3. User or Role 생성
        4. 생성한 사용자에 대해 프로그래밍 엑세스를 활성화하고, Access key, secret key를 다운로드

### Access key & Secret key 사용
    발급받은 Access Key, Secret Key, Region 값은 애플리케이션의 환경 설정 파일
    application.properties or application.yml or AWS SDK에서 직접 설정하게 된다.
    
    ex) 
        cloud:
          aws:
            credentials:
                accessKey: access-key
                secretKey: secret-key
            region
                static: us-west-2

### Java 코드에서 사용
    이 값을 Spring의 @Value 어노테이션을 사용해 주입.
    ex)
        @Value("${cloud.aws.credentials.accessKey}")
        private String accessKey;
        
        @Value("${cloud.aws.credentials.secretKey}")
        private String secretKey;
        
        @Value("${cloud.aws.region.static}")     
        private String region;

    - Access Key, Secret Key, Region 값을 설정 파일(application.properties) 또는 Spring의 
        @Value 로 주입하고, AWS SDK를 사용하여 AmazonS3Client를 생성한 후,
        API 호출을 하여 S3 버킷에 파일을 업로드하거나 다운로드하는 API를 호출하여 AWS 리소스에 접근한다.

    - 이렇게 자격 증명과 리전 설정이 올바르게 되어 있어야 AWS S3와 같은 서비스에 접근할 수 있게 된다.