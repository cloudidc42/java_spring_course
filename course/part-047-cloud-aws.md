# Part 047: Deploying Spring Boot to AWS

## Table of Contents
1. [AWS Overview for Java Developers](#aws-overview)
2. [Spring Cloud AWS Setup](#spring-cloud-aws-setup)
3. [S3 Integration](#s3-integration)
4. [SQS with Spring Cloud AWS](#sqs-integration)
5. [SNS Publishing](#sns-publishing)
6. [RDS Configuration and Connection Pooling](#rds-configuration)
7. [ElastiCache (Redis) as Spring Cache](#elasticache-redis)
8. [AWS Secrets Manager](#aws-secrets-manager)
9. [EC2 Deployment](#ec2-deployment)
10. [ECS Deployment](#ecs-deployment)
11. [Real Example: File Upload Service](#real-example)
12. [Summary](#summary)

---

## 1. AWS Overview for Java Developers {#aws-overview}

AWS provides a comprehensive set of managed services that pair naturally with Spring Boot applications.

| Service | Purpose | Spring Integration |
|---------|---------|-------------------|
| **EC2** | Virtual machines | Manual deployment |
| **ECS** | Container orchestration (Docker) | Spring Boot Docker |
| **EKS** | Managed Kubernetes | Spring on K8s |
| **Lambda** | Serverless functions | Spring Cloud Function |
| **RDS** | Managed relational databases | Spring Data JPA |
| **ElastiCache** | Managed Redis/Memcached | Spring Cache |
| **S3** | Object storage | Spring Cloud AWS |
| **SQS** | Message queuing | Spring Cloud AWS |
| **SNS** | Pub/Sub notifications | Spring Cloud AWS |
| **Secrets Manager** | Credential management | Spring Cloud AWS |
| **Parameter Store** | Configuration management | Spring Cloud AWS |

### Architecture Patterns on AWS

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS Region                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    VPC (Virtual Private Cloud)           │    │
│  │  ┌──────────────┐     ┌──────────────────────────────┐  │    │
│  │  │ Public Subnet │     │      Private Subnet          │  │    │
│  │  │  ┌─────────┐ │     │  ┌──────────┐  ┌─────────┐  │  │    │
│  │  │  │   ALB   │ │ ──► │  │   ECS    │  │   RDS   │  │  │    │
│  │  │  │  (Load  │ │     │  │ (Spring) │  │(PostgreS│  │  │    │
│  │  │  │Balancer)│ │     │  └────┬─────┘  │   QL)   │  │  │    │
│  │  │  └─────────┘ │     │       │        └─────────┘  │  │    │
│  │  └──────────────┘     │  ┌────▼─────┐               │  │    │
│  │                        │  │ElastiCache│               │  │    │
│  │                        │  │  (Redis) │               │  │    │
│  │                        │  └──────────┘               │  │    │
│  │                        └──────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                          ┌─────┐  ┌─────┐                       │
│                          │ S3  │  │ SQS │                        │
│                          └─────┘  └─────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Spring Cloud AWS Setup {#spring-cloud-aws-setup}

### Maven Dependencies

```xml
<!-- pom.xml -->
<properties>
    <java.version>21</java.version>
    <spring-cloud-aws.version>3.1.1</spring-cloud-aws.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.awspring.cloud</groupId>
            <artifactId>spring-cloud-aws-dependencies</artifactId>
            <version>${spring-cloud-aws.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Core AWS -->
    <dependency>
        <groupId>io.awspring.cloud</groupId>
        <artifactId>spring-cloud-aws-starter</artifactId>
    </dependency>

    <!-- S3 -->
    <dependency>
        <groupId>io.awspring.cloud</groupId>
        <artifactId>spring-cloud-aws-starter-s3</artifactId>
    </dependency>

    <!-- SQS -->
    <dependency>
        <groupId>io.awspring.cloud</groupId>
        <artifactId>spring-cloud-aws-starter-sqs</artifactId>
    </dependency>

    <!-- SNS -->
    <dependency>
        <groupId>io.awspring.cloud</groupId>
        <artifactId>spring-cloud-aws-starter-sns</artifactId>
    </dependency>

    <!-- Parameter Store and Secrets Manager -->
    <dependency>
        <groupId>io.awspring.cloud</groupId>
        <artifactId>spring-cloud-aws-starter-parameter-store</artifactId>
    </dependency>
    <dependency>
        <groupId>io.awspring.cloud</groupId>
        <artifactId>spring-cloud-aws-starter-secrets-manager</artifactId>
    </dependency>

    <!-- RDS -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
    </dependency>

    <!-- Redis / ElastiCache -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
</dependencies>
```

### Application Configuration

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: file-upload-service

  # Cloud-specific config loaded from Parameter Store or Secrets Manager
  config:
    import:
      - "optional:aws-secretsmanager:/myapp/prod/"
      - "optional:aws-parameterstore:/myapp/"

spring:
  cloud:
    aws:
      region:
        static: us-east-1
      credentials:
        # In production, use IAM roles (no explicit credentials needed)
        # For local dev, profile or env vars
        instance-profile: true
      s3:
        enabled: true
      sqs:
        enabled: true
      sns:
        enabled: true

# S3
app:
  s3:
    bucket: my-file-uploads-bucket
    presigned-url-expiry: 3600  # seconds

  sqs:
    upload-notification-queue: file-upload-notifications
    dlq: file-upload-dlq

  sns:
    upload-topic-arn: arn:aws:sns:us-east-1:123456789:file-uploaded

---
# Local development profile
spring:
  config:
    activate:
      on-profile: local

spring:
  cloud:
    aws:
      region:
        static: us-east-1
      credentials:
        access-key: ${AWS_ACCESS_KEY_ID:test}
        secret-key: ${AWS_SECRET_ACCESS_KEY:test}
      # LocalStack endpoint for local testing
      endpoint: http://localhost:4566
```

---

## 3. S3 Integration {#s3-integration}

### S3 Configuration

```java
package com.example.aws.config;

import io.awspring.cloud.s3.S3Template;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;
import software.amazon.awssdk.auth.credentials.DefaultCredentialsProvider;
import software.amazon.awssdk.regions.Region;

@Configuration
public class S3Config {

    @Value("${spring.cloud.aws.region.static}")
    private String region;

    @Bean
    public S3Presigner s3Presigner() {
        return S3Presigner.builder()
                .region(Region.of(region))
                .credentialsProvider(DefaultCredentialsProvider.create())
                .build();
    }
}
```

### S3 Service

```java
package com.example.aws.service;

import io.awspring.cloud.s3.S3Template;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;
import software.amazon.awssdk.services.s3.presigner.model.GetObjectPresignRequest;
import software.amazon.awssdk.services.s3.presigner.model.PutObjectPresignRequest;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.time.Duration;
import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class S3Service {

    private final S3Template s3Template;
    private final S3Client s3Client;
    private final S3Presigner s3Presigner;

    @Value("${app.s3.bucket}")
    private String bucketName;

    @Value("${app.s3.presigned-url-expiry}")
    private long presignedUrlExpiry;

    /**
     * Upload a file to S3
     */
    public String uploadFile(MultipartFile file, String folder) throws IOException {
        String key = generateKey(folder, file.getOriginalFilename());

        log.info("Uploading file to S3: bucket={}, key={}, size={}",
                bucketName, key, file.getSize());

        try (InputStream inputStream = file.getInputStream()) {
            s3Template.upload(bucketName, key, inputStream,
                    io.awspring.cloud.s3.ObjectMetadata.builder()
                            .contentType(file.getContentType())
                            .contentLength(file.getSize())
                            .build());
        }

        log.info("File uploaded successfully: {}", key);
        return key;
    }

    /**
     * Download a file from S3
     */
    public InputStream downloadFile(String key) {
        log.info("Downloading file from S3: bucket={}, key={}", bucketName, key);
        return s3Template.download(bucketName, key).getInputStream();
    }

    /**
     * Delete a file from S3
     */
    public void deleteFile(String key) {
        log.info("Deleting file from S3: bucket={}, key={}", bucketName, key);
        s3Client.deleteObject(DeleteObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build());
    }

    /**
     * Generate a presigned URL for downloading (valid for configured duration)
     */
    public URL generatePresignedDownloadUrl(String key) {
        GetObjectRequest getObjectRequest = GetObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
                .signatureDuration(Duration.ofSeconds(presignedUrlExpiry))
                .getObjectRequest(getObjectRequest)
                .build();

        URL url = s3Presigner.presignGetObject(presignRequest).url();
        log.debug("Generated presigned download URL for key={}: {}", key, url);
        return url;
    }

    /**
     * Generate a presigned URL for uploading directly from client
     */
    public URL generatePresignedUploadUrl(String key, String contentType) {
        PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .contentType(contentType)
                .build();

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(Duration.ofSeconds(presignedUrlExpiry))
                .putObjectRequest(putObjectRequest)
                .build();

        return s3Presigner.presignPutObject(presignRequest).url();
    }

    /**
     * Check if a file exists in S3
     */
    public boolean fileExists(String key) {
        try {
            s3Client.headObject(HeadObjectRequest.builder()
                    .bucket(bucketName)
                    .key(key)
                    .build());
            return true;
        } catch (NoSuchKeyException e) {
            return false;
        }
    }

    /**
     * List all files in a folder
     */
    public List<String> listFiles(String prefix) {
        ListObjectsV2Request request = ListObjectsV2Request.builder()
                .bucket(bucketName)
                .prefix(prefix)
                .build();

        return s3Client.listObjectsV2(request)
                .contents()
                .stream()
                .map(S3Object::key)
                .collect(Collectors.toList());
    }

    /**
     * Copy a file within S3
     */
    public String copyFile(String sourceKey, String destinationFolder) {
        String destinationKey = generateKey(destinationFolder,
                sourceKey.substring(sourceKey.lastIndexOf('/') + 1));

        s3Client.copyObject(CopyObjectRequest.builder()
                .sourceBucket(bucketName)
                .sourceKey(sourceKey)
                .destinationBucket(bucketName)
                .destinationKey(destinationKey)
                .build());

        return destinationKey;
    }

    private String generateKey(String folder, String originalFilename) {
        String extension = "";
        if (originalFilename != null && originalFilename.contains(".")) {
            extension = originalFilename.substring(originalFilename.lastIndexOf("."));
        }
        return folder + "/" + UUID.randomUUID() + extension;
    }
}
```

---

## 4. SQS with Spring Cloud AWS {#sqs-integration}

### SQS Configuration

```java
package com.example.aws.config;

import io.awspring.cloud.sqs.config.SqsMessageListenerContainerFactory;
import io.awspring.cloud.sqs.listener.acknowledgement.handler.AcknowledgementMode;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.services.sqs.SqsAsyncClient;

@Configuration
public class SqsConfig {

    /**
     * Custom SQS listener container factory
     */
    @Bean
    public SqsMessageListenerContainerFactory<Object> defaultSqsListenerContainerFactory(
            SqsAsyncClient sqsAsyncClient) {
        return SqsMessageListenerContainerFactory
                .builder()
                .configure(options -> options
                        .maxConcurrentMessages(10)
                        .maxMessagesPerPoll(10)
                        .acknowledgementMode(AcknowledgementMode.ON_SUCCESS)
                        .pollTimeout(java.time.Duration.ofSeconds(20))
                )
                .sqsAsyncClient(sqsAsyncClient)
                .build();
    }
}
```

### SQS Message Models

```java
package com.example.aws.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.Instant;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class FileUploadNotification {

    @JsonProperty("fileKey")
    private String fileKey;

    @JsonProperty("fileName")
    private String fileName;

    @JsonProperty("fileSize")
    private long fileSize;

    @JsonProperty("contentType")
    private String contentType;

    @JsonProperty("uploadedBy")
    private String uploadedBy;

    @JsonProperty("uploadedAt")
    private Instant uploadedAt;

    @JsonProperty("bucketName")
    private String bucketName;
}
```

### SQS Listener (Consumer)

```java
package com.example.aws.listener;

import com.example.aws.dto.FileUploadNotification;
import com.example.aws.service.FileProcessingService;
import io.awspring.cloud.sqs.annotation.SqsListener;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class FileUploadListener {

    private final FileProcessingService fileProcessingService;

    /**
     * Listen for file upload notifications
     * Automatically deletes the message on success
     */
    @SqsListener(value = "${app.sqs.upload-notification-queue}",
                 factory = "defaultSqsListenerContainerFactory")
    public void handleFileUpload(FileUploadNotification notification,
                                  @Header("MessageId") String messageId) {
        log.info("Received file upload notification: messageId={}, fileKey={}",
                messageId, notification.getFileKey());

        try {
            fileProcessingService.processUploadedFile(notification);
            log.info("Successfully processed file: {}", notification.getFileKey());
        } catch (Exception e) {
            log.error("Failed to process file: {}, error: {}",
                    notification.getFileKey(), e.getMessage(), e);
            // Re-throw to trigger message visibility timeout (retry)
            throw e;
        }
    }

    /**
     * Listen for DLQ messages for monitoring/alerting
     */
    @SqsListener(value = "${app.sqs.dlq}")
    public void handleDeadLetter(String rawMessage,
                                  @Header("MessageId") String messageId) {
        log.error("Dead letter received: messageId={}, body={}", messageId, rawMessage);
        // Send alert, store in DB for manual review, etc.
    }
}
```

### SQS Producer (Sender)

```java
package com.example.aws.service;

import com.example.aws.dto.FileUploadNotification;
import io.awspring.cloud.sqs.operations.SqsTemplate;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final SqsTemplate sqsTemplate;

    @Value("${app.sqs.upload-notification-queue}")
    private String uploadNotificationQueue;

    public void sendFileUploadNotification(FileUploadNotification notification) {
        log.info("Sending file upload notification for key: {}", notification.getFileKey());

        sqsTemplate.send(to -> to
                .queue(uploadNotificationQueue)
                .payload(notification)
                .messageGroupId("file-uploads")  // For FIFO queues
                .messageDeduplicationId(UUID.randomUUID().toString())
        );

        log.info("Notification sent for file: {}", notification.getFileKey());
    }
}
```

---

## 5. SNS Publishing {#sns-publishing}

```java
package com.example.aws.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.awspring.cloud.sns.core.SnsTemplate;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Service;

import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class SnsService {

    private final SnsTemplate snsTemplate;
    private final ObjectMapper objectMapper;

    @Value("${app.sns.upload-topic-arn}")
    private String uploadTopicArn;

    /**
     * Publish a notification to SNS topic
     * All subscribers (SQS, Lambda, HTTP endpoints) will receive this
     */
    public void publishFileUploadEvent(String fileKey, String userId, Map<String, String> metadata) {
        try {
            Map<String, Object> event = Map.of(
                    "eventType", "FILE_UPLOADED",
                    "fileKey", fileKey,
                    "userId", userId,
                    "metadata", metadata
            );

            String messageBody = objectMapper.writeValueAsString(event);

            snsTemplate.send(to -> to
                    .topicArn(uploadTopicArn)
                    .subject("File Upload Event")
                    .message(messageBody)
                    .messageAttributes(Map.of(
                            "eventType", io.awspring.cloud.sns.core.SnsNotification
                                    .of("FILE_UPLOADED").getPayload()
                    ))
            );

            log.info("SNS event published for file: {}", fileKey);

        } catch (JsonProcessingException e) {
            log.error("Failed to serialize SNS message", e);
            throw new RuntimeException("Failed to publish SNS event", e);
        }
    }
}
```

---

## 6. RDS Configuration and Connection Pooling {#rds-configuration}

### HikariCP Configuration for RDS

```java
package com.example.aws.config;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;

@Configuration
public class DatabaseConfig {

    @Value("${spring.datasource.url}")
    private String jdbcUrl;

    @Value("${spring.datasource.username}")
    private String username;

    @Value("${spring.datasource.password}")
    private String password;

    @Bean
    @Primary
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(jdbcUrl);
        config.setUsername(username);
        config.setPassword(password);
        config.setDriverClassName("org.postgresql.Driver");

        // Connection pool settings optimized for RDS
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30_000);      // 30 seconds
        config.setIdleTimeout(600_000);           // 10 minutes
        config.setMaxLifetime(1_800_000);         // 30 minutes
        config.setLeakDetectionThreshold(60_000); // 1 minute

        // Connection validation
        config.setConnectionTestQuery("SELECT 1");
        config.setValidationTimeout(5_000);

        // RDS-specific SSL settings
        config.addDataSourceProperty("ssl", "true");
        config.addDataSourceProperty("sslfactory",
                "org.postgresql.ssl.NonValidatingFactory");

        // Performance tuning
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");

        // Pool name for monitoring
        config.setPoolName("rds-hikari-pool");
        config.setRegisterMbeans(true);

        return new HikariDataSource(config);
    }
}
```

### Application YAML for RDS

```yaml
# application-aws.yml
spring:
  datasource:
    url: jdbc:postgresql://${RDS_HOSTNAME}:${RDS_PORT:5432}/${RDS_DB_NAME}
    username: ${RDS_USERNAME}
    password: ${RDS_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: false
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
        batch_versioned_data: true
    show-sql: false
    open-in-view: false
```

---

## 7. ElastiCache (Redis) as Spring Cache {#elasticache-redis}

### Redis Configuration

```java
package com.example.aws.config;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceClientConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;
import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableCaching
public class RedisConfig {

    @Value("${spring.redis.host}")
    private String redisHost;

    @Value("${spring.redis.port:6379}")
    private int redisPort;

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        // ElastiCache supports TLS in-transit encryption
        LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
                .useSsl()             // Enable for ElastiCache with TLS
                .and()
                .commandTimeout(Duration.ofSeconds(5))
                .shutdownTimeout(Duration.ZERO)
                .build();

        RedisStandaloneConfiguration serverConfig =
                new RedisStandaloneConfiguration(redisHost, redisPort);

        return new LettuceConnectionFactory(serverConfig, clientConfig);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer(objectMapper()));
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer(objectMapper()));
        template.afterPropertiesSet();
        return template;
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        // Default cache configuration
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(30))
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer(objectMapper())))
                .disableCachingNullValues();

        // Per-cache TTL configurations
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        cacheConfigurations.put("files", defaultConfig.entryTtl(Duration.ofHours(1)));
        cacheConfigurations.put("users", defaultConfig.entryTtl(Duration.ofMinutes(15)));
        cacheConfigurations.put("presigned-urls", defaultConfig.entryTtl(Duration.ofMinutes(50)));

        return RedisCacheManager.builder(factory)
                .cacheDefaults(defaultConfig)
                .withInitialCacheConfigurations(cacheConfigurations)
                .build();
    }

    private ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.activateDefaultTyping(
                LaissezFaireSubTypeValidator.instance,
                ObjectMapper.DefaultTyping.NON_FINAL,
                JsonTypeInfo.As.PROPERTY
        );
        return mapper;
    }
}
```

### Using Cache in Services

```java
package com.example.aws.service;

import com.example.aws.entity.FileMetadata;
import com.example.aws.repository.FileMetadataRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class FileMetadataService {

    private final FileMetadataRepository repository;
    private final RedisTemplate<String, Object> redisTemplate;

    @Cacheable(value = "files", key = "#fileKey")
    public FileMetadata getFileMetadata(String fileKey) {
        log.info("Cache miss for fileKey: {}, loading from DB", fileKey);
        return repository.findByFileKey(fileKey)
                .orElseThrow(() -> new RuntimeException("File not found: " + fileKey));
    }

    @CachePut(value = "files", key = "#result.fileKey")
    public FileMetadata saveFileMetadata(FileMetadata metadata) {
        return repository.save(metadata);
    }

    @CacheEvict(value = "files", key = "#fileKey")
    public void deleteFileMetadata(String fileKey) {
        repository.deleteByFileKey(fileKey);
    }

    @CacheEvict(value = "files", allEntries = true)
    public void clearAllFileCache() {
        log.info("Clearing all file cache entries");
    }

    // Manual Redis operations for complex scenarios
    public void incrementDownloadCount(String fileKey) {
        String counterKey = "download:count:" + fileKey;
        redisTemplate.opsForValue().increment(counterKey);
        redisTemplate.expire(counterKey, 7, TimeUnit.DAYS);
    }

    public Long getDownloadCount(String fileKey) {
        String counterKey = "download:count:" + fileKey;
        Object value = redisTemplate.opsForValue().get(counterKey);
        return value != null ? Long.parseLong(value.toString()) : 0L;
    }
}
```

---

## 8. AWS Secrets Manager {#aws-secrets-manager}

### Secrets Manager Configuration

```java
package com.example.aws.config;

import io.awspring.cloud.secretsmanager.SecretsManagerConfigDataLoader;
import org.springframework.context.annotation.Configuration;

// Configuration is done through application.yml
// spring.config.import=aws-secretsmanager:/myapp/prod/

@Configuration
public class SecretsManagerConfig {
    // Spring Cloud AWS automatically loads secrets from AWS Secrets Manager
    // The secrets are mapped to Spring properties
    // e.g., secret named "/myapp/prod/" with JSON:
    // {"spring.datasource.password": "secret123", "app.api.key": "mykey"}
    // will be available as @Value("${spring.datasource.password}")
}
```

### Programmatic Secrets Access

```java
package com.example.aws.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueResponse;

import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class SecretsService {

    private final SecretsManagerClient secretsManagerClient;
    private final ObjectMapper objectMapper;

    /**
     * Retrieve a secret value by name
     */
    public String getSecretString(String secretName) {
        GetSecretValueRequest request = GetSecretValueRequest.builder()
                .secretId(secretName)
                .build();

        GetSecretValueResponse response = secretsManagerClient.getSecretValue(request);
        return response.secretString();
    }

    /**
     * Retrieve a secret as a key-value map
     */
    @SuppressWarnings("unchecked")
    public Map<String, String> getSecretAsMap(String secretName) {
        String secretString = getSecretString(secretName);
        try {
            return objectMapper.readValue(secretString, Map.class);
        } catch (Exception e) {
            log.error("Failed to parse secret as map: {}", secretName, e);
            throw new RuntimeException("Failed to parse secret", e);
        }
    }

    /**
     * Retrieve a specific key from a secret
     */
    public String getSecretValue(String secretName, String key) {
        return getSecretAsMap(secretName).get(key);
    }
}
```

### bootstrap.yml for Secret Loading

```yaml
# src/main/resources/bootstrap.yml
# Needed for early secret loading before app context
spring:
  cloud:
    aws:
      secretsmanager:
        enabled: true
      region:
        static: us-east-1

  config:
    import:
      - "aws-secretsmanager:/myapp/database"
      - "aws-secretsmanager:/myapp/api-keys"
```

---

## 9. EC2 Deployment {#ec2-deployment}

### User Data Script for EC2

```bash
#!/bin/bash
# ec2-user-data.sh
# This runs when the EC2 instance starts for the first time

set -e
LOG_FILE="/var/log/app-setup.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "Starting application setup..."

# Update system packages
yum update -y
log "System updated"

# Install Java 21
yum install -y java-21-amazon-corretto-headless
java -version
log "Java 21 installed"

# Install CloudWatch agent for log shipping
yum install -y amazon-cloudwatch-agent
log "CloudWatch agent installed"

# Create app user and directories
useradd -r -s /bin/false appuser
mkdir -p /opt/app /var/log/app
chown appuser:appuser /opt/app /var/log/app
log "App user and directories created"

# Download application JAR from S3
APP_BUCKET="my-deployments-bucket"
APP_VERSION="${APP_VERSION:-latest}"
JAR_FILE="/opt/app/app.jar"

aws s3 cp "s3://${APP_BUCKET}/releases/${APP_VERSION}/app.jar" "$JAR_FILE"
chown appuser:appuser "$JAR_FILE"
log "Application JAR downloaded"

# Create application config
cat > /opt/app/application-prod.yml <<EOF
spring:
  profiles:
    active: prod
server:
  port: 8080
EOF

# Create systemd service
cat > /etc/systemd/system/app.service <<'EOF'
[Unit]
Description=Spring Boot Application
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/app
ExecStart=/usr/bin/java \
  -Xms512m -Xmx1g \
  -Dspring.profiles.active=prod \
  -Dspring.config.location=/opt/app/ \
  -jar /opt/app/app.jar
SuccessExitStatus=143
TimeoutStopSec=10
Restart=on-failure
RestartSec=5

# Security hardening
NoNewPrivileges=yes
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
EOF

# Configure CloudWatch agent
cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json <<'EOF'
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/app/application.log",
            "log_group_name": "/ec2/myapp",
            "log_stream_name": "{instance_id}/application",
            "timezone": "UTC"
          }
        ]
      }
    }
  },
  "metrics": {
    "metrics_collected": {
      "mem": { "measurement": ["mem_used_percent"] },
      "disk": { "measurement": ["disk_used_percent"], "resources": ["/"] }
    }
  }
}
EOF

# Start services
systemctl daemon-reload
systemctl enable app
systemctl start app
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s

log "Application setup complete"
```

---

## 10. ECS Deployment {#ecs-deployment}

### Dockerfile

```dockerfile
# Dockerfile
FROM amazoncorretto:21-alpine AS builder

WORKDIR /build
COPY pom.xml .
COPY src ./src

# Install Maven and build
RUN apk add --no-cache maven && \
    mvn clean package -DskipTests -q

# Runtime image
FROM amazoncorretto:21-alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY --from=builder /build/target/*.jar app.jar

RUN chown appuser:appgroup app.jar

USER appuser

# JVM tuning for containers
ENV JAVA_OPTS="-Xms256m -Xmx512m \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+UseG1GC \
  -Djava.security.egd=file:/dev/./urandom"

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### ECS Task Definition (JSON)

```json
{
  "family": "my-spring-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "spring-app",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/my-spring-app:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "SPRING_PROFILES_ACTIVE",
          "value": "prod"
        },
        {
          "name": "AWS_REGION",
          "value": "us-east-1"
        }
      ],
      "secrets": [
        {
          "name": "SPRING_DATASOURCE_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:myapp/database:password::"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-spring-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "wget -q --spider http://localhost:8080/actuator/health || exit 1"],
        "interval": 30,
        "timeout": 10,
        "retries": 3,
        "startPeriod": 60
      },
      "essential": true
    }
  ]
}
```

### ECS Deployment Script

```bash
#!/bin/bash
# deploy-ecs.sh

set -e

AWS_REGION="us-east-1"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
ECR_REPOSITORY="my-spring-app"
IMAGE_TAG="${GITHUB_SHA:-$(git rev-parse --short HEAD)}"
IMAGE_URI="${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
ECS_CLUSTER="my-cluster"
ECS_SERVICE="my-spring-app-service"
TASK_FAMILY="my-spring-app"

echo "Building and pushing Docker image..."
aws ecr get-login-password --region "$AWS_REGION" | \
  docker login --username AWS --password-stdin "$ECR_REGISTRY"

docker build -t "$IMAGE_URI" .
docker push "$IMAGE_URI"

# Also tag as latest
docker tag "$IMAGE_URI" "${ECR_REGISTRY}/${ECR_REPOSITORY}:latest"
docker push "${ECR_REGISTRY}/${ECR_REPOSITORY}:latest"

echo "Updating ECS task definition..."
TASK_DEFINITION=$(aws ecs describe-task-definition \
  --task-definition "$TASK_FAMILY" \
  --query 'taskDefinition' \
  --output json)

# Update image in task definition
NEW_TASK_DEF=$(echo "$TASK_DEFINITION" | \
  jq --arg IMAGE "$IMAGE_URI" \
  '.containerDefinitions[0].image = $IMAGE |
   del(.taskDefinitionArn, .revision, .status, .requiresAttributes,
       .placementConstraints, .compatibilities, .registeredAt, .registeredBy)')

NEW_TASK_ARN=$(aws ecs register-task-definition \
  --cli-input-json "$NEW_TASK_DEF" \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text)

echo "Deploying to ECS service..."
aws ecs update-service \
  --cluster "$ECS_CLUSTER" \
  --service "$ECS_SERVICE" \
  --task-definition "$NEW_TASK_ARN" \
  --force-new-deployment

echo "Waiting for deployment to complete..."
aws ecs wait services-stable \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE"

echo "Deployment complete! Task: $NEW_TASK_ARN"
```

---

## 11. Real Example: File Upload Service with S3 + SQS {#real-example}

### Entity

```java
package com.example.aws.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.Instant;

@Entity
@Table(name = "file_metadata",
       indexes = {
           @Index(name = "idx_file_key", columnList = "file_key", unique = true),
           @Index(name = "idx_uploaded_by", columnList = "uploaded_by")
       })
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class FileMetadata {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(name = "file_key", nullable = false, unique = true)
    private String fileKey;

    @Column(name = "file_name", nullable = false)
    private String fileName;

    @Column(name = "content_type")
    private String contentType;

    @Column(name = "file_size")
    private Long fileSize;

    @Column(name = "uploaded_by", nullable = false)
    private String uploadedBy;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    @Builder.Default
    private FileStatus status = FileStatus.UPLOADING;

    @Column(name = "download_count")
    @Builder.Default
    private Long downloadCount = 0L;

    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at")
    private Instant updatedAt;

    public enum FileStatus {
        UPLOADING, AVAILABLE, PROCESSING, FAILED, DELETED
    }
}
```

### Repository

```java
package com.example.aws.repository;

import com.example.aws.entity.FileMetadata;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface FileMetadataRepository extends JpaRepository<FileMetadata, String> {

    Optional<FileMetadata> findByFileKey(String fileKey);

    List<FileMetadata> findByUploadedByOrderByCreatedAtDesc(String uploadedBy);

    @Modifying
    @Query("UPDATE FileMetadata f SET f.status = :status WHERE f.fileKey = :fileKey")
    int updateStatus(String fileKey, FileMetadata.FileStatus status);

    @Modifying
    @Query("UPDATE FileMetadata f SET f.downloadCount = f.downloadCount + 1 WHERE f.fileKey = :fileKey")
    int incrementDownloadCount(String fileKey);

    void deleteByFileKey(String fileKey);
}
```

### Upload Controller

```java
package com.example.aws.controller;

import com.example.aws.dto.FileUploadResponse;
import com.example.aws.dto.PresignedUrlResponse;
import com.example.aws.entity.FileMetadata;
import com.example.aws.service.FileUploadService;
import jakarta.validation.constraints.NotBlank;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.io.InputStreamResource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.InputStream;
import java.util.List;

@Slf4j
@RestController
@RequestMapping("/api/v1/files")
@RequiredArgsConstructor
@Validated
public class FileUploadController {

    private final FileUploadService fileUploadService;

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    @ResponseStatus(HttpStatus.CREATED)
    public FileUploadResponse uploadFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam(defaultValue = "uploads") String folder,
            @AuthenticationPrincipal UserDetails user) {

        log.info("Upload request: file={}, folder={}, user={}",
                file.getOriginalFilename(), folder, user.getUsername());

        return fileUploadService.uploadFile(file, folder, user.getUsername());
    }

    @GetMapping("/{fileKey}/download")
    public ResponseEntity<InputStreamResource> downloadFile(
            @PathVariable @NotBlank String fileKey,
            @AuthenticationPrincipal UserDetails user) {

        FileMetadata metadata = fileUploadService.getFileMetadata(fileKey);
        InputStream stream = fileUploadService.downloadFile(fileKey, user.getUsername());

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION,
                        "attachment; filename=\"" + metadata.getFileName() + "\"")
                .contentType(MediaType.parseMediaType(
                        metadata.getContentType() != null ? metadata.getContentType() : "application/octet-stream"))
                .body(new InputStreamResource(stream));
    }

    @GetMapping("/{fileKey}/presigned-url")
    public PresignedUrlResponse getPresignedUrl(
            @PathVariable @NotBlank String fileKey,
            @AuthenticationPrincipal UserDetails user) {

        return fileUploadService.generatePresignedUrl(fileKey, user.getUsername());
    }

    @GetMapping
    public List<FileMetadata> listFiles(
            @AuthenticationPrincipal UserDetails user) {
        return fileUploadService.listUserFiles(user.getUsername());
    }

    @DeleteMapping("/{fileKey}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteFile(
            @PathVariable @NotBlank String fileKey,
            @AuthenticationPrincipal UserDetails user) {

        fileUploadService.deleteFile(fileKey, user.getUsername());
    }
}
```

### File Upload Service (Orchestrator)

```java
package com.example.aws.service;

import com.example.aws.dto.FileUploadNotification;
import com.example.aws.dto.FileUploadResponse;
import com.example.aws.dto.PresignedUrlResponse;
import com.example.aws.entity.FileMetadata;
import com.example.aws.repository.FileMetadataRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;

import java.io.InputStream;
import java.net.URL;
import java.time.Instant;
import java.util.Arrays;
import java.util.List;
import java.util.Set;

@Slf4j
@Service
@RequiredArgsConstructor
public class FileUploadService {

    private static final Set<String> ALLOWED_CONTENT_TYPES = Set.of(
            "image/jpeg", "image/png", "image/gif", "image/webp",
            "application/pdf", "text/plain",
            "application/zip", "application/x-zip-compressed"
    );
    private static final long MAX_FILE_SIZE = 50 * 1024 * 1024; // 50 MB

    private final S3Service s3Service;
    private final NotificationService notificationService;
    private final SnsService snsService;
    private final FileMetadataRepository repository;
    private final FileMetadataService metadataService;

    @Transactional
    public FileUploadResponse uploadFile(MultipartFile file, String folder, String userId) {
        validateFile(file);

        // 1. Upload to S3
        String fileKey;
        try {
            fileKey = s3Service.uploadFile(file, folder);
        } catch (Exception e) {
            log.error("S3 upload failed for user={}", userId, e);
            throw new RuntimeException("File upload failed", e);
        }

        // 2. Save metadata to DB
        FileMetadata metadata = FileMetadata.builder()
                .fileKey(fileKey)
                .fileName(file.getOriginalFilename())
                .contentType(file.getContentType())
                .fileSize(file.getSize())
                .uploadedBy(userId)
                .status(FileMetadata.FileStatus.AVAILABLE)
                .build();

        FileMetadata saved = metadataService.saveFileMetadata(metadata);

        // 3. Send SQS notification for async processing
        FileUploadNotification notification = FileUploadNotification.builder()
                .fileKey(fileKey)
                .fileName(file.getOriginalFilename())
                .fileSize(file.getSize())
                .contentType(file.getContentType())
                .uploadedBy(userId)
                .uploadedAt(Instant.now())
                .build();

        notificationService.sendFileUploadNotification(notification);

        // 4. Publish SNS event for other systems
        snsService.publishFileUploadEvent(fileKey, userId, java.util.Map.of(
                "fileName", file.getOriginalFilename(),
                "folder", folder
        ));

        return FileUploadResponse.builder()
                .fileId(saved.getId())
                .fileKey(fileKey)
                .fileName(file.getOriginalFilename())
                .fileSize(file.getSize())
                .message("File uploaded successfully")
                .build();
    }

    public InputStream downloadFile(String fileKey, String userId) {
        FileMetadata metadata = metadataService.getFileMetadata(fileKey);
        repository.incrementDownloadCount(fileKey);
        metadataService.incrementDownloadCount(fileKey);
        return s3Service.downloadFile(metadata.getFileKey());
    }

    public PresignedUrlResponse generatePresignedUrl(String fileKey, String userId) {
        FileMetadata metadata = metadataService.getFileMetadata(fileKey);
        URL presignedUrl = s3Service.generatePresignedDownloadUrl(metadata.getFileKey());

        return PresignedUrlResponse.builder()
                .url(presignedUrl.toString())
                .expiresIn(3600)
                .build();
    }

    public FileMetadata getFileMetadata(String fileKey) {
        return metadataService.getFileMetadata(fileKey);
    }

    public List<FileMetadata> listUserFiles(String userId) {
        return repository.findByUploadedByOrderByCreatedAtDesc(userId);
    }

    @Transactional
    public void deleteFile(String fileKey, String userId) {
        FileMetadata metadata = metadataService.getFileMetadata(fileKey);
        s3Service.deleteFile(metadata.getFileKey());
        repository.updateStatus(fileKey, FileMetadata.FileStatus.DELETED);
        metadataService.deleteFileMetadata(fileKey);
    }

    private void validateFile(MultipartFile file) {
        if (file.isEmpty()) {
            throw new IllegalArgumentException("File cannot be empty");
        }
        if (file.getSize() > MAX_FILE_SIZE) {
            throw new IllegalArgumentException("File size exceeds maximum allowed: 50MB");
        }
        if (file.getContentType() != null && !ALLOWED_CONTENT_TYPES.contains(file.getContentType())) {
            throw new IllegalArgumentException("File type not allowed: " + file.getContentType());
        }
    }
}
```

### File Processing Service (SQS Consumer Logic)

```java
package com.example.aws.service;

import com.example.aws.dto.FileUploadNotification;
import com.example.aws.entity.FileMetadata;
import com.example.aws.repository.FileMetadataRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@Service
@RequiredArgsConstructor
public class FileProcessingService {

    private final FileMetadataRepository repository;
    private final S3Service s3Service;

    @Transactional
    public void processUploadedFile(FileUploadNotification notification) {
        log.info("Processing file: {}", notification.getFileKey());

        // Update status to processing
        repository.updateStatus(notification.getFileKey(), FileMetadata.FileStatus.PROCESSING);

        try {
            // Simulate file processing (virus scan, thumbnail generation, etc.)
            performFileProcessing(notification);

            // Update status to available
            repository.updateStatus(notification.getFileKey(), FileMetadata.FileStatus.AVAILABLE);
            log.info("File processing complete: {}", notification.getFileKey());

        } catch (Exception e) {
            log.error("File processing failed: {}", notification.getFileKey(), e);
            repository.updateStatus(notification.getFileKey(), FileMetadata.FileStatus.FAILED);
            throw e;
        }
    }

    private void performFileProcessing(FileUploadNotification notification) {
        // Add your processing logic:
        // - Virus scanning
        // - Image resizing/thumbnail generation
        // - Document conversion
        // - Content indexing
        log.info("Running file processing pipeline for: {}, type: {}",
                notification.getFileKey(), notification.getContentType());
    }
}
```

### DTOs

```java
package com.example.aws.dto;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class FileUploadResponse {
    private String fileId;
    private String fileKey;
    private String fileName;
    private long fileSize;
    private String message;
}
```

```java
package com.example.aws.dto;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class PresignedUrlResponse {
    private String url;
    private long expiresIn;
}
```

---

## 12. Summary {#summary}

| Concept | Key Points |
|---------|-----------|
| **Spring Cloud AWS** | Auto-configures AWS SDK clients; use IAM roles in production |
| **S3** | `S3Template` for simple ops; `S3Client` for advanced; presigned URLs for direct client access |
| **SQS** | `@SqsListener` auto-acknowledges on success; use DLQ for failed messages |
| **SNS** | Fan-out pattern; one publish reaches multiple subscribers (SQS, Lambda, HTTP) |
| **RDS** | HikariCP with tuned pool settings; use SSL; never hardcode credentials |
| **ElastiCache** | Lettuce with TLS; `@Cacheable`/`@CachePut`/`@CacheEvict` per cache name |
| **Secrets Manager** | Load at startup via `spring.config.import`; rotate credentials without restart |
| **EC2** | User data for bootstrapping; systemd for process management; CloudWatch for logs |
| **ECS/Fargate** | Prefer Fargate (serverless containers); use task IAM roles; health checks |
| **Credentials** | Never use access keys in production; always prefer IAM instance/task roles |

### IAM Policy Minimum Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::my-file-uploads-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": ["sqs:SendMessage", "sqs:ReceiveMessage", "sqs:DeleteMessage",
                 "sqs:GetQueueAttributes"],
      "Resource": "arn:aws:sqs:us-east-1:*:file-upload-*"
    },
    {
      "Effect": "Allow",
      "Action": ["sns:Publish"],
      "Resource": "arn:aws:sns:us-east-1:*:file-uploaded"
    },
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:us-east-1:*:secret:myapp/*"
    }
  ]
}
```

---

> **Next: Part 048 - Deploying Spring Boot to Google Cloud** — Cloud Run, Pub/Sub, Cloud Storage, Cloud SQL, and fully managed container deployments on GCP.
