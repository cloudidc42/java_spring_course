# Part 020: Maven & Gradle - Build Tools

## เนื้อหาในส่วนนี้
- Maven: POM, Lifecycle, Plugins, Dependencies
- Maven Multi-module Projects
- Gradle: Build Scripts, Tasks, Plugins
- Gradle Kotlin DSL
- Dependency Management และ Version Catalogs
- Publishing Artifacts
- CI/CD Integration

---

## 1. Maven Fundamentals

### Project Object Model (POM)

```xml
<!-- pom.xml - Complete structure -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    
    <!-- Project Coordinates (GAV) -->
    <groupId>com.example</groupId>
    <artifactId>my-application</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    
    <!-- Project Metadata -->
    <name>My Application</name>
    <description>A sample Java application</description>
    <url>https://example.com/my-app</url>
    
    <!-- Project Properties -->
    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        
        <!-- Dependency versions as properties -->
        <spring.version>6.1.1</spring.version>
        <junit.version>5.10.1</junit.version>
        <mockito.version>5.7.0</mockito.version>
    </properties>
    
    <!-- Dependencies -->
    <dependencies>
        <!-- Compile scope (default) -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring.version}</version>
        </dependency>
        
        <!-- Test scope - only for testing -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
        
        <!-- Provided scope - available at compile, not packaged -->
        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version>
            <scope>provided</scope>
        </dependency>
        
        <!-- Runtime scope - not needed for compile, needed for runtime -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.2.0</version>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Optional dependency -->
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>32.1.3-jre</version>
            <optional>true</optional>
        </dependency>
        
        <!-- Exclude transitive dependency -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>${spring.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>commons-logging</groupId>
                    <artifactId>commons-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
    
    <!-- Build Configuration -->
    <build>
        <sourceDirectory>src/main/java</sourceDirectory>
        <testSourceDirectory>src/test/java</testSourceDirectory>
        
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>  <!-- Replace ${property} in resource files -->
            </resource>
        </resources>
        
        <plugins>
            <!-- Compiler plugin -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>21</source>
                    <target>21</target>
                    <compilerArgs>
                        <arg>--enable-preview</arg>
                    </compilerArgs>
                </configuration>
            </plugin>
            
            <!-- Surefire - run unit tests -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.1.2</version>
                <configuration>
                    <includes>
                        <include>**/*Test.java</include>
                        <include>**/*Tests.java</include>
                        <include>**/*TestCase.java</include>
                    </includes>
                    <excludes>
                        <exclude>**/*IntegrationTest.java</exclude>
                    </excludes>
                </configuration>
            </plugin>
            
            <!-- Failsafe - run integration tests -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>3.1.2</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>integration-test</goal>
                            <goal>verify</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            
            <!-- JAR plugin - create executable JAR -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.3.0</version>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>com.example.Main</mainClass>
                            <addClasspath>true</addClasspath>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
            
            <!-- Assembly plugin - fat JAR with all dependencies -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.6.0</version>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>com.example.Main</mainClass>
                        </manifest>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
    
    <!-- Profiles - different configurations per environment -->
    <profiles>
        <profile>
            <id>development</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <db.url>jdbc:h2:mem:devdb</db.url>
            </properties>
        </profile>
        
        <profile>
            <id>production</id>
            <activation>
                <property>
                    <name>env</name>
                    <value>prod</value>
                </property>
            </activation>
            <properties>
                <db.url>jdbc:mysql://prod-host:3306/mydb</db.url>
            </properties>
        </profile>
        
        <profile>
            <id>integration-tests</id>
            <build>
                <plugins>
                    <plugin>
                        <artifactId>maven-failsafe-plugin</artifactId>
                        <executions>
                            <execution>
                                <goals>
                                    <goal>integration-test</goal>
                                    <goal>verify</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
    
    <!-- Distribution Management -->
    <distributionManagement>
        <repository>
            <id>releases</id>
            <url>https://nexus.example.com/repository/releases/</url>
        </repository>
        <snapshotRepository>
            <id>snapshots</id>
            <url>https://nexus.example.com/repository/snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
    
</project>
```

---

## 2. Maven Build Lifecycle

```
Maven has 3 built-in lifecycles:

DEFAULT LIFECYCLE (most common):
────────────────────────────────
validate      → Validate project is correct
compile       → Compile source code
test-compile  → Compile test sources
test          → Run unit tests
package       → Package compiled code (JAR/WAR)
verify        → Run integration tests
install       → Install to local repository (~/.m2)
deploy        → Deploy to remote repository

CLEAN LIFECYCLE:
────────────────
pre-clean     → Before clean
clean         → Delete target/ directory
post-clean    → After clean

SITE LIFECYCLE:
───────────────
site          → Generate project site documentation
site-deploy   → Deploy site to server
```

```bash
# Common Maven commands
mvn compile                 # Compile only
mvn test                    # Compile + test
mvn package                 # Compile + test + package
mvn install                 # package + install to ~/.m2
mvn deploy                  # install + deploy to remote

mvn clean package           # Clean then package
mvn clean install -DskipTests  # Skip tests

# Run specific test
mvn test -Dtest=MyServiceTest
mvn test -Dtest=MyServiceTest#myTestMethod

# Run with profile
mvn package -Pproduction
mvn package -Pintegration-tests

# Display dependency tree
mvn dependency:tree

# Show effective POM (after inheritance resolution)
mvn help:effective-pom

# Update all dependencies to latest
mvn versions:use-latest-versions

# Check for dependency updates
mvn versions:display-dependency-updates
```

---

## 3. Maven Dependency Management

```xml
<!-- Parent POM - for dependency version management -->
<project>
    <groupId>com.example</groupId>
    <artifactId>my-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <!-- dependencyManagement - declare versions without adding deps -->
    <dependencyManagement>
        <dependencies>
            <!-- Import BOM (Bill of Materials) -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.2.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            
            <!-- After importing BOM, child modules don't need versions -->
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>my-common</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <!-- Modules -->
    <modules>
        <module>my-common</module>
        <module>my-service</module>
        <module>my-web</module>
    </modules>
</project>

<!-- Child POM - inherits from parent -->
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>my-parent</artifactId>
        <version>1.0.0</version>
    </parent>
    
    <artifactId>my-service</artifactId>
    
    <dependencies>
        <!-- Version managed by parent's dependencyManagement -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <!-- No version needed - inherited from BOM -->
        </dependency>
        
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>my-common</artifactId>
            <!-- No version needed - managed by parent -->
        </dependency>
    </dependencies>
</project>
```

---

## 4. Maven Multi-module Project

```
Project Structure:
my-ecommerce/                    ← Parent POM
├── pom.xml
├── ecommerce-domain/            ← Domain objects
│   ├── pom.xml
│   └── src/main/java/
├── ecommerce-repository/        ← Data access
│   ├── pom.xml
│   └── src/main/java/
├── ecommerce-service/           ← Business logic
│   ├── pom.xml
│   └── src/main/java/
└── ecommerce-web/               ← Web layer
    ├── pom.xml
    └── src/main/java/
```

```xml
<!-- my-ecommerce/pom.xml (Parent) -->
<project>
    <groupId>com.example</groupId>
    <artifactId>my-ecommerce</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    
    <modules>
        <module>ecommerce-domain</module>
        <module>ecommerce-repository</module>
        <module>ecommerce-service</module>
        <module>ecommerce-web</module>
    </modules>
    
    <properties>
        <java.version>21</java.version>
        <spring.version>6.1.1</spring.version>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>ecommerce-domain</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>ecommerce-repository</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>

<!-- ecommerce-service/pom.xml -->
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>my-ecommerce</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    
    <artifactId>ecommerce-service</artifactId>
    
    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>ecommerce-domain</artifactId>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>ecommerce-repository</artifactId>
        </dependency>
    </dependencies>
</project>
```

```bash
# Build entire multi-module project
mvn clean install

# Build from root, run only specific module's tests
mvn test -pl ecommerce-service

# Build only changed modules and their dependents
mvn clean install -pl ecommerce-domain,ecommerce-service -am

# -am = --also-make (build dependencies too)
# -amd = --also-make-dependents (build dependents too)
```

---

## 5. Gradle Fundamentals

### build.gradle (Groovy DSL)

```groovy
// build.gradle

plugins {
    id 'java'
    id 'application'
    id 'jacoco'
}

group = 'com.example'
version = '1.0.0-SNAPSHOT'

java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()
    maven {
        url 'https://repo.spring.io/milestone'
    }
}

// Dependency configurations
dependencies {
    // Implementation - compile + runtime
    implementation 'org.springframework:spring-context:6.1.1'
    implementation 'com.google.guava:guava:32.1.3-jre'
    
    // API - exposes dependency to consumers
    api 'org.slf4j:slf4j-api:2.0.9'
    
    // CompileOnly - compile but not runtime (like Maven 'provided')
    compileOnly 'jakarta.servlet:jakarta.servlet-api:6.0.0'
    
    // RuntimeOnly - runtime but not compile (like Maven 'runtime')
    runtimeOnly 'com.mysql:mysql-connector-j:8.2.0'
    
    // TestImplementation
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.1'
    testImplementation 'org.mockito:mockito-junit-jupiter:5.7.0'
    
    // TestRuntimeOnly
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

application {
    mainClass = 'com.example.Main'
}

test {
    useJUnitPlatform()
    
    testLogging {
        events "passed", "skipped", "failed"
        showExceptions true
        exceptionFormat "full"
    }
    
    // Run tests in parallel
    maxParallelForks = Runtime.runtime.availableProcessors()
}

jacoco {
    toolVersion = "0.8.11"
}

jacocoTestReport {
    reports {
        xml.required = true
        html.required = true
    }
    dependsOn test
}

// Custom task
tasks.register('generateSources') {
    group = 'build'
    description = 'Generates source files'
    
    doLast {
        println "Generating sources..."
    }
}

// Task dependencies
compileJava.dependsOn generateSources

// Configure existing task
jar {
    manifest {
        attributes 'Main-Class': 'com.example.Main'
    }
    
    // Fat JAR
    from {
        configurations.runtimeClasspath.collect {
            it.isDirectory() ? it : zipTree(it)
        }
    }
    
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
}
```

---

## 6. Gradle Kotlin DSL

```kotlin
// build.gradle.kts

plugins {
    java
    application
    jacoco
    id("com.github.ben-manes.versions") version "0.50.0"
}

group = "com.example"
version = "1.0.0-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(21))
    }
}

repositories {
    mavenCentral()
}

val springVersion = "6.1.1"
val junitVersion = "5.10.1"

dependencies {
    implementation("org.springframework:spring-context:$springVersion")
    implementation("org.slf4j:slf4j-api:2.0.9")
    runtimeOnly("ch.qos.logback:logback-classic:1.4.14")
    
    testImplementation("org.junit.jupiter:junit-jupiter:$junitVersion")
    testImplementation("org.mockito:mockito-junit-jupiter:5.7.0")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

application {
    mainClass.set("com.example.Main")
}

tasks.test {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = "0.80".toBigDecimal()
            }
        }
    }
}

// Custom task
tasks.register("printDependencies") {
    group = "help"
    description = "Print all dependencies"
    
    doLast {
        configurations.runtimeClasspath.get().forEach {
            println(it.name)
        }
    }
}

// Configure fat JAR
tasks.jar {
    manifest {
        attributes["Main-Class"] = "com.example.Main"
    }
    from(configurations.runtimeClasspath.get().map { 
        if (it.isDirectory) it else zipTree(it) 
    })
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
}
```

---

## 7. Gradle Multi-project Build

```kotlin
// settings.gradle.kts (root project)
rootProject.name = "my-ecommerce"

include(
    "ecommerce-domain",
    "ecommerce-repository", 
    "ecommerce-service",
    "ecommerce-web"
)

// Enable version catalog
dependencyResolutionManagement {
    versionCatalogs {
        create("libs") {
            from(files("gradle/libs.versions.toml"))
        }
    }
}
```

```toml
# gradle/libs.versions.toml (Version Catalog)
[versions]
spring = "6.1.1"
springBoot = "3.2.0"
junit = "5.10.1"
mockito = "5.7.0"
lombok = "1.18.30"

[libraries]
spring-context = { module = "org.springframework:spring-context", version.ref = "spring" }
spring-webmvc = { module = "org.springframework:spring-webmvc", version.ref = "spring" }
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit" }
mockito-core = { module = "org.mockito:mockito-core", version.ref = "mockito" }
lombok = { module = "org.projectlombok:lombok", version.ref = "lombok" }

[bundles]
testing = ["junit-jupiter", "mockito-core"]

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "springBoot" }
```

```kotlin
// build.gradle.kts (root)
plugins {
    java
}

// Shared configuration for all subprojects
subprojects {
    apply(plugin = "java")
    
    repositories {
        mavenCentral()
    }
    
    java {
        sourceCompatibility = JavaVersion.VERSION_21
    }
    
    dependencies {
        // Available to all subprojects
        testImplementation(libs.junit.jupiter)
        testImplementation(libs.mockito.core)
    }
    
    tasks.test {
        useJUnitPlatform()
    }
}

// ecommerce-service/build.gradle.kts
dependencies {
    implementation(project(":ecommerce-domain"))
    implementation(project(":ecommerce-repository"))
    implementation(libs.spring.context)
}
```

```bash
# Gradle commands
./gradlew build              # Compile + test + package
./gradlew clean build        # Clean then build
./gradlew test               # Run tests
./gradlew test --tests "com.example.MyTest"  # Specific test
./gradlew test --tests "com.example.*"       # Pattern

./gradlew dependencies       # Show dependency tree
./gradlew dependencies --configuration compileClasspath

./gradlew tasks              # List all available tasks
./gradlew tasks --all        # Include all tasks

# Multi-project
./gradlew :ecommerce-service:build   # Build specific module
./gradlew :ecommerce-service:test    # Test specific module

# Parallel build
./gradlew build --parallel

# Build scan (requires Gradle Enterprise)
./gradlew build --scan

# Daemon management
./gradlew --stop             # Stop Gradle daemon
./gradlew build --no-daemon  # Run without daemon
```

---

## 8. Gradle Custom Plugins

```kotlin
// buildSrc/build.gradle.kts (custom plugin project)
plugins {
    `kotlin-dsl`
}

repositories {
    mavenCentral()
}
```

```kotlin
// buildSrc/src/main/kotlin/java-conventions.gradle.kts
// Convention plugin - shared configuration

plugins {
    java
    checkstyle
    jacoco
}

java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()
}

checkstyle {
    toolVersion = "10.12.6"
    configFile = file("${rootDir}/config/checkstyle/checkstyle.xml")
}

jacoco {
    toolVersion = "0.8.11"
}

tasks.test {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)
}

tasks.jacocoTestReport {
    reports {
        xml.required.set(true)
    }
}

// Usage in any subproject:
// plugins {
//     id("java-conventions")
// }
```

```kotlin
// Custom task plugin
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.DefaultTask
import org.gradle.api.tasks.TaskAction
import org.gradle.api.file.DirectoryProperty
import org.gradle.api.tasks.OutputDirectory

// Custom task type
abstract class GenerateVersionTask : DefaultTask() {
    
    @get:OutputDirectory
    abstract val outputDir: DirectoryProperty
    
    @TaskAction
    fun generate() {
        val file = outputDir.get().asFile.resolve("Version.java")
        file.parentFile.mkdirs()
        file.writeText("""
            package com.example.generated;
            
            public class Version {
                public static final String VERSION = "${project.version}";
                public static final String BUILD_TIME = "${java.time.Instant.now()}";
            }
        """.trimIndent())
        println("Generated Version.java with version ${project.version}")
    }
}

// Custom plugin
class MyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        val extension = project.extensions.create("myPlugin", MyPluginExtension::class.java)
        
        val generateVersion = project.tasks.register("generateVersion", GenerateVersionTask::class.java) {
            group = "build"
            description = "Generate version information"
            outputDir.set(project.layout.buildDirectory.dir("generated-sources/main"))
        }
        
        project.tasks.named("compileJava") {
            dependsOn(generateVersion)
        }
        
        project.sourceSets {
            named("main") {
                java.srcDir(generateVersion.map { it.outputDir })
            }
        }
    }
}

open class MyPluginExtension {
    var packageName: String = "com.example.generated"
}
```

---

## 9. Dependency Management Best Practices

```kotlin
// Enforce dependency versions (lock file)
// gradle/dependency-locks/compileClasspath.lockfile

// build.gradle.kts
dependencyLocking {
    lockAllConfigurations()
}

// Generate lock files:
// ./gradlew dependencies --write-locks

// Update specific dependency:
// ./gradlew dependencies --update-locks com.example:my-lib
```

```xml
<!-- Maven: Enforce plugin to ban certain versions -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <version>3.4.1</version>
    <executions>
        <execution>
            <id>enforce</id>
            <goals>
                <goal>enforce</goal>
            </goals>
            <configuration>
                <rules>
                    <!-- Require minimum Maven version -->
                    <requireMavenVersion>
                        <version>[3.8,)</version>
                    </requireMavenVersion>
                    
                    <!-- Require minimum Java version -->
                    <requireJavaVersion>
                        <version>[21,)</version>
                    </requireJavaVersion>
                    
                    <!-- Prevent duplicate dependencies -->
                    <banDuplicatePomDependencyVersions/>
                    
                    <!-- No SNAPSHOT dependencies in releases -->
                    <requireReleaseDeps>
                        <onlyWhenRelease>true</onlyWhenRelease>
                    </requireReleaseDeps>
                    
                    <!-- Ban specific dependency -->
                    <bannedDependencies>
                        <excludes>
                            <exclude>commons-logging:commons-logging</exclude>
                        </excludes>
                    </bannedDependencies>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

---

## 10. Publishing Artifacts

### Maven Publishing

```xml
<!-- Publishing to Maven Central (via Sonatype OSSRH) -->
<build>
    <plugins>
        <!-- Source JAR -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-source-plugin</artifactId>
            <version>3.3.0</version>
            <executions>
                <execution>
                    <id>attach-sources</id>
                    <goals>
                        <goal>jar-no-fork</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        
        <!-- Javadoc JAR -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-javadoc-plugin</artifactId>
            <version>3.6.3</version>
            <executions>
                <execution>
                    <id>attach-javadocs</id>
                    <goals>
                        <goal>jar</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        
        <!-- GPG Signing (required for Maven Central) -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-gpg-plugin</artifactId>
            <version>3.1.0</version>
            <executions>
                <execution>
                    <id>sign-artifacts</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>sign</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        
        <!-- Nexus Staging -->
        <plugin>
            <groupId>org.sonatype.plugins</groupId>
            <artifactId>nexus-staging-maven-plugin</artifactId>
            <version>1.6.13</version>
            <extensions>true</extensions>
            <configuration>
                <serverId>ossrh</serverId>
                <nexusUrl>https://s01.oss.sonatype.org/</nexusUrl>
                <autoReleaseAfterClose>true</autoReleaseAfterClose>
            </configuration>
        </plugin>
    </plugins>
</build>

<!-- Add to settings.xml (~/.m2/settings.xml) -->
<!--
<servers>
    <server>
        <id>ossrh</id>
        <username>your-sonatype-username</username>
        <password>your-sonatype-password</password>
    </server>
</servers>
-->
```

### Gradle Publishing

```kotlin
// build.gradle.kts
plugins {
    java
    `maven-publish`
    signing
}

java {
    withJavadocJar()
    withSourcesJar()
}

publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])
            
            pom {
                name.set("My Library")
                description.set("A sample library")
                url.set("https://github.com/example/my-lib")
                
                licenses {
                    license {
                        name.set("MIT License")
                        url.set("https://opensource.org/licenses/MIT")
                    }
                }
                
                developers {
                    developer {
                        id.set("johndoe")
                        name.set("John Doe")
                        email.set("john@example.com")
                    }
                }
                
                scm {
                    connection.set("scm:git:git://github.com/example/my-lib.git")
                    developerConnection.set("scm:git:ssh://github.com:example/my-lib.git")
                    url.set("https://github.com/example/my-lib/tree/main")
                }
            }
        }
    }
    
    repositories {
        maven {
            name = "OSSRH"
            url = if (version.toString().endsWith("SNAPSHOT")) {
                uri("https://s01.oss.sonatype.org/content/repositories/snapshots/")
            } else {
                uri("https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/")
            }
            credentials {
                username = project.findProperty("ossrhUsername") as String?
                password = project.findProperty("ossrhPassword") as String?
            }
        }
    }
}

signing {
    sign(publishing.publications["mavenJava"])
}

// Publish: ./gradlew publishMavenJavaPublicationToOSSRHRepository
```

---

## 11. CI/CD Integration

### GitHub Actions with Maven

```yaml
# .github/workflows/maven.yml
name: Java CI with Maven

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        java: [17, 21]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up JDK ${{ matrix.java }}
      uses: actions/setup-java@v4
      with:
        java-version: ${{ matrix.java }}
        distribution: 'temurin'
        cache: maven
    
    - name: Build with Maven
      run: mvn -B package --file pom.xml
    
    - name: Run tests
      run: mvn test
    
    - name: Generate coverage report
      run: mvn jacoco:report
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: target/site/jacoco/jacoco.xml
    
    - name: Upload artifact
      uses: actions/upload-artifact@v4
      with:
        name: app-jar-${{ matrix.java }}
        path: target/*.jar
  
  release:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up JDK
      uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
        server-id: ossrh
        server-username: MAVEN_USERNAME
        server-password: MAVEN_PASSWORD
        gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
        gpg-passphrase: MAVEN_GPG_PASSPHRASE
    
    - name: Publish to OSSRH
      run: mvn --no-transfer-progress deploy -P release
      env:
        MAVEN_USERNAME: ${{ secrets.OSSRH_USERNAME }}
        MAVEN_PASSWORD: ${{ secrets.OSSRH_TOKEN }}
        MAVEN_GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
```

### GitHub Actions with Gradle

```yaml
# .github/workflows/gradle.yml
name: Java CI with Gradle

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up JDK 21
      uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
    
    - name: Setup Gradle
      uses: gradle/actions/setup-gradle@v3
      with:
        gradle-version: wrapper
    
    - name: Build and test
      run: ./gradlew build
    
    - name: Test Report
      uses: dorny/test-reporter@v1
      if: success() || failure()
      with:
        name: JUnit Tests
        path: '**/build/test-results/test/TEST-*.xml'
        reporter: java-junit
    
    - name: Code coverage
      run: ./gradlew jacocoTestReport
    
    - name: SonarQube analysis
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      run: ./gradlew sonar
        -Dsonar.projectKey=my-project
        -Dsonar.host.url=https://sonarcloud.io
        -Dsonar.organization=my-org
```

---

## 12. Complete Project Setup Script

```bash
#!/bin/bash
# setup-maven-project.sh - Create a new Maven project

PROJECT_NAME=$1
GROUP_ID=${2:-"com.example"}

if [ -z "$PROJECT_NAME" ]; then
    echo "Usage: $0 <project-name> [group-id]"
    exit 1
fi

# Create project using Maven archetype
mvn archetype:generate \
    -DgroupId="$GROUP_ID" \
    -DartifactId="$PROJECT_NAME" \
    -DarchetypeArtifactId=maven-archetype-quickstart \
    -DarchetypeVersion=1.4 \
    -DinteractiveMode=false

cd "$PROJECT_NAME"

# Create standard directory structure
mkdir -p src/main/resources
mkdir -p src/test/resources
mkdir -p src/main/java/"${GROUP_ID//.//}"/{config,controller,service,repository,model,exception}
mkdir -p src/test/java/"${GROUP_ID//.//}"/{unit,integration}

# Create .gitignore
cat > .gitignore << 'EOF'
target/
.idea/
*.iml
.classpath
.project
.settings/
*.class
*.jar
*.war
*.log
.env
EOF

echo "Project $PROJECT_NAME created successfully!"
echo "Run: cd $PROJECT_NAME && mvn clean install"
```

```bash
#!/bin/bash
# setup-gradle-project.sh - Create a new Gradle project

PROJECT_NAME=$1

mkdir -p "$PROJECT_NAME"
cd "$PROJECT_NAME"

# Initialize Gradle project
gradle init \
    --type java-application \
    --dsl kotlin \
    --test-framework junit-jupiter \
    --project-name "$PROJECT_NAME" \
    --package com.example

# Create GitHub Actions workflow
mkdir -p .github/workflows
cat > .github/workflows/build.yml << 'EOF'
name: Build
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
    - uses: gradle/actions/setup-gradle@v3
    - run: ./gradlew build
EOF

echo "Gradle project $PROJECT_NAME created!"
```

---

## 13. Maven Wrapper and Gradle Wrapper

```bash
# Maven Wrapper - ensures consistent Maven version
mvn wrapper:wrapper                  # Generate mvnw
mvn wrapper:wrapper -Dmaven=3.9.5   # Specific version

# Use wrapper
./mvnw clean install     # Unix
mvnw.cmd clean install   # Windows

# Gradle Wrapper - already included in new projects
gradle wrapper                       # Regenerate wrapper
gradle wrapper --gradle-version 8.5  # Specific version

# Use wrapper (ALWAYS prefer this in team projects)
./gradlew build          # Unix
gradlew.bat build        # Windows
```

```properties
# gradle/wrapper/gradle-wrapper.properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

---

## 14. SonarQube Integration

```xml
<!-- Maven SonarQube -->
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.10.0.2594</version>
</plugin>

<!-- Run:
mvn sonar:sonar \
  -Dsonar.projectKey=my-project \
  -Dsonar.host.url=https://sonarcloud.io \
  -Dsonar.organization=my-org \
  -Dsonar.login=$SONAR_TOKEN
-->
```

```kotlin
// Gradle SonarQube
plugins {
    id("org.sonarqube") version "4.4.1.3373"
}

sonar {
    properties {
        property("sonar.projectKey", "my-project")
        property("sonar.organization", "my-org")
        property("sonar.host.url", "https://sonarcloud.io")
        property("sonar.coverage.jacoco.xmlReportPaths", 
            "${layout.buildDirectory.get()}/reports/jacoco/test/jacocoTestReport.xml")
        property("sonar.exclusions", "**/generated/**,**/test/**")
    }
}

// Run: ./gradlew sonar -Dsonar.token=$SONAR_TOKEN
```

---

## สรุป Part 020

| หัวข้อ | Maven | Gradle |
|--------|-------|--------|
| Configuration | XML (pom.xml) | Groovy/Kotlin DSL |
| Build Lifecycle | Fixed phases | Flexible task graph |
| Performance | Slower | Faster (incremental + caching) |
| Multi-project | modules | multi-project |
| Plugins | Maven plugins | Gradle plugins |
| Wrapper | mvnw | gradlew |
| Convention | Strong conventions | Flexible |
| IDE Support | Excellent | Excellent |

### เมื่อเลือก Maven หรือ Gradle?
- **Maven**: ถ้า team คุ้นเคย, Spring Boot projects (both work), ต้องการ strict conventions
- **Gradle**: ต้องการ performance, Android projects, complex build logic, Kotlin projects

---

**Part 021:** Spring Framework Fundamentals - IoC Container & Dependency Injection
- ApplicationContext, BeanFactory
- Bean definitions, scopes, lifecycle
- @Component, @Service, @Repository, @Controller
- Constructor injection, setter injection, field injection
- @Configuration, @Bean, @Autowired, @Value
