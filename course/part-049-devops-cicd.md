# Part 049: DevOps and CI/CD for Spring Boot

## Table of Contents
1. [GitHub Actions Workflows](#github-actions)
2. [GitLab CI/CD Pipeline](#gitlab-cicd)
3. [Jenkins Pipeline](#jenkins)
4. [SonarQube Static Analysis](#sonarqube)
5. [Code Coverage with JaCoCo](#jacoco)
6. [OWASP Dependency Check](#owasp)
7. [Docker Image Scanning](#docker-scanning)
8. [Environment-Specific Deployments](#environments)
9. [Blue/Green Deployment](#blue-green)
10. [Canary Deployment](#canary)
11. [Rollback Strategy](#rollback)
12. [CI Notifications](#notifications)
13. [Real Example: Complete GitHub Actions Pipeline](#real-example)
14. [Summary](#summary)

---

## 1. GitHub Actions Workflows {#github-actions}

### Project Structure

```
.github/
└── workflows/
    ├── ci.yml          # Build, test, analyze on every PR
    ├── cd-staging.yml  # Deploy to staging on merge to main
    ├── cd-prod.yml     # Deploy to production on release tag
    └── security.yml    # Weekly security scans
```

### Basic CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  JAVA_VERSION: "21"
  MAVEN_OPTS: "-Xmx1g -XX:+TieredCompilation -XX:TieredStopAtLevel=1"

jobs:
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for SonarQube blame

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Build with Maven
        run: mvn compile -q

      - name: Run unit tests
        run: |
          mvn test \
            -Dspring.profiles.active=test \
            -Dsurefire.failIfNoSpecifiedTests=false

      - name: Run integration tests
        run: |
          mvn verify \
            -Dspring.profiles.active=integration-test \
            -Dspring.datasource.url=jdbc:postgresql://localhost:5432/testdb \
            -Dspring.datasource.username=testuser \
            -Dspring.datasource.password=testpass \
            -Dspring.redis.host=localhost \
            -P integration-test
        env:
          SPRING_PROFILES_ACTIVE: integration-test

      - name: Generate JaCoCo report
        run: mvn jacoco:report

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: target/site/jacoco/jacoco.xml
          fail_ci_if_error: true

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: |
            target/surefire-reports/
            target/failsafe-reports/
          retention-days: 7

      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: target/surefire-reports/*.xml

  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: SonarQube Analysis
        run: |
          mvn sonar:sonar \
            -Dsonar.projectKey=my-spring-app \
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
            -Dsonar.login=${{ secrets.SONAR_TOKEN }} \
            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Check SonarQube Quality Gate
        uses: sonarsource/sonarqube-quality-gate-action@master
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: OWASP Dependency Check
        run: |
          mvn dependency-check:check \
            -DfailBuildOnCVSS=7 \
            -DsuppressionFile=owasp-suppressions.xml
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}

      - name: Upload OWASP Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: owasp-report
          path: target/dependency-check-report.html
          retention-days: 30

  docker-build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [build-and-test, code-quality]
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Extract metadata for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64

      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

---

## 2. GitLab CI/CD Pipeline {#gitlab-cicd}

```yaml
# .gitlab-ci.yml
image: eclipse-temurin:21-jdk

stages:
  - build
  - test
  - analyze
  - package
  - deploy-staging
  - smoke-test
  - deploy-prod

variables:
  MAVEN_OPTS: "-Xmx1g -Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  MAVEN_CLI_OPTS: "--batch-mode --no-transfer-progress"
  DOCKER_REGISTRY: registry.gitlab.com/$CI_PROJECT_PATH

cache:
  key: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"
  paths:
    - .m2/repository/
    - target/

# ──────────────── BUILD ────────────────
compile:
  stage: build
  script:
    - mvn $MAVEN_CLI_OPTS compile
  artifacts:
    paths:
      - target/classes/
    expire_in: 1 hour

# ──────────────── TEST ────────────────
unit-tests:
  stage: test
  services:
    - name: postgres:16-alpine
      alias: postgres
    - name: redis:7-alpine
      alias: redis
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
    SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/testdb
    SPRING_DATASOURCE_USERNAME: testuser
    SPRING_DATASOURCE_PASSWORD: testpass
    SPRING_REDIS_HOST: redis
  script:
    - mvn $MAVEN_CLI_OPTS test
  coverage: '/Total.*?([0-9]{1,3})%/'
  artifacts:
    when: always
    reports:
      junit: target/surefire-reports/TEST-*.xml
      coverage_report:
        coverage_format: cobertura
        path: target/site/jacoco/jacoco.xml
    paths:
      - target/site/jacoco/
    expire_in: 1 week

# ──────────────── ANALYZE ────────────────
sonarqube:
  stage: analyze
  script:
    - mvn $MAVEN_CLI_OPTS sonar:sonar
      -Dsonar.projectKey=$CI_PROJECT_PATH_SLUG
      -Dsonar.host.url=$SONAR_URL
      -Dsonar.login=$SONAR_TOKEN
      -Dsonar.gitlab.project_id=$CI_PROJECT_ID
      -Dsonar.gitlab.commit_sha=$CI_COMMIT_SHA
      -Dsonar.gitlab.ref_name=$CI_COMMIT_REF_NAME
  allow_failure: false
  rules:
    - if: $CI_MERGE_REQUEST_ID
    - if: $CI_COMMIT_BRANCH == "main"

owasp-check:
  stage: analyze
  script:
    - mvn $MAVEN_CLI_OPTS dependency-check:check
      -DfailBuildOnCVSS=7
  artifacts:
    when: always
    paths:
      - target/dependency-check-report.html
    expire_in: 1 month

# ──────────────── PACKAGE ────────────────
docker-build:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    IMAGE_TAG: $DOCKER_REGISTRY:$CI_COMMIT_SHORT_SHA
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build --platform linux/amd64 -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
    - docker tag $IMAGE_TAG $DOCKER_REGISTRY:latest
    - docker push $DOCKER_REGISTRY:latest
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_TAG

trivy-scan:
  stage: package
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image
      --format template
      --template "@/contrib/gitlab.tpl"
      --output gl-container-scanning-report.json
      --severity CRITICAL,HIGH
      --exit-code 1
      $DOCKER_REGISTRY:$CI_COMMIT_SHORT_SHA
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json

# ──────────────── STAGING ────────────────
deploy-staging:
  stage: deploy-staging
  image: google/cloud-sdk:alpine
  environment:
    name: staging
    url: https://staging.myapp.com
  script:
    - gcloud auth activate-service-account --key-file=$GCP_SERVICE_KEY
    - gcloud config set project $GCP_PROJECT_ID
    - gcloud run deploy my-app-staging
      --image $DOCKER_REGISTRY:$CI_COMMIT_SHORT_SHA
      --region us-central1
      --platform managed
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  needs: [docker-build, trivy-scan]

smoke-tests:
  stage: smoke-test
  script:
    - |
      STAGING_URL="https://staging.myapp.com"
      curl -sf "${STAGING_URL}/actuator/health" || exit 1
      curl -sf "${STAGING_URL}/actuator/info" || exit 1
      echo "All smoke tests passed"
  needs: [deploy-staging]

# ──────────────── PRODUCTION ────────────────
deploy-prod:
  stage: deploy-prod
  image: google/cloud-sdk:alpine
  environment:
    name: production
    url: https://myapp.com
  script:
    - gcloud auth activate-service-account --key-file=$GCP_SERVICE_KEY
    - gcloud run deploy my-app
      --image $DOCKER_REGISTRY:$CI_COMMIT_SHORT_SHA
      --region us-central1
      --platform managed
  when: manual
  rules:
    - if: $CI_COMMIT_TAG
  needs: [smoke-tests]
```

---

## 3. Jenkins Pipeline {#jenkins}

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: eclipse-temurin:21-jdk
    command: ['sleep', 'infinity']
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "1"
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
    volumeMounts:
    - name: docker-graph-storage
      mountPath: /var/lib/docker
  volumes:
  - name: docker-graph-storage
    emptyDir: {}
"""
        }
    }

    environment {
        MAVEN_OPTS = '-Xmx1g -XX:+TieredCompilation -XX:TieredStopAtLevel=1'
        REGISTRY   = 'my-registry.example.com'
        IMAGE_NAME = "${REGISTRY}/my-spring-app"
        SONAR_URL  = credentials('sonar-url')
        SONAR_TOKEN = credentials('sonar-token')
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        ansiColor('xterm')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_MSG = sh(
                        script: 'git log -1 --pretty=%B',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn compile -q --no-transfer-progress'
                }
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        container('maven') {
                            sh '''
                                mvn test \
                                    --no-transfer-progress \
                                    -Dspring.profiles.active=test
                            '''
                        }
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                            jacoco(
                                execPattern: 'target/**/*.exec',
                                classPattern: 'target/classes',
                                sourcePattern: 'src/main/java',
                                minimumLineCoverage: '70'
                            )
                        }
                    }
                }

                stage('OWASP Check') {
                    steps {
                        container('maven') {
                            sh '''
                                mvn dependency-check:check \
                                    --no-transfer-progress \
                                    -DfailBuildOnCVSS=8
                            '''
                        }
                    }
                    post {
                        always {
                            publishHTML(target: [
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'target',
                                reportFiles: 'dependency-check-report.html',
                                reportName: 'OWASP Report'
                            ])
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    sh """
                        mvn sonar:sonar \
                            --no-transfer-progress \
                            -Dsonar.projectKey=my-spring-app \
                            -Dsonar.host.url=${SONAR_URL} \
                            -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Package') {
            steps {
                container('maven') {
                    sh 'mvn package -DskipTests -q --no-transfer-progress'
                }
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Docker Build & Push') {
            when {
                anyOf {
                    branch 'main'
                    tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
                }
            }
            steps {
                container('docker') {
                    script {
                        def imageTag = "${IMAGE_NAME}:${env.GIT_COMMIT[0..7]}"
                        sh """
                            docker build -t ${imageTag} .
                            docker push ${imageTag}
                            docker tag ${imageTag} ${IMAGE_NAME}:latest
                            docker push ${IMAGE_NAME}:latest
                        """
                        env.DOCKER_TAG = imageTag
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            when { branch 'main' }
            steps {
                script {
                    echo "Deploying ${env.DOCKER_TAG} to staging..."
                    // Add your deployment commands
                }
            }
        }

        stage('Deploy to Production') {
            when { tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP' }
            input {
                message "Deploy ${TAG_NAME} to production?"
                ok "Deploy"
                submitter "release-manager"
            }
            steps {
                script {
                    echo "Deploying ${TAG_NAME} to production..."
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: ":white_check_mark: *${env.JOB_NAME}* build #${env.BUILD_NUMBER} succeeded\n${env.GIT_COMMIT_MSG}\n<${env.BUILD_URL}|View Build>"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: ":x: *${env.JOB_NAME}* build #${env.BUILD_NUMBER} FAILED\n${env.GIT_COMMIT_MSG}\n<${env.BUILD_URL}|View Build>"
            )
            emailext(
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build failed. <a href='${env.BUILD_URL}'>View build</a>",
                to: "${env.CHANGE_AUTHOR_EMAIL}",
                mimeType: 'text/html'
            )
        }
    }
}
```

---

## 4. SonarQube Static Analysis {#sonarqube}

### Maven Plugin Configuration

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>4.0.0.4121</version>
</plugin>

<properties>
    <!-- SonarQube configuration -->
    <sonar.projectKey>com.example:my-spring-app</sonar.projectKey>
    <sonar.projectName>My Spring App</sonar.projectName>
    <sonar.language>java</sonar.language>
    <sonar.java.source>21</sonar.java.source>

    <!-- Coverage -->
    <sonar.coverage.jacoco.xmlReportPaths>
        ${project.build.directory}/site/jacoco/jacoco.xml
    </sonar.coverage.jacoco.xmlReportPaths>

    <!-- Exclusions -->
    <sonar.exclusions>
        **/generated/**,
        **/dto/**,
        **/*Application.java,
        **/config/**
    </sonar.exclusions>
    <sonar.coverage.exclusions>
        **/dto/**,
        **/entity/**,
        **/*Application.java
    </sonar.coverage.exclusions>
    <sonar.test.exclusions>
        src/test/**
    </sonar.test.exclusions>

    <!-- Quality gate -->
    <sonar.qualitygate.wait>true</sonar.qualitygate.wait>
    <sonar.qualitygate.timeout>300</sonar.qualitygate.timeout>
</properties>
```

### sonar-project.properties

```properties
# sonar-project.properties
sonar.projectKey=my-spring-app
sonar.projectName=My Spring Application
sonar.projectVersion=1.0

sonar.sources=src/main/java
sonar.tests=src/test/java
sonar.java.binaries=target/classes
sonar.java.test.binaries=target/test-classes

sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

# Code quality thresholds (enforced by Quality Gate in SonarQube UI)
sonar.issue.ignore.multicriteria=e1,e2
sonar.issue.ignore.multicriteria.e1.ruleKey=java:S1192
sonar.issue.ignore.multicriteria.e1.resourceKey=**/*Constants.java
sonar.issue.ignore.multicriteria.e2.ruleKey=java:S2699
sonar.issue.ignore.multicriteria.e2.resourceKey=**/integration/**/*Test.java
```

---

## 5. Code Coverage with JaCoCo {#jacoco}

### JaCoCo Maven Plugin

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <!-- Initialize agent before tests -->
        <execution>
            <id>prepare-agent</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>

        <!-- Generate report after tests -->
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>

        <!-- Enforce minimum coverage -->
        <execution>
            <id>check</id>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.70</minimum>
                            </limit>
                            <limit>
                                <counter>BRANCH</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.60</minimum>
                            </limit>
                        </limits>
                    </rule>
                    <rule>
                        <!-- Per-class minimum -->
                        <element>CLASS</element>
                        <excludes>
                            <exclude>**/dto/**</exclude>
                            <exclude>**/entity/**</exclude>
                            <exclude>**/*Application</exclude>
                        </excludes>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.60</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
                <excludes>
                    <exclude>**/dto/**</exclude>
                    <exclude>**/entity/**</exclude>
                    <exclude>**/*Application.class</exclude>
                    <exclude>**/config/**</exclude>
                </excludes>
            </configuration>
        </execution>

        <!-- For integration tests -->
        <execution>
            <id>prepare-agent-integration</id>
            <goals><goal>prepare-agent-integration</goal></goals>
        </execution>
        <execution>
            <id>report-integration</id>
            <phase>post-integration-test</phase>
            <goals><goal>report-integration</goal></goals>
        </execution>

        <!-- Merge unit + integration test coverage -->
        <execution>
            <id>merge-results</id>
            <phase>verify</phase>
            <goals><goal>merge</goal></goals>
            <configuration>
                <fileSets>
                    <fileSet>
                        <directory>${project.build.directory}</directory>
                        <includes><include>*.exec</include></includes>
                    </fileSet>
                </fileSets>
                <destFile>${project.build.directory}/merged.exec</destFile>
            </configuration>
        </execution>
        <execution>
            <id>merged-report</id>
            <phase>verify</phase>
            <goals><goal>report</goal></goals>
            <configuration>
                <dataFile>${project.build.directory}/merged.exec</dataFile>
                <outputDirectory>${project.reporting.outputDirectory}/jacoco-merged</outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
```

---

## 6. OWASP Dependency Check {#owasp}

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.0.9</version>
    <configuration>
        <!-- Fail if any dependency has CVSS >= 7 (High/Critical) -->
        <failBuildOnCVSS>7</failBuildOnCVSS>

        <!-- Use NVD API key to avoid rate limiting -->
        <nvdApiKey>${env.NVD_API_KEY}</nvdApiKey>

        <!-- Report formats -->
        <formats>
            <format>HTML</format>
            <format>JSON</format>
            <format>SARIF</format>
        </formats>

        <!-- Suppression file for known false positives -->
        <suppressionFile>owasp-suppressions.xml</suppressionFile>

        <!-- Skip test-scoped dependencies -->
        <skipTestScope>false</skipTestScope>

        <!-- Analyzers -->
        <ossindexAnalyzerEnabled>true</ossindexAnalyzerEnabled>
        <retireJsAnalyzerEnabled>false</retireJsAnalyzerEnabled>
        <nodeAnalyzerEnabled>false</nodeAnalyzerEnabled>
    </configuration>
    <executions>
        <execution>
            <goals><goal>check</goal></goals>
        </execution>
    </executions>
</plugin>
```

### OWASP Suppressions File

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- owasp-suppressions.xml -->
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">

    <!-- Example: Suppress a known false positive -->
    <suppress>
        <notes>
            CVE-2023-12345 does not affect this version of the library
            in our configuration. Reviewed by security team on 2024-01-15.
        </notes>
        <cve>CVE-2023-12345</cve>
    </suppress>

    <!-- Suppress for specific package -->
    <suppress>
        <notes>Test-only dependency, not exposed in production</notes>
        <packageUrl regex="true">^pkg:maven/org\.springframework\.boot/spring-boot-test.*</packageUrl>
        <cve>CVE-2023-99999</cve>
    </suppress>

</suppressions>
```

---

## 7. Docker Image Scanning in CI {#docker-scanning}

### Trivy Integration in GitHub Actions

```yaml
# .github/workflows/security.yml
name: Security Scans

on:
  schedule:
    - cron: '0 2 * * 1'  # Weekly on Monday at 2 AM
  push:
    branches: [main]

jobs:
  trivy-scan:
    name: Trivy Container Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image for scanning
        run: docker build -t scan-target:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: scan-target:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          ignore-unfixed: true
          exit-code: 1

      - name: Upload results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

  grype-scan:
    name: Grype Container Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Scan with Grype
        uses: anchore/scan-action@v3
        with:
          path: "."
          fail-build: true
          severity-cutoff: high
          acs-report-enable: true

      - name: Upload Anchore report
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: results.sarif

  secret-scan:
    name: Secret Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Scan for secrets with Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 8. Environment-Specific Deployments {#environments}

### Spring Boot Profile per Environment

```yaml
# src/main/resources/application.yml (base)
spring:
  application:
    name: my-spring-app
  jpa:
    open-in-view: false

server:
  port: 8080

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

---
# Dev profile
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:postgresql://localhost:5432/devdb
    username: devuser
    password: devpass
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: update
logging:
  level:
    com.example: DEBUG

---
# Staging profile
spring:
  config:
    activate:
      on-profile: staging
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
logging:
  level:
    com.example: INFO

---
# Production profile
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
  jpa:
    hibernate:
      ddl-auto: validate
  redis:
    host: ${REDIS_HOST}
    password: ${REDIS_PASSWORD}
logging:
  level:
    root: WARN
    com.example: INFO
```

### GitHub Actions Environment Deployment

```yaml
# .github/workflows/cd-staging.yml
name: Deploy Staging

on:
  push:
    branches: [main]

jobs:
  deploy:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          # Example: deploy to k8s
          echo "${{ secrets.KUBE_CONFIG }}" > kubeconfig.yaml
          kubectl --kubeconfig=kubeconfig.yaml set image \
            deployment/my-app \
            my-app=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --namespace=staging

      - name: Wait for rollout
        run: |
          kubectl --kubeconfig=kubeconfig.yaml rollout status \
            deployment/my-app --namespace=staging --timeout=300s

      - name: Smoke tests
        run: |
          STAGING_URL="https://staging.myapp.com"
          for i in $(seq 1 5); do
            STATUS=$(curl -sf -o /dev/null -w "%{http_code}" \
              "${STAGING_URL}/actuator/health")
            if [ "$STATUS" = "200" ]; then
              echo "Health check passed"
              exit 0
            fi
            echo "Attempt $i failed (status=$STATUS), retrying..."
            sleep 10
          done
          echo "Health check failed after 5 attempts"
          exit 1
```

---

## 9. Blue/Green Deployment {#blue-green}

### Blue/Green on Kubernetes

```yaml
# k8s/blue-green.yaml
# Blue deployment (currently live)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
  namespace: production
  labels:
    app: my-app
    slot: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      slot: blue
  template:
    metadata:
      labels:
        app: my-app
        slot: blue
    spec:
      containers:
        - name: my-app
          image: my-registry/my-app:v1.0.0
          ports:
            - containerPort: 8080
---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
  namespace: production
  labels:
    app: my-app
    slot: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      slot: green
  template:
    metadata:
      labels:
        app: my-app
        slot: green
    spec:
      containers:
        - name: my-app
          image: my-registry/my-app:v2.0.0
          ports:
            - containerPort: 8080
---
# Service points to active slot
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
spec:
  selector:
    app: my-app
    slot: blue    # ← Change to "green" to switch traffic
  ports:
    - port: 80
      targetPort: 8080
```

### Blue/Green Switch Script

```bash
#!/bin/bash
# blue-green-switch.sh

set -e

NAMESPACE="production"
SERVICE_NAME="my-app"
NEW_IMAGE="$1"

if [ -z "$NEW_IMAGE" ]; then
    echo "Usage: $0 <new-image>"
    exit 1
fi

# Determine current and new slots
CURRENT_SLOT=$(kubectl get service "$SERVICE_NAME" -n "$NAMESPACE" \
    -o jsonpath='{.spec.selector.slot}')

if [ "$CURRENT_SLOT" = "blue" ]; then
    NEW_SLOT="green"
else
    NEW_SLOT="blue"
fi

echo "Current slot: $CURRENT_SLOT, New slot: $NEW_SLOT"

# Update new slot deployment with new image
echo "Deploying ${NEW_IMAGE} to ${NEW_SLOT}..."
kubectl set image deployment/"$SERVICE_NAME-$NEW_SLOT" \
    "$SERVICE_NAME=$NEW_IMAGE" \
    -n "$NAMESPACE"

# Wait for new slot to be ready
echo "Waiting for ${NEW_SLOT} to be ready..."
kubectl rollout status deployment/"$SERVICE_NAME-$NEW_SLOT" \
    -n "$NAMESPACE" --timeout=300s

# Run smoke tests against new slot before switching
echo "Running smoke tests on new slot..."
NEW_SLOT_IP=$(kubectl get pod -n "$NAMESPACE" \
    -l "app=$SERVICE_NAME,slot=$NEW_SLOT" \
    -o jsonpath='{.items[0].status.podIP}')

if ! curl -sf "http://${NEW_SLOT_IP}:8080/actuator/health"; then
    echo "Smoke test FAILED on new slot. Aborting switch."
    exit 1
fi

# Switch service to new slot
echo "Switching traffic to ${NEW_SLOT}..."
kubectl patch service "$SERVICE_NAME" -n "$NAMESPACE" \
    -p "{\"spec\":{\"selector\":{\"slot\":\"${NEW_SLOT}\"}}}"

echo "Traffic switched to ${NEW_SLOT}"
echo "Previous slot ($CURRENT_SLOT) is still running for instant rollback"
echo "To rollback: kubectl patch service $SERVICE_NAME -n $NAMESPACE -p '{\"spec\":{\"selector\":{\"slot\":\"$CURRENT_SLOT\"}}}'"
```

---

## 10. Canary Deployment {#canary}

### Canary with Nginx Ingress

```yaml
# k8s/canary.yaml
# Stable deployment (95% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-stable
  namespace: production
spec:
  replicas: 9  # 9 pods = 90% of traffic (weight-based)
  selector:
    matchLabels:
      app: my-app
      track: stable
  template:
    metadata:
      labels:
        app: my-app
        track: stable
    spec:
      containers:
        - name: my-app
          image: my-registry/my-app:v1.0.0
          ports:
            - containerPort: 8080
---
# Canary deployment (5% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
  namespace: production
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 1  # 1 pod = ~10% of traffic
  selector:
    matchLabels:
      app: my-app
      track: canary
  template:
    metadata:
      labels:
        app: my-app
        track: canary
    spec:
      containers:
        - name: my-app
          image: my-registry/my-app:v2.0.0
          ports:
            - containerPort: 8080
---
# Main service routes to both stable and canary via shared label
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
spec:
  selector:
    app: my-app   # Both deployments match this
  ports:
    - port: 80
      targetPort: 8080
---
# Nginx Ingress canary based on HTTP header
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-canary
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
    nginx.ingress.kubernetes.io/canary-weight: "5"
spec:
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-canary
                port:
                  number: 80
```

---

## 11. Rollback Strategy {#rollback}

### Kubernetes Rollback

```bash
#!/bin/bash
# rollback.sh

NAMESPACE="production"
DEPLOYMENT="my-app"

echo "=== Deployment History ==="
kubectl rollout history deployment/"$DEPLOYMENT" -n "$NAMESPACE"

echo ""
read -p "Enter revision to rollback to (or press Enter for previous): " REVISION

if [ -z "$REVISION" ]; then
    echo "Rolling back to previous revision..."
    kubectl rollout undo deployment/"$DEPLOYMENT" -n "$NAMESPACE"
else
    echo "Rolling back to revision $REVISION..."
    kubectl rollout undo deployment/"$DEPLOYMENT" \
        -n "$NAMESPACE" \
        --to-revision="$REVISION"
fi

echo "Waiting for rollback to complete..."
kubectl rollout status deployment/"$DEPLOYMENT" \
    -n "$NAMESPACE" --timeout=120s

echo "Rollback complete!"
kubectl get pods -n "$NAMESPACE" -l "app=$DEPLOYMENT"
```

### Health Check Integration for Rollback

```java
package com.example.cicd.health;

import lombok.RequiredArgsConstructor;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

/**
 * Custom health indicators used for deployment gates
 * - /actuator/health/liveness → Kubernetes liveness probe
 * - /actuator/health/readiness → Kubernetes readiness probe
 */
@Component("database")
@RequiredArgsConstructor
public class DatabaseHealthIndicator implements HealthIndicator {

    private final JdbcTemplate jdbcTemplate;

    @Override
    public Health health() {
        try {
            jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "reachable")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

```yaml
# application.yml health configuration
management:
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState,diskSpace
        readiness:
          include: readinessState,database,redis
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

---

## 12. Slack/Email Notifications from CI {#notifications}

### Slack Notification Action

```yaml
# Reusable notification step in GitHub Actions
- name: Send Slack notification on success
  if: success()
  uses: slackapi/slack-github-action@v1.25.0
  with:
    channel-id: 'C01234ABCDE'
    slack-message: |
      :white_check_mark: *Deployment Successful*
      *App:* `${{ github.repository }}`
      *Environment:* `${{ env.ENVIRONMENT }}`
      *Version:* `${{ github.sha }}`
      *Deployed by:* `${{ github.actor }}`
      *Branch:* `${{ github.ref_name }}`
      <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>
  env:
    SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

- name: Send Slack notification on failure
  if: failure()
  uses: slackapi/slack-github-action@v1.25.0
  with:
    channel-id: 'C01234ABCDE'
    slack-message: |
      :x: *Deployment FAILED*
      *App:* `${{ github.repository }}`
      *Environment:* `${{ env.ENVIRONMENT }}`
      *Failed step:* `${{ github.job }}`
      *Triggered by:* `${{ github.actor }}`
      <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>
  env:
    SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### Email Notification via Spring

```java
package com.example.cicd.notification;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.thymeleaf.TemplateEngine;
import org.thymeleaf.context.Context;

import jakarta.mail.MessagingException;
import jakarta.mail.internet.MimeMessage;

@Slf4j
@Service
@RequiredArgsConstructor
public class EmailNotificationService {

    private final JavaMailSender mailSender;
    private final TemplateEngine templateEngine;

    @Async
    public void sendDeploymentNotification(String to, String environment,
                                            String version, boolean success) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");

            helper.setTo(to);
            helper.setSubject(String.format("[%s] Deployment to %s %s",
                    success ? "SUCCESS" : "FAILURE", environment,
                    success ? "succeeded" : "FAILED"));

            Context context = new Context();
            context.setVariable("environment", environment);
            context.setVariable("version", version);
            context.setVariable("success", success);
            context.setVariable("timestamp", java.time.Instant.now());

            String html = templateEngine.process("deployment-notification", context);
            helper.setText(html, true);

            mailSender.send(message);
            log.info("Deployment notification sent to {}", to);

        } catch (MessagingException e) {
            log.error("Failed to send deployment notification", e);
        }
    }
}
```

---

## 13. Real Example: Complete GitHub Actions Pipeline from Build to Prod {#real-example}

```yaml
# .github/workflows/full-pipeline.yml
name: Full Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  release:
    types: [published]

env:
  JAVA_VERSION: "21"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ==================== 1. BUILD & TEST ====================
  build-test:
    name: Build & Test
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5

    outputs:
      version: ${{ steps.version.outputs.version }}

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set version
        id: version
        run: |
          if [[ "${{ github.event_name }}" == "release" ]]; then
            VERSION="${{ github.event.release.tag_name }}"
          else
            VERSION="${{ github.sha }}"
          fi
          echo "version=$VERSION" >> $GITHUB_OUTPUT

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: Run all tests with coverage
        run: |
          mvn verify \
            -Dspring.profiles.active=test \
            -Dspring.datasource.url=jdbc:postgresql://localhost:5432/testdb \
            -Dspring.datasource.username=testuser \
            -Dspring.datasource.password=testpass \
            --no-transfer-progress

      - name: Upload test artifacts
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ github.run_id }}
          path: |
            target/surefire-reports/
            target/site/jacoco/
          retention-days: 7

      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: jar-${{ github.run_id }}
          path: target/*.jar
          retention-days: 1

  # ==================== 2. SECURITY ANALYSIS ====================
  security:
    name: Security Analysis
    runs-on: ubuntu-latest
    needs: build-test
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: OWASP Dependency Check
        run: |
          mvn dependency-check:check -DfailBuildOnCVSS=8 --no-transfer-progress
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}
        continue-on-error: false

      - name: SonarQube Analysis
        run: |
          mvn sonar:sonar \
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
            -Dsonar.login=${{ secrets.SONAR_TOKEN }} \
            --no-transfer-progress
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # ==================== 3. BUILD DOCKER IMAGE ====================
  docker:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: [build-test, security]
    if: github.event_name != 'pull_request'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Download JAR
        uses: actions/download-artifact@v4
        with:
          name: jar-${{ github.run_id }}
          path: target/

      - uses: docker/setup-buildx-action@v3

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,format=long
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: sarif
          output: trivy.sarif
          severity: CRITICAL,HIGH
          exit-code: 1

      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy.sarif

  # ==================== 4. DEPLOY STAGING ====================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: docker
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.myapp.com

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          IMAGE="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}"
          echo "Deploying $IMAGE to staging"
          # Add actual deployment command here

      - name: Smoke tests
        run: |
          sleep 30  # Wait for service to start
          for i in $(seq 1 10); do
            STATUS=$(curl -sf -o /dev/null -w "%{http_code}" \
              "https://staging.myapp.com/actuator/health" || echo "000")
            if [ "$STATUS" = "200" ]; then
              echo "Health check passed"
              exit 0
            fi
            echo "Attempt $i: status=$STATUS, waiting..."
            sleep 10
          done
          echo "Health check failed"
          exit 1

      - name: Run integration tests against staging
        run: |
          mvn failsafe:integration-test failsafe:verify \
            -Dtest.base-url=https://staging.myapp.com \
            --no-transfer-progress

  # ==================== 5. DEPLOY PRODUCTION ====================
  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.event_name == 'release'
    environment:
      name: production
      url: https://myapp.com

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to production
        run: |
          IMAGE="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.event.release.tag_name }}"
          echo "Deploying $IMAGE to production"

      - name: Notify Slack - Success
        if: success()
        uses: slackapi/slack-github-action@v1.25.0
        with:
          channel-id: ${{ secrets.SLACK_DEPLOY_CHANNEL }}
          slack-message: |
            :rocket: *Production Deployment Successful*
            Version: `${{ github.event.release.tag_name }}`
            By: `${{ github.actor }}`
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

      - name: Notify Slack - Failure
        if: failure()
        uses: slackapi/slack-github-action@v1.25.0
        with:
          channel-id: ${{ secrets.SLACK_DEPLOY_CHANNEL }}
          slack-message: |
            :fire: *Production Deployment FAILED*
            Version: `${{ github.event.release.tag_name }}`
            By: `${{ github.actor }}`
            <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Details>
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## 14. Summary {#summary}

| Tool / Concept | Purpose | Key Config |
|---------------|---------|-----------|
| **GitHub Actions** | Cloud-native CI/CD; free for public repos | `.github/workflows/` YAML files |
| **GitLab CI** | Built-in CI/CD with review apps | `.gitlab-ci.yml` stages/jobs |
| **Jenkins** | Self-hosted; declarative Jenkinsfile | Kubernetes agents; parallel stages |
| **SonarQube** | Static analysis; quality gates | `sonar.qualitygate.wait=true` |
| **JaCoCo** | Code coverage reporting and enforcement | `minimumLineCoverage`, `check` goal |
| **OWASP** | Known CVE scanning in dependencies | `failBuildOnCVSS=7` for high severity |
| **Trivy / Grype** | Container image vulnerability scanning | Exit code 1 on CRITICAL/HIGH |
| **Blue/Green** | Zero-downtime deployment with instant rollback | Service selector switch |
| **Canary** | Gradual traffic shift to new version | Weight-based or header-based routing |
| **Rollback** | `kubectl rollout undo` or slot switch | Always keep N-1 version running |
| **Notifications** | Slack + email on CI events | Slack bot token; JavaMailSender |

### Pipeline Duration Targets

| Stage | Target Duration |
|-------|----------------|
| Build | < 2 minutes |
| Unit Tests | < 3 minutes |
| Docker Build | < 3 minutes |
| Security Scan | < 5 minutes |
| Staging Deploy + Smoke | < 5 minutes |
| **Total to Staging** | **< 15 minutes** |

---

> **Next: Part 050 - API Security Best Practices** — OWASP API Top 10, rate limiting, request signing, input validation, SQL injection prevention, and a hardened REST API example.
