# Part 035: Kubernetes for Spring Boot

## Introduction

Kubernetes (K8s) is the industry-standard container orchestration platform. It automates deployment, scaling, and management of containerized applications. For Spring Boot microservices, Kubernetes provides the infrastructure backbone: self-healing, rolling updates, service discovery, and horizontal scaling — all declaratively configured in YAML.

This part covers everything you need to deploy real Spring Boot applications to Kubernetes, from architecture fundamentals to Helm charts.

---

## 1. Kubernetes Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                         KUBERNETES CLUSTER                      │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     CONTROL PLANE                        │  │
│  │                                                          │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │  │
│  │  │  API Server  │  │  Scheduler  │  │ Controller Mgr  │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │  │
│  │  ┌─────────────┐                                         │  │
│  │  │    etcd     │  (distributed key-value store)         │  │
│  │  └─────────────┘                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   WORKER 1   │  │   WORKER 2   │  │      WORKER 3        │  │
│  │              │  │              │  │                      │  │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────────────┐  │  │
│  │  │  Pod   │  │  │  │  Pod   │  │  │  │      Pod       │  │  │
│  │  │ [App]  │  │  │  │ [App]  │  │  │  │    [App]       │  │  │
│  │  └────────┘  │  │  └────────┘  │  │  └────────────────┘  │  │
│  │  kubelet     │  │  kubelet     │  │  kubelet             │  │
│  │  kube-proxy  │  │  kube-proxy  │  │  kube-proxy          │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Key Concepts:**

| Component | Role |
|-----------|------|
| **Pod** | Smallest deployable unit; one or more containers sharing network/storage |
| **Node** | Worker machine (VM or bare metal) running Pods |
| **Cluster** | Set of Nodes managed by Control Plane |
| **Control Plane** | Brain of Kubernetes; API Server, Scheduler, Controller Manager, etcd |
| **Deployment** | Declares desired state; manages ReplicaSets |
| **Service** | Stable network endpoint for a group of Pods |
| **Namespace** | Virtual cluster for resource isolation |
| **ConfigMap** | Non-sensitive configuration data |
| **Secret** | Sensitive configuration data (base64-encoded) |

---

## 2. kubectl Basics

### Installation and Context Setup

```bash
# Install kubectl (Linux)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# View cluster info
kubectl cluster-info

# View current context
kubectl config current-context

# Switch context (e.g., between minikube and production)
kubectl config use-context minikube

# List all contexts
kubectl config get-contexts
```

### Essential kubectl Commands

```bash
# ─── PODS ───────────────────────────────────────────────────────────────
kubectl get pods                         # List pods in current namespace
kubectl get pods -n my-namespace         # List pods in specific namespace
kubectl get pods -o wide                 # Show node placement
kubectl describe pod my-pod-xyz          # Detailed info + events
kubectl logs my-pod-xyz                  # View logs
kubectl logs my-pod-xyz -f               # Follow logs (tail)
kubectl logs my-pod-xyz -c my-container  # Specific container in multi-container pod
kubectl exec -it my-pod-xyz -- /bin/sh   # Shell into pod
kubectl delete pod my-pod-xyz            # Delete pod (Deployment recreates it)

# ─── DEPLOYMENTS ────────────────────────────────────────────────────────
kubectl get deployments
kubectl describe deployment my-app
kubectl scale deployment my-app --replicas=5
kubectl rollout status deployment/my-app
kubectl rollout history deployment/my-app
kubectl rollout undo deployment/my-app

# ─── SERVICES ───────────────────────────────────────────────────────────
kubectl get services
kubectl get svc                          # Short form
kubectl describe svc my-service

# ─── APPLY / DELETE ─────────────────────────────────────────────────────
kubectl apply -f deployment.yaml         # Create or update resource
kubectl delete -f deployment.yaml        # Delete resource from file
kubectl apply -f ./k8s/                  # Apply all YAML in directory

# ─── DEBUGGING ──────────────────────────────────────────────────────────
kubectl get events --sort-by='.lastTimestamp'
kubectl top pods                         # CPU/memory usage (requires metrics-server)
kubectl top nodes
kubectl port-forward svc/my-service 8080:80  # Forward local port to service
```

---

## 3. Spring Boot Application Setup

### Sample Spring Boot Application

```java
// src/main/java/com/example/k8sdemo/K8sDemoApplication.java
package com.example.k8sdemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class K8sDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(K8sDemoApplication.class, args);
    }
}
```

```java
// src/main/java/com/example/k8sdemo/controller/HealthController.java
package com.example.k8sdemo.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.Map;

@RestController
public class HealthController {

    @Value("${app.version:1.0.0}")
    private String appVersion;

    @Value("${app.environment:local}")
    private String environment;

    @GetMapping("/")
    public Map<String, String> home() throws UnknownHostException {
        return Map.of(
            "version", appVersion,
            "environment", environment,
            "host", InetAddress.getLocalHost().getHostName(),
            "status", "UP"
        );
    }

    @GetMapping("/api/hello")
    public Map<String, String> hello() {
        return Map.of("message", "Hello from Kubernetes!");
    }
}
```

```yaml
# src/main/resources/application.yaml
spring:
  application:
    name: k8s-demo

app:
  version: ${APP_VERSION:1.0.0}
  environment: ${APP_ENVIRONMENT:local}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

### Dockerfile (Multi-stage Build)

```dockerfile
# Dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN apk add --no-cache maven && mvn clean package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Create non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080

# Use exec form to handle SIGTERM properly
ENTRYPOINT ["java", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "app.jar"]
```

```bash
# Build and push image
docker build -t myregistry/k8s-demo:1.0.0 .
docker push myregistry/k8s-demo:1.0.0
```

---

## 4. Deployment YAML for Spring Boot

### Basic Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo
  namespace: production
  labels:
    app: k8s-demo
    version: "1.0.0"
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: k8s-demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Allow 1 extra pod during update
      maxUnavailable: 0    # Never have fewer than desired replicas
  template:
    metadata:
      labels:
        app: k8s-demo
        version: "1.0.0"
    spec:
      # Grace period for shutdown
      terminationGracePeriodSeconds: 60

      # Security context for the pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000

      containers:
        - name: k8s-demo
          image: myregistry/k8s-demo:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
              name: http

          # Environment variables from ConfigMap and Secret
          envFrom:
            - configMapRef:
                name: k8s-demo-config
            - secretRef:
                name: k8s-demo-secrets

          # Additional individual env vars
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production"
            - name: APP_VERSION
              value: "1.0.0"
            # Expose pod info to the application
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName

          # Resource limits and requests
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"

          # Health probes (covered in detail below)
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
            timeoutSeconds: 3

          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 30   # Allow up to 5 min to start

          # Mount ConfigMap as file
          volumeMounts:
            - name: app-config
              mountPath: /app/config
              readOnly: true

      volumes:
        - name: app-config
          configMap:
            name: k8s-demo-files

      # Pod anti-affinity: spread across nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - k8s-demo
                topologyKey: kubernetes.io/hostname
```

---

## 5. Service Types

### ClusterIP (Internal only)

```yaml
# k8s/service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: k8s-demo
  namespace: production
  labels:
    app: k8s-demo
spec:
  type: ClusterIP      # Default; only reachable within cluster
  selector:
    app: k8s-demo      # Routes to pods with this label
  ports:
    - name: http
      port: 80         # Service port
      targetPort: 8080  # Container port
      protocol: TCP
```

### NodePort (External via node IP)

```yaml
# k8s/service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: k8s-demo-nodeport
  namespace: production
spec:
  type: NodePort
  selector:
    app: k8s-demo
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080   # Range: 30000–32767; omit for random assignment
```

### LoadBalancer (Cloud provider external LB)

```yaml
# k8s/service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: k8s-demo-lb
  namespace: production
  annotations:
    # AWS-specific annotation for NLB
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app: k8s-demo
  ports:
    - port: 80
      targetPort: 8080
```

### Headless Service (for StatefulSets/direct pod access)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: k8s-demo-headless
spec:
  clusterIP: None    # Headless: no virtual IP assigned
  selector:
    app: k8s-demo
  ports:
    - port: 8080
```

---

## 6. ConfigMap and Secret for Spring Boot

### ConfigMap

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: k8s-demo-config
  namespace: production
data:
  # Key-value pairs become environment variables
  APP_ENVIRONMENT: "production"
  APP_LOG_LEVEL: "INFO"
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres-service:5432/appdb"
  SERVER_PORT: "8080"

---
# ConfigMap as a mounted file
apiVersion: v1
kind: ConfigMap
metadata:
  name: k8s-demo-files
  namespace: production
data:
  application-k8s.yaml: |
    spring:
      datasource:
        url: jdbc:postgresql://postgres-service:5432/appdb
        hikari:
          maximum-pool-size: 10
          minimum-idle: 2
      jpa:
        hibernate:
          ddl-auto: validate
    logging:
      level:
        root: INFO
        com.example: DEBUG
```

### Secret

```bash
# Create secret from literals (base64-encoded automatically)
kubectl create secret generic k8s-demo-secrets \
  --from-literal=SPRING_DATASOURCE_PASSWORD=mysecretpassword \
  --from-literal=JWT_SECRET=my-jwt-secret-key \
  --namespace=production

# Or via YAML (values must be manually base64-encoded)
# echo -n "mysecretpassword" | base64
```

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: k8s-demo-secrets
  namespace: production
type: Opaque
data:
  SPRING_DATASOURCE_PASSWORD: bXlzZWNyZXRwYXNzd29yZA==  # base64
  JWT_SECRET: bXktand0LXNlY3JldC1rZXk=

# For Docker registry credentials
---
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

### Reading ConfigMap as File in Spring Boot

```java
// Spring Boot auto-loads files from /app/config/ if using Spring Config
// Or explicitly configure:

// src/main/resources/application.yaml
spring:
  config:
    import: "optional:file:/app/config/application-k8s.yaml"
```

---

## 7. Horizontal Pod Autoscaler (HPA)

### Setting Up Metrics Server

```bash
# Install metrics-server (required for HPA CPU/memory scaling)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### CPU-Based HPA

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: k8s-demo-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: k8s-demo
  minReplicas: 2
  maxReplicas: 10
  metrics:
    # Scale on CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # Scale out when average CPU > 70%
    # Scale on memory utilization
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # Wait 60s before scaling up again
      policies:
        - type: Pods
          value: 2                     # Add at most 2 pods per period
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
      policies:
        - type: Percent
          value: 25                    # Remove at most 25% of pods
          periodSeconds: 60
```

### Custom Metrics HPA (with Prometheus Adapter)

```yaml
# Scale based on custom metric (e.g., HTTP request rate from Prometheus)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: k8s-demo-hpa-custom
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: k8s-demo
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"   # Scale when avg > 100 req/s per pod
```

---

## 8. Ingress with nginx Controller

### Install nginx Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# For local (minikube)
minikube addons enable ingress
```

### Ingress Resource

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: k8s-demo-ingress
  namespace: production
  annotations:
    # nginx annotations
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    # TLS certificate (cert-manager)
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.myapp.com
      secretName: api-myapp-tls   # cert-manager creates this
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: k8s-demo
                port:
                  number: 80
```

---

## 9. Readiness/Liveness Probes with Spring Actuator

### Spring Boot Actuator Configuration

```yaml
# src/main/resources/application.yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true   # Enables /actuator/health/liveness and /readiness
      group:
        liveness:
          include: livenessState,diskSpace
        readiness:
          include: readinessState,db,redis
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

### Custom Health Indicators

```java
// src/main/java/com/example/k8sdemo/health/DatabaseHealthIndicator.java
package com.example.k8sdemo.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    private final JdbcTemplate jdbcTemplate;

    public DatabaseHealthIndicator(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Health health() {
        try {
            Integer result = jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            if (result != null && result == 1) {
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "Reachable")
                    .build();
            }
            return Health.down().withDetail("message", "Unexpected result").build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

```java
// src/main/java/com/example/k8sdemo/health/ReadinessController.java
package com.example.k8sdemo.health;

import org.springframework.boot.availability.ApplicationAvailability;
import org.springframework.boot.availability.AvailabilityChangeEvent;
import org.springframework.boot.availability.ReadinessState;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Allows manual control of readiness state for graceful shutdown.
 */
@RestController
@RequestMapping("/internal")
public class ReadinessController {

    private final ApplicationEventPublisher eventPublisher;
    private final ApplicationAvailability availability;

    public ReadinessController(ApplicationEventPublisher eventPublisher,
                                ApplicationAvailability availability) {
        this.eventPublisher = eventPublisher;
        this.availability = availability;
    }

    @PostMapping("/readiness/refuse")
    public String refuseTraffic() {
        AvailabilityChangeEvent.publish(eventPublisher, this, ReadinessState.REFUSING_TRAFFIC);
        return "Application marked as not ready";
    }

    @PostMapping("/readiness/accept")
    public String acceptTraffic() {
        AvailabilityChangeEvent.publish(eventPublisher, this, ReadinessState.ACCEPTING_TRAFFIC);
        return "Application marked as ready";
    }
}
```

### Probe Configuration in Kubernetes

```yaml
containers:
  - name: k8s-demo
    # Startup probe: prevents liveness failures during slow startup
    startupProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
      failureThreshold: 30      # 30 * 10s = 5 minutes max startup time
      successThreshold: 1

    # Liveness probe: kills and restarts the container if it fails
    livenessProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
      initialDelaySeconds: 0    # startupProbe handles initial delay
      periodSeconds: 10
      failureThreshold: 3       # 3 consecutive failures = restart
      successThreshold: 1
      timeoutSeconds: 5

    # Readiness probe: removes pod from Service endpoints if it fails
    readinessProbe:
      httpGet:
        path: /actuator/health/readiness
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      failureThreshold: 3
      successThreshold: 1
      timeoutSeconds: 3
```

---

## 10. Rolling Updates and Rollbacks

### Performing a Rolling Update

```bash
# Update the image (triggers rolling update)
kubectl set image deployment/k8s-demo k8s-demo=myregistry/k8s-demo:2.0.0 -n production

# Watch the rollout
kubectl rollout status deployment/k8s-demo -n production

# View rollout history
kubectl rollout history deployment/k8s-demo -n production

# View specific revision details
kubectl rollout history deployment/k8s-demo --revision=3 -n production
```

### Rollback

```bash
# Rollback to previous version
kubectl rollout undo deployment/k8s-demo -n production

# Rollback to specific revision
kubectl rollout undo deployment/k8s-demo --to-revision=2 -n production
```

### Annotate Deployments for History

```yaml
spec:
  template:
    metadata:
      annotations:
        kubernetes.io/change-cause: "Deploy version 2.0.0: Add order tracking feature"
```

### Blue-Green Deployment Pattern

```yaml
# Blue (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: k8s-demo
      slot: blue
  template:
    metadata:
      labels:
        app: k8s-demo
        slot: blue
    spec:
      containers:
        - name: k8s-demo
          image: myregistry/k8s-demo:1.0.0

---
# Green (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: k8s-demo
      slot: green
  template:
    metadata:
      labels:
        app: k8s-demo
        slot: green
    spec:
      containers:
        - name: k8s-demo
          image: myregistry/k8s-demo:2.0.0

---
# Service: switch traffic by changing selector
apiVersion: v1
kind: Service
metadata:
  name: k8s-demo
spec:
  selector:
    app: k8s-demo
    slot: blue    # Change to "green" to switch traffic
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# Switch traffic to green
kubectl patch service k8s-demo -p '{"spec":{"selector":{"slot":"green"}}}'
```

---

## 11. Namespace Isolation

### Creating Namespaces

```yaml
# k8s/namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: prod
```

### ResourceQuota per Namespace

```yaml
# k8s/resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
```

### LimitRange (Default limits per pod)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
```

### NetworkPolicy (Pod-level firewall)

```yaml
# k8s/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: k8s-demo-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: k8s-demo
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow traffic from nginx ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
    # Allow traffic from other services in same namespace
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
  egress:
    # Allow access to database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Allow DNS
    - to: []
      ports:
        - protocol: UDP
          port: 53
```

---

## 12. Helm Chart Basics for Spring Boot

### Helm Chart Structure

```
my-spring-app/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default values
├── values-production.yaml  # Environment overrides
├── templates/
│   ├── _helpers.tpl        # Template helpers/functions
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   └── NOTES.txt
└── charts/                 # Sub-charts (dependencies)
```

### Chart.yaml

```yaml
# my-spring-app/Chart.yaml
apiVersion: v2
name: my-spring-app
description: A Helm chart for Spring Boot microservice
type: application
version: 1.0.0          # Chart version
appVersion: "2.0.0"     # Application version
keywords:
  - spring-boot
  - java
maintainers:
  - name: Dev Team
    email: dev@company.com
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

### values.yaml

```yaml
# my-spring-app/values.yaml
replicaCount: 2

image:
  repository: myregistry/k8s-demo
  pullPolicy: IfNotPresent
  tag: ""  # Overrides appVersion

nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.myapp.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.myapp.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  SPRING_PROFILES_ACTIVE: "production"
  APP_LOG_LEVEL: "INFO"

secrets:
  SPRING_DATASOURCE_PASSWORD: ""
  JWT_SECRET: ""

postgresql:
  enabled: true
  auth:
    database: appdb
    username: appuser

probes:
  liveness:
    path: /actuator/health/liveness
    initialDelaySeconds: 30
    periodSeconds: 10
  readiness:
    path: /actuator/health/readiness
    initialDelaySeconds: 20
    periodSeconds: 5
```

### templates/_helpers.tpl

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "my-spring-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-spring-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-spring-app.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ include "my-spring-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

### templates/deployment.yaml

```yaml
# my-spring-app/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-spring-app.fullname" . }}
  labels:
    {{- include "my-spring-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "my-spring-app.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        {{- include "my-spring-app.labels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          envFrom:
            - configMapRef:
                name: {{ include "my-spring-app.fullname" . }}-config
            - secretRef:
                name: {{ include "my-spring-app.fullname" . }}-secrets
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
```

### Helm Commands

```bash
# Install
helm install my-release ./my-spring-app \
  --namespace production \
  --create-namespace \
  --values values-production.yaml \
  --set image.tag=2.0.0

# Upgrade
helm upgrade my-release ./my-spring-app \
  --namespace production \
  --values values-production.yaml \
  --set image.tag=2.1.0

# Rollback
helm rollback my-release 1 --namespace production

# Dry run (see what would be applied)
helm upgrade --dry-run --debug my-release ./my-spring-app

# List releases
helm list --all-namespaces

# Uninstall
helm uninstall my-release --namespace production
```

---

## 13. Real Example: Deploy Microservices to Kubernetes

### Scenario: E-Commerce Platform with 3 Services

```
┌─────────────────────────────────────────────────────────────────┐
│                     KUBERNETES CLUSTER                          │
│                                                                 │
│  Ingress (nginx) ──► user-service:80                           │
│                 ──► order-service:80                           │
│                 ──► product-service:80                         │
│                                                                 │
│  order-service ──► user-service (ClusterIP)                    │
│  order-service ──► product-service (ClusterIP)                 │
│                                                                 │
│  All services ──► postgres (StatefulSet)                       │
│  All services ──► redis (StatefulSet)                          │
└─────────────────────────────────────────────────────────────────┘
```

### User Service

```java
// src/main/java/com/example/userservice/UserServiceApplication.java
package com.example.userservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

```java
// src/main/java/com/example/userservice/controller/UserController.java
package com.example.userservice.controller;

import com.example.userservice.model.User;
import com.example.userservice.service.UserService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }
}
```

### Complete Kubernetes Manifests

```yaml
# k8s/complete/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    app: ecommerce

---
# k8s/complete/postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: ecommerce
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: ecommerce
            - name: POSTGRES_USER
              value: ecomuser
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: ecommerce
spec:
  clusterIP: None   # Headless for StatefulSet
  selector:
    app: postgres
  ports:
    - port: 5432

---
# k8s/complete/user-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: myregistry/user-service:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_DATASOURCE_URL
              value: jdbc:postgresql://postgres:5432/ecommerce
            - name: SPRING_DATASOURCE_USERNAME
              value: ecomuser
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: ecommerce
spec:
  type: ClusterIP
  selector:
    app: user-service
  ports:
    - port: 80
      targetPort: 8080

---
# k8s/complete/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: api.ecommerce.com
      http:
        paths:
          - path: /users(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: user-service
                port:
                  number: 80
          - path: /orders(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /products(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: product-service
                port:
                  number: 80
```

### Deployment Script

```bash
#!/bin/bash
# deploy.sh
set -e

NAMESPACE=ecommerce
VERSION=$1

if [ -z "$VERSION" ]; then
  echo "Usage: ./deploy.sh <version>"
  exit 1
fi

echo "Deploying version $VERSION to namespace $NAMESPACE"

# Apply namespace and secrets first
kubectl apply -f k8s/complete/namespace.yaml
kubectl apply -f k8s/complete/secrets.yaml

# Deploy infrastructure
kubectl apply -f k8s/complete/postgres-statefulset.yaml
kubectl wait --for=condition=ready pod -l app=postgres -n $NAMESPACE --timeout=120s

# Deploy services with new image tag
for service in user-service order-service product-service; do
  kubectl set image deployment/$service $service=myregistry/$service:$VERSION -n $NAMESPACE
  kubectl rollout status deployment/$service -n $NAMESPACE --timeout=300s
done

# Apply ingress
kubectl apply -f k8s/complete/ingress.yaml

echo "Deployment complete!"
kubectl get pods -n $NAMESPACE
```

---

## Summary Table

| Topic | Command/Concept | Key Points |
|-------|----------------|------------|
| Architecture | Pod → Node → Cluster | Smallest unit is Pod |
| Deployment | `kubectl apply -f` | Declarative YAML |
| Service types | ClusterIP, NodePort, LoadBalancer | ClusterIP for internal |
| ConfigMap | Env vars or mounted files | Non-sensitive config |
| Secret | Base64-encoded values | Database passwords, API keys |
| HPA | autoscaling/v2 | CPU/memory/custom metrics |
| Ingress | nginx controller | SSL termination, routing |
| Probes | liveness, readiness, startup | Spring Actuator endpoints |
| Rolling update | `kubectl set image` | Zero-downtime deploys |
| Rollback | `kubectl rollout undo` | Quick recovery |
| Namespaces | Resource isolation | Per-environment separation |
| Helm | `helm install/upgrade` | Templated deployments |

---

## What's Next

**Part 036: RabbitMQ with Spring Boot** — Learn event-driven messaging with AMQP. We'll cover Exchange types, Dead Letter Queues, message acknowledgment, and build a full order-processing system with retry and DLQ patterns.

---

*End of Part 035: Kubernetes for Spring Boot*
