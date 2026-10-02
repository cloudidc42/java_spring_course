# Part 048: Deploying Spring Boot to Google Cloud

## Table of Contents
1. [GCP Overview for Java Developers](#gcp-overview)
2. [Spring Cloud GCP Setup](#spring-cloud-gcp-setup)
3. [Cloud Storage (GCS) Integration](#cloud-storage)
4. [Cloud Pub/Sub with Spring](#pubsub)
5. [Cloud SQL Configuration](#cloud-sql)
6. [Memorystore (Redis) as Cache](#memorystore)
7. [Secret Manager Integration](#secret-manager)
8. [Cloud Run Deployment](#cloud-run)
9. [GKE Deployment](#gke-deployment)
10. [Cloud Build CI/CD](#cloud-build)
11. [Real Example: Event-Driven App on Cloud Run](#real-example)
12. [Summary](#summary)

---

## 1. GCP Overview for Java Developers {#gcp-overview}

| Service | AWS Equivalent | Purpose |
|---------|---------------|---------|
| **Cloud Run** | ECS Fargate / Lambda | Serverless containers |
| **GKE** | EKS | Managed Kubernetes |
| **Cloud SQL** | RDS | Managed PostgreSQL/MySQL |
| **Memorystore** | ElastiCache | Managed Redis |
| **Cloud Storage** | S3 | Object storage |
| **Pub/Sub** | SQS + SNS | Messaging & event streaming |
| **Secret Manager** | Secrets Manager | Credential storage |
| **Cloud Build** | CodeBuild | CI/CD pipelines |
| **Artifact Registry** | ECR | Container image storage |
| **Cloud Run Jobs** | AWS Batch | One-off batch tasks |

### Cloud Run Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        GCP Project                            │
│                                                               │
│  ┌───────────────┐     ┌─────────────────────────────────┐  │
│  │ Cloud Build   │────►│         Cloud Run                │  │
│  │ (CI/CD)       │     │  ┌─────────────────────────┐    │  │
│  └───────────────┘     │  │  Spring Boot Container  │    │  │
│                         │  │  (Auto-scales 0→N)      │    │  │
│  ┌───────────────┐     │  └──────────┬──────────────┘    │  │
│  │Artifact Reg.  │────►│             │                    │  │
│  │(Docker images)│     └─────────────┼────────────────────┘  │
│  └───────────────┘                   │                        │
│                         ┌────────────▼────────────────────┐  │
│  ┌───────────────┐     │  ┌──────────┐  ┌─────────────┐  │  │
│  │Secret Manager │     │  │Cloud SQL │  │ Memorystore │  │  │
│  │(credentials)  │     │  │(Postgres)│  │  (Redis)    │  │  │
│  └───────────────┘     │  └──────────┘  └─────────────┘  │  │
│                         └────────────────────────────────────┘  │
│  ┌───────────────┐     ┌─────────────────────────────────┐  │
│  │  Cloud Storage│     │          Pub/Sub                 │  │
│  │    (GCS)      │     │   Topics & Subscriptions         │  │
│  └───────────────┘     └─────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Spring Cloud GCP Setup {#spring-cloud-gcp-setup}

### Maven Dependencies

```xml
<!-- pom.xml -->
<properties>
    <java.version>21</java.version>
    <spring-cloud-gcp.version>5.3.0</spring-cloud-gcp.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.google.cloud</groupId>
            <artifactId>spring-cloud-gcp-dependencies</artifactId>
            <version>${spring-cloud-gcp.version}</version>
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

    <!-- Core GCP -->
    <dependency>
        <groupId>com.google.cloud</groupId>
        <artifactId>spring-cloud-gcp-starter</artifactId>
    </dependency>

    <!-- Cloud Storage -->
    <dependency>
        <groupId>com.google.cloud</groupId>
        <artifactId>spring-cloud-gcp-starter-storage</artifactId>
    </dependency>

    <!-- Pub/Sub -->
    <dependency>
        <groupId>com.google.cloud</groupId>
        <artifactId>spring-cloud-gcp-starter-pubsub</artifactId>
    </dependency>

    <!-- Cloud SQL (PostgreSQL) -->
    <dependency>
        <groupId>com.google.cloud</groupId>
        <artifactId>spring-cloud-gcp-starter-sql-postgresql</artifactId>
    </dependency>

    <!-- Secret Manager -->
    <dependency>
        <groupId>com.google.cloud</groupId>
        <artifactId>spring-cloud-gcp-starter-secretmanager</artifactId>
    </dependency>

    <!-- JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- Redis / Memorystore -->
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
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

### Application Configuration

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: event-driven-service

  config:
    import:
      - "optional:sm://"  # Load all secrets from Secret Manager

  cloud:
    gcp:
      project-id: my-gcp-project-id
      credentials:
        location: classpath:gcp-credentials.json  # For local dev only
        # In Cloud Run: uses Application Default Credentials automatically

      # Cloud SQL
      sql:
        database-name: my_database
        instance-connection-name: my-gcp-project-id:us-central1:my-postgres-instance
        enabled: true

      # Pub/Sub
      pubsub:
        enabled: true
        subscriber:
          parallel-pull-count: 2
          max-ack-extension-period: 600  # seconds
          executor-threads: 4

  # Secret Manager properties
  # Properties prefixed with sm:// are loaded from Secret Manager
  datasource:
    password: ${sm://my-postgres-password}
    username: ${sm://my-postgres-username}

# App-specific config
app:
  gcs:
    bucket: my-documents-bucket
    presigned-url-expiry: 3600

  pubsub:
    document-uploaded-topic: document-uploaded
    document-processed-subscription: document-process-sub
    notification-topic: user-notifications
```

---

## 3. Cloud Storage (GCS) Integration {#cloud-storage}

### GCS Service

```java
package com.example.gcp.service;

import com.google.cloud.storage.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.gcp.storage.GoogleStorageResource;
import org.springframework.core.io.Resource;
import org.springframework.core.io.ResourceLoader;
import org.springframework.core.io.WritableResource;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.URL;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class GcsService {

    private final Storage storage;
    private final ResourceLoader resourceLoader;

    @Value("${app.gcs.bucket}")
    private String bucketName;

    @Value("${app.gcs.presigned-url-expiry}")
    private long presignedUrlExpiry;

    /**
     * Upload file to GCS using Spring Resource abstraction
     */
    public String uploadFile(MultipartFile file, String folder) throws IOException {
        String key = generateKey(folder, file.getOriginalFilename());
        String gcsUrl = "gs://" + bucketName + "/" + key;

        Resource gcsResource = resourceLoader.getResource(gcsUrl);
        try (OutputStream outputStream = ((WritableResource) gcsResource).getOutputStream();
             InputStream inputStream = file.getInputStream()) {
            inputStream.transferTo(outputStream);
        }

        log.info("Uploaded file to GCS: {}", gcsUrl);
        return key;
    }

    /**
     * Upload file using GCS SDK directly (more control)
     */
    public String uploadFileDirect(MultipartFile file, String folder) throws IOException {
        String key = generateKey(folder, file.getOriginalFilename());

        BlobId blobId = BlobId.of(bucketName, key);
        BlobInfo blobInfo = BlobInfo.newBuilder(blobId)
                .setContentType(file.getContentType())
                .build();

        try (InputStream inputStream = file.getInputStream()) {
            storage.createFrom(blobInfo, inputStream);
        }

        log.info("Uploaded file to GCS: gs://{}/{}", bucketName, key);
        return key;
    }

    /**
     * Download file from GCS
     */
    public InputStream downloadFile(String key) {
        BlobId blobId = BlobId.of(bucketName, key);
        Blob blob = storage.get(blobId);

        if (blob == null) {
            throw new RuntimeException("File not found in GCS: " + key);
        }

        return new java.io.ByteArrayInputStream(blob.getContent());
    }

    /**
     * Delete a file from GCS
     */
    public boolean deleteFile(String key) {
        BlobId blobId = BlobId.of(bucketName, key);
        boolean deleted = storage.delete(blobId);
        log.info("Delete GCS file: key={}, success={}", key, deleted);
        return deleted;
    }

    /**
     * Generate a signed URL for download (valid for configured duration)
     */
    public URL generateSignedDownloadUrl(String key) {
        BlobInfo blobInfo = BlobInfo.newBuilder(BlobId.of(bucketName, key)).build();

        URL signedUrl = storage.signUrl(
                blobInfo,
                presignedUrlExpiry,
                TimeUnit.SECONDS,
                Storage.SignUrlOption.withV4Signature()
        );

        log.debug("Generated signed URL for key: {}", key);
        return signedUrl;
    }

    /**
     * Generate a signed URL for upload (client-direct upload)
     */
    public URL generateSignedUploadUrl(String key, String contentType) {
        BlobInfo blobInfo = BlobInfo.newBuilder(BlobId.of(bucketName, key))
                .setContentType(contentType)
                .build();

        return storage.signUrl(
                blobInfo,
                presignedUrlExpiry,
                TimeUnit.SECONDS,
                Storage.SignUrlOption.httpMethod(HttpMethod.PUT),
                Storage.SignUrlOption.withContentType(),
                Storage.SignUrlOption.withV4Signature()
        );
    }

    /**
     * Check if object exists
     */
    public boolean exists(String key) {
        BlobId blobId = BlobId.of(bucketName, key);
        Blob blob = storage.get(blobId);
        return blob != null && blob.exists();
    }

    /**
     * List all objects with a given prefix
     */
    public List<String> listObjects(String prefix) {
        Page<Blob> blobs = storage.list(bucketName,
                Storage.BlobListOption.prefix(prefix));

        List<String> keys = new ArrayList<>();
        for (Blob blob : blobs.iterateAll()) {
            keys.add(blob.getName());
        }
        return keys;
    }

    /**
     * Copy object within GCS
     */
    public String copyObject(String sourceKey, String destinationFolder) {
        String destinationKey = generateKey(destinationFolder,
                sourceKey.substring(sourceKey.lastIndexOf('/') + 1));

        storage.copy(Storage.CopyRequest.newBuilder()
                .setSource(BlobId.of(bucketName, sourceKey))
                .setTarget(BlobId.of(bucketName, destinationKey))
                .build());

        return destinationKey;
    }

    /**
     * Move object (copy + delete)
     */
    public String moveObject(String sourceKey, String destinationFolder) {
        String destinationKey = copyObject(sourceKey, destinationFolder);
        deleteFile(sourceKey);
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

## 4. Cloud Pub/Sub with Spring {#pubsub}

### Pub/Sub Configuration

```java
package com.example.gcp.config;

import com.google.cloud.spring.pubsub.core.PubSubTemplate;
import com.google.cloud.spring.pubsub.integration.AckMode;
import com.google.cloud.spring.pubsub.integration.inbound.PubSubInboundChannelAdapter;
import com.google.cloud.spring.pubsub.integration.outbound.PubSubMessageHandler;
import com.google.cloud.spring.pubsub.support.BasicAcknowledgeablePubsubMessage;
import com.google.cloud.spring.pubsub.support.GcpPubSubHeaders;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.channel.PublishSubscribeChannel;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHandler;

@Slf4j
@Configuration
public class PubSubConfig {

    @Value("${app.pubsub.document-processed-subscription}")
    private String documentProcessedSubscription;

    /**
     * Inbound channel for messages from Pub/Sub subscription
     */
    @Bean
    public MessageChannel documentProcessingInputChannel() {
        return new PublishSubscribeChannel();
    }

    /**
     * Adapter that bridges Pub/Sub subscription to Spring channel
     */
    @Bean
    public PubSubInboundChannelAdapter documentProcessingAdapter(
            @Qualifier("documentProcessingInputChannel") MessageChannel inputChannel,
            PubSubTemplate pubSubTemplate) {

        PubSubInboundChannelAdapter adapter = new PubSubInboundChannelAdapter(
                pubSubTemplate, documentProcessedSubscription);
        adapter.setOutputChannel(inputChannel);
        adapter.setAckMode(AckMode.MANUAL);  // We'll ack manually after processing
        adapter.setPayloadType(String.class);
        return adapter;
    }
}
```

### Pub/Sub Publisher

```java
package com.example.gcp.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.google.cloud.spring.pubsub.core.PubSubTemplate;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.CompletableFuture;

@Slf4j
@Service
@RequiredArgsConstructor
public class PubSubPublisher {

    private final PubSubTemplate pubSubTemplate;
    private final ObjectMapper objectMapper;

    @Value("${app.pubsub.document-uploaded-topic}")
    private String documentUploadedTopic;

    @Value("${app.pubsub.notification-topic}")
    private String notificationTopic;

    /**
     * Publish a document upload event
     */
    public CompletableFuture<String> publishDocumentUploaded(
            String documentId, String gcsPath, String userId) {

        Map<String, Object> event = new HashMap<>();
        event.put("eventType", "DOCUMENT_UPLOADED");
        event.put("documentId", documentId);
        event.put("gcsPath", gcsPath);
        event.put("userId", userId);
        event.put("timestamp", java.time.Instant.now().toString());

        Map<String, String> attributes = new HashMap<>();
        attributes.put("eventType", "DOCUMENT_UPLOADED");
        attributes.put("userId", userId);

        try {
            String payload = objectMapper.writeValueAsString(event);
            log.info("Publishing document upload event: topic={}, documentId={}",
                    documentUploadedTopic, documentId);

            return pubSubTemplate.publish(documentUploadedTopic, payload, attributes)
                    .thenApply(messageId -> {
                        log.info("Published event: messageId={}", messageId);
                        return messageId;
                    });

        } catch (JsonProcessingException e) {
            log.error("Failed to serialize Pub/Sub message", e);
            return CompletableFuture.failedFuture(e);
        }
    }

    /**
     * Publish a user notification
     */
    public CompletableFuture<String> publishNotification(
            String userId, String message, String type) {

        Map<String, Object> notification = Map.of(
                "userId", userId,
                "message", message,
                "type", type,
                "timestamp", java.time.Instant.now().toString()
        );

        try {
            String payload = objectMapper.writeValueAsString(notification);
            return pubSubTemplate.publish(notificationTopic, payload,
                    Map.of("notificationType", type));
        } catch (JsonProcessingException e) {
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

### Pub/Sub Subscriber (Listener)

```java
package com.example.gcp.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.google.cloud.spring.pubsub.support.BasicAcknowledgeablePubsubMessage;
import com.google.cloud.spring.pubsub.support.GcpPubSubHeaders;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.messaging.Message;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Service;

import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class DocumentProcessingListener {

    private final DocumentService documentService;
    private final ObjectMapper objectMapper;

    /**
     * Listen to messages on the documentProcessingInputChannel
     * Message comes from Pub/Sub via Spring Integration adapter
     */
    @ServiceActivator(inputChannel = "documentProcessingInputChannel")
    public void handleDocumentUpload(Message<String> message) {
        BasicAcknowledgeablePubsubMessage originalMessage =
                message.getHeaders().get(GcpPubSubHeaders.ORIGINAL_MESSAGE,
                        BasicAcknowledgeablePubsubMessage.class);

        String messageId = message.getHeaders().getId() != null
                ? message.getHeaders().getId().toString() : "unknown";

        log.info("Received Pub/Sub message: messageId={}", messageId);

        try {
            @SuppressWarnings("unchecked")
            Map<String, Object> event = objectMapper.readValue(
                    message.getPayload(), Map.class);

            String documentId = (String) event.get("documentId");
            String gcsPath = (String) event.get("gcsPath");
            String userId = (String) event.get("userId");

            log.info("Processing document: documentId={}, userId={}", documentId, userId);

            documentService.processDocument(documentId, gcsPath, userId);

            // Acknowledge message on successful processing
            if (originalMessage != null) {
                originalMessage.ack();
                log.info("Message acknowledged: documentId={}", documentId);
            }

        } catch (Exception e) {
            log.error("Failed to process Pub/Sub message: {}", e.getMessage(), e);

            // Nack the message - it will be redelivered based on subscription retry policy
            if (originalMessage != null) {
                originalMessage.nack();
            }
        }
    }
}
```

### Direct Pull Subscription (Alternative Pattern)

```java
package com.example.gcp.service;

import com.google.cloud.spring.pubsub.core.PubSubTemplate;
import com.google.cloud.spring.pubsub.support.AcknowledgeablePubsubMessage;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class PubSubPullService {

    private final PubSubTemplate pubSubTemplate;

    /**
     * Alternative: pull messages on a schedule
     * Useful when you want more control over consumption rate
     */
    @Scheduled(fixedDelay = 5000)
    public void pullMessages() {
        List<AcknowledgeablePubsubMessage> messages =
                pubSubTemplate.pull("document-process-sub", 10, false);

        if (!messages.isEmpty()) {
            log.info("Pulled {} messages from Pub/Sub", messages.size());

            for (AcknowledgeablePubsubMessage message : messages) {
                try {
                    String payload = message.getPubsubMessage().getData().toStringUtf8();
                    log.info("Processing pulled message: {}", payload);

                    // Process message...
                    processMessage(payload);

                    message.ack();
                } catch (Exception e) {
                    log.error("Failed to process message, nacking", e);
                    message.nack();
                }
            }
        }
    }

    private void processMessage(String payload) {
        // Processing logic here
    }
}
```

---

## 5. Cloud SQL Configuration {#cloud-sql}

### Cloud SQL with Spring Cloud GCP

```yaml
# application-gcp.yml
spring:
  cloud:
    gcp:
      sql:
        database-name: my_database
        instance-connection-name: my-project:us-central1:my-postgres
        enabled: true
        # Cloud SQL Auth Proxy handles connection automatically
        # No need to specify host/port

  datasource:
    username: ${sm://postgres-username}
    password: ${sm://postgres-password}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
    show-sql: false
    open-in-view: false
```

### Manual Cloud SQL Connection (without Spring Cloud GCP)

```java
package com.example.gcp.config;

import com.google.cloud.sql.core.CoreSocketFactory;
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

import javax.sql.DataSource;

@Configuration
@Profile("manual-cloudsql")
public class CloudSqlManualConfig {

    @Value("${app.cloudsql.instance-connection-name}")
    private String instanceConnectionName;

    @Value("${app.cloudsql.database-name}")
    private String databaseName;

    @Value("${app.cloudsql.username}")
    private String username;

    @Value("${app.cloudsql.password}")
    private String password;

    @Bean
    public DataSource cloudSqlDataSource() {
        HikariConfig config = new HikariConfig();

        // Cloud SQL Auth Proxy socket factory
        config.addDataSourceProperty("socketFactory",
                "com.google.cloud.sql.postgres.SocketFactory");
        config.addDataSourceProperty("cloudSqlInstance", instanceConnectionName);

        config.setJdbcUrl(String.format(
                "jdbc:postgresql:///%s?cloudSqlInstance=%s&socketFactory=com.google.cloud.sql.postgres.SocketFactory",
                databaseName, instanceConnectionName));
        config.setUsername(username);
        config.setPassword(password);
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(2);
        config.setConnectionTimeout(30_000);
        config.setIdleTimeout(600_000);
        config.setMaxLifetime(1_800_000);
        config.setPoolName("cloud-sql-pool");

        return new HikariDataSource(config);
    }
}
```

---

## 6. Memorystore (Redis) as Cache {#memorystore}

```yaml
# application-gcp.yml additions
spring:
  redis:
    host: ${REDIS_HOST:10.0.0.3}  # Memorystore private IP
    port: ${REDIS_PORT:6379}
    # Memorystore supports AUTH (password)
    password: ${sm://redis-auth-string}
    # TLS is not supported on basic tier Memorystore
    ssl: false
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 5000ms
      shutdown-timeout: 200ms
```

```java
package com.example.gcp.config;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;
import java.util.Map;

@Configuration
@EnableCaching
public class MemorystoreConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModule(new JavaTimeModule());
        objectMapper.activateDefaultTyping(
                LaissezFaireSubTypeValidator.instance,
                ObjectMapper.DefaultTyping.NON_FINAL,
                JsonTypeInfo.As.PROPERTY
        );

        GenericJackson2JsonRedisSerializer serializer =
                new GenericJackson2JsonRedisSerializer(objectMapper);

        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(30))
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(serializer))
                .disableCachingNullValues();

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(defaultConfig)
                .withInitialCacheConfigurations(Map.of(
                        "documents", defaultConfig.entryTtl(Duration.ofHours(2)),
                        "users", defaultConfig.entryTtl(Duration.ofMinutes(15)),
                        "signed-urls", defaultConfig.entryTtl(Duration.ofMinutes(50))
                ))
                .build();
    }
}
```

---

## 7. Secret Manager Integration {#secret-manager}

### Secret Manager Configuration

```yaml
# bootstrap.yml
spring:
  cloud:
    gcp:
      project-id: my-gcp-project-id
      secretmanager:
        enabled: true
        # All sm:// references are resolved at startup
```

### Programmatic Access to Secret Manager

```java
package com.example.gcp.service;

import com.google.cloud.secretmanager.v1.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class SecretManagerService {

    private final SecretManagerServiceClient secretManagerClient;

    @Value("${spring.cloud.gcp.project-id}")
    private String projectId;

    /**
     * Get secret value by name
     */
    public String getSecret(String secretName) {
        SecretVersionName secretVersionName = SecretVersionName.of(
                projectId, secretName, "latest");

        AccessSecretVersionResponse response =
                secretManagerClient.accessSecretVersion(secretVersionName);

        return response.getPayload().getData().toStringUtf8();
    }

    /**
     * Get a specific version of a secret
     */
    public String getSecret(String secretName, String version) {
        SecretVersionName secretVersionName = SecretVersionName.of(
                projectId, secretName, version);

        return secretManagerClient.accessSecretVersion(secretVersionName)
                .getPayload().getData().toStringUtf8();
    }

    /**
     * Create or update a secret
     */
    public void setSecret(String secretName, String value) {
        ProjectName projectName = ProjectName.of(projectId);
        SecretName name = SecretName.of(projectId, secretName);

        // Try to get existing secret, create if not exists
        try {
            secretManagerClient.getSecret(name);
        } catch (Exception e) {
            log.info("Creating new secret: {}", secretName);
            secretManagerClient.createSecret(projectName, secretName,
                    Secret.newBuilder()
                            .setReplication(Replication.newBuilder()
                                    .setAutomatic(Replication.Automatic.getDefaultInstance())
                                    .build())
                            .build());
        }

        // Add a new version
        SecretPayload payload = SecretPayload.newBuilder()
                .setData(com.google.protobuf.ByteString.copyFromUtf8(value))
                .build();

        secretManagerClient.addSecretVersion(name, payload);
        log.info("Secret updated: {}", secretName);
    }
}
```

### Using Secrets via @Value

```java
package com.example.gcp.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    // sm:// prefix fetches from GCP Secret Manager
    @Value("${sm://my-api-key}")
    private String apiKey;

    @Value("${sm://my-db-password}")
    private String dbPassword;

    // With specific version
    @Value("${sm://my-api-key/versions/1}")
    private String oldApiKey;
}
```

---

## 8. Cloud Run Deployment {#cloud-run}

### Dockerfile for Cloud Run

```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /workspace/app

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
RUN ./mvnw dependency:go-offline -q

COPY src src
RUN ./mvnw package -DskipTests -q

# Extract layers for faster startup
RUN java -Djarmode=layertools -jar target/*.jar extract

# Runtime image
FROM eclipse-temurin:21-jre-alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app

# Copy layers for faster restarts
COPY --from=builder /workspace/app/dependencies/ ./
COPY --from=builder /workspace/app/spring-boot-loader/ ./
COPY --from=builder /workspace/app/snapshot-dependencies/ ./
COPY --from=builder /workspace/app/application/ ./

RUN chown -R appuser:appgroup /app
USER appuser

# Cloud Run requires PORT env var
ENV PORT=8080
EXPOSE ${PORT}

ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 \
  -XX:+ExitOnOutOfMemoryError -Djava.security.egd=file:/dev/./urandom"

ENTRYPOINT ["sh", "-c", "java ${JAVA_OPTS} -server org.springframework.boot.loader.JarLauncher"]
```

### Cloud Run Service YAML

```yaml
# cloud-run-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: event-driven-service
  namespace: my-gcp-project-id
  labels:
    cloud.googleapis.com/location: us-central1
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/execution-environment: gen2
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "20"
        run.googleapis.com/cpu-throttling: "false"  # Always-on CPU
        run.googleapis.com/startup-cpu-boost: "true"
    spec:
      serviceAccountName: my-service-account@my-project.iam.gserviceaccount.com
      containers:
        - image: us-central1-docker.pkg.dev/my-project/my-repo/event-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "gcp"
            - name: GOOGLE_CLOUD_PROJECT
              value: "my-gcp-project-id"
          resources:
            limits:
              cpu: "2"
              memory: "1Gi"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
      timeoutSeconds: 300
      containerConcurrency: 80
  traffic:
    - percent: 100
      latestRevision: true
```

### Cloud Run Deployment Script

```bash
#!/bin/bash
# deploy-cloudrun.sh

set -e

PROJECT_ID="my-gcp-project-id"
REGION="us-central1"
SERVICE_NAME="event-driven-service"
IMAGE_NAME="event-service"
REGISTRY="us-central1-docker.pkg.dev/${PROJECT_ID}/my-repo/${IMAGE_NAME}"
TAG="${GITHUB_SHA:-$(git rev-parse --short HEAD)}"
IMAGE_TAG="${REGISTRY}:${TAG}"

echo "Authenticating to Google Cloud..."
gcloud auth configure-docker us-central1-docker.pkg.dev --quiet

echo "Building Docker image..."
docker build --platform linux/amd64 -t "$IMAGE_TAG" .
docker push "$IMAGE_TAG"

# Also tag as latest
docker tag "$IMAGE_TAG" "${REGISTRY}:latest"
docker push "${REGISTRY}:latest"

echo "Deploying to Cloud Run..."
gcloud run deploy "$SERVICE_NAME" \
  --image "$IMAGE_TAG" \
  --region "$REGION" \
  --platform managed \
  --service-account "cloud-run-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --set-env-vars "SPRING_PROFILES_ACTIVE=gcp,GOOGLE_CLOUD_PROJECT=${PROJECT_ID}" \
  --memory "1Gi" \
  --cpu "2" \
  --min-instances 1 \
  --max-instances 20 \
  --concurrency 80 \
  --timeout 300 \
  --allow-unauthenticated \
  --quiet

echo "Getting service URL..."
SERVICE_URL=$(gcloud run services describe "$SERVICE_NAME" \
  --region "$REGION" \
  --format "value(status.url)")

echo "Deployment complete! Service URL: $SERVICE_URL"

# Test health check
echo "Checking health..."
curl -sf "${SERVICE_URL}/actuator/health" && echo "Health check passed"
```

---

## 9. GKE Deployment {#gke-deployment}

### Kubernetes Manifests

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-driven-service
  namespace: production
  labels:
    app: event-driven-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: event-driven-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: event-driven-service
      annotations:
        # Workload Identity - uses GCP service account for pod
        iam.gke.io/gcp-service-account: "my-sa@my-project.iam.gserviceaccount.com"
    spec:
      serviceAccountName: ksa-event-service  # K8s SA linked to GCP SA
      containers:
        - name: event-driven-service
          image: us-central1-docker.pkg.dev/my-project/my-repo/event-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "gcp"
            - name: GOOGLE_CLOUD_PROJECT
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 30
            failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  name: event-driven-service
  namespace: production
spec:
  selector:
    app: event-driven-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: event-driven-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: event-driven-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## 10. Cloud Build CI/CD {#cloud-build}

### cloudbuild.yaml

```yaml
# cloudbuild.yaml
steps:
  # Step 1: Run tests
  - name: "maven:3.9-eclipse-temurin-21"
    id: "test"
    entrypoint: "mvn"
    args:
      - "test"
      - "-Dspring.profiles.active=test"
    env:
      - "MAVEN_OPTS=-Xmx1g"

  # Step 2: Build JAR
  - name: "maven:3.9-eclipse-temurin-21"
    id: "build"
    entrypoint: "mvn"
    args:
      - "package"
      - "-DskipTests"
      - "-q"
    waitFor: ["test"]

  # Step 3: Build Docker image
  - name: "gcr.io/cloud-builders/docker"
    id: "docker-build"
    args:
      - "build"
      - "--platform"
      - "linux/amd64"
      - "-t"
      - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/event-service:$COMMIT_SHA"
      - "-t"
      - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/event-service:latest"
      - "."
    waitFor: ["build"]

  # Step 4: Push to Artifact Registry
  - name: "gcr.io/cloud-builders/docker"
    id: "docker-push"
    args:
      - "push"
      - "--all-tags"
      - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/event-service"
    waitFor: ["docker-build"]

  # Step 5: Deploy to Cloud Run (staging)
  - name: "gcr.io/google.com/cloudsdktool/cloud-sdk"
    id: "deploy-staging"
    entrypoint: "gcloud"
    args:
      - "run"
      - "deploy"
      - "event-service-staging"
      - "--image"
      - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/event-service:$COMMIT_SHA"
      - "--region"
      - "us-central1"
      - "--platform"
      - "managed"
      - "--quiet"
    waitFor: ["docker-push"]

  # Step 6: Run smoke tests against staging
  - name: "curlimages/curl"
    id: "smoke-test"
    entrypoint: "sh"
    args:
      - "-c"
      - |
        STAGING_URL=$$(gcloud run services describe event-service-staging \
          --region us-central1 --format 'value(status.url)')
        curl -sf "$${STAGING_URL}/actuator/health" || exit 1
        echo "Smoke test passed"
    waitFor: ["deploy-staging"]

  # Step 7: Deploy to production (manual approval via Cloud Build triggers)
  - name: "gcr.io/google.com/cloudsdktool/cloud-sdk"
    id: "deploy-prod"
    entrypoint: "gcloud"
    args:
      - "run"
      - "deploy"
      - "event-service"
      - "--image"
      - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/event-service:$COMMIT_SHA"
      - "--region"
      - "us-central1"
      - "--platform"
      - "managed"
      - "--quiet"
    waitFor: ["smoke-test"]

options:
  machineType: "E2_HIGHCPU_8"
  logging: CLOUD_LOGGING_ONLY

timeout: "1200s"

images:
  - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/event-service:$COMMIT_SHA"
  - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/event-service:latest"
```

---

## 11. Real Example: Event-Driven App on Cloud Run with Pub/Sub {#real-example}

### Document Entity

```java
package com.example.gcp.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;

import java.time.Instant;

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

    @Column(nullable = false)
    private String name;

    @Column(name = "gcs_path", nullable = false, unique = true)
    private String gcsPath;

    @Column(name = "content_type")
    private String contentType;

    @Column(name = "file_size")
    private Long fileSize;

    @Column(name = "uploaded_by", nullable = false)
    private String uploadedBy;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    @Builder.Default
    private DocumentStatus status = DocumentStatus.PENDING;

    @Column(name = "processed_text", columnDefinition = "TEXT")
    private String processedText;  // Extracted text content

    @Column(name = "thumbnail_path")
    private String thumbnailPath;

    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;

    @Column(name = "processed_at")
    private Instant processedAt;

    public enum DocumentStatus {
        PENDING, PROCESSING, PROCESSED, FAILED
    }
}
```

### Document Controller

```java
package com.example.gcp.controller;

import com.example.gcp.dto.DocumentResponse;
import com.example.gcp.dto.DocumentUploadRequest;
import com.example.gcp.entity.Document;
import com.example.gcp.service.DocumentUploadService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.List;

@Slf4j
@RestController
@RequestMapping("/api/v1/documents")
@RequiredArgsConstructor
public class DocumentController {

    private final DocumentUploadService uploadService;

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    @ResponseStatus(HttpStatus.ACCEPTED)
    public DocumentResponse uploadDocument(
            @RequestParam("file") MultipartFile file,
            @AuthenticationPrincipal Jwt jwt) throws IOException {

        String userId = jwt.getSubject();
        log.info("Document upload: user={}, file={}", userId, file.getOriginalFilename());

        return uploadService.uploadDocument(file, userId);
    }

    @GetMapping("/{id}")
    public DocumentResponse getDocument(
            @PathVariable String id,
            @AuthenticationPrincipal Jwt jwt) {

        return uploadService.getDocument(id, jwt.getSubject());
    }

    @GetMapping
    public List<DocumentResponse> listDocuments(
            @AuthenticationPrincipal Jwt jwt) {
        return uploadService.listUserDocuments(jwt.getSubject());
    }
}
```

### Document Upload Service (Full Orchestration)

```java
package com.example.gcp.service;

import com.example.gcp.dto.DocumentResponse;
import com.example.gcp.entity.Document;
import com.example.gcp.repository.DocumentRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class DocumentUploadService {

    private final GcsService gcsService;
    private final PubSubPublisher pubSubPublisher;
    private final DocumentRepository repository;

    private static final long MAX_SIZE = 20 * 1024 * 1024L; // 20 MB

    @Transactional
    @CacheEvict(value = "user-documents", key = "#userId")
    public DocumentResponse uploadDocument(MultipartFile file, String userId) throws IOException {
        if (file.getSize() > MAX_SIZE) {
            throw new IllegalArgumentException("File too large: max 20MB");
        }

        // Upload to GCS
        String gcsPath = gcsService.uploadFile(file, "documents/" + userId);
        log.info("Document uploaded to GCS: path={}", gcsPath);

        // Save metadata
        Document document = Document.builder()
                .name(file.getOriginalFilename())
                .gcsPath(gcsPath)
                .contentType(file.getContentType())
                .fileSize(file.getSize())
                .uploadedBy(userId)
                .status(Document.DocumentStatus.PENDING)
                .build();

        Document saved = repository.save(document);

        // Publish to Pub/Sub for async processing
        pubSubPublisher.publishDocumentUploaded(saved.getId(), gcsPath, userId)
                .thenAccept(msgId ->
                        log.info("Published upload event: docId={}, msgId={}", saved.getId(), msgId))
                .exceptionally(e -> {
                    log.error("Failed to publish event for doc: {}", saved.getId(), e);
                    return null;
                });

        return DocumentResponse.fromEntity(saved);
    }

    @Cacheable(value = "documents", key = "#id")
    public DocumentResponse getDocument(String id, String userId) {
        Document doc = repository.findById(id)
                .orElseThrow(() -> new RuntimeException("Document not found: " + id));

        if (!doc.getUploadedBy().equals(userId)) {
            throw new RuntimeException("Access denied");
        }

        return DocumentResponse.fromEntity(doc);
    }

    @Cacheable(value = "user-documents", key = "#userId")
    public List<DocumentResponse> listUserDocuments(String userId) {
        return repository.findByUploadedByOrderByCreatedAtDesc(userId)
                .stream()
                .map(DocumentResponse::fromEntity)
                .collect(Collectors.toList());
    }
}
```

### Document Processing Service (Pub/Sub Consumer)

```java
package com.example.gcp.service;

import com.example.gcp.entity.Document;
import com.example.gcp.repository.DocumentRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;

@Slf4j
@Service
@RequiredArgsConstructor
public class DocumentService {

    private final DocumentRepository repository;
    private final GcsService gcsService;
    private final PubSubPublisher pubSubPublisher;

    @Transactional
    @CacheEvict(value = "documents", key = "#documentId")
    public void processDocument(String documentId, String gcsPath, String userId) {
        Document document = repository.findById(documentId)
                .orElseThrow(() -> new RuntimeException("Document not found: " + documentId));

        repository.save(document.toBuilder()
                .status(Document.DocumentStatus.PROCESSING)
                .build());

        try {
            // Download and process document
            var inputStream = gcsService.downloadFile(gcsPath);
            String extractedText = extractTextFromDocument(document.getContentType(), inputStream);

            // Generate thumbnail if image
            String thumbnailPath = null;
            if (document.getContentType() != null &&
                    document.getContentType().startsWith("image/")) {
                thumbnailPath = generateThumbnail(gcsPath);
            }

            // Save results
            repository.save(document.toBuilder()
                    .status(Document.DocumentStatus.PROCESSED)
                    .processedText(extractedText)
                    .thumbnailPath(thumbnailPath)
                    .processedAt(Instant.now())
                    .build());

            // Notify user via Pub/Sub
            pubSubPublisher.publishNotification(userId,
                    "Document '" + document.getName() + "' has been processed",
                    "DOCUMENT_PROCESSED");

            log.info("Document processed: id={}", documentId);

        } catch (Exception e) {
            log.error("Processing failed for document: {}", documentId, e);
            repository.save(document.toBuilder()
                    .status(Document.DocumentStatus.FAILED)
                    .build());
            throw new RuntimeException("Document processing failed", e);
        }
    }

    private String extractTextFromDocument(String contentType, java.io.InputStream stream) {
        // Integration point: use Apache Tika, PDFBox, etc.
        log.info("Extracting text from document type: {}", contentType);
        return "Extracted text placeholder";
    }

    private String generateThumbnail(String gcsPath) {
        // Integration point: use ImageMagick, Thumbnailator, etc.
        String thumbnailPath = gcsPath.replace("/documents/", "/thumbnails/")
                + "_thumb.jpg";
        log.info("Generated thumbnail: {}", thumbnailPath);
        return thumbnailPath;
    }
}
```

### Document Repository

```java
package com.example.gcp.repository;

import com.example.gcp.entity.Document;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface DocumentRepository extends JpaRepository<Document, String> {

    List<Document> findByUploadedByOrderByCreatedAtDesc(String uploadedBy);

    List<Document> findByStatus(Document.DocumentStatus status);

    @Modifying
    @Query("UPDATE Document d SET d.status = :status WHERE d.id = :id")
    int updateStatus(String id, Document.DocumentStatus status);
}
```

### DocumentResponse DTO

```java
package com.example.gcp.dto;

import com.example.gcp.entity.Document;
import lombok.Builder;
import lombok.Data;

import java.time.Instant;

@Data
@Builder
public class DocumentResponse {
    private String id;
    private String name;
    private String contentType;
    private Long fileSize;
    private String status;
    private String processedText;
    private String thumbnailPath;
    private Instant createdAt;
    private Instant processedAt;

    public static DocumentResponse fromEntity(Document doc) {
        return DocumentResponse.builder()
                .id(doc.getId())
                .name(doc.getName())
                .contentType(doc.getContentType())
                .fileSize(doc.getFileSize())
                .status(doc.getStatus().name())
                .processedText(doc.getProcessedText())
                .thumbnailPath(doc.getThumbnailPath())
                .createdAt(doc.getCreatedAt())
                .processedAt(doc.getProcessedAt())
                .build();
    }
}
```

---

## 12. Summary {#summary}

| Concept | Key Points |
|---------|-----------|
| **Spring Cloud GCP** | Auto-configures GCP clients; uses Application Default Credentials in Cloud Run |
| **Cloud Storage** | Spring Resource abstraction with `gs://` URLs; SDK for advanced operations |
| **Pub/Sub** | `PubSubTemplate` for publishing; Spring Integration adapter for subscription |
| **Cloud SQL** | Socket factory handles Auth Proxy connection; specify instance connection name |
| **Memorystore** | Standard Redis integration; no TLS on basic tier; private VPC IP |
| **Secret Manager** | `sm://secret-name` in properties; resolved at startup automatically |
| **Cloud Run** | Serverless containers; auto-scale to 0; Workload Identity for GCP access |
| **GKE** | Full Kubernetes; Workload Identity maps K8s SA to GCP SA |
| **Cloud Build** | Native CI/CD; uses substitution variables (`$PROJECT_ID`, `$COMMIT_SHA`) |
| **IAM Best Practices** | Minimal permissions; prefer service accounts; never use user credentials in code |

### Required IAM Roles for Cloud Run Service Account

```
roles/cloudsql.client          → Cloud SQL connections
roles/storage.objectAdmin      → GCS read/write
roles/pubsub.publisher         → Pub/Sub publish
roles/pubsub.subscriber        → Pub/Sub subscribe
roles/secretmanager.secretAccessor → Secret Manager read
```

---

> **Next: Part 049 - DevOps and CI/CD for Spring Boot** — GitHub Actions, GitLab CI, Jenkins, SonarQube, OWASP checks, blue/green deployments, and complete pipelines.
