##  Joo_Cafe - S3UploaderService class 분석 및 학습

### Amazon S3
    AWS에서 제공하는 확장 간으하고 안정적인 객체 스토리지 서비스입니다.
    S3는 데이터를 인터넷을 통해 안전하게 저장하고 검색할 수 있도록 설계되었으며, 사용자가 원하는 양의 데이터를 언제 어디서든 접근할 수 있게 해준다. 

### 주요 특징
    1. 객체 스토리지 (Object storage)    
        - Amazon S3는 데이터를 객체 단위로 저장한다. 
        
        - 각 객체는 데이터 자체, 메타데이터, 그리고 객체를 식별하는 고유 키로 구성

        - 파일 하나가 객체에 해당되며, 객체는 S3 내의 버킷에 저장된다.

    2. 버킷(Bucket)
        - 버킷은 S3에서 객체를 저장하는 컨테이너 역할을 한다.
        모든 S3 객체는 버킷에 저장되며, 각 버킷은 전 세계적으로 고유한 이름을 가져야 한다.

        - 버킷은 사용자 계정에 귀속되며, 버킷 안에 저장된 데이터는 디렉토리 구조와 유사하게 관리.    

    3. 확장성
        - S3는 무한하게 확장 가능하고, 사용자는 저장 용량이나 처리량에 제한 없이 데이터를 저장하고 엑세스할 수 있다.

    4. 보안
        - 데이터 전송 시 HTTPS를 사용하여 데이터를 암호화, 저장 시 서버 측 암포화를 통해 데이터를 보호.

        - 접근 제어 목록(ACL), 버킷 정책, IAM(Identity and Access Management)등을 통해 세밀한 접근 제어를 설정.

    5. 비용 효율성    
        - S3는 사용한 만큼 비용을 지불하는 페이퍼 유즈 모델을 사용,
        저장한 데이터의 양과 요청한 데이터 엑세스 횟수에 따라 비용이 청구.

    6. 버전 관리
        - S3는 객체의 버전 관리를 지원하여, 객체의 수정 및 삭제에 따른 이전 버전을 보존할 수 있따. 이를 통해 실수로 삭제되거나 덮어쓰여진 데이터를 복구.

### 사용 사례
    1. 백업 및 복구        
        - 주용한 데이터의 백업을 위해 S3를 사용하여 데이터를 안전하게 저장하고 필요 시 복구.

    2. 콘텐츠 저장 및 배포
        - 웹 사이트 이미지, 비디오, 문서 등의 정적 콘텐츠를 S3에 저장하고, 이를 CDN(CloudFront)과 결합하여 전 세계적으로 빠르고 안전하게 배포.
    
    3. 백엔드 데이터 저장소
        - 모바일 및 웹 애플리케이션의 파일 저장소로 S3를 사용할 수 있으며, 이를 통해 사용자의 파일을 안전하게 관리.

### Amazon S3의 기본적인 동작 방식
    1. 업로드
        - 데이터를 S3 버킷에 업로드할 때, 각 파일은 객체로 저장되며 고유한 키를 갖게 된다.

    2. 다운로드 
        - 저장된 객체는 HTTP OR HTTPS를 통해 언제든지 다운로드할 수 있으며, 퍼블릭 엑세스를 허용한 경우 누구나 접근이 가능.

    3. 삭제 
        - 더 이상 필요하지 않은 객체는 언제든지 삭제, 삭제된 객체는 버전 관리 기능이 활성화된 경우 복구할 수 있다.

    - S3는 이와 같은 기능을 통해 대규모 데이터를 안전하고 효율적으로 저장, 관리할 수 있는 서비스이다.
### AmazonS3 class code

    import com.amazonaws.SdkClientException;
    import com.amazonaws.services.s3.AmazonS3Client;
    import com.amazonaws.services.s3.model.CannedAccessControlList;
    import com.amazonaws.services.s3.model.ObjectMetadata;
    import com.amazonaws.services.s3.model.PutObjectRequest;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.stereotype.Service;
    import org.springframework.web.multipart.MultipartFile;

    import java.io.IOException;
    import java.io.InputStream;

    @Service
    public class S3UploaderService {

        private final AmazonS3Client amazonS3Client;
        private final String bucketName;

        // @Value("${cloud.aws.s3.bucket}") 어노테이션을 사용할려면 생성자를 따로 만들어야 함
        @Autowired
        public S3UploaderService(AmazonS3Client amazonS3Client,
            @Value("${cloud.aws.s3.bucket}") String bucketName) {
            this.amazonS3Client = amazonS3Client;
            this.bucketName = bucketName;
        }

        // multipartFile = 업로드할 파일, dirName = 업로드할 디렉토리 이름
        // 이미지 파일 업로드 
        public String uploadFileToS3(MultipartFile multipartFile, String dirName) throws IOException {
            String fileName = dirName + "/" + multipartFile.getOriginalFilename();

            // 업로드할 파일의 메타데이터 객체를 생성
            ObjectMetadata metadata = new ObjectMetadata();
            // 파일의 크기를 메타데이터에 설정
            metadata.setContentLength(multipartFile.getSize());
            // 파일의 콘텐츠 타입(형식)을 메타데이터에 설정
            metadata.setContentType(multipartFile.getContentType());

            try (InputStream inputStream = multipartFile.getInputStream()) {
                // 파일을 AmazonS3 버킷에 업로드
                amazonS3Client.putObject(
                    new PutObjectRequest(bucketName, fileName, inputStream, metadata)
                        .withCannedAcl(CannedAccessControlList.PublicRead)
                );

            }
            return amazonS3Client.getUrl(bucketName, fileName).toString();
        }

        // 이미지 파일 삭제
        public void deleteFile(String picture) throws IOException {
            try {
                // picture 값에서 파일의 경로와 이름 부분을 추출하여 키로 사용
                String key = picture.substring(picture.lastIndexOf("/") + 1);
                amazonS3Client.deleteObject(bucketName, key);
            } catch (SdkClientException e) {
                throw new IOException("Error deleting file from S3", e);
            }
        }
    }

    ● 코드 해석
    위에 코드는 Amazon S3와 통합하여 파일을 업로드하고 삭제하는 서비스를 구현한 것이다. 주로 파일(ex_ img)을 AWS S3에 업로드하고, 필요한 경우 해당 파일을 삭제하는 기능을 제공.    

    1. 클래스 개요
       - 위 클래스는 s3UploaderService 로, 파일을 AWS S3 버팃에 업로드하고 삭제하는 기능을 제공.

       - AmazonS3Client 객체를 사용하여 S3와 상호작용한다.

    2. 생성자

        private final AmazonS3Client amazonS3Client;
        private final String bucketName;

        // @Value("${cloud.aws.s3.bucket}") 어노테이션을 사용할려면 생성자를 따로 만들어야 함
        @Autowired
        public S3UploaderService(AmazonS3Client amazonS3Client,
            @Value("${cloud.aws.s3.bucket}") String bucketName) {
            this.amazonS3Client = amazonS3Client;
            this.bucketName = bucketName;
        }

        - 생성자를 통해 AmazonS3Client 와 버킷 이름(bucketName)을 주입 받는다.

        - @Value("${cloud.aws.s3.bucket}") 은 Spring의 @Valud 어노테이션을 사용하여 application.properties or yml 파일에서 S3 버킷 이름을 가져온다.

        - @Autowired 를 통해 생성자가 의존성 주입을 받도록 설정

    3. 파일 업로드 메서드(uploadFileToS3)

        // multipartFile = 업로드할 파일, dirName = 업로드할 디렉토리 이름
        // 이미지 파일 업로드 
        public String uploadFileToS3(MultipartFile multipartFile, String dirName) throws IOException {
            String fileName = dirName + "/" + multipartFile.getOriginalFilename();

            // 업로드할 파일의 메타데이터 객체를 생성
            ObjectMetadata metadata = new ObjectMetadata();
            // 파일의 크기를 메타데이터에 설정
            metadata.setContentLength(multipartFile.getSize());
            // 파일의 콘텐츠 타입(형식)을 메타데이터에 설정
            metadata.setContentType(multipartFile.getContentType());

            try (InputStream inputStream = multipartFile.getInputStream()) {
                // 파일을 AmazonS3 버킷에 업로드
                amazonS3Client.putObject(
                    new PutObjectRequest(bucketName, fileName, inputStream, metadata)
                        .withCannedAcl(CannedAccessControlList.PublicRead)
                );

            }
            return amazonS3Client.getUrl(bucketName, fileName).toString();
        }

        - 파라미터
          - MultipartFile multipartFile 업로드할 파일
          - String dirName 파일이 저장될 S3 버킷 내의 디렉토리 이름을 나타낸다.

        - 업로드 과정
          - 업로드할 파일의 전체 경로(fileName)를 설정한다. (dirName/파일이름)
          - ObjectMetadata 객체를 생성하여 파일의 크기와 콘텐츠 타입을 설정.
          - InputStream을 통해 파일을 읽어오고, amazonS3Client.putObjec() 메서드를 사용해 파일을 S3 버킷에 업로드 한다.
          - 업로드가 완료되면 S3에서 해당 파일에 접근할 수 있는 URL을 반환
          
    4. 파일 삭제 메서드 (deleteFile)
    
        // 이미지 파일 삭제 jiyeon-23.08.25
        // 메서드는 구현했지만 S3확인했을때 이미지가 삭제되고있진 않습니다(추후 다시 구현예정)
        public void deleteFile(String picture) throws IOException {
            try {
                // picture 값에서 파일의 경로와 이름 부분을 추출하여 키로 사용
                String key = picture.substring(picture.lastIndexOf("/") + 1);
                amazonS3Client.deleteObject(bucketName, key);
            } catch (SdkClientException e) {
                throw new IOException("Error deleting file from S3", e);
            }
        }          

        - 파라미터
          - String picture 삭제할 파일의 URL 또는 경로

          - String key = picture.substring(picture.lastIndexOf("/") +1);
            - 파일의 경로에서 파일 이름을 추출하는 기능.

              - picture ex : "https://example-bucket.s3.amazonaws.com/images/profile_picture.png"   

              - picture : file 전체 URL이나 경로를 포함하는 문자열

              - picture.lastIndexOf("/") 는 문자열에서 마지막 '/'의 위치를 찾는다. 이는 파일 경로에서 마지막 '/'가 파일 이름의 시작 지점을 나타내기 때문.

              - picture.substring(picture.lastIndexOf("/") + 1)는 마지막 '/'ㄴ의 다음 문자부터 문자열의 끝까지를 추출하여 파일 이름을 얻습니다.

              - 따라서 picture.substring(picture.lastIndexOf("/") + 1)의 결과는 "profile_picture.png"가 됩니다.
          
        - 삭제 과정
          - URL에서 파일의 경로를 추출하여 S3에서 파일을 삭제하는 데 사용되는 키를 설정.
          - amazonS3Client.deleteObject() 메서드를 사용해 해당 파일을 삭제.
          - SdkClientException 예외가 발생하면, IOException 으로 래핑하여 처리.

### S3 파일 업로드 흐름
    S3UploaderServcie 클래스는 AWS에 파일을 업로드하고 삭제하는 기능을 제공한다. 이 과정을 통해 파일이 어떻게 업로드되고 삭제되는지 흐름을 이해할 수 있다.

    1. S3에 파일 업로드 흐름
    
        ● 파일 준비 
           - 사용자가 업로드할 파일을 선택하면, MultipartFile 객체로 Spring 애플리케이션에 전달된다.

        ● 메타데이터 설정
           - 업로드할 파일의 크기, 타입 등의 정보를 포함하는 ObjectMetadata 객체를 생성한다. 이 메타데이터는 S3에 파일을 저장할 때 함께 저장되어 파일의 특성을 정의한다.

           - objectMetadata : Multipart에서 (contentType, size)가 metadata에 해당된다.

        ● 파일 업로드
           - S3 클라이언트를 통해 putObject() 메서드를 호출하여 파일을 S3에 업로드한다.

           - 이 메서드에 업로드할 파일의 bucketName, dir, InputStream(파일의 내용), metadata를 전달한다.

           - PutObjectRequest 객체는 파일 업로드 요청을 캡슐화하며, 이 요청은 S3로 전송된다.

           - withCannedAcl(CannedAccessControlList.PublicRead) 설정을 통해 업로드된 파일이 퍼블릭하게 접근될 수 있도록 권한을 설정.

        ● 파일 URL 반환 
           - 파일이 성공적으로 업로드되면, amazonS3Client.getUrl() 메서드를 사용하여 업로드된 파일의 URL을 생성하고 이를 반환.

           - 이 URL은 파일에 접근할 수 있는 웹 주소로, 클라이언트에서 해당 파일을 표시하거나 다운로드할 때 사용된다.

    2. S3에서 파일 삭제 흐름

        ● 파일의 키 추출
           - 삭제할 파일의 URL에서 파일 이름을 추출, 이 이름은 S3에서 파일으 구분하는 **키(key)로 사용된다.

        ● 파일 삭제 요청
           - S3 클라이언트의 deleteObject() 메서드를 사용하여 S3에서 파일으 삭제한다.

           - 이 메서드에는 버킷 이름과 삭제할 파일의 키가 전달된다.

        ● 삭제 확인
           - 삭제 요청이 성공적으로 처리되면 S3에서 해당 파일이 삭제된다. 
           S3는 삭제된 파일에 더 이상 접근할 수 없도록 보장.

    3. 전체 흐름 요약

        ● 업로드
           - 사용자가 파일을 업로드하면, Spring 애플리케이션이 이를 S3에 업로드 요청을 보낸다.

           - 파일은 지정된 버킷과 경로에 저장되고, 접근 가능한 URL이 생성.

        ● 삭제
           - 사용자가 파일 삭제를 요청하면, Spring 애플리케이션이 S3에 삭제 요청을 보낸다.

           - S3는 해당 파일을 삭제하여 더 이상 접근할 수 없도록 한다.     


### 주요 개념 및 기술

    ● @Valid
        - Spring에서 외부 설정 값을 주입뱓을 때 사용, 이 경우 S3 버킷 이름을 주입받는다.

    ● AmazonS3Client
        - AWS SDK의 S3 클라이언트로, S3에 대한 모든 작업(업로드, 삭제 등)을 처리.

        ● SDK        

          ● 라이브러리
          - SDK는 주로 API 호출을 포함하는 라이브러리를 제공한다.
          - 이 라이브러리는 특정 플랫폼이나 서비스와 상호작용하는 데 필요한 기능들을 캡슐화.

          ● 도구
          - SDK에는 코드 편집기, 디버거, 컴파일러, 그리고 다른 개발 도구가 포함될 수 있다. 이 도구들은 소프트웨어 개발을 쉽게 해주며, 효율적인 개발 환경을 제공.

          ● 문서화
          - SDK는 API와 사용법에 대한 문서화도 제공한다. 이는 개발자가 SDK를 사용하여 애플리케이션을 개발하는 방법을 이해하고, 구현에 필요한 모든 정보를 얻을 수 있게 해준다.

          ● 클라우드 서비스와의 연동
          - amazon S3 SDK는 AWS S3 서비스와 상호작용할 수 있는 기능을 제공하며, 개발자가 파일 업로드, 다운로드, 삭제와 같은 작업을 할 수 있도록 돕는다.

        ● SDK 정리
        - SDK는 특정 플랫폼 또는 서비스와 상호작용하기 위한 모든 도구를 제공하는 개발 키트이다. 이는 개발자가 특정 환경에서 애플리케이션을 쉽게 개발, 테스트, 배포할 수 있도록 설계된 도구 모음으로, 라이브러리 도구, 문서화 등이 포함되어 있어 개발을 간소화하고 효율성을 높인다.

    ● MultipartFile
        - Spring에서 제공하는 인터페이스로, HTTP 요청에서 전송된 파일을 처리.

    ● ObjectRequest
        - S3에 업로드할 때 사용하는 요청 객체.

    ● PutObjectRequest
        - S3에 파일을 업로드할 때 사용하는 요청 객체.           

    ● CannedAccessControlList.PublicRead
        - 파일을 퍼블릭하게 접근 가능하도록 설정.
            즉, 업로드된 파일을 누구나 읽을 수 잇따.  

    사용 예시
    ● 파일 업로드

        @PostMapping("/upload")              
        public ResponseEntity<String> uploadFile(@RequestParam("file") MultipartFile file) {
            String url = s3UploadService.uploadFileToS3(file, "images");
            return ResponseEntity.ok(url);
        }

    ● 파일 삭제

        @DeleteMapping("/delete")    
        public ReponseEntity<Void> delete(@RequestParam("fileUrl") String fileUrll) throws IOException {
            s3UploaderService.deleteFile(fileUrl);
            return ReposneEntity.noContent().build();
        }

### 느낀점
    AWS를 학원에서 서버 배포 목적으로 EC2..?를 사용을 해본 경험이 있다.
    클론 코딩이지만 이 프로젝트 클래스에서 사용하는 목적은 파일 업로드, 다운로드, 삭제, 대용량 데이터(img) 관리에 
    amazonS3 사용이 유용하다고 알게되었다.

    좀 더, aws에 관해서 찾아보고 학습하여 amazonS3를 활용해 내 프로젝트에 적용해 보고 싶다.