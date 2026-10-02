# Part 081: Spring Boot on AWS Lambda

## Overview

AWS Lambda lets you run code without provisioning servers. But Spring Boot was designed for long-running processes, not ephemeral functions. This creates tension: Spring's dependency injection, auto-configuration, and large classpath all contribute to slow cold starts. This part covers how to use `spring-cloud-function` to write Lambda functions in a Spring-idiomatic way, minimize cold start times with GraalVM native or SnapStart, and build a complete image processing pipeline.

---

## 1. Serverless Concepts

### Cold Start vs Warm Start

```
Cold Start (first invocation or after idle):
    Lambda creates a new execution environment
    ↓
    Download code package (ms)
    ↓
    Initialize JVM (100-500ms for native, up to 3s for Spring)
    ↓
    Spring application context starts (500ms - 5s)
    ↓
    Handle request

Warm Start (subsequent invocations):
    Reuse existing execution environment
    ↓
    Handle request (ms)
```

### Cost Model

```
Cost = invocations × (duration × memory_GB × $0.0000166667)
     + invocations × $0.0000002

Example: 1M requests × 500ms × 512MB = $4.27/month
```

---

## 2. Project Setup

### Maven Dependencies

```xml
<!-- pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<properties>
    <java.version>21</java.version>
    <spring-cloud.version>2023.0.0</spring-cloud.version>
    <aws-lambda-java.version>1.2.3</aws-lambda-java.version>
</properties>

<dependencies>
    <!-- Spring Cloud Function core -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-function-context</artifactId>
    </dependency>

    <!-- Lambda adapter -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-function-adapter-aws</artifactId>
    </dependency>

    <!-- AWS Lambda Java events (API Gateway, SQS, S3, etc.) -->
    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-lambda-java-events</artifactId>
        <version>3.11.3</version>
    </dependency>

    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-lambda-java-core</artifactId>
        <version>${aws-lambda-java.version}</version>
    </dependency>

    <!-- AWS SDK v2 -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3</artifactId>
        <version>2.21.41</version>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>sqs</artifactId>
        <version>2.21.41</version>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>ssm</artifactId>
        <version>2.21.41</version>
    </dependency>

    <!-- Image processing -->
    <dependency>
        <groupId>net.coobird</groupId>
        <artifactId>thumbnailator</artifactId>
        <version>0.4.20</version>
    </dependency>

    <!-- Reduce classpath size for Lambda -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-logging</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-log4j2</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <!-- Thin JAR for Lambda layers approach -->
                <layout>ZIP</layout>
            </configuration>
        </plugin>
    </plugins>
</build>
```

---

## 3. Basic Lambda Function

### Application Entry Point

```java
// src/main/java/com/example/lambda/ImageProcessorApplication.java
package com.example.lambda;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ImageProcessorApplication {
    public static void main(String[] args) {
        SpringApplication.run(ImageProcessorApplication.class, args);
    }
}
```

### Function as a Bean

```java
// src/main/java/com/example/lambda/functions/ResizeImageFunction.java
package com.example.lambda.functions;

import com.example.lambda.model.ResizeRequest;
import com.example.lambda.model.ResizeResult;
import com.example.lambda.service.ImageService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.function.Function;

@Configuration
public class ResizeImageFunction {

    private static final Logger log = LoggerFactory.getLogger(ResizeImageFunction.class);

    private final ImageService imageService;

    public ResizeImageFunction(ImageService imageService) {
        this.imageService = imageService;
    }

    /**
     * Spring Cloud Function uses the bean name as the function name.
     * This function processes image resize requests.
     */
    @Bean
    public Function<ResizeRequest, ResizeResult> resizeImage() {
        return request -> {
            log.info("Processing resize request: sourceKey={}, targetWidth={}",
                    request.sourceKey(), request.targetWidth());

            return imageService.resizeImage(request);
        };
    }
}
```

### Model Classes

```java
// src/main/java/com/example/lambda/model/ResizeRequest.java
package com.example.lambda.model;

public record ResizeRequest(
        String sourceBucket,
        String sourceKey,
        String destinationBucket,
        String destinationKey,
        int targetWidth,
        int targetHeight,
        String outputFormat   // "jpg", "png", "webp"
) {}
```

```java
// src/main/java/com/example/lambda/model/ResizeResult.java
package com.example.lambda.model;

public record ResizeResult(
        String destinationKey,
        long originalSizeBytes,
        long outputSizeBytes,
        int outputWidth,
        int outputHeight,
        long processingTimeMs
) {}
```

---

## 4. Lambda with API Gateway

### API Gateway Event Handler

```java
// src/main/java/com/example/lambda/handlers/ApiGatewayHandler.java
package com.example.lambda.handlers;

import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;
import com.example.lambda.model.ResizeRequest;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.function.adapter.aws.FunctionInvoker;

import java.util.Map;

/**
 * Lambda handler for API Gateway integration.
 * This is the class you set as Lambda's handler in AWS console.
 * Handler: com.example.lambda.handlers.ApiGatewayHandler::handleRequest
 */
public class ApiGatewayHandler extends FunctionInvoker {

    private static final Logger log = LoggerFactory.getLogger(ApiGatewayHandler.class);
}
```

### REST-style Function

```java
// src/main/java/com/example/lambda/functions/OrderApiFunction.java
package com.example.lambda.functions;

import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Map;
import java.util.function.Function;

@Configuration
public class OrderApiFunction {

    private static final Logger log = LoggerFactory.getLogger(OrderApiFunction.class);

    private final ObjectMapper objectMapper;
    private final OrderRepository orderRepository;

    public OrderApiFunction(ObjectMapper objectMapper, OrderRepository orderRepository) {
        this.objectMapper = objectMapper;
        this.orderRepository = orderRepository;
    }

    @Bean
    public Function<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> orders() {
        return request -> {
            log.info("API Gateway request: method={}, path={}",
                    request.getHttpMethod(), request.getPath());

            try {
                return switch (request.getHttpMethod()) {
                    case "GET" -> handleGetOrders(request);
                    case "POST" -> handleCreateOrder(request);
                    default -> buildResponse(405, Map.of("error", "Method Not Allowed"));
                };
            } catch (Exception e) {
                log.error("Error processing request", e);
                return buildResponse(500, Map.of("error", "Internal Server Error"));
            }
        };
    }

    private APIGatewayProxyResponseEvent handleGetOrders(
            APIGatewayProxyRequestEvent request) throws Exception {

        String customerId = request.getPathParameters() != null
                ? request.getPathParameters().get("customerId")
                : null;

        Object result = customerId != null
                ? orderRepository.findByCustomerId(customerId)
                : orderRepository.findAll();

        return buildResponse(200, result);
    }

    private APIGatewayProxyResponseEvent handleCreateOrder(
            APIGatewayProxyRequestEvent request) throws Exception {

        CreateOrderRequest orderRequest = objectMapper.readValue(
                request.getBody(), CreateOrderRequest.class);

        Order order = orderRepository.save(new Order(orderRequest));
        return buildResponse(201, order);
    }

    private APIGatewayProxyResponseEvent buildResponse(int statusCode, Object body) {
        try {
            return new APIGatewayProxyResponseEvent()
                    .withStatusCode(statusCode)
                    .withHeaders(Map.of(
                            "Content-Type", "application/json",
                            "Access-Control-Allow-Origin", "*"
                    ))
                    .withBody(objectMapper.writeValueAsString(body));
        } catch (Exception e) {
            return new APIGatewayProxyResponseEvent()
                    .withStatusCode(500)
                    .withBody("{\"error\":\"Serialization error\"}");
        }
    }

    public record CreateOrderRequest(String customerId, java.util.List<String> productIds) {}
}
```

---

## 5. Lambda with SQS Trigger

### SQS Event Processor

```java
// src/main/java/com/example/lambda/functions/SqsOrderProcessor.java
package com.example.lambda.functions;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.SQSEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.function.adapter.aws.FunctionInvoker;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.List;
import java.util.function.Function;

@Configuration
public class SqsOrderProcessor {

    private static final Logger log = LoggerFactory.getLogger(SqsOrderProcessor.class);

    private final ObjectMapper objectMapper;
    private final OrderService orderService;

    public SqsOrderProcessor(ObjectMapper objectMapper, OrderService orderService) {
        this.objectMapper = objectMapper;
        this.orderService = orderService;
    }

    /**
     * SQS batch processing.
     * Returns failed message IDs to allow partial batch failures.
     * Lambda will re-process only failed messages.
     */
    @Bean
    public Function<SQSEvent, SQSBatchResponse> processOrderEvents() {
        return sqsEvent -> {
            List<SQSBatchResponse.BatchItemFailure> failures = new ArrayList<>();

            for (SQSEvent.SQSMessage message : sqsEvent.getRecords()) {
                String messageId = message.getMessageId();

                try {
                    log.info("Processing SQS message: {}", messageId);

                    OrderEvent event = objectMapper.readValue(
                            message.getBody(), OrderEvent.class);

                    orderService.processEvent(event);

                    log.info("Successfully processed message: {}", messageId);

                } catch (Exception e) {
                    log.error("Failed to process message: {}", messageId, e);
                    // Return this message ID as failure for re-processing
                    failures.add(new SQSBatchResponse.BatchItemFailure(messageId));
                }
            }

            log.info("Batch processing complete: total={}, failures={}",
                    sqsEvent.getRecords().size(), failures.size());

            return new SQSBatchResponse(failures);
        };
    }

    public record OrderEvent(
            String eventType,
            String orderId,
            String customerId,
            String payload
    ) {}

    // Simple response model (use actual AWS SDK class in production)
    public record SQSBatchResponse(
            List<BatchItemFailure> batchItemFailures
    ) {
        public record BatchItemFailure(String itemIdentifier) {}
    }
}
```

---

## 6. Lambda with S3 Trigger

### S3 Event Processor (Image Processing Pipeline)

```java
// src/main/java/com/example/lambda/functions/S3ImageProcessor.java
package com.example.lambda.functions;

import com.amazonaws.services.lambda.runtime.events.S3Event;
import com.amazonaws.services.lambda.runtime.events.models.s3.S3EventNotification;
import com.example.lambda.service.ImageService;
import com.example.lambda.service.SqsNotificationService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.function.Consumer;

@Configuration
public class S3ImageProcessor {

    private static final Logger log = LoggerFactory.getLogger(S3ImageProcessor.class);

    private final ImageService imageService;
    private final SqsNotificationService sqsService;

    @Value("${OUTPUT_BUCKET:processed-images}")
    private String outputBucket;

    public S3ImageProcessor(ImageService imageService,
                             SqsNotificationService sqsService) {
        this.imageService = imageService;
        this.sqsService = sqsService;
    }

    /**
     * Triggered when an image is uploaded to S3.
     * Resizes to multiple dimensions and sends notification.
     */
    @Bean
    public Consumer<S3Event> processUploadedImages() {
        return s3Event -> {
            for (S3EventNotification.S3EventNotificationRecord record :
                    s3Event.getRecords()) {

                String bucket = record.getS3().getBucket().getName();
                String key = java.net.URLDecoder.decode(
                        record.getS3().getObject().getKey(),
                        java.nio.charset.StandardCharsets.UTF_8
                );
                long size = record.getS3().getObject().getSizeAsLong();

                log.info("Processing uploaded image: bucket={}, key={}, size={}",
                        bucket, key, size);

                // Skip non-image files
                if (!isImageFile(key)) {
                    log.info("Skipping non-image file: {}", key);
                    continue;
                }

                try {
                    // Generate multiple sizes
                    int[][] targetSizes = {
                            {1920, 1080},  // Full HD
                            {1280, 720},   // HD
                            {640, 480},    // SD
                            {200, 200},    // Thumbnail
                    };

                    for (int[] size2d : targetSizes) {
                        String sizeLabel = size2d[0] + "x" + size2d[1];
                        String destKey = "resized/" + sizeLabel + "/" + key;

                        imageService.resizeAndUpload(
                                bucket, key,
                                outputBucket, destKey,
                                size2d[0], size2d[1]
                        );

                        log.info("Resized to {}: {}", sizeLabel, destKey);
                    }

                    // Notify downstream systems
                    sqsService.sendProcessingComplete(bucket, key, outputBucket);

                } catch (Exception e) {
                    log.error("Failed to process image: bucket={}, key={}", bucket, key, e);
                    sqsService.sendProcessingFailed(bucket, key, e.getMessage());
                    throw new RuntimeException("Image processing failed", e);
                }
            }
        };
    }

    private boolean isImageFile(String key) {
        String lower = key.toLowerCase();
        return lower.endsWith(".jpg") || lower.endsWith(".jpeg")
                || lower.endsWith(".png") || lower.endsWith(".gif")
                || lower.endsWith(".webp") || lower.endsWith(".bmp");
    }
}
```

### Image Service

```java
// src/main/java/com/example/lambda/service/ImageService.java
package com.example.lambda.service;

import net.coobird.thumbnailator.Thumbnails;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.GetObjectRequest;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.time.Instant;

@Service
public class ImageService {

    private static final Logger log = LoggerFactory.getLogger(ImageService.class);

    private final S3Client s3Client;

    public ImageService(S3Client s3Client) {
        this.s3Client = s3Client;
    }

    public void resizeAndUpload(String sourceBucket, String sourceKey,
                                 String destBucket, String destKey,
                                 int targetWidth, int targetHeight) {
        Instant start = Instant.now();

        // Download from S3
        byte[] originalBytes = downloadFromS3(sourceBucket, sourceKey);
        log.debug("Downloaded {} bytes from s3://{}/{}", originalBytes.length, sourceBucket, sourceKey);

        // Resize in memory
        byte[] resizedBytes = resize(originalBytes, targetWidth, targetHeight, sourceKey);
        log.debug("Resized to {} bytes ({}x{})", resizedBytes.length, targetWidth, targetHeight);

        // Upload to S3
        uploadToS3(destBucket, destKey, resizedBytes, getContentType(sourceKey));
        log.debug("Uploaded to s3://{}/{}", destBucket, destKey);

        long elapsed = Instant.now().toEpochMilli() - start.toEpochMilli();
        log.info("Resize complete: {}ms, original={}KB, resized={}KB",
                elapsed, originalBytes.length / 1024, resizedBytes.length / 1024);
    }

    private byte[] downloadFromS3(String bucket, String key) {
        GetObjectRequest getRequest = GetObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .build();

        return s3Client.getObjectAsBytes(getRequest).asByteArray();
    }

    private byte[] resize(byte[] imageBytes, int width, int height, String filename) {
        try {
            ByteArrayInputStream input = new ByteArrayInputStream(imageBytes);
            ByteArrayOutputStream output = new ByteArrayOutputStream();

            String format = getOutputFormat(filename);

            Thumbnails.of(input)
                    .size(width, height)
                    .keepAspectRatio(true)
                    .outputQuality(0.85)
                    .outputFormat(format)
                    .toOutputStream(output);

            return output.toByteArray();
        } catch (Exception e) {
            throw new RuntimeException("Failed to resize image", e);
        }
    }

    private void uploadToS3(String bucket, String key, byte[] bytes, String contentType) {
        PutObjectRequest putRequest = PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .contentType(contentType)
                .contentLength((long) bytes.length)
                .build();

        s3Client.putObject(putRequest, RequestBody.fromBytes(bytes));
    }

    private String getOutputFormat(String filename) {
        String lower = filename.toLowerCase();
        if (lower.endsWith(".png")) return "png";
        if (lower.endsWith(".webp")) return "webp";
        return "jpg";
    }

    private String getContentType(String filename) {
        String lower = filename.toLowerCase();
        if (lower.endsWith(".png")) return "image/png";
        if (lower.endsWith(".webp")) return "image/webp";
        if (lower.endsWith(".gif")) return "image/gif";
        return "image/jpeg";
    }
}
```

---

## 7. AWS SDK Configuration

```java
// src/main/java/com/example/lambda/config/AwsConfig.java
package com.example.lambda.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.auth.credentials.DefaultCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.ssm.SsmClient;
import software.amazon.awssdk.http.urlconnection.UrlConnectionHttpClient;

@Configuration
public class AwsConfig {

    @Value("${AWS_REGION:us-east-1}")
    private String region;

    @Bean
    public S3Client s3Client() {
        return S3Client.builder()
                .region(Region.of(region))
                .credentialsProvider(DefaultCredentialsProvider.create())
                // Use URL connection client (lighter than Apache HTTP for Lambda)
                .httpClientBuilder(UrlConnectionHttpClient.builder())
                .build();
    }

    @Bean
    public SqsClient sqsClient() {
        return SqsClient.builder()
                .region(Region.of(region))
                .credentialsProvider(DefaultCredentialsProvider.create())
                .httpClientBuilder(UrlConnectionHttpClient.builder())
                .build();
    }

    @Bean
    public SsmClient ssmClient() {
        return SsmClient.builder()
                .region(Region.of(region))
                .credentialsProvider(DefaultCredentialsProvider.create())
                .httpClientBuilder(UrlConnectionHttpClient.builder())
                .build();
    }
}
```

---

## 8. SSM Parameter Store Integration

```java
// src/main/java/com/example/lambda/config/SsmParameterLoader.java
package com.example.lambda.config;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.services.ssm.SsmClient;
import software.amazon.awssdk.services.ssm.model.GetParameterRequest;
import software.amazon.awssdk.services.ssm.model.GetParametersByPathRequest;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class SsmParameterLoader {

    private static final Logger log = LoggerFactory.getLogger(SsmParameterLoader.class);

    private final SsmClient ssmClient;

    @Value("${SSM_PARAMETER_PATH:/myapp/prod}")
    private String parameterPath;

    public SsmParameterLoader(SsmClient ssmClient) {
        this.ssmClient = ssmClient;
    }

    /**
     * Load all parameters under a path at startup.
     * Cached for the lifetime of the Lambda execution environment (warm invocations reuse).
     */
    @Bean
    public Map<String, String> ssmParameters() {
        Map<String, String> params = new HashMap<>();

        try {
            GetParametersByPathRequest request = GetParametersByPathRequest.builder()
                    .path(parameterPath)
                    .withDecryption(true)
                    .recursive(true)
                    .build();

            ssmClient.getParametersByPath(request).parameters().forEach(param -> {
                // Strip the path prefix from key
                String key = param.name().replace(parameterPath + "/", "");
                params.put(key, param.value());
                log.info("Loaded SSM parameter: {}", key);
            });

        } catch (Exception e) {
            log.warn("Failed to load SSM parameters from path: {}", parameterPath, e);
        }

        return params;
    }

    /**
     * Get a single parameter with caching in a WeakHashMap.
     */
    public String getParameter(String name) {
        try {
            return ssmClient.getParameter(
                    GetParameterRequest.builder()
                            .name(name)
                            .withDecryption(true)
                            .build()
            ).parameter().value();
        } catch (Exception e) {
            log.error("Failed to get SSM parameter: {}", name, e);
            throw new RuntimeException("Failed to get parameter: " + name, e);
        }
    }
}
```

---

## 9. SnapStart for Reduced Cold Starts

### Enable SnapStart (Corretto runtime only)

```yaml
# SAM template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: java21
    Architectures: [x86_64]
    MemorySize: 1024
    Timeout: 30
    Environment:
      Variables:
        SPRING_CLOUD_FUNCTION_DEFINITION: processUploadedImages

Resources:
  ImageProcessorFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: com.example.lambda.handlers.S3EventHandler::handleRequest
      CodeUri: target/image-processor.jar
      SnapStart:
        ApplyOn: PublishedVersions    # Enable SnapStart
      Environment:
        Variables:
          OUTPUT_BUCKET: !Ref ProcessedImagesBucket
          QUEUE_URL: !Ref ProcessingNotificationQueue
          AWS_REGION: !Ref AWS::Region
      Events:
        S3Upload:
          Type: S3
          Properties:
            Bucket: !Ref UploadBucket
            Events: s3:ObjectCreated:*
            Filter:
              S3Key:
                Rules:
                  - Name: suffix
                    Value: .jpg
                  - Name: suffix
                    Value: .png
      Policies:
        - S3ReadPolicy:
            BucketName: !Ref UploadBucket
        - S3WritePolicy:
            BucketName: !Ref ProcessedImagesBucket
        - SQSSendMessagePolicy:
            QueueName: !GetAtt ProcessingNotificationQueue.QueueName
```

### CRaC Support for SnapStart

```java
// src/main/java/com/example/lambda/snapstart/SnapStartConfig.java
package com.example.lambda.snapstart;

import org.crac.Context;
import org.crac.Core;
import org.crac.Resource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import software.amazon.awssdk.services.s3.S3Client;

/**
 * CRaC (Coordinated Restore at Checkpoint) support for Lambda SnapStart.
 * Called before taking a snapshot and after restoring from one.
 */
@Component
public class SnapStartConfig implements Resource {

    private static final Logger log = LoggerFactory.getLogger(SnapStartConfig.class);

    private final S3Client s3Client;

    public SnapStartConfig(S3Client s3Client) {
        this.s3Client = s3Client;
        Core.getGlobalContext().register(this);
    }

    @Override
    public void beforeCheckpoint(Context<? extends Resource> context) throws Exception {
        log.info("SnapStart: before checkpoint - closing connections");
        // Close any connections before snapshot is taken
        // AWS SDK connections can't be serialized
        s3Client.close();
    }

    @Override
    public void afterRestore(Context<? extends Resource> context) throws Exception {
        log.info("SnapStart: after restore - reinitializing connections");
        // Re-initialize connections after restore
        // The S3Client will be re-created from the bean factory
    }
}
```

---

## 10. Function Routing

### Multiple Functions in One Lambda

```yaml
# application.yml
spring:
  cloud:
    function:
      routing-expression: headers['X-Function']

# OR for routing by event type:
# routing-expression: payload.eventType
```

```java
// src/main/java/com/example/lambda/functions/FunctionCatalog.java
package com.example.lambda.functions;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.function.Function;

@Configuration
public class FunctionCatalog {

    @Bean("uppercase")
    public Function<String, String> uppercase() {
        return String::toUpperCase;
    }

    @Bean("lowercase")
    public Function<String, String> lowercase() {
        return String::toLowerCase;
    }

    @Bean("reverse")
    public Function<String, String> reverse() {
        return s -> new StringBuilder(s).reverse().toString();
    }

    // Function composition
    @Bean("uppercaseReverse")
    public Function<String, String> uppercaseReverse() {
        return uppercase().andThen(reverse());
    }
}
```

---

## 11. Lambda Layers for Shared Dependencies

### Build Layer Script

```bash
#!/bin/bash
# build-layer.sh

# Create layer directory structure
mkdir -p lambda-layer/java/lib

# Copy all dependency JARs to layer
mvn dependency:copy-dependencies -DoutputDirectory=lambda-layer/java/lib

# Remove JARs that are part of the function code
rm lambda-layer/java/lib/my-function*.jar

# Create ZIP
cd lambda-layer
zip -r ../lambda-layer.zip java/
```

### SAM Template for Layer

```yaml
# template.yaml (layers section)
Layers:
  SpringDependenciesLayer:
    Type: AWS::Serverless::LayerVersion
    Properties:
      LayerName: spring-dependencies
      Description: Spring Boot and AWS SDK dependencies
      ContentUri: lambda-layer.zip
      CompatibleRuntimes:
        - java21
      RetentionPolicy: Retain
```

---

## 12. Cost Optimization

### Memory Tuning

```java
// src/main/java/com/example/lambda/util/MemoryOptimizer.java
package com.example.lambda.util;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.context.event.ApplicationStartedEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

import java.lang.management.ManagementFactory;
import java.lang.management.MemoryMXBean;

/**
 * Logs memory usage after startup to help tune Lambda memory setting.
 * Rule of thumb: set Lambda memory to 1.5x actual usage.
 */
@Component
public class MemoryOptimizer {

    private static final Logger log = LoggerFactory.getLogger(MemoryOptimizer.class);

    @EventListener(ApplicationStartedEvent.class)
    public void logMemoryAfterStartup() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        long usedMB = memoryBean.getHeapMemoryUsage().getUsed() / (1024 * 1024);
        long maxMB = memoryBean.getHeapMemoryUsage().getMax() / (1024 * 1024);

        log.info("Memory after startup: {}MB used / {}MB max", usedMB, maxMB);
        log.info("Recommended Lambda memory: {}MB", (long) (usedMB * 1.5));
    }
}
```

### Tiered Compilation (Faster Startup)

```bash
# Lambda environment variable for faster JVM startup
JAVA_TOOL_OPTIONS=-XX:+TieredCompilation -XX:TieredStopAtLevel=1 -Xss512k -XX:MaxRAM=256m
```

```yaml
# application.yml for Lambda
spring:
  main:
    # Faster startup: don't initialize web server in Lambda
    web-application-type: none
    # Enable lazy initialization
    lazy-initialization: true
  jmx:
    enabled: false  # No JMX in Lambda
  jackson:
    mapper:
      default-view-inclusion: false
```

---

## 13. Local Testing with SAM CLI

### SAM Test Events

```json
// events/s3-event.json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": {
          "name": "my-upload-bucket",
          "arn": "arn:aws:s3:::my-upload-bucket"
        },
        "object": {
          "key": "uploads/photo.jpg",
          "size": 1024000
        }
      }
    }
  ]
}
```

```json
// events/sqs-event.json
{
  "Records": [
    {
      "messageId": "059f36b4-87a3-44ab-83d2-661975830a7d",
      "receiptHandle": "AQEBwJnKyrHigUMZj6reyNurDd9...",
      "body": "{\"eventType\":\"ORDER_PLACED\",\"orderId\":\"123\",\"customerId\":\"456\"}",
      "attributes": {
        "ApproximateReceiveCount": "1",
        "SentTimestamp": "1573602047670"
      },
      "messageAttributes": {},
      "md5OfBody": "e4e68fb7bd0e697a0ae8f1bb342846b0",
      "eventSource": "aws:sqs",
      "eventSourceARN": "arn:aws:sqs:us-east-1:123456789:MyQueue",
      "awsRegion": "us-east-1"
    }
  ]
}
```

```bash
# Build and test locally with SAM
mvn package -DskipTests

# Start local API
sam local start-api

# Invoke with S3 event
sam local invoke ImageProcessorFunction --event events/s3-event.json

# Invoke with SQS event
sam local invoke SqsProcessorFunction --event events/sqs-event.json

# Watch logs
sam logs -n ImageProcessorFunction --stack-name image-processor --tail

# Deploy to AWS
sam deploy --guided
```

---

## 14. Complete Image Processing Pipeline

### Architecture

```
S3 (uploads/)
    ↓ trigger
Lambda (ImageProcessor)
    ↓ resize to 4 sizes
S3 (processed/)
    ↓ notify
SQS (processing-complete)
    ↓ trigger
Lambda (NotificationProcessor)
    ↓
DynamoDB (image-metadata) + SNS (webhooks)
```

### Pipeline Orchestrator

```java
// src/main/java/com/example/lambda/pipeline/ImagePipeline.java
package com.example.lambda.pipeline;

import com.amazonaws.services.lambda.runtime.events.S3Event;
import com.amazonaws.services.lambda.runtime.events.models.s3.S3EventNotification;
import com.example.lambda.service.ImageService;
import com.example.lambda.service.MetadataService;
import com.example.lambda.service.SqsNotificationService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Map;
import java.util.function.Consumer;

@Configuration
public class ImagePipeline {

    private static final Logger log = LoggerFactory.getLogger(ImagePipeline.class);

    private final ImageService imageService;
    private final MetadataService metadataService;
    private final SqsNotificationService notificationService;

    @Value("${OUTPUT_BUCKET:processed-images}")
    private String outputBucket;

    @Value("${QUEUE_URL}")
    private String queueUrl;

    public ImagePipeline(ImageService imageService,
                          MetadataService metadataService,
                          SqsNotificationService notificationService) {
        this.imageService = imageService;
        this.metadataService = metadataService;
        this.notificationService = notificationService;
    }

    @Bean
    public Consumer<S3Event> imageProcessingPipeline() {
        return s3Event -> {
            for (S3EventNotification.S3EventNotificationRecord record : s3Event.getRecords()) {
                String sourceBucket = record.getS3().getBucket().getName();
                String sourceKey = decodeKey(record.getS3().getObject().getKey());

                log.info("Pipeline started: s3://{}/{}", sourceBucket, sourceKey);

                ProcessingContext ctx = new ProcessingContext(sourceBucket, sourceKey);

                // Step 1: Validate
                if (!isValidImage(sourceKey)) {
                    log.warn("Invalid file type, skipping: {}", sourceKey);
                    continue;
                }

                // Step 2: Extract metadata
                ctx.setMetadata(imageService.extractMetadata(sourceBucket, sourceKey));

                // Step 3: Generate variants
                Map<String, String> generatedKeys = imageService.generateVariants(
                        sourceBucket, sourceKey, outputBucket,
                        new int[][]{{1920, 1080}, {1280, 720}, {640, 360}, {200, 200}}
                );

                ctx.setGeneratedKeys(generatedKeys);

                // Step 4: Store metadata
                metadataService.save(ctx);

                // Step 5: Notify downstream
                notificationService.sendProcessingComplete(ctx, queueUrl);

                log.info("Pipeline complete: {} variants generated for {}",
                        generatedKeys.size(), sourceKey);
            }
        };
    }

    private String decodeKey(String key) {
        return java.net.URLDecoder.decode(key, java.nio.charset.StandardCharsets.UTF_8);
    }

    private boolean isValidImage(String key) {
        String lower = key.toLowerCase();
        return lower.matches(".*\\.(jpg|jpeg|png|gif|webp)$");
    }

    public static class ProcessingContext {
        private final String sourceBucket;
        private final String sourceKey;
        private Map<String, Object> metadata;
        private Map<String, String> generatedKeys;

        public ProcessingContext(String sourceBucket, String sourceKey) {
            this.sourceBucket = sourceBucket;
            this.sourceKey = sourceKey;
        }

        public String getSourceBucket() { return sourceBucket; }
        public String getSourceKey() { return sourceKey; }
        public Map<String, Object> getMetadata() { return metadata; }
        public void setMetadata(Map<String, Object> metadata) { this.metadata = metadata; }
        public Map<String, String> getGeneratedKeys() { return generatedKeys; }
        public void setGeneratedKeys(Map<String, String> generatedKeys) { this.generatedKeys = generatedKeys; }
    }
}
```

---

## Summary

| Topic | Key Point |
|-------|-----------|
| Cold Start | Use SnapStart, lazy init, and native compilation to minimize |
| Function Type | Use `java.util.function.Function` for request/response |
| Consumer | Use `java.util.function.Consumer` for fire-and-forget (S3/SQS) |
| Routing | Use `spring.cloud.function.definition` or routing expression |
| Memory | Set to 1.5x actual usage; more memory = more CPU = faster |
| Dependencies | Use Lambda layers to separate large JARs |
| Config | SSM Parameter Store for secrets; env vars for simple config |
| Testing | SAM CLI for local testing; TestContainers for integration tests |
| SnapStart | Only on published versions; requires CRaC hooks for connections |
| Cost | Balance memory (cost per ms) vs cold start (user experience) |

### Cold Start Comparison

| Approach | Cold Start | Notes |
|----------|-----------|-------|
| Spring Boot + full auto-config | 5-10s | Not suitable for Lambda |
| Spring Boot + lazy init + no web | 2-4s | Acceptable |
| Spring Boot + SnapStart | 300ms-1s | Best with Spring |
| Micronaut / Quarkus | 200-500ms | Purpose-built for serverless |
| GraalVM native | 50-200ms | Best cold start, complex build |

---

## Next Part Preview

**Part 082: Multi-Module Spring Boot Projects** - We'll build an e-commerce monorepo with 8 Maven modules, covering shared libraries, API contracts, parent POM management, parallel builds, and CI/CD configuration.
