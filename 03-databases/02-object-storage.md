# Object Storage — Files Don't Belong in Your Database

## Why Object Storage

Storing files in a database (as BLOBs) wastes resources, bloats backups, and kills query performance. Object storage (S3, MinIO) stores files separately and gives you URLs to reference them.

## Step 1: AWS S3 SDK with Spring Boot

```xml
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
    <version>2.25.0</version>
</dependency>
```

```java
@Configuration
public class S3Config {
    @Bean
    public S3Client s3Client(
            @Value("${aws.s3.region}") String region,
            @Value("${aws.s3.endpoint:}") String endpoint) {
        var builder = S3Client.builder()
            .region(Region.of(region))
            .credentialsProvider(DefaultCredentialsChain.builder().build());
        if (!endpoint.isEmpty()) {
            builder.endpointOverride(URI.create(endpoint))
                   .forcePathStyle(true);
        }
        return builder.build();
    }
}
```

## Step 2: Document Storage Service

```java
@Service
@RequiredArgsConstructor
public class DocumentStorageService {
    private final S3Client s3Client;
    @Value("${aws.s3.bucket}") private String bucket;

    public String upload(String key, InputStream data,
            long size, String contentType) {
        s3Client.putObject(PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .contentType(contentType)
                .contentLength(size)
                .build(),
            RequestBody.fromInputStream(data, size));
        return key;
    }

    public byte[] download(String key) {
        var response = s3Client.getObject(GetObjectRequest.builder()
            .bucket(bucket)
            .key(key)
            .build());
        try {
            return response.readAllBytes();
        } catch (IOException e) {
            throw new StorageException("Failed to read object", e);
        }
    }

    public void delete(String key) {
        s3Client.deleteObject(DeleteObjectRequest.builder()
            .bucket(bucket)
            .key(key)
            .build());
    }

    public String generatePresignedUrl(String key, Duration validity) {
        var request = GetObjectPresignedRequest.builder()
            .signatureDuration(validity)
            .getObjectRequest(GetObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .build())
            .build();
        return s3Client.utilities()
            .getPresignedUrl(request)
            .url()
            .toString();
    }
}
```

## Step 3: Upload Endpoint

```java
@RestController
@RequestMapping("/api/documents")
@RequiredArgsConstructor
public class DocumentController {
    private final DocumentStorageService storage;
    private final DocumentMetadataRepository metadataRepo;

    @PostMapping("/upload")
    public ResponseEntity<DocumentResponse> upload(
            @RequestParam("file") MultipartFile file,
            @RequestParam("category") String category) {
        var key = "%s/%s/%s".formatted(
            category, LocalDate.now(), file.getOriginalFilename());

        storage.upload(key, file.getInputStream(),
            file.getSize(), file.getContentType());

        var metadata = new DocumentMetadata();
        metadata.setStorageKey(key);
        metadata.setFileName(file.getOriginalFilename());
        metadata.setContentType(file.getContentType());
        metadata.setSize(file.getSize());
        metadata.setCategory(category);
        metadataRepo.save(metadata);

        return ResponseEntity.ok(new DocumentResponse(
            metadata.getId(), key, file.getOriginalFilename()));
    }

    @GetMapping("/{id}/download-url")
    public ResponseEntity<Map<String, String>> getDownloadUrl(
            @PathVariable Long id) {
        var metadata = metadataRepo.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Not found"));
        var url = storage.generatePresignedUrl(
            metadata.getStorageKey(), Duration.ofMinutes(15));
        return ResponseEntity.ok(Map.of("url", url));
    }
}
```

## Step 4: MinIO for Local Development

```yaml
# docker-compose.yml
services:
  minio:
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
```

```yaml
# application-dev.yml
aws:
  s3:
    region: us-east-1
    endpoint: http://localhost:9000
    bucket: local-documents
```

MinIO is S3-compatible. The same code works with both — just change the endpoint.

## Key Points

- Store files in S3/MinIO, store metadata (file name, key, size) in your database
- Presigned URLs let clients download directly from S3 — your server is not a proxy
- Use structured keys: `category/date/filename` for organization
- MinIO for local dev, S3 for production — identical API
- Always set content type and size on upload for proper HTTP responses
