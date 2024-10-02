Spring Boot에서 AWS S3에 이미지를 업로드하고 불러오는 기능을 구현하려면 다음과 같은 과정을 따를 수 있습니다.

### 1. AWS S3 설정
1. **S3 버킷 생성**: AWS S3 콘솔에 접속하여 버킷을 생성합니다.
2. **IAM 사용자 생성 및 권한 부여**: S3 버킷에 접근할 수 있도록, S3FullAccess 권한이 있는 IAM 사용자를 생성하고, 접근 키 (Access Key)와 비밀 키 (Secret Key)를 받아 둡니다.

### 2. Spring Boot 프로젝트에 AWS SDK 설정

#### 1) Maven 의존성 추가
`pom.xml` 파일에 AWS SDK를 추가합니다.

```xml
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
</dependency>
```

#### 2) AWS S3 설정 파일 작성
`application.yml` 또는 `application.properties`에 AWS S3 관련 설정을 추가합니다.

```yaml
cloud:
  aws:
    credentials:
      accessKey: YOUR_AWS_ACCESS_KEY
      secretKey: YOUR_AWS_SECRET_KEY
    s3:
      bucket: YOUR_S3_BUCKET_NAME
    region:
      static: YOUR_AWS_REGION
```

#### 3) S3 클라이언트 구성
S3와의 상호작용을 위한 `AmazonS3` 클라이언트를 Spring Bean으로 설정합니다.

```java
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class S3Config {

    @Value("${cloud.aws.credentials.accessKey}")
    private String accessKey;

    @Value("${cloud.aws.credentials.secretKey}")
    private String secretKey;

    @Value("${cloud.aws.region.static}")
    private String region;

    @Bean
    public S3Client s3Client() {
        AwsBasicCredentials awsCreds = AwsBasicCredentials.create(accessKey, secretKey);

        return S3Client.builder()
                .region(Region.of(region))
                .credentialsProvider(StaticCredentialsProvider.create(awsCreds))
                .build();
    }
}
```

### 3. S3 이미지 업로드 및 다운로드 서비스 구현

#### 1) S3 이미지 업로드 서비스

```java
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;
import software.amazon.awssdk.services.s3.model.PutObjectResponse;

import java.io.IOException;
import java.nio.file.Paths;

@Service
public class S3Service {

    private final S3Client s3Client;
    private final String bucketName;

    public S3Service(S3Client s3Client, @Value("${cloud.aws.s3.bucket}") String bucketName) {
        this.s3Client = s3Client;
        this.bucketName = bucketName;
    }

    public String uploadFile(MultipartFile file) throws IOException {
        String key = Paths.get("family-profiles", file.getOriginalFilename()).toString();

        PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        s3Client.putObject(putObjectRequest, file.getInputStream());

        return key;
    }
}
```

#### 2) S3 이미지 다운로드 서비스

```java
import software.amazon.awssdk.services.s3.model.GetObjectRequest;
import org.springframework.stereotype.Service;

import java.io.InputStream;

@Service
public class S3DownloadService {

    private final S3Client s3Client;
    private final String bucketName;

    public S3DownloadService(S3Client s3Client, @Value("${cloud.aws.s3.bucket}") String bucketName) {
        this.s3Client = s3Client;
        this.bucketName = bucketName;
    }

    public InputStream downloadFile(String key) {
        GetObjectRequest getObjectRequest = GetObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        return s3Client.getObject(getObjectRequest);
    }
}
```

### 4. 이미지 업로드 API 구현

```java
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import java.io.IOException;

@RestController
@RequestMapping("/api/v1/family")
public class FamilyController {

    private final S3Service s3Service;

    public FamilyController(S3Service s3Service) {
        this.s3Service = s3Service;
    }

    @PostMapping("/upload-profile")
    public ResponseEntity<String> uploadProfileImage(@RequestParam("file") MultipartFile file) {
        try {
            String fileUrl = s3Service.uploadFile(file);
            return ResponseEntity.ok(fileUrl);
        } catch (IOException e) {
            return ResponseEntity.status(500).body("File upload failed");
        }
    }
}
```

### 5. 프론트엔드 이미지 업로드 처리 (Vue.js)
프론트엔드에서 이미지를 선택하고 S3로 업로드하는 과정은 기존에 base64로 처리한 것과 유사하게 파일을 선택하면 됩니다.

```js
async function uploadFamilyProfile() {
    const formData = new FormData();
    formData.append("file", fileInput.value.files[0]);

    try {
        const response = await axios.post("/api/v1/family/upload-profile", formData, {
            headers: {
                "Content-Type": "multipart/form-data",
            },
        });
        console.log("Uploaded Image URL:", response.data);
    } catch (error) {
        console.error("Image upload failed:", error);
    }
}
```

### 6. 결론
이렇게 하면 이미지를 S3에 업로드하고, 프론트엔드에서는 업로드된 이미지를 S3에서 직접 불러올 수 있게 됩니다.