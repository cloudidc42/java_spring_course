# Part 034: Docker & Containerization for Spring Boot

## เนื้อหาในส่วนนี้
- Docker Fundamentals
- Dockerfile for Spring Boot
- Multi-stage Builds
- Docker Compose for Development
- Docker Networking & Volumes
- Health Checks & Graceful Shutdown
- Best Practices for Production
- Docker with Microservices
- Container Security

---

## 1. Docker Fundamentals

```bash
# Core Docker Commands
docker version          # Check Docker version
docker info             # System-wide info
docker ps               # Running containers
docker ps -a            # All containers
docker images           # Local images

# Run a container
docker run -d \
  --name my-app \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  my-spring-app:latest

# Build an image
docker build -t my-spring-app:1.0.0 .

# View logs
docker logs my-app -f   # Follow logs
docker logs my-app --tail=100

# Execute command in container
docker exec -it my-app /bin/sh

# Stop & Remove
docker stop my-app
docker rm my-app
docker rmi my-spring-app:1.0.0

# Clean up
docker system prune -f          # Remove unused resources
docker image prune -f           # Remove dangling images
docker volume prune -f          # Remove unused volumes
```

---

## 2. Dockerfile for Spring Boot

### Simple Dockerfile (Not Recommended for Production)

```dockerfile
# Simple - not efficient (whole fat jar)
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY target/myapp-1.0.0.jar app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

### Layered Dockerfile (Recommended)

```dockerfile
# Stage 1: Extract layers from the jar
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /build
COPY target/myapp-1.0.0.jar app.jar

# Spring Boot creates layered jar structure
RUN java -Djarmode=layertools -jar app.jar extract

# Stage 2: Final image
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Create non-root user for security
RUN addgroup --system spring && adduser --system spring --ingroup spring
USER spring:spring

# Copy layers in order (least to most frequently changed)
COPY --from=builder /build/dependencies/ ./
COPY --from=builder /build/spring-boot-loader/ ./
COPY --from=builder /build/snapshot-dependencies/ ./
COPY --from=builder /build/application/ ./

# JVM settings
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+ExitOnOutOfMemoryError"

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]
```

### Enable Layered Jar in Spring Boot

```xml
<!-- pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <layers>
                    <enabled>true</enabled>
                </layers>
                <image>
                    <!-- Cloud Native Buildpacks (alternative to Dockerfile) -->
                    <name>myregistry/${project.artifactId}:${project.version}</name>
                </image>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Cloud Native Buildpacks (No Dockerfile Needed)

```bash
# Build image using Buildpacks (no Dockerfile needed)
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=myapp:latest

# Or with Gradle
./gradlew bootBuildImage --imageName=myapp:latest
```

---

## 3. Docker Compose for Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Spring Boot App
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder    # Multi-stage target
    image: myapp:dev
    container_name: spring-app
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/appdb
      SPRING_DATASOURCE_USERNAME: appuser
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD:-secret}
      SPRING_REDIS_HOST: redis
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-network
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    container_name: app-postgres
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./docker/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: app-redis
    command: redis-server --requirepass ${REDIS_PASSWORD:-redispass} --maxmemory 256mb
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network

  # Kafka + Zookeeper
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - app-network

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    networks:
      - app-network

  # pgAdmin (Database UI)
  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    networks:
      - app-network
    profiles:
      - tools   # Only start with --profile tools

  # Zipkin (Distributed Tracing UI)
  zipkin:
    image: openzipkin/zipkin:3
    ports:
      - "9411:9411"
    networks:
      - app-network
    profiles:
      - monitoring

volumes:
  postgres-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

```bash
# Docker Compose commands
docker compose up -d              # Start all services in background
docker compose up -d app postgres # Start specific services
docker compose down               # Stop and remove containers
docker compose down -v            # Also remove volumes

# With profiles
docker compose --profile tools up -d      # Include pgAdmin
docker compose --profile monitoring up -d  # Include Zipkin

# Logs
docker compose logs app -f        # Follow app logs
docker compose logs -f            # All services

# Scale a service
docker compose up -d --scale app=3

# Rebuild and restart
docker compose up -d --build app
```

---

## 4. Spring Boot Docker Configuration

```yaml
# application-docker.yml (active when SPRING_PROFILES_ACTIVE=docker)
spring:
  datasource:
    url: jdbc:postgresql://postgres:5432/appdb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000
  
  redis:
    host: redis
    port: 6379
    password: ${REDIS_PASSWORD}
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 8
  
  kafka:
    bootstrap-servers: kafka:9092
  
  jpa:
    hibernate:
      ddl-auto: validate   # Never auto-create in production!

server:
  port: 8080
  shutdown: graceful       # Wait for ongoing requests
  
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s  # Give 20s to finish requests

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true   # Enable /actuator/health/liveness and /readiness
```

---

## 5. Graceful Shutdown

```java
import org.springframework.context.event.EventListener;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.stereotype.Component;
import jakarta.annotation.PreDestroy;
import java.util.concurrent.*;

@Component
public class GracefulShutdownHandler {
    
    private final ExecutorService taskExecutor = Executors.newFixedThreadPool(5);
    private volatile boolean shuttingDown = false;
    
    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        System.out.println("Application ready, accepting traffic");
    }
    
    public boolean isShuttingDown() { return shuttingDown; }
    
    @PreDestroy
    public void onShutdown() throws InterruptedException {
        System.out.println("Graceful shutdown initiated...");
        shuttingDown = true;
        
        // Stop accepting new tasks
        taskExecutor.shutdown();
        
        // Wait for ongoing tasks (max 20s)
        if (!taskExecutor.awaitTermination(20, TimeUnit.SECONDS)) {
            System.err.println("Force shutdown after 20s timeout");
            taskExecutor.shutdownNow();
        }
        
        System.out.println("Graceful shutdown complete");
    }
}

// Health endpoint integration
@Component
class AppHealthIndicator implements org.springframework.boot.actuate.health.HealthIndicator {
    
    private final GracefulShutdownHandler shutdownHandler;
    
    AppHealthIndicator(GracefulShutdownHandler shutdownHandler) {
        this.shutdownHandler = shutdownHandler;
    }
    
    @Override
    public org.springframework.boot.actuate.health.Health health() {
        if (shutdownHandler.isShuttingDown()) {
            return org.springframework.boot.actuate.health.Health.outOfService()
                .withDetail("reason", "Application shutting down")
                .build();
        }
        return org.springframework.boot.actuate.health.Health.up().build();
    }
}
```

---

## 6. Multi-Service Docker Compose (Microservices)

```yaml
# docker-compose-microservices.yml
version: '3.8'

services:
  # Infrastructure
  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"
    networks: [ms-network]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8761/actuator/health"]
      interval: 20s
      retries: 5
      start_period: 30s

  config-server:
    build: ./config-server
    ports:
      - "8888:8888"
    networks: [ms-network]
    depends_on:
      eureka-server:
        condition: service_healthy
    environment:
      EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE: http://eureka-server:8761/eureka/

  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    networks: [ms-network]
    depends_on:
      eureka-server:
        condition: service_healthy
    environment:
      EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE: http://eureka-server:8761/eureka/

  # Microservices
  user-service:
    build: ./user-service
    networks: [ms-network]
    depends_on:
      postgres:
        condition: service_healthy
      eureka-server:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/userdb
      EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE: http://eureka-server:8761/eureka/
    deploy:
      replicas: 2   # Run 2 instances

  order-service:
    build: ./order-service
    networks: [ms-network]
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_started
      eureka-server:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/orderdb
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE: http://eureka-server:8761/eureka/

  # Databases & Infrastructure
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_MULTIPLE_DATABASES: userdb,orderdb,productdb
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./docker/create-multiple-dbs.sh:/docker-entrypoint-initdb.d/create-dbs.sh
    networks: [ms-network]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    networks: [ms-network]

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks: [ms-network]

  zipkin:
    image: openzipkin/zipkin:3
    ports:
      - "9411:9411"
    networks: [ms-network]

volumes:
  postgres-data:

networks:
  ms-network:
    driver: bridge
```

```bash
# Script to create multiple databases (docker/create-multiple-dbs.sh)
#!/bin/bash
set -e
set -u

function create_user_and_database() {
    local database=$1
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
        CREATE DATABASE $database;
        GRANT ALL PRIVILEGES ON DATABASE $database TO $POSTGRES_USER;
EOSQL
}

if [ -n "$POSTGRES_MULTIPLE_DATABASES" ]; then
    for db in $(echo $POSTGRES_MULTIPLE_DATABASES | tr ',' ' '); do
        create_user_and_database $db
    done
fi
```

---

## 7. Container Security Best Practices

```dockerfile
# Security-hardened Dockerfile
FROM eclipse-temurin:21-jre-alpine AS base

# Install security updates
RUN apk update && apk upgrade --no-cache

# Create dedicated user (no shell, no home)
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup -s /sbin/nologin

# Set working directory with correct ownership
WORKDIR /app
RUN chown appuser:appgroup /app

FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM base
# Copy application files owned by appuser
COPY --chown=appuser:appgroup --from=builder /build/dependencies/ ./
COPY --chown=appuser:appgroup --from=builder /build/spring-boot-loader/ ./
COPY --chown=appuser:appgroup --from=builder /build/snapshot-dependencies/ ./
COPY --chown=appuser:appgroup --from=builder /build/application/ ./

# Switch to non-root user
USER appuser:appgroup

# Read-only filesystem (security)
# Add --read-only flag when running: docker run --read-only --tmpfs /tmp

# No new privileges
# Add --security-opt=no-new-privileges flag when running

EXPOSE 8080

# JVM security settings
ENV JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+ExitOnOutOfMemoryError \
  -Djava.security.egd=file:/dev/./urandom \
  -Dfile.encoding=UTF-8"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]
```

```bash
# Run container securely
docker run -d \
  --name my-app \
  --read-only \
  --tmpfs /tmp \
  --security-opt=no-new-privileges \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  --memory=512m \
  --cpus=1.0 \
  my-spring-app:latest

# Scan image for vulnerabilities
docker scout cves my-spring-app:latest
# Or use Trivy
trivy image my-spring-app:latest
```

---

## 8. .dockerignore

```
# .dockerignore
.git
.gitignore
.mvn
target/
!target/*.jar    # Keep the jar
*.md
*.log
logs/
.idea/
*.iml
.vscode/
node_modules/
docker/
docker-compose*.yml
Dockerfile*
```

---

## 9. Building & Publishing Images

```bash
# Build with version tag
VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
docker build -t myorg/myapp:${VERSION} -t myorg/myapp:latest .

# Push to Docker Hub
docker login
docker push myorg/myapp:${VERSION}
docker push myorg/myapp:latest

# Push to AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com
docker tag myapp:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# CI/CD with GitHub Actions
# .github/workflows/docker.yml
```

```yaml
# .github/workflows/docker.yml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build-push:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven
      
      - name: Build with Maven
        run: ./mvnw package -DskipTests
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: myorg/myapp
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64  # Multi-arch
```

---

## 10. JVM Container Tuning

```bash
# Recommended JVM flags for containers
JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -XX:+ExitOnOutOfMemoryError \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/app/dumps/heap.hprof \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+UseStringDeduplication \
  -Xss256k \
  -Djava.awt.headless=true"

# For Java 21 with Virtual Threads
JAVA_OPTS="$JAVA_OPTS -Djdk.tracePinnedThreads=full"

# Resource limits in docker-compose
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
```

---

## สรุป Part 034

| Topic | Command / Config |
|-------|-----------------|
| Build image | `docker build -t app:1.0 .` |
| Layered jar | `spring-boot-maven-plugin` with layers |
| Compose up | `docker compose up -d` |
| Non-root | `USER appuser:appgroup` in Dockerfile |
| Memory | `-XX:MaxRAMPercentage=75.0` |
| Health check | `HEALTHCHECK CMD wget /actuator/health` |
| Security scan | `trivy image myapp:latest` |

---

**Part 035:** Kubernetes สำหรับ Spring Boot
- Pods, Deployments, Services
- ConfigMaps & Secrets
- Horizontal Pod Autoscaler
- Ingress Controller
- Helm Charts
- Rolling Updates & Rollbacks
