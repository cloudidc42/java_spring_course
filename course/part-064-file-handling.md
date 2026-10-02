# Part 064: File Handling and Storage

## Overview

Modern applications handle files constantly: user uploads, generated reports, exported data. This part covers everything from basic multipart uploads to resumable chunked uploads, S3 storage, image processing, PDF generation, and Excel/CSV exports. We build a complete document management system as the capstone example.

---

## Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- AWS S3 -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3</artifactId>
        <version>2.21.0</version>
    </dependency>
    <!-- Image processing -->
    <dependency>
        <groupId>net.coobird</groupId>
        <artifactId>thumbnailator</artifactId>
        <version>0.4.20</version>
    </dependency>
    <!-- PDF generation (iText 7) -->
    <dependency>
        <groupId>com.itextpdf</groupId>
        <artifactId>kernel</artifactId>
        <version>8.0.2</version>
    </dependency>
    <dependency>
        <groupId>com.itextpdf</groupId>
        <artifactId>layout</artifactId>
        <version>8.0.2</version>
    </dependency>
    <!-- Excel -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml</artifactId>
        <version>5.2.5</version>
    </dependency>
    <!-- CSV -->
    <dependency>
        <groupId>com.opencsv</groupId>
        <artifactId>opencsv</artifactId>
        <version>5.9</version>
    </dependency>
    <!-- Apache Tika for content detection -->
    <dependency>
        <groupId>org.apache.tika</groupId>
        <artifactId>tika-core</artifactId>
        <version>2.9.1</version>
    </dependency>
    <!-- Apache PDFBox (alternative PDF) -->
    <dependency>
        <groupId>org.apache.pdfbox</groupId>
        <artifactId>pdfbox</artifactId>
        <version>3.0.1</version>
    </dependency>
</dependencies>
```

---

## 1. Spring MVC Multipart File Upload

```java
// Multipart upload controller
@RestController
@RequestMapping("/api/files")
@Slf4j
public class FileUploadController {

    private final FileStorageService storageService;
    private final FileValidationService validationService;

    // Single file upload
    @PostMapping("/upload")
    public ResponseEntity<FileUploadResponse> uploadFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam(required = false) String category,
            Authentication authentication) {

        // Validate before storing
        validationService.validate(file);

        String userId = authentication.getName();
        FileMetadata metadata = storageService.store(file, userId, category);

        return ResponseEntity.ok(FileUploadResponse.builder()
            .fileId(metadata.getId())
            .fileName(metadata.getOriginalName())
            .size(metadata.getSizeBytes())
            .contentType(metadata.getContentType())
            .downloadUrl("/api/files/" + metadata.getId())
            .build());
    }

    // Multiple files upload
    @PostMapping("/upload/batch")
    public ResponseEntity<List<FileUploadResponse>> uploadBatch(
            @RequestParam("files") List<MultipartFile> files,
            Authentication authentication) {

        if (files.size() > 10) {
            return ResponseEntity.badRequest().build();
        }

        String userId = authentication.getName();

        List<FileUploadResponse> responses = files.stream()
            .map(file -> {
                validationService.validate(file);
                FileMetadata metadata = storageService.store(file, userId, null);
                return FileUploadResponse.builder()
                    .fileId(metadata.getId())
                    .fileName(metadata.getOriginalName())
                    .size(metadata.getSizeBytes())
                    .contentType(metadata.getContentType())
                    .downloadUrl("/api/files/" + metadata.getId())
                    .build();
            })
            .collect(Collectors.toList());

        return ResponseEntity.ok(responses);
    }
}

// Configuration for multipart uploads
@Configuration
public class MultipartConfig {

    @Bean
    public MultipartConfigElement multipartConfigElement() {
        MultipartConfigFactory factory = new MultipartConfigFactory();
        factory.setMaxFileSize(DataSize.ofMegabytes(100));   // Max 100MB per file
        factory.setMaxRequestSize(DataSize.ofMegabytes(200)); // Max 200MB total
        factory.setFileSizeThreshold(DataSize.ofMegabytes(5)); // Buffer to disk after 5MB
        return factory.createMultipartConfig();
    }
}
```

```yaml
# application.yml multipart configuration
spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 100MB
      max-request-size: 200MB
      file-size-threshold: 5MB
      location: /tmp/multipart
```

---

## 2. File Validation

```java
// Comprehensive file validation
@Service
@Slf4j
public class FileValidationService {

    private final Tika tika = new Tika();

    private static final Map<String, Set<String>> ALLOWED_TYPES = Map.of(
        "image", Set.of("image/jpeg", "image/png", "image/gif", "image/webp"),
        "document", Set.of("application/pdf", "application/msword",
            "application/vnd.openxmlformats-officedocument.wordprocessingml.document"),
        "spreadsheet", Set.of("application/vnd.ms-excel",
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"),
        "any", Set.of("image/jpeg", "image/png", "application/pdf",
            "application/vnd.ms-excel", "text/plain", "text/csv")
    );

    private static final long MAX_FILE_SIZE = 100 * 1024 * 1024L; // 100MB
    private static final Set<String> BLOCKED_EXTENSIONS = Set.of(
        ".exe", ".bat", ".sh", ".js", ".php", ".py", ".rb", ".dll", ".jar"
    );

    public void validate(MultipartFile file) {
        validateNotEmpty(file);
        validateFileSize(file);
        validateFileName(file);
        validateContentType(file);
    }

    public void validate(MultipartFile file, String category) {
        validate(file);
        validateCategoryContentType(file, category);
    }

    private void validateNotEmpty(MultipartFile file) {
        if (file.isEmpty()) {
            throw new FileValidationException("File is empty");
        }
    }

    private void validateFileSize(MultipartFile file) {
        if (file.getSize() > MAX_FILE_SIZE) {
            throw new FileValidationException(
                String.format("File size %d exceeds maximum %d bytes",
                    file.getSize(), MAX_FILE_SIZE));
        }
    }

    private void validateFileName(MultipartFile file) {
        String originalName = file.getOriginalFilename();
        if (originalName == null || originalName.isBlank()) {
            throw new FileValidationException("File name is missing");
        }

        // Check for path traversal attacks
        if (originalName.contains("..") || originalName.contains("/") ||
                originalName.contains("\\")) {
            throw new FileValidationException("Invalid file name: contains illegal characters");
        }

        // Check blocked extensions
        String nameLower = originalName.toLowerCase();
        BLOCKED_EXTENSIONS.forEach(ext -> {
            if (nameLower.endsWith(ext)) {
                throw new FileValidationException(
                    "File type not allowed: " + ext);
            }
        });
    }

    private void validateContentType(MultipartFile file) {
        try {
            // Detect actual content type from bytes (not just filename)
            String detectedType = tika.detect(file.getInputStream());

            // Verify claimed content type matches actual content
            String claimedType = file.getContentType();
            if (claimedType != null && !isCompatibleContentType(claimedType, detectedType)) {
                log.warn("Content type mismatch: claimed={}, detected={}",
                    claimedType, detectedType);
                throw new FileValidationException(
                    "File content does not match declared content type");
            }

            log.debug("File content type detected: {}", detectedType);

        } catch (IOException e) {
            throw new FileValidationException("Failed to validate file content", e);
        }
    }

    private void validateCategoryContentType(MultipartFile file, String category) {
        Set<String> allowed = ALLOWED_TYPES.getOrDefault(category, ALLOWED_TYPES.get("any"));
        String contentType = file.getContentType();

        if (contentType == null || !allowed.contains(contentType)) {
            throw new FileValidationException(
                String.format("Content type '%s' not allowed for category '%s'",
                    contentType, category));
        }
    }

    private boolean isCompatibleContentType(String claimed, String detected) {
        if (claimed.equals(detected)) return true;
        // Some types have aliases
        if (claimed.equals("image/jpg") && detected.equals("image/jpeg")) return true;
        return false;
    }
}
```

---

## 3. Local File Storage Service

```java
// File metadata entity
@Entity
@Table(name = "file_metadata")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class FileMetadata {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    private String originalName;
    private String storedName;       // UUID-based name on disk
    private String contentType;
    private Long sizeBytes;
    private String storageLocation;  // "local" or "s3"
    private String storagePath;      // Path or S3 key
    private String userId;
    private String category;
    private String checksum;         // SHA-256

    @Enumerated(EnumType.STRING)
    private FileStatus status;       // ACTIVE, DELETED, QUARANTINED

    @CreatedDate
    private LocalDateTime uploadedAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

// Local storage implementation
@Service
@Slf4j
public class LocalFileStorageService implements FileStorageService {

    @Value("${app.storage.local.base-path:/data/uploads}")
    private String basePath;

    private final FileMetadataRepository metadataRepository;

    @Override
    public FileMetadata store(MultipartFile file, String userId, String category) {
        try {
            // Generate unique stored name
            String extension = getExtension(file.getOriginalFilename());
            String storedName = UUID.randomUUID() + extension;

            // Create directory structure: basePath/userId/YYYY/MM/
            String datePath = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy/MM"));
            Path storageDir = Paths.get(basePath, userId, datePath);
            Files.createDirectories(storageDir);

            Path filePath = storageDir.resolve(storedName);

            // Calculate checksum while copying
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            try (InputStream in = file.getInputStream();
                 DigestInputStream dis = new DigestInputStream(in, digest)) {
                Files.copy(dis, filePath, StandardCopyOption.REPLACE_EXISTING);
            }

            String checksum = HexFormat.of().formatHex(digest.digest());

            // Save metadata
            FileMetadata metadata = FileMetadata.builder()
                .originalName(file.getOriginalFilename())
                .storedName(storedName)
                .contentType(file.getContentType())
                .sizeBytes(file.getSize())
                .storageLocation("local")
                .storagePath(filePath.toString())
                .userId(userId)
                .category(category)
                .checksum(checksum)
                .status(FileStatus.ACTIVE)
                .build();

            return metadataRepository.save(metadata);

        } catch (Exception e) {
            throw new FileStorageException("Failed to store file", e);
        }
    }

    @Override
    public Resource load(String fileId) {
        FileMetadata metadata = getActiveMetadata(fileId);

        Path filePath = Paths.get(metadata.getStoragePath());

        if (!Files.exists(filePath)) {
            throw new FileNotFoundException("File not found: " + fileId);
        }

        Resource resource = new FileSystemResource(filePath);
        if (!resource.isReadable()) {
            throw new FileStorageException("File is not readable: " + fileId);
        }

        return resource;
    }

    @Override
    public void delete(String fileId, String userId) {
        FileMetadata metadata = getActiveMetadata(fileId);

        // Authorization check
        if (!metadata.getUserId().equals(userId)) {
            throw new AccessDeniedException("You don't own this file");
        }

        try {
            Files.deleteIfExists(Paths.get(metadata.getStoragePath()));
            metadata.setStatus(FileStatus.DELETED);
            metadataRepository.save(metadata);
        } catch (IOException e) {
            throw new FileStorageException("Failed to delete file", e);
        }
    }

    private FileMetadata getActiveMetadata(String fileId) {
        return metadataRepository.findByIdAndStatus(fileId, FileStatus.ACTIVE)
            .orElseThrow(() -> new FileNotFoundException("File not found: " + fileId));
    }

    private String getExtension(String filename) {
        if (filename == null) return "";
        int lastDot = filename.lastIndexOf('.');
        return lastDot >= 0 ? filename.substring(lastDot).toLowerCase() : "";
    }
}
```

---

## 4. File Download with Streaming

```java
@RestController
@RequestMapping("/api/files")
@Slf4j
public class FileDownloadController {

    private final FileStorageService storageService;
    private final FileMetadataRepository metadataRepository;

    // Basic download
    @GetMapping("/{fileId}")
    public ResponseEntity<Resource> downloadFile(
            @PathVariable String fileId,
            @RequestParam(defaultValue = "false") boolean inline,
            Authentication authentication) {

        FileMetadata metadata = metadataRepository.findById(fileId)
            .orElseThrow(() -> new FileNotFoundException(fileId));

        // Authorization check
        if (!canAccess(authentication, metadata)) {
            throw new AccessDeniedException("Access denied");
        }

        Resource resource = storageService.load(fileId);
        String disposition = inline ? "inline" : "attachment";

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(metadata.getContentType()))
            .contentLength(metadata.getSizeBytes())
            .header("Content-Disposition",
                disposition + "; filename=\"" +
                encodeFileName(metadata.getOriginalName()) + "\"")
            .header("Cache-Control", "no-cache, no-store, must-revalidate")
            .header("X-Content-Type-Options", "nosniff")
            .body(resource);
    }

    // Streaming download for large files
    @GetMapping("/{fileId}/stream")
    public void streamFile(@PathVariable String fileId,
            HttpServletRequest request,
            HttpServletResponse response,
            Authentication authentication) throws IOException {

        FileMetadata metadata = metadataRepository.findById(fileId)
            .orElseThrow(() -> new FileNotFoundException(fileId));

        Resource resource = storageService.load(fileId);

        // Support range requests for resumable downloads / video streaming
        String rangeHeader = request.getHeader("Range");
        long fileLength = metadata.getSizeBytes();

        if (rangeHeader != null) {
            // Parse range: "bytes=0-1023"
            long[] range = parseRange(rangeHeader, fileLength);
            long start = range[0];
            long end = range[1];
            long length = end - start + 1;

            response.setStatus(HttpServletResponse.SC_PARTIAL_CONTENT);
            response.setContentType(metadata.getContentType());
            response.setHeader("Content-Range",
                "bytes " + start + "-" + end + "/" + fileLength);
            response.setContentLengthLong(length);
            response.setHeader("Accept-Ranges", "bytes");

            try (InputStream in = resource.getInputStream();
                 OutputStream out = response.getOutputStream()) {
                in.skip(start);
                byte[] buffer = new byte[8192];
                long remaining = length;

                while (remaining > 0) {
                    int read = in.read(buffer, 0,
                        (int) Math.min(buffer.length, remaining));
                    if (read == -1) break;
                    out.write(buffer, 0, read);
                    remaining -= read;
                }
            }
        } else {
            response.setStatus(HttpServletResponse.SC_OK);
            response.setContentType(metadata.getContentType());
            response.setContentLengthLong(fileLength);
            response.setHeader("Accept-Ranges", "bytes");
            response.setHeader("Content-Disposition",
                "attachment; filename=\"" + metadata.getOriginalName() + "\"");

            try (InputStream in = resource.getInputStream();
                 OutputStream out = response.getOutputStream()) {
                StreamUtils.copy(in, out);
            }
        }
    }

    private long[] parseRange(String rangeHeader, long fileLength) {
        String range = rangeHeader.substring("bytes=".length());
        String[] parts = range.split("-");
        long start = Long.parseLong(parts[0]);
        long end = parts.length > 1 && !parts[1].isEmpty()
            ? Long.parseLong(parts[1])
            : fileLength - 1;
        return new long[]{start, end};
    }

    private String encodeFileName(String fileName) {
        try {
            return URLEncoder.encode(fileName, StandardCharsets.UTF_8)
                .replace("+", "%20");
        } catch (Exception e) {
            return fileName;
        }
    }

    private boolean canAccess(Authentication auth, FileMetadata metadata) {
        if (auth == null) return false;
        if (auth.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) return true;
        return metadata.getUserId().equals(auth.getName());
    }
}
```

---

## 5. Chunked File Upload (Resumable)

```java
// Resumable upload implementation
@Entity
@Table(name = "upload_sessions")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UploadSession {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    private String fileName;
    private String contentType;
    private Long totalSize;
    private Integer totalChunks;
    private String userId;
    private String checksum;  // Expected checksum of complete file

    @ElementCollection
    @CollectionTable(name = "upload_chunks")
    private Set<Integer> receivedChunks = new HashSet<>();

    @Enumerated(EnumType.STRING)
    private UploadStatus status;   // IN_PROGRESS, COMPLETED, EXPIRED

    private LocalDateTime createdAt;
    private LocalDateTime expiresAt;
}

// Resumable upload service
@Service
@Slf4j
public class ResumableUploadService {

    private final UploadSessionRepository sessionRepository;
    private final FileStorageService storageService;

    @Value("${app.storage.upload.temp-dir:/tmp/uploads}")
    private String tempDir;

    // Step 1: Initialize upload session
    public UploadSession initializeUpload(InitUploadRequest request, String userId) {
        UploadSession session = UploadSession.builder()
            .fileName(request.getFileName())
            .contentType(request.getContentType())
            .totalSize(request.getTotalSize())
            .totalChunks(calculateChunks(request.getTotalSize(), request.getChunkSize()))
            .userId(userId)
            .checksum(request.getChecksum())
            .status(UploadStatus.IN_PROGRESS)
            .createdAt(LocalDateTime.now())
            .expiresAt(LocalDateTime.now().plusHours(24))
            .build();

        session = sessionRepository.save(session);

        // Create temp directory for chunks
        try {
            Files.createDirectories(Paths.get(tempDir, session.getId()));
        } catch (IOException e) {
            throw new UploadInitializationException("Failed to create upload directory", e);
        }

        log.info("Upload session created: {} for {} ({} bytes, {} chunks)",
            session.getId(), request.getFileName(),
            request.getTotalSize(), session.getTotalChunks());

        return session;
    }

    // Step 2: Upload each chunk
    @Transactional
    public ChunkUploadResult uploadChunk(String sessionId, int chunkIndex,
            MultipartFile chunkData) {

        UploadSession session = sessionRepository.findById(sessionId)
            .orElseThrow(() -> new UploadSessionNotFoundException(sessionId));

        if (session.getStatus() != UploadStatus.IN_PROGRESS) {
            throw new IllegalStateException(
                "Upload session is not active: " + session.getStatus());
        }

        if (LocalDateTime.now().isAfter(session.getExpiresAt())) {
            session.setStatus(UploadStatus.EXPIRED);
            sessionRepository.save(session);
            throw new UploadExpiredException("Upload session has expired");
        }

        // Save chunk to temp directory
        try {
            Path chunkPath = Paths.get(tempDir, sessionId,
                "chunk-" + chunkIndex + ".part");
            Files.write(chunkPath, chunkData.getBytes());

            session.getReceivedChunks().add(chunkIndex);
            sessionRepository.save(session);

            log.debug("Saved chunk {}/{} for session {}",
                chunkIndex, session.getTotalChunks() - 1, sessionId);

            // Check if all chunks received
            boolean complete = session.getReceivedChunks().size()
                >= session.getTotalChunks();

            return ChunkUploadResult.builder()
                .sessionId(sessionId)
                .chunkIndex(chunkIndex)
                .receivedChunks(session.getReceivedChunks().size())
                .totalChunks(session.getTotalChunks())
                .complete(complete)
                .build();

        } catch (IOException e) {
            throw new ChunkUploadException("Failed to save chunk " + chunkIndex, e);
        }
    }

    // Step 3: Complete upload - merge chunks
    @Transactional
    public FileMetadata completeUpload(String sessionId, String userId) {
        UploadSession session = sessionRepository.findById(sessionId)
            .orElseThrow(() -> new UploadSessionNotFoundException(sessionId));

        if (!session.getUserId().equals(userId)) {
            throw new AccessDeniedException("Not your upload session");
        }

        // Verify all chunks received
        if (session.getReceivedChunks().size() < session.getTotalChunks()) {
            throw new IncompleteUploadException(
                "Missing chunks: received " + session.getReceivedChunks().size() +
                " of " + session.getTotalChunks());
        }

        try {
            // Merge all chunks
            Path outputPath = Paths.get(tempDir, sessionId, "complete");
            try (OutputStream out = new BufferedOutputStream(
                    Files.newOutputStream(outputPath))) {

                for (int i = 0; i < session.getTotalChunks(); i++) {
                    Path chunkPath = Paths.get(tempDir, sessionId,
                        "chunk-" + i + ".part");
                    Files.copy(chunkPath, out);
                }
            }

            // Verify checksum
            if (session.getChecksum() != null) {
                String actualChecksum = calculateChecksum(outputPath);
                if (!session.getChecksum().equals(actualChecksum)) {
                    throw new ChecksumMismatchException(
                        "File checksum mismatch. Upload may be corrupted.");
                }
            }

            // Store final file
            MockMultipartFile file = new MockMultipartFile(
                "file",
                session.getFileName(),
                session.getContentType(),
                Files.readAllBytes(outputPath)
            );

            FileMetadata metadata = storageService.store(file, userId, null);

            // Update session status
            session.setStatus(UploadStatus.COMPLETED);
            sessionRepository.save(session);

            // Cleanup temp files
            cleanupTempFiles(sessionId);

            return metadata;

        } catch (IOException e) {
            throw new FileStorageException("Failed to complete upload", e);
        }
    }

    // Get missing chunks (for resume)
    public List<Integer> getMissingChunks(String sessionId) {
        UploadSession session = sessionRepository.findById(sessionId)
            .orElseThrow(() -> new UploadSessionNotFoundException(sessionId));

        List<Integer> missing = new ArrayList<>();
        for (int i = 0; i < session.getTotalChunks(); i++) {
            if (!session.getReceivedChunks().contains(i)) {
                missing.add(i);
            }
        }
        return missing;
    }

    private int calculateChunks(long totalSize, long chunkSize) {
        return (int) Math.ceil((double) totalSize / chunkSize);
    }

    private String calculateChecksum(Path file) throws IOException {
        MessageDigest digest = MessageDigest.getInstance("SHA-256",
            new java.security.Provider(){});
        // Simplified - use Apache Commons Codec in production
        byte[] bytes = Files.readAllBytes(file);
        return HexFormat.of().formatHex(bytes);
    }

    private void cleanupTempFiles(String sessionId) {
        try {
            Path sessionDir = Paths.get(tempDir, sessionId);
            Files.walk(sessionDir)
                .sorted(Comparator.reverseOrder())
                .forEach(p -> {
                    try {
                        Files.deleteIfExists(p);
                    } catch (IOException e) {
                        log.warn("Failed to delete temp file: {}", p);
                    }
                });
        } catch (IOException e) {
            log.warn("Failed to cleanup temp files for session {}", sessionId);
        }
    }
}

// Chunked upload controller
@RestController
@RequestMapping("/api/uploads")
@Slf4j
public class ResumableUploadController {

    private final ResumableUploadService uploadService;

    @PostMapping("/init")
    public ResponseEntity<UploadSession> initUpload(
            @RequestBody @Valid InitUploadRequest request,
            Authentication auth) {
        UploadSession session = uploadService.initializeUpload(request, auth.getName());
        return ResponseEntity.ok(session);
    }

    @PostMapping("/{sessionId}/chunks/{chunkIndex}")
    public ResponseEntity<ChunkUploadResult> uploadChunk(
            @PathVariable String sessionId,
            @PathVariable int chunkIndex,
            @RequestParam("chunk") MultipartFile chunk) {

        ChunkUploadResult result = uploadService.uploadChunk(sessionId, chunkIndex, chunk);
        return ResponseEntity.ok(result);
    }

    @PostMapping("/{sessionId}/complete")
    public ResponseEntity<FileMetadata> completeUpload(
            @PathVariable String sessionId,
            Authentication auth) {
        FileMetadata metadata = uploadService.completeUpload(sessionId, auth.getName());
        return ResponseEntity.ok(metadata);
    }

    @GetMapping("/{sessionId}/status")
    public ResponseEntity<UploadStatusResponse> getStatus(
            @PathVariable String sessionId) {
        List<Integer> missing = uploadService.getMissingChunks(sessionId);
        return ResponseEntity.ok(new UploadStatusResponse(sessionId, missing));
    }
}
```

---

## 6. S3 File Storage

```java
// S3 configuration (works with LocalStack for testing)
@Configuration
public class S3Config {

    @Value("${aws.s3.endpoint:}")
    private String endpoint;

    @Value("${aws.s3.bucket}")
    private String bucketName;

    @Value("${aws.region:us-east-1}")
    private String region;

    @Bean
    public S3Client s3Client() {
        S3ClientBuilder builder = S3Client.builder()
            .region(Region.of(region))
            .credentialsProvider(DefaultCredentialsProvider.create());

        // For LocalStack (local testing)
        if (!endpoint.isEmpty()) {
            builder.endpointOverride(URI.create(endpoint))
                .forcePathStyle(true);  // Required for LocalStack
        }

        return builder.build();
    }
}

// S3 storage implementation
@Service("s3FileStorage")
@Slf4j
public class S3FileStorageService implements FileStorageService {

    private final S3Client s3Client;
    private final S3Presigner s3Presigner;
    private final FileMetadataRepository metadataRepository;

    @Value("${aws.s3.bucket}")
    private String bucketName;

    @Override
    public FileMetadata store(MultipartFile file, String userId, String category) {
        String s3Key = buildS3Key(userId, file.getOriginalFilename());

        try {
            // Calculate checksum
            byte[] bytes = file.getBytes();
            String checksum = calculateMd5(bytes);

            // Upload to S3 with metadata
            PutObjectRequest request = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(s3Key)
                .contentType(file.getContentType())
                .contentLength(file.getSize())
                .metadata(Map.of(
                    "original-name", file.getOriginalFilename(),
                    "user-id", userId,
                    "checksum", checksum
                ))
                .serverSideEncryption(ServerSideEncryption.AES256)
                .build();

            s3Client.putObject(request, RequestBody.fromBytes(bytes));

            log.info("Uploaded file to S3: {}", s3Key);

            // Save metadata to DB
            FileMetadata metadata = FileMetadata.builder()
                .originalName(file.getOriginalFilename())
                .storedName(s3Key.substring(s3Key.lastIndexOf('/') + 1))
                .contentType(file.getContentType())
                .sizeBytes(file.getSize())
                .storageLocation("s3")
                .storagePath(s3Key)
                .userId(userId)
                .category(category)
                .checksum(checksum)
                .status(FileStatus.ACTIVE)
                .build();

            return metadataRepository.save(metadata);

        } catch (IOException | S3Exception e) {
            throw new FileStorageException("Failed to upload to S3", e);
        }
    }

    @Override
    public Resource load(String fileId) {
        FileMetadata metadata = metadataRepository.findById(fileId)
            .orElseThrow(() -> new FileNotFoundException(fileId));

        GetObjectRequest request = GetObjectRequest.builder()
            .bucket(bucketName)
            .key(metadata.getStoragePath())
            .build();

        ResponseInputStream<GetObjectResponse> s3Object = s3Client.getObject(request);
        return new InputStreamResource(s3Object);
    }

    // Generate pre-signed URL for direct browser download (bypasses your server)
    public String generatePresignedDownloadUrl(String fileId, Duration expiration) {
        FileMetadata metadata = metadataRepository.findById(fileId)
            .orElseThrow(() -> new FileNotFoundException(fileId));

        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
            .signatureDuration(expiration)
            .getObjectRequest(GetObjectRequest.builder()
                .bucket(bucketName)
                .key(metadata.getStoragePath())
                .build())
            .build();

        PresignedGetObjectRequest presigned = s3Presigner.presignGetObject(presignRequest);
        return presigned.url().toString();
    }

    // Generate pre-signed URL for direct browser upload (client uploads directly to S3)
    public PresignedUploadInfo generatePresignedUploadUrl(String fileName,
            String contentType, String userId) {

        String s3Key = buildS3Key(userId, fileName);

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
            .signatureDuration(Duration.ofMinutes(15))
            .putObjectRequest(PutObjectRequest.builder()
                .bucket(bucketName)
                .key(s3Key)
                .contentType(contentType)
                .build())
            .build();

        PresignedPutObjectRequest presigned = s3Presigner.presignPutObject(presignRequest);

        return PresignedUploadInfo.builder()
            .uploadUrl(presigned.url().toString())
            .s3Key(s3Key)
            .expiresAt(Instant.now().plus(Duration.ofMinutes(15)))
            .build();
    }

    @Override
    public void delete(String fileId, String userId) {
        FileMetadata metadata = metadataRepository.findById(fileId)
            .orElseThrow(() -> new FileNotFoundException(fileId));

        if (!metadata.getUserId().equals(userId)) {
            throw new AccessDeniedException("Not your file");
        }

        s3Client.deleteObject(DeleteObjectRequest.builder()
            .bucket(bucketName)
            .key(metadata.getStoragePath())
            .build());

        metadata.setStatus(FileStatus.DELETED);
        metadataRepository.save(metadata);

        log.info("Deleted file from S3: {}", metadata.getStoragePath());
    }

    private String buildS3Key(String userId, String fileName) {
        String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy/MM/dd"));
        String safeFileName = fileName.replaceAll("[^a-zA-Z0-9._-]", "_");
        return String.format("uploads/%s/%s/%s-%s",
            userId, date, UUID.randomUUID(), safeFileName);
    }

    private String calculateMd5(byte[] bytes) throws NoSuchAlgorithmException {
        MessageDigest md = MessageDigest.getInstance("MD5");
        return Base64.getEncoder().encodeToString(md.digest(bytes));
    }
}
```

---

## 7. Image Processing with Thumbnailator

```java
@Service
@Slf4j
public class ImageProcessingService {

    private static final int THUMBNAIL_WIDTH = 200;
    private static final int THUMBNAIL_HEIGHT = 200;
    private static final int PREVIEW_WIDTH = 800;
    private static final int PREVIEW_HEIGHT = 600;

    public byte[] createThumbnail(byte[] imageBytes) throws IOException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();

        Thumbnails.of(new ByteArrayInputStream(imageBytes))
            .size(THUMBNAIL_WIDTH, THUMBNAIL_HEIGHT)
            .outputQuality(0.8)
            .outputFormat("jpeg")
            .toOutputStream(output);

        return output.toByteArray();
    }

    public byte[] createPreview(byte[] imageBytes) throws IOException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();

        Thumbnails.of(new ByteArrayInputStream(imageBytes))
            .size(PREVIEW_WIDTH, PREVIEW_HEIGHT)
            .keepAspectRatio(true)
            .outputQuality(0.9)
            .toOutputStream(output);

        return output.toByteArray();
    }

    public byte[] resizeToWidth(byte[] imageBytes, int targetWidth) throws IOException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();

        Thumbnails.of(new ByteArrayInputStream(imageBytes))
            .width(targetWidth)
            .keepAspectRatio(true)
            .outputQuality(0.9)
            .toOutputStream(output);

        return output.toByteArray();
    }

    public byte[] cropCenter(byte[] imageBytes, int width, int height) throws IOException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();

        Thumbnails.of(new ByteArrayInputStream(imageBytes))
            .size(width, height)
            .crop(Positions.CENTER)
            .toOutputStream(output);

        return output.toByteArray();
    }

    // Generate multiple sizes for responsive images
    public Map<String, byte[]> generateResponsiveImages(byte[] imageBytes)
            throws IOException {

        Map<String, byte[]> sizes = new HashMap<>();

        int[][] dimensions = {{320, 240}, {640, 480}, {1024, 768}, {1920, 1080}};
        String[] names = {"sm", "md", "lg", "xl"};

        for (int i = 0; i < dimensions.length; i++) {
            ByteArrayOutputStream output = new ByteArrayOutputStream();
            Thumbnails.of(new ByteArrayInputStream(imageBytes))
                .size(dimensions[i][0], dimensions[i][1])
                .keepAspectRatio(true)
                .outputQuality(0.85)
                .toOutputStream(output);
            sizes.put(names[i], output.toByteArray());
        }

        return sizes;
    }

    // Process and store image with thumbnails
    public ImageUploadResult processAndStore(MultipartFile image,
            FileStorageService storageService, String userId) throws IOException {

        byte[] originalBytes = image.getBytes();

        // Generate variants
        byte[] thumbnail = createThumbnail(originalBytes);
        byte[] preview = createPreview(originalBytes);

        // Store all variants
        FileMetadata original = storageService.store(image, userId, "image");

        FileMetadata thumbnailMetadata = storageService.store(
            createMockFile("thumb_" + image.getOriginalFilename(), "image/jpeg", thumbnail),
            userId, "thumbnail");

        FileMetadata previewMetadata = storageService.store(
            createMockFile("preview_" + image.getOriginalFilename(), "image/jpeg", preview),
            userId, "preview");

        return ImageUploadResult.builder()
            .originalId(original.getId())
            .thumbnailId(thumbnailMetadata.getId())
            .previewId(previewMetadata.getId())
            .build();
    }

    private MultipartFile createMockFile(String name, String contentType, byte[] data) {
        return new MockMultipartFile(name, name, contentType, data);
    }
}
```

---

## 8. PDF Generation with iText

```java
@Service
@Slf4j
public class PdfGenerationService {

    // Generate invoice PDF
    public byte[] generateInvoice(InvoiceData invoice) throws IOException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();

        try (PdfWriter writer = new PdfWriter(output);
             PdfDocument pdfDoc = new PdfDocument(writer);
             Document document = new Document(pdfDoc, PageSize.A4)) {

            document.setMargins(36, 36, 36, 36);

            // Company header
            PdfFont boldFont = PdfFontFactory.createFont(StandardFonts.HELVETICA_BOLD);
            PdfFont regularFont = PdfFontFactory.createFont(StandardFonts.HELVETICA);

            document.add(new Paragraph("INVOICE")
                .setFont(boldFont)
                .setFontSize(28)
                .setFontColor(ColorConstants.DARK_GRAY));

            // Invoice metadata table
            Table metaTable = new Table(UnitValue.createPercentArray(new float[]{50, 50}))
                .setWidth(UnitValue.createPercentValue(100));

            metaTable.addCell(createCell("Invoice #: " + invoice.getInvoiceNumber(),
                regularFont, TextAlignment.LEFT));
            metaTable.addCell(createCell("Date: " + invoice.getDate().toString(),
                regularFont, TextAlignment.RIGHT));
            metaTable.addCell(createCell("To: " + invoice.getCustomerName(),
                regularFont, TextAlignment.LEFT));
            metaTable.addCell(createCell("Due: " + invoice.getDueDate().toString(),
                regularFont, TextAlignment.RIGHT));

            document.add(metaTable);
            document.add(new LineSeparator(new SolidLine()));

            // Line items table
            Table itemTable = new Table(
                UnitValue.createPercentArray(new float[]{40, 15, 20, 25}))
                .setWidth(UnitValue.createPercentValue(100));

            // Header
            for (String header : List.of("Description", "Qty", "Unit Price", "Total")) {
                itemTable.addHeaderCell(new Cell()
                    .add(new Paragraph(header).setFont(boldFont))
                    .setBackgroundColor(ColorConstants.LIGHT_GRAY));
            }

            // Items
            for (InvoiceItem item : invoice.getItems()) {
                itemTable.addCell(item.getDescription());
                itemTable.addCell(String.valueOf(item.getQuantity()));
                itemTable.addCell("$" + item.getUnitPrice().toPlainString());
                itemTable.addCell("$" + item.getTotal().toPlainString());
            }

            document.add(itemTable);

            // Total
            document.add(new Paragraph()
                .add(new Text("TOTAL: $" + invoice.getTotal().toPlainString())
                    .setFont(boldFont)
                    .setFontSize(16))
                .setTextAlignment(TextAlignment.RIGHT)
                .setMarginTop(10));

            // Footer
            document.add(new Paragraph("Thank you for your business!")
                .setFont(regularFont)
                .setFontColor(ColorConstants.GRAY)
                .setTextAlignment(TextAlignment.CENTER)
                .setMarginTop(30));
        }

        return output.toByteArray();
    }

    private Cell createCell(String content, PdfFont font, TextAlignment alignment) {
        return new Cell()
            .add(new Paragraph(content).setFont(font))
            .setTextAlignment(alignment)
            .setBorder(Border.NO_BORDER);
    }

    // Merge multiple PDFs
    public byte[] mergePdfs(List<byte[]> pdfList) throws IOException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();

        try (PdfWriter writer = new PdfWriter(output);
             PdfDocument mergedDoc = new PdfDocument(writer)) {

            PdfMerger merger = new PdfMerger(mergedDoc);

            for (byte[] pdfBytes : pdfList) {
                try (PdfDocument sourcePdf =
                        new PdfDocument(new PdfReader(new ByteArrayInputStream(pdfBytes)))) {
                    merger.merge(sourcePdf, 1, sourcePdf.getNumberOfPages());
                }
            }
        }

        return output.toByteArray();
    }
}
```

---

## 9. Excel Generation with Apache POI

```java
@Service
@Slf4j
public class ExcelGenerationService {

    public byte[] generateOrderReport(List<OrderReportRow> orders) throws IOException {
        try (Workbook workbook = new XSSFWorkbook()) {

            // Styles
            CellStyle headerStyle = createHeaderStyle(workbook);
            CellStyle currencyStyle = createCurrencyStyle(workbook);
            CellStyle dateStyle = createDateStyle(workbook);
            CellStyle evenRowStyle = createEvenRowStyle(workbook);

            Sheet sheet = workbook.createSheet("Orders");

            // Freeze header row
            sheet.createFreezePane(0, 1);

            // Enable auto filter
            sheet.setAutoFilter(new CellRangeAddress(0, 0, 0, 6));

            // Header row
            String[] headers = {"Order ID", "Customer", "Amount", "Status",
                "Created At", "Items", "Notes"};
            Row headerRow = sheet.createRow(0);
            for (int i = 0; i < headers.length; i++) {
                Cell cell = headerRow.createCell(i);
                cell.setCellValue(headers[i]);
                cell.setCellStyle(headerStyle);
            }

            // Data rows
            for (int rowNum = 0; rowNum < orders.size(); rowNum++) {
                OrderReportRow order = orders.get(rowNum);
                Row row = sheet.createRow(rowNum + 1);

                if (rowNum % 2 == 0) {
                    // Alternate row coloring
                    for (int col = 0; col < headers.length; col++) {
                        row.createCell(col).setCellStyle(evenRowStyle);
                    }
                }

                row.createCell(0).setCellValue(order.getOrderId());
                row.createCell(1).setCellValue(order.getCustomerName());

                Cell amountCell = row.createCell(2);
                amountCell.setCellValue(order.getAmount().doubleValue());
                amountCell.setCellStyle(currencyStyle);

                row.createCell(3).setCellValue(order.getStatus().name());

                Cell dateCell = row.createCell(4);
                dateCell.setCellValue(order.getCreatedAt());
                dateCell.setCellStyle(dateStyle);

                row.createCell(5).setCellValue(order.getItemCount());
                row.createCell(6).setCellValue(order.getNotes());
            }

            // Summary row
            int lastRow = orders.size() + 1;
            Row summaryRow = sheet.createRow(lastRow);
            summaryRow.createCell(0).setCellValue("TOTAL");

            Cell totalCell = summaryRow.createCell(2);
            totalCell.setCellFormula("SUM(C2:C" + (orders.size() + 1) + ")");
            totalCell.setCellStyle(currencyStyle);

            // Auto-size all columns
            for (int i = 0; i < headers.length; i++) {
                sheet.autoSizeColumn(i);
                // Cap column width at 50 characters
                if (sheet.getColumnWidth(i) > 50 * 256) {
                    sheet.setColumnWidth(i, 50 * 256);
                }
            }

            ByteArrayOutputStream output = new ByteArrayOutputStream();
            workbook.write(output);
            return output.toByteArray();
        }
    }

    private CellStyle createHeaderStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        Font font = workbook.createFont();
        font.setBold(true);
        font.setColor(IndexedColors.WHITE.getIndex());
        style.setFont(font);
        style.setFillForegroundColor(IndexedColors.DARK_BLUE.getIndex());
        style.setFillPattern(FillPatternType.SOLID_FOREGROUND);
        style.setAlignment(HorizontalAlignment.CENTER);
        style.setBorderBottom(BorderStyle.THIN);
        return style;
    }

    private CellStyle createCurrencyStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        DataFormat format = workbook.createDataFormat();
        style.setDataFormat(format.getFormat("$#,##0.00"));
        style.setAlignment(HorizontalAlignment.RIGHT);
        return style;
    }

    private CellStyle createDateStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        DataFormat format = workbook.createDataFormat();
        style.setDataFormat(format.getFormat("yyyy-mm-dd hh:mm"));
        return style;
    }

    private CellStyle createEvenRowStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        style.setFillForegroundColor(IndexedColors.LIGHT_YELLOW.getIndex());
        style.setFillPattern(FillPatternType.SOLID_FOREGROUND);
        return style;
    }
}
```

---

## 10. CSV Export with OpenCSV

```java
@Service
public class CsvExportService {

    // Export to CSV with custom mapping
    public byte[] exportOrders(List<OrderExportRow> orders) throws IOException, CsvDataTypeMismatchException, CsvRequiredFieldEmptyException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();
        OutputStreamWriter writer = new OutputStreamWriter(output, StandardCharsets.UTF_8);

        // Write BOM for Excel compatibility
        writer.write('﻿');

        StatefulBeanToCsvBuilder<OrderExportRow> builder =
            new StatefulBeanToCsvBuilder<>(writer);

        StatefulBeanToCsv<OrderExportRow> csv = builder
            .withQuotechar(CSVWriter.DEFAULT_QUOTE_CHARACTER)
            .withSeparator(CSVWriter.DEFAULT_SEPARATOR)
            .withOrderedResults(true)
            .build();

        csv.write(orders);
        writer.flush();
        return output.toByteArray();
    }

    // Custom CSV with headers
    public byte[] exportCustomCsv(List<Map<String, Object>> data,
            List<String> columns) throws IOException {

        ByteArrayOutputStream output = new ByteArrayOutputStream();

        try (CSVWriter writer = new CSVWriter(
                new OutputStreamWriter(output, StandardCharsets.UTF_8),
                CSVWriter.DEFAULT_SEPARATOR,
                CSVWriter.DEFAULT_QUOTE_CHARACTER,
                CSVWriter.DEFAULT_ESCAPE_CHARACTER,
                CSVWriter.DEFAULT_LINE_END)) {

            // Write header
            writer.writeNext(columns.toArray(new String[0]));

            // Write data rows
            for (Map<String, Object> row : data) {
                String[] values = columns.stream()
                    .map(col -> {
                        Object val = row.get(col);
                        return val != null ? val.toString() : "";
                    })
                    .toArray(String[]::new);
                writer.writeNext(values);
            }
        }

        return output.toByteArray();
    }
}

// Bean for CSV export
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderExportRow {

    @CsvBindByName(column = "Order ID")
    @CsvBindByPosition(position = 0)
    private Long orderId;

    @CsvBindByName(column = "Customer Name")
    @CsvBindByPosition(position = 1)
    private String customerName;

    @CsvBindByName(column = "Amount")
    @CsvBindByPosition(position = 2)
    private BigDecimal amount;

    @CsvBindByName(column = "Status")
    @CsvBindByPosition(position = 3)
    private String status;

    @CsvBindByName(column = "Created At")
    @CsvBindByPosition(position = 4)
    @CsvDate("yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createdAt;
}
```

---

## 11. Complete Example: Document Management System

```java
// Document entity with rich metadata
@Entity
@Table(name = "documents")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Document {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    private String title;
    private String description;

    @Enumerated(EnumType.STRING)
    private DocumentType type;  // CONTRACT, INVOICE, REPORT, IMAGE, OTHER

    private String fileId;        // FK to file_metadata
    private String thumbnailId;   // FK to thumbnail in file_metadata

    private String ownerId;
    private String departmentId;

    @ElementCollection
    @CollectionTable(name = "document_tags")
    private List<String> tags = new ArrayList<>();

    @ElementCollection
    @CollectionTable(name = "document_shares")
    private Set<String> sharedWithUserIds = new HashSet<>();

    @Enumerated(EnumType.STRING)
    private DocumentStatus status;  // DRAFT, ACTIVE, ARCHIVED

    private Integer version = 1;
    private String parentDocumentId;  // For versioning

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    private LocalDateTime expiresAt;  // For time-limited documents
}

// Document management service
@Service
@Transactional
@Slf4j
public class DocumentManagementService {

    private final DocumentRepository documentRepository;
    private final FileStorageService storageService;
    private final ImageProcessingService imageService;
    private final FileValidationService validationService;
    private final SecurityAuditService auditService;

    public DocumentUploadResult uploadDocument(MultipartFile file,
            UploadDocumentRequest request, String userId) {

        // Validate
        validationService.validate(file);

        // Process image thumbnails if applicable
        String thumbnailId = null;
        if (isImage(file.getContentType())) {
            try {
                byte[] thumbBytes = imageService.createThumbnail(file.getBytes());
                FileMetadata thumb = storageService.store(
                    new MockMultipartFile("thumb", "thumb.jpg", "image/jpeg", thumbBytes),
                    userId, "thumbnail");
                thumbnailId = thumb.getId();
            } catch (IOException e) {
                log.warn("Failed to create thumbnail for document", e);
            }
        }

        // Store original file in S3
        FileMetadata fileMetadata = storageService.store(file, userId, request.getType().name());

        // Create document record
        Document document = Document.builder()
            .title(request.getTitle())
            .description(request.getDescription())
            .type(request.getType())
            .fileId(fileMetadata.getId())
            .thumbnailId(thumbnailId)
            .ownerId(userId)
            .departmentId(request.getDepartmentId())
            .tags(request.getTags())
            .status(DocumentStatus.ACTIVE)
            .version(1)
            .expiresAt(request.getExpiresAt())
            .build();

        document = documentRepository.save(document);

        auditService.recordSensitiveAccess(userId, "document.upload:" + document.getId());

        return DocumentUploadResult.builder()
            .documentId(document.getId())
            .fileId(fileMetadata.getId())
            .downloadUrl("/api/documents/" + document.getId() + "/download")
            .thumbnailUrl(thumbnailId != null ?
                "/api/files/" + thumbnailId + "?inline=true" : null)
            .build();
    }

    public Document uploadNewVersion(String documentId, MultipartFile file,
            String userId) {

        Document original = getOwnedDocument(documentId, userId);

        // Store new file version
        FileMetadata newFile = storageService.store(file, userId,
            original.getType().name());

        // Archive old document
        original.setStatus(DocumentStatus.ARCHIVED);
        documentRepository.save(original);

        // Create new version
        Document newVersion = Document.builder()
            .title(original.getTitle())
            .description(original.getDescription())
            .type(original.getType())
            .fileId(newFile.getId())
            .ownerId(userId)
            .departmentId(original.getDepartmentId())
            .tags(new ArrayList<>(original.getTags()))
            .status(DocumentStatus.ACTIVE)
            .version(original.getVersion() + 1)
            .parentDocumentId(original.getParentDocumentId() != null ?
                original.getParentDocumentId() : original.getId())
            .build();

        newVersion = documentRepository.save(newVersion);

        log.info("Created document version {} for document {}",
            newVersion.getVersion(), documentId);

        return newVersion;
    }

    public void shareDocument(String documentId, String ownerId, String targetUserId) {
        Document document = getOwnedDocument(documentId, ownerId);
        document.getSharedWithUserIds().add(targetUserId);
        documentRepository.save(document);

        log.info("Document {} shared with user {}", documentId, targetUserId);
    }

    public Page<Document> searchDocuments(DocumentSearchRequest search, Pageable pageable) {
        return documentRepository.search(
            search.getQuery(),
            search.getType(),
            search.getTags(),
            search.getOwnerId(),
            pageable
        );
    }

    public List<Document> getDocumentVersionHistory(String documentId) {
        Document current = documentRepository.findById(documentId).orElseThrow();
        String rootId = current.getParentDocumentId() != null ?
            current.getParentDocumentId() : current.getId();

        return documentRepository.findVersionHistory(rootId);
    }

    private Document getOwnedDocument(String documentId, String userId) {
        Document doc = documentRepository.findById(documentId)
            .orElseThrow(() -> new DocumentNotFoundException(documentId));

        if (!doc.getOwnerId().equals(userId)) {
            throw new AccessDeniedException("You don't own this document");
        }

        return doc;
    }

    private boolean isImage(String contentType) {
        return contentType != null && contentType.startsWith("image/");
    }
}

// Document controller
@RestController
@RequestMapping("/api/documents")
@Slf4j
public class DocumentController {

    private final DocumentManagementService documentService;

    @PostMapping
    public ResponseEntity<DocumentUploadResult> uploadDocument(
            @RequestParam("file") MultipartFile file,
            @RequestParam String title,
            @RequestParam(required = false) String description,
            @RequestParam DocumentType type,
            @RequestParam(required = false) List<String> tags,
            Authentication auth) {

        UploadDocumentRequest request = UploadDocumentRequest.builder()
            .title(title).description(description)
            .type(type).tags(tags != null ? tags : List.of())
            .build();

        DocumentUploadResult result = documentService.uploadDocument(
            file, request, auth.getName());

        return ResponseEntity.status(HttpStatus.CREATED).body(result);
    }

    @GetMapping("/{id}/download")
    public ResponseEntity<Resource> downloadDocument(@PathVariable String id,
            Authentication auth) {
        return storageService.loadAsResponse(id, auth.getName());
    }

    @PostMapping("/{id}/versions")
    public ResponseEntity<Document> uploadNewVersion(
            @PathVariable String id,
            @RequestParam("file") MultipartFile file,
            Authentication auth) {
        Document updated = documentService.uploadNewVersion(id, file, auth.getName());
        return ResponseEntity.ok(updated);
    }

    @GetMapping("/{id}/versions")
    public ResponseEntity<List<Document>> getVersionHistory(@PathVariable String id) {
        return ResponseEntity.ok(documentService.getDocumentVersionHistory(id));
    }

    @PostMapping("/{id}/share/{targetUserId}")
    public ResponseEntity<Void> share(@PathVariable String id,
            @PathVariable String targetUserId, Authentication auth) {
        documentService.shareDocument(id, auth.getName(), targetUserId);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/export/csv")
    @PreAuthorize("hasRole('ADMIN')")
    public void exportCsv(HttpServletResponse response, Authentication auth)
            throws Exception {
        List<OrderExportRow> data = documentService.getAllDocumentsForExport();
        byte[] csv = csvExportService.exportOrders(data);

        response.setContentType("text/csv");
        response.setHeader("Content-Disposition", "attachment; filename=\"documents.csv\"");
        response.getOutputStream().write(csv);
    }

    @GetMapping("/export/excel")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<byte[]> exportExcel() throws IOException {
        List<OrderReportRow> data = documentService.getDocumentReportRows();
        byte[] excel = excelService.generateOrderReport(data);

        return ResponseEntity.ok()
            .header("Content-Disposition", "attachment; filename=\"documents.xlsx\"")
            .contentType(MediaType.parseMediaType(
                "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"))
            .body(excel);
    }
}
```

---

## LocalStack Integration Testing

```java
// Integration test with LocalStack (fake AWS)
@SpringBootTest
@Testcontainers
class S3StorageIntegrationTest {

    @Container
    static LocalStackContainer localStack = new LocalStackContainer(
        DockerImageName.parse("localstack/localstack:3.0"))
        .withServices(S3);

    @DynamicPropertySource
    static void configureS3(DynamicPropertyRegistry registry) {
        registry.add("aws.s3.endpoint", () -> localStack.getEndpointOverride(S3).toString());
        registry.add("aws.s3.bucket", () -> "test-bucket");
        registry.add("aws.region", () -> localStack.getRegion());
    }

    @Autowired
    private S3Client s3Client;

    @Autowired
    private S3FileStorageService storageService;

    @BeforeEach
    void createBucket() {
        s3Client.createBucket(CreateBucketRequest.builder()
            .bucket("test-bucket").build());
    }

    @Test
    void storeAndRetrieveFile() throws Exception {
        MockMultipartFile file = new MockMultipartFile(
            "test.txt", "test.txt", "text/plain", "Hello World".getBytes());

        FileMetadata stored = storageService.store(file, "user123", "document");

        assertThat(stored.getId()).isNotNull();
        assertThat(stored.getStoragePath()).contains("uploads/user123");

        // Verify we can read it back
        Resource loaded = storageService.load(stored.getId());
        assertThat(loaded.isReadable()).isTrue();

        String content = new String(loaded.getInputStream().readAllBytes());
        assertThat(content).isEqualTo("Hello World");
    }

    @Test
    void generatePresignedUrl() {
        // ... test presigned URL generation
    }
}
```

---

## Summary

| Feature | Library/API | Use Case |
|---------|------------|---------|
| Multipart upload | Spring MVC | Simple file upload |
| Chunked upload | Custom implementation | Large files, resumable |
| File streaming | Spring MVC Resource | Large file download |
| File validation | Apache Tika | Content type detection |
| Local storage | Java NIO | Development, small scale |
| S3 storage | AWS SDK v2 | Production, scalable |
| Pre-signed URLs | S3Presigner | Direct browser uploads |
| Image processing | Thumbnailator | Resizing, thumbnails |
| PDF generation | iText 7 | Invoices, reports |
| Excel generation | Apache POI | Data exports |
| CSV export | OpenCSV | Tabular data |
| Local S3 testing | LocalStack + Testcontainers | Integration tests |

## Course Completion

Congratulations! You have completed Parts 059–064 covering advanced microservices patterns, resilience, security, API design, scheduling, and file handling. These patterns form the foundation of production-grade Spring Boot applications.

**Recommended next topics:**
- Observability (distributed tracing with Jaeger/Zipkin)
- GraphQL with Spring Boot
- gRPC services
- Event sourcing with Axon Framework
- Production deployment strategies (Blue/Green, Canary)
