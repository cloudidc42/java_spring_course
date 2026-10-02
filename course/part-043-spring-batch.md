# Part 043: Spring Batch for Bulk Processing

## Overview

Spring Batch provides a framework for robust, enterprise-grade batch processing. It handles common concerns like transaction management, restart/retry logic, skip policies, and parallel processing. This part builds from basics to a production-ready ETL pipeline processing millions of rows.

---

## 1. Project Setup

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>
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
        <groupId>com.opencsv</groupId>
        <artifactId>opencsv</artifactId>
        <version>5.8</version>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  batch:
    job:
      enabled: false  # Don't run jobs on startup automatically
    jdbc:
      initialize-schema: always  # Creates Spring Batch meta tables
  datasource:
    url: jdbc:postgresql://localhost:5432/batchdb
    username: postgres
    password: secret
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: false
```

---

## 2. Spring Batch Architecture Overview

```
Job
├── JobRepository (persists job metadata)
├── JobLauncher (starts jobs)
└── Step 1
    ├── ItemReader (read one item at a time)
    ├── ItemProcessor (transform the item)
    └── ItemWriter (write a chunk of items)
        └── Chunk (e.g., 1000 items per transaction)
```

---

## 3. Domain Model

```java
package com.example.batch.domain;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDate;

// Input CSV record
public record CustomerCsvRecord(
    String customerId,
    String firstName,
    String lastName,
    String email,
    String phone,
    String country,
    String city,
    String segment,
    BigDecimal totalPurchases,
    LocalDate joinDate
) {}
```

```java
package com.example.batch.domain;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.Instant;
import java.time.LocalDate;

@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String customerId;

    private String firstName;
    private String lastName;

    @Column(unique = true)
    private String email;

    private String phone;
    private String country;
    private String city;
    private String segment;

    @Column(precision = 15, scale = 2)
    private BigDecimal totalPurchases;

    private LocalDate joinDate;

    @Enumerated(EnumType.STRING)
    private CustomerTier tier;

    @Column(nullable = false)
    private Instant processedAt = Instant.now();

    public enum CustomerTier {
        BRONZE, SILVER, GOLD, PLATINUM
    }

    // Getters and setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }
    public String getFirstName() { return firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }
    public String getLastName() { return lastName; }
    public void setLastName(String lastName) { this.lastName = lastName; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
    public String getCountry() { return country; }
    public void setCountry(String country) { this.country = country; }
    public String getCity() { return city; }
    public void setCity(String city) { this.city = city; }
    public String getSegment() { return segment; }
    public void setSegment(String segment) { this.segment = segment; }
    public BigDecimal getTotalPurchases() { return totalPurchases; }
    public void setTotalPurchases(BigDecimal totalPurchases) { this.totalPurchases = totalPurchases; }
    public LocalDate getJoinDate() { return joinDate; }
    public void setJoinDate(LocalDate joinDate) { this.joinDate = joinDate; }
    public CustomerTier getTier() { return tier; }
    public void setTier(CustomerTier tier) { this.tier = tier; }
    public Instant getProcessedAt() { return processedAt; }
    public void setProcessedAt(Instant processedAt) { this.processedAt = processedAt; }
}
```

```java
package com.example.batch.domain;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "invalid_records")
public class InvalidRecord {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String rawData;
    private String errorMessage;
    private String sourceFile;
    private long lineNumber;
    private Instant recordedAt = Instant.now();

    public InvalidRecord() {}

    public InvalidRecord(String rawData, String errorMessage, String sourceFile, long lineNumber) {
        this.rawData = rawData;
        this.errorMessage = errorMessage;
        this.sourceFile = sourceFile;
        this.lineNumber = lineNumber;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getRawData() { return rawData; }
    public void setRawData(String rawData) { this.rawData = rawData; }
    public String getErrorMessage() { return errorMessage; }
    public void setErrorMessage(String errorMessage) { this.errorMessage = errorMessage; }
    public String getSourceFile() { return sourceFile; }
    public void setSourceFile(String sourceFile) { this.sourceFile = sourceFile; }
    public long getLineNumber() { return lineNumber; }
    public void setLineNumber(long lineNumber) { this.lineNumber = lineNumber; }
    public Instant getRecordedAt() { return recordedAt; }
    public void setRecordedAt(Instant recordedAt) { this.recordedAt = recordedAt; }
}
```

---

## 4. ItemReader Implementations

### 4.1 FlatFileItemReader (CSV)

```java
package com.example.batch.reader;

import com.example.batch.domain.CustomerCsvRecord;
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.builder.FlatFileItemReaderBuilder;
import org.springframework.batch.item.file.mapping.BeanWrapperFieldSetMapper;
import org.springframework.batch.item.file.mapping.DefaultLineMapper;
import org.springframework.batch.item.file.mapping.FieldSetMapper;
import org.springframework.batch.item.file.transform.DelimitedLineTokenizer;
import org.springframework.batch.item.file.transform.FieldSet;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.FileSystemResource;
import org.springframework.validation.BindException;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

@Configuration
public class CustomerCsvReaderConfig {

    private static final String[] CSV_HEADERS = {
        "customerId", "firstName", "lastName", "email", "phone",
        "country", "city", "segment", "totalPurchases", "joinDate"
    };

    @Bean
    public FlatFileItemReader<CustomerCsvRecord> customerCsvReader() {
        return new FlatFileItemReaderBuilder<CustomerCsvRecord>()
            .name("customerCsvReader")
            .resource(new FileSystemResource("data/customers.csv"))
            .linesToSkip(1)  // Skip header row
            .lineMapper(customerLineMapper())
            .build();
    }

    private DefaultLineMapper<CustomerCsvRecord> customerLineMapper() {
        DefaultLineMapper<CustomerCsvRecord> mapper = new DefaultLineMapper<>();

        DelimitedLineTokenizer tokenizer = new DelimitedLineTokenizer();
        tokenizer.setNames(CSV_HEADERS);
        tokenizer.setDelimiter(",");
        tokenizer.setQuoteCharacter('"');
        tokenizer.setStrict(false); // Don't fail on missing fields

        mapper.setLineTokenizer(tokenizer);
        mapper.setFieldSetMapper(customerFieldSetMapper());

        return mapper;
    }

    private FieldSetMapper<CustomerCsvRecord> customerFieldSetMapper() {
        return fieldSet -> new CustomerCsvRecord(
            fieldSet.readString("customerId"),
            fieldSet.readString("firstName"),
            fieldSet.readString("lastName"),
            fieldSet.readString("email"),
            fieldSet.readString("phone"),
            fieldSet.readString("country"),
            fieldSet.readString("city"),
            fieldSet.readString("segment"),
            new BigDecimal(fieldSet.readString("totalPurchases").isEmpty()
                ? "0" : fieldSet.readString("totalPurchases")),
            fieldSet.readString("joinDate").isEmpty() ? null :
                LocalDate.parse(fieldSet.readString("joinDate"),
                    DateTimeFormatter.ofPattern("yyyy-MM-dd"))
        );
    }
}
```

### 4.2 JdbcCursorItemReader

```java
package com.example.batch.reader;

import com.example.batch.domain.Customer;
import org.springframework.batch.item.database.JdbcCursorItemReader;
import org.springframework.batch.item.database.builder.JdbcCursorItemReaderBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.BeanPropertyRowMapper;

import javax.sql.DataSource;

@Configuration
public class JdbcReaderConfig {

    @Bean
    public JdbcCursorItemReader<Customer> customerJdbcReader(DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder<Customer>()
            .name("customerJdbcReader")
            .dataSource(dataSource)
            .sql("""
                SELECT c.id, c.customer_id, c.first_name, c.last_name,
                       c.email, c.country, c.segment, c.total_purchases, c.tier
                FROM customers c
                WHERE c.tier IS NULL
                ORDER BY c.id
                """)
            .rowMapper(new BeanPropertyRowMapper<>(Customer.class))
            .fetchSize(1000)  // Fetch 1000 rows at a time from DB
            .build();
    }
}
```

### 4.3 JpaPagingItemReader

```java
package com.example.batch.reader;

import com.example.batch.domain.Customer;
import jakarta.persistence.EntityManagerFactory;
import org.springframework.batch.item.database.JpaPagingItemReader;
import org.springframework.batch.item.database.builder.JpaPagingItemReaderBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Map;

@Configuration
public class JpaReaderConfig {

    @Bean
    public JpaPagingItemReader<Customer> customerJpaReader(EntityManagerFactory emf) {
        return new JpaPagingItemReaderBuilder<Customer>()
            .name("customerJpaReader")
            .entityManagerFactory(emf)
            .queryString("""
                SELECT c FROM Customer c
                WHERE c.tier IS NULL
                ORDER BY c.id
                """)
            .pageSize(1000)
            .build();
    }

    // With parameters
    @Bean
    public JpaPagingItemReader<Customer> customerJpaReaderByCountry(
            EntityManagerFactory emf, String country) {
        return new JpaPagingItemReaderBuilder<Customer>()
            .name("customerJpaReaderByCountry")
            .entityManagerFactory(emf)
            .queryString("""
                SELECT c FROM Customer c
                WHERE c.country = :country
                  AND c.tier IS NULL
                ORDER BY c.id
                """)
            .parameterValues(Map.of("country", country))
            .pageSize(500)
            .build();
    }
}
```

---

## 5. ItemProcessor Implementations

### 5.1 Customer Tier Processor

```java
package com.example.batch.processor;

import com.example.batch.domain.Customer;
import com.example.batch.domain.CustomerCsvRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.time.Instant;

@Component
public class CustomerEnrichmentProcessor
        implements ItemProcessor<CustomerCsvRecord, Customer> {

    private static final Logger log = LoggerFactory.getLogger(CustomerEnrichmentProcessor.class);

    private static final BigDecimal PLATINUM_THRESHOLD = new BigDecimal("10000");
    private static final BigDecimal GOLD_THRESHOLD = new BigDecimal("5000");
    private static final BigDecimal SILVER_THRESHOLD = new BigDecimal("1000");

    @Override
    public Customer process(CustomerCsvRecord item) {
        // Return null to SKIP this record (it won't be written)
        if (item.email() == null || item.email().isBlank()) {
            log.warn("Skipping customer {} - missing email", item.customerId());
            return null;
        }

        if (!isValidEmail(item.email())) {
            log.warn("Skipping customer {} - invalid email: {}", item.customerId(), item.email());
            return null;
        }

        Customer customer = new Customer();
        customer.setCustomerId(item.customerId());
        customer.setFirstName(capitalize(item.firstName()));
        customer.setLastName(capitalize(item.lastName()));
        customer.setEmail(item.email().toLowerCase().trim());
        customer.setPhone(normalizePhone(item.phone()));
        customer.setCountry(item.country());
        customer.setCity(item.city());
        customer.setSegment(item.segment());
        customer.setTotalPurchases(item.totalPurchases());
        customer.setJoinDate(item.joinDate());
        customer.setTier(calculateTier(item.totalPurchases()));
        customer.setProcessedAt(Instant.now());

        return customer;
    }

    private Customer.CustomerTier calculateTier(BigDecimal totalPurchases) {
        if (totalPurchases == null) return Customer.CustomerTier.BRONZE;
        if (totalPurchases.compareTo(PLATINUM_THRESHOLD) >= 0) return Customer.CustomerTier.PLATINUM;
        if (totalPurchases.compareTo(GOLD_THRESHOLD) >= 0) return Customer.CustomerTier.GOLD;
        if (totalPurchases.compareTo(SILVER_THRESHOLD) >= 0) return Customer.CustomerTier.SILVER;
        return Customer.CustomerTier.BRONZE;
    }

    private boolean isValidEmail(String email) {
        return email != null && email.matches("^[^@]+@[^@]+\\.[^@]+$");
    }

    private String capitalize(String name) {
        if (name == null || name.isBlank()) return name;
        return Character.toUpperCase(name.charAt(0)) +
               name.substring(1).toLowerCase();
    }

    private String normalizePhone(String phone) {
        if (phone == null) return null;
        return phone.replaceAll("[^+\\d]", "");
    }
}
```

### 5.2 CompositeItemProcessor

```java
package com.example.batch.processor;

import com.example.batch.domain.Customer;
import com.example.batch.domain.CustomerCsvRecord;
import org.springframework.batch.item.support.CompositeItemProcessor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
public class ProcessorConfig {

    @Bean
    public CompositeItemProcessor<CustomerCsvRecord, Customer> compositeProcessor(
            CustomerEnrichmentProcessor enrichmentProcessor,
            CustomerValidationProcessor validationProcessor,
            CustomerNormalizationProcessor normalizationProcessor
    ) {
        CompositeItemProcessor<CustomerCsvRecord, Customer> composite =
            new CompositeItemProcessor<>();
        composite.setDelegates(List.of(
            enrichmentProcessor,
            validationProcessor,
            normalizationProcessor
        ));
        return composite;
    }
}
```

```java
package com.example.batch.processor;

import com.example.batch.domain.Customer;
import com.example.batch.domain.CustomerCsvRecord;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.stereotype.Component;

@Component
public class CustomerValidationProcessor
        implements ItemProcessor<CustomerCsvRecord, Customer> {

    @Override
    public Customer process(CustomerCsvRecord item) {
        // Additional validation logic
        if (item.customerId() == null || item.customerId().isBlank()) {
            return null; // Skip records without ID
        }
        // This processor passes through (another processor does the actual mapping)
        // In a real composite, each processor would work on a specific transformation
        return null; // Should not reach here - composite handles ordering
    }
}
```

```java
package com.example.batch.processor;

import com.example.batch.domain.Customer;
import com.example.batch.domain.CustomerCsvRecord;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.stereotype.Component;

@Component
public class CustomerNormalizationProcessor
        implements ItemProcessor<CustomerCsvRecord, Customer> {

    @Override
    public Customer process(CustomerCsvRecord item) {
        // Normalization: another step in the composite
        return null;
    }
}
```

---

## 6. ItemWriter Implementations

### 6.1 JdbcBatchItemWriter

```java
package com.example.batch.writer;

import com.example.batch.domain.Customer;
import org.springframework.batch.item.database.JdbcBatchItemWriter;
import org.springframework.batch.item.database.builder.JdbcBatchItemWriterBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;

@Configuration
public class CustomerJdbcWriterConfig {

    @Bean
    public JdbcBatchItemWriter<Customer> customerJdbcWriter(DataSource dataSource) {
        return new JdbcBatchItemWriterBuilder<Customer>()
            .dataSource(dataSource)
            .sql("""
                INSERT INTO customers
                    (customer_id, first_name, last_name, email, phone,
                     country, city, segment, total_purchases, join_date, tier, processed_at)
                VALUES
                    (:customerId, :firstName, :lastName, :email, :phone,
                     :country, :city, :segment, :totalPurchases, :joinDate, :tier, :processedAt)
                ON CONFLICT (customer_id) DO UPDATE SET
                    first_name = EXCLUDED.first_name,
                    last_name = EXCLUDED.last_name,
                    total_purchases = EXCLUDED.total_purchases,
                    tier = EXCLUDED.tier,
                    processed_at = EXCLUDED.processed_at
                """)
            .beanMapped()  // Uses getter names to map to :paramName
            .build();
    }
}
```

### 6.2 JpaItemWriter

```java
package com.example.batch.writer;

import com.example.batch.domain.Customer;
import jakarta.persistence.EntityManagerFactory;
import org.springframework.batch.item.database.JpaItemWriter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CustomerJpaWriterConfig {

    @Bean
    public JpaItemWriter<Customer> customerJpaWriter(EntityManagerFactory emf) {
        JpaItemWriter<Customer> writer = new JpaItemWriter<>();
        writer.setEntityManagerFactory(emf);
        writer.setUsePersist(false); // Use merge (upsert behavior)
        return writer;
    }
}
```

### 6.3 Custom FlatFileItemWriter (CSV Output)

```java
package com.example.batch.writer;

import com.example.batch.domain.Customer;
import org.springframework.batch.item.file.FlatFileItemWriter;
import org.springframework.batch.item.file.builder.FlatFileItemWriterBuilder;
import org.springframework.batch.item.file.transform.BeanWrapperFieldExtractor;
import org.springframework.batch.item.file.transform.DelimitedLineAggregator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.FileSystemResource;

@Configuration
public class CustomerCsvWriterConfig {

    @Bean
    public FlatFileItemWriter<Customer> customerCsvWriter() {
        String[] fields = {
            "customerId", "firstName", "lastName", "email",
            "country", "segment", "tier", "totalPurchases"
        };

        BeanWrapperFieldExtractor<Customer> extractor = new BeanWrapperFieldExtractor<>();
        extractor.setNames(fields);

        DelimitedLineAggregator<Customer> aggregator = new DelimitedLineAggregator<>();
        aggregator.setDelimiter(",");
        aggregator.setFieldExtractor(extractor);

        return new FlatFileItemWriterBuilder<Customer>()
            .name("customerCsvWriter")
            .resource(new FileSystemResource("output/customers-processed.csv"))
            .headerCallback(writer -> writer.write(
                "customerId,firstName,lastName,email,country,segment,tier,totalPurchases"
            ))
            .lineAggregator(aggregator)
            .build();
    }
}
```

---

## 7. Job and Step Configuration

### 7.1 Basic Job Definition

```java
package com.example.batch.job;

import com.example.batch.domain.Customer;
import com.example.batch.domain.CustomerCsvRecord;
import com.example.batch.listener.CustomerJobListener;
import com.example.batch.listener.CustomerStepListener;
import com.example.batch.processor.CustomerEnrichmentProcessor;
import com.example.batch.writer.CustomerJdbcWriterConfig;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.database.JdbcBatchItemWriter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class CustomerImportJobConfig {

    @Bean
    public Job customerImportJob(
            JobRepository jobRepository,
            Step customerImportStep,
            CustomerJobListener jobListener
    ) {
        return new JobBuilder("customerImportJob", jobRepository)
            .incrementer(new RunIdIncrementer()) // Auto-increment run ID for re-runs
            .listener(jobListener)
            .start(customerImportStep)
            .build();
    }

    @Bean
    public Step customerImportStep(
            JobRepository jobRepository,
            PlatformTransactionManager transactionManager,
            FlatFileItemReader<CustomerCsvRecord> customerCsvReader,
            CustomerEnrichmentProcessor processor,
            JdbcBatchItemWriter<Customer> customerJdbcWriter,
            CustomerStepListener stepListener
    ) {
        return new StepBuilder("customerImportStep", jobRepository)
            .<CustomerCsvRecord, Customer>chunk(1000, transactionManager)
            .reader(customerCsvReader)
            .processor(processor)
            .writer(customerJdbcWriter)
            .listener(stepListener)
            // Skip policy: skip bad records, continue processing
            .faultTolerant()
            .skipLimit(100)  // Skip up to 100 bad records
            .skip(Exception.class)
            .noSkip(OutOfMemoryError.class)
            // Retry policy: retry transient failures
            .retryLimit(3)
            .retry(org.springframework.dao.DeadlockLoserDataAccessException.class)
            .build();
    }
}
```

---

## 8. Job Listeners and Step Listeners

### 8.1 Job Execution Listener

```java
package com.example.batch.listener;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.batch.core.BatchStatus;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobExecutionListener;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Component
public class CustomerJobListener implements JobExecutionListener {

    private static final Logger log = LoggerFactory.getLogger(CustomerJobListener.class);

    @Override
    public void beforeJob(JobExecution jobExecution) {
        log.info("Starting job: {} | Run ID: {} | Params: {}",
            jobExecution.getJobInstance().getJobName(),
            jobExecution.getId(),
            jobExecution.getJobParameters()
        );
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        BatchStatus status = jobExecution.getStatus();
        long durationMs = Duration.between(
            jobExecution.getStartTime(),
            jobExecution.getEndTime()
        ).toMillis();

        log.info("Job completed: {} | Status: {} | Duration: {} ms",
            jobExecution.getJobInstance().getJobName(),
            status,
            durationMs
        );

        if (status == BatchStatus.COMPLETED) {
            log.info("Write count: {} | Read count: {} | Skip count: {}",
                jobExecution.getStepExecutions().stream()
                    .mapToLong(s -> s.getWriteCount()).sum(),
                jobExecution.getStepExecutions().stream()
                    .mapToLong(s -> s.getReadCount()).sum(),
                jobExecution.getStepExecutions().stream()
                    .mapToLong(s -> s.getSkipCount()).sum()
            );
        } else if (status == BatchStatus.FAILED) {
            log.error("Job FAILED with exceptions:");
            jobExecution.getAllFailureExceptions()
                .forEach(ex -> log.error("  - {}: {}", ex.getClass().getSimpleName(), ex.getMessage()));
        }
    }
}
```

### 8.2 Step Execution Listener

```java
package com.example.batch.listener;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.batch.core.ExitStatus;
import org.springframework.batch.core.StepExecution;
import org.springframework.batch.core.StepExecutionListener;
import org.springframework.stereotype.Component;

@Component
public class CustomerStepListener implements StepExecutionListener {

    private static final Logger log = LoggerFactory.getLogger(CustomerStepListener.class);

    @Override
    public void beforeStep(StepExecution stepExecution) {
        log.info("Starting step: {}", stepExecution.getStepName());
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        log.info("""
            Step '{}' finished:
              Read:     {}
              Written:  {}
              Filtered: {}
              Skipped:  {} (read: {}, process: {}, write: {})
              Commits:  {}
              Rollbacks:{}
            """,
            stepExecution.getStepName(),
            stepExecution.getReadCount(),
            stepExecution.getWriteCount(),
            stepExecution.getFilterCount(),
            stepExecution.getSkipCount(),
            stepExecution.getReadSkipCount(),
            stepExecution.getProcessSkipCount(),
            stepExecution.getWriteSkipCount(),
            stepExecution.getCommitCount(),
            stepExecution.getRollbackCount()
        );

        // Can modify the exit status
        if (stepExecution.getSkipCount() > 0) {
            log.warn("Skipped {} records - check invalid_records table for details",
                stepExecution.getSkipCount());
        }

        return stepExecution.getExitStatus();
    }
}
```

---

## 9. Partitioned Steps for Parallel Processing

```java
package com.example.batch.partition;

import org.springframework.batch.core.partition.support.Partitioner;
import org.springframework.batch.item.ExecutionContext;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.HashMap;
import java.util.Map;

@Component
public class CustomerPartitioner implements Partitioner {

    private final DataSource dataSource;

    public CustomerPartitioner(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Map<String, ExecutionContext> result = new HashMap<>();

        // Get min/max ID range from DB
        long minId = 1L;
        long maxId = 1_000_000L;

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "SELECT MIN(id), MAX(id) FROM customers WHERE tier IS NULL")) {
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                minId = rs.getLong(1);
                maxId = rs.getLong(2);
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to get ID range", e);
        }

        long targetSize = (maxId - minId) / gridSize + 1;

        long start = minId;
        long end = start + targetSize - 1;

        for (int i = 0; i < gridSize; i++) {
            ExecutionContext context = new ExecutionContext();
            context.putLong("minId", start);
            context.putLong("maxId", Math.min(end, maxId));
            result.put("partition" + i, context);

            start += targetSize;
            end += targetSize;
        }

        return result;
    }
}
```

```java
package com.example.batch.job;

import com.example.batch.domain.Customer;
import com.example.batch.partition.CustomerPartitioner;
import jakarta.persistence.EntityManagerFactory;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.batch.core.partition.support.TaskExecutorPartitionHandler;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.database.JpaPagingItemReader;
import org.springframework.batch.item.database.builder.JpaPagingItemReaderBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskExecutor;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.transaction.PlatformTransactionManager;

import java.util.Map;

@Configuration
public class PartitionedJobConfig {

    @Bean
    public TaskExecutor partitionTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(8);
        executor.setThreadNamePrefix("partition-");
        executor.initialize();
        return executor;
    }

    @Bean
    public Step partitionManagerStep(
            JobRepository jobRepository,
            CustomerPartitioner partitioner,
            Step partitionWorkerStep
    ) {
        TaskExecutorPartitionHandler partitionHandler = new TaskExecutorPartitionHandler();
        partitionHandler.setTaskExecutor(partitionTaskExecutor());
        partitionHandler.setStep(partitionWorkerStep);
        partitionHandler.setGridSize(8); // 8 parallel partitions

        return new StepBuilder("partitionManagerStep", jobRepository)
            .partitioner("partitionWorkerStep", partitioner)
            .partitionHandler(partitionHandler)
            .build();
    }

    @Bean
    public Step partitionWorkerStep(
            JobRepository jobRepository,
            PlatformTransactionManager transactionManager,
            JpaPagingItemReader<Customer> partitionedReader,
            TierAssignmentProcessor tierProcessor,
            org.springframework.batch.item.database.JpaItemWriter<Customer> jpaWriter
    ) {
        return new StepBuilder("partitionWorkerStep", jobRepository)
            .<Customer, Customer>chunk(500, transactionManager)
            .reader(partitionedReader)
            .processor(tierProcessor)
            .writer(jpaWriter)
            .build();
    }

    // StepScope: creates a new instance for EACH step execution (required for partitioning)
    @Bean
    @StepScope
    public JpaPagingItemReader<Customer> partitionedReader(
            EntityManagerFactory emf,
            @Value("#{stepExecutionContext['minId']}") Long minId,
            @Value("#{stepExecutionContext['maxId']}") Long maxId
    ) {
        return new JpaPagingItemReaderBuilder<Customer>()
            .name("partitionedCustomerReader")
            .entityManagerFactory(emf)
            .queryString("""
                SELECT c FROM Customer c
                WHERE c.id BETWEEN :minId AND :maxId
                  AND c.tier IS NULL
                ORDER BY c.id
                """)
            .parameterValues(Map.of("minId", minId, "maxId", maxId))
            .pageSize(500)
            .build();
    }

    @Bean
    public Job tierAssignmentJob(
            JobRepository jobRepository,
            Step partitionManagerStep
    ) {
        return new JobBuilder("tierAssignmentJob", jobRepository)
            .incrementer(new RunIdIncrementer())
            .start(partitionManagerStep)
            .build();
    }
}
```

```java
package com.example.batch.job;

import com.example.batch.domain.Customer;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;

@Component
public class TierAssignmentProcessor implements ItemProcessor<Customer, Customer> {

    @Override
    public Customer process(Customer customer) {
        BigDecimal purchases = customer.getTotalPurchases();
        if (purchases == null) {
            customer.setTier(Customer.CustomerTier.BRONZE);
        } else if (purchases.compareTo(new BigDecimal("10000")) >= 0) {
            customer.setTier(Customer.CustomerTier.PLATINUM);
        } else if (purchases.compareTo(new BigDecimal("5000")) >= 0) {
            customer.setTier(Customer.CustomerTier.GOLD);
        } else if (purchases.compareTo(new BigDecimal("1000")) >= 0) {
            customer.setTier(Customer.CustomerTier.SILVER);
        } else {
            customer.setTier(Customer.CustomerTier.BRONZE);
        }
        return customer;
    }
}
```

---

## 10. Job Parameters and Execution

```java
package com.example.batch.controller;

import org.springframework.batch.core.*;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.batch.core.repository.JobExecutionAlreadyRunningException;
import org.springframework.batch.core.repository.JobInstanceAlreadyCompleteException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.Instant;
import java.util.Map;

@RestController
@RequestMapping("/api/batch")
public class BatchController {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job customerImportJob;

    @Autowired
    private Job tierAssignmentJob;

    @PostMapping("/import/customers")
    public ResponseEntity<?> importCustomers(
            @RequestParam String filePath,
            @RequestParam(defaultValue = "false") boolean force
    ) {
        try {
            JobParameters params = new JobParametersBuilder()
                .addString("filePath", filePath)
                .addString("runAt", Instant.now().toString())  // Ensures unique run
                .addLong("timestamp", System.currentTimeMillis())
                .toJobParameters();

            JobExecution execution = jobLauncher.run(customerImportJob, params);

            return ResponseEntity.ok(Map.of(
                "jobId", execution.getId(),
                "status", execution.getStatus().toString(),
                "startTime", execution.getStartTime()
            ));

        } catch (JobInstanceAlreadyCompleteException e) {
            return ResponseEntity.badRequest()
                .body(Map.of("error", "Job already completed for these parameters"));
        } catch (JobExecutionAlreadyRunningException e) {
            return ResponseEntity.badRequest()
                .body(Map.of("error", "Job is already running"));
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                .body(Map.of("error", e.getMessage()));
        }
    }

    @GetMapping("/status/{jobId}")
    public ResponseEntity<?> getJobStatus(@PathVariable Long jobId) {
        // In production, query the JobRepository
        return ResponseEntity.ok(Map.of("jobId", jobId, "status", "COMPLETED"));
    }
}
```

---

## 11. Job Restart and Skip Policy

```java
package com.example.batch.config;

import com.example.batch.listener.SkippedItemListener;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

import java.io.IOException;

@Configuration
public class FaultTolerantStepConfig {

    @Bean
    public Step faultTolerantImportStep(
            JobRepository jobRepository,
            PlatformTransactionManager transactionManager,
            ItemReader<?> reader,
            ItemWriter<?> writer,
            SkippedItemListener skippedItemListener
    ) {
        return new StepBuilder("faultTolerantImportStep", jobRepository)
            .chunk(1000, transactionManager)
            .reader(reader)
            .writer(writer)
            .faultTolerant()
            // Skip configuration
            .skipLimit(500)
            .skip(Exception.class)
            .noSkip(OutOfMemoryError.class)
            .noSkip(IllegalStateException.class) // Hard failures
            // Retry configuration
            .retryLimit(3)
            .retry(java.sql.SQLException.class)
            .retry(org.springframework.dao.TransientDataAccessException.class)
            // Listener to log skipped items
            .listener(skippedItemListener)
            // Allow restart from last checkpoint
            .allowStartIfComplete(false)  // Don't re-run completed steps
            .build();
    }
}
```

```java
package com.example.batch.listener;

import com.example.batch.domain.CustomerCsvRecord;
import com.example.batch.domain.InvalidRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.batch.core.SkipListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import java.time.Instant;

@Component
public class SkippedItemListener implements SkipListener<CustomerCsvRecord, Object> {

    private static final Logger log = LoggerFactory.getLogger(SkippedItemListener.class);

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public void onSkipInRead(Throwable t) {
        log.warn("Skipped during read: {}", t.getMessage());
    }

    @Override
    public void onSkipInProcess(CustomerCsvRecord item, Throwable t) {
        log.warn("Skipped during process - customerId: {} | Error: {}",
            item.customerId(), t.getMessage());

        // Save bad record to audit table
        jdbcTemplate.update(
            """
            INSERT INTO invalid_records (raw_data, error_message, source_file, line_number, recorded_at)
            VALUES (?, ?, ?, ?, ?)
            """,
            item.toString(),
            t.getMessage(),
            "customers.csv",
            -1,
            Instant.now()
        );
    }

    @Override
    public void onSkipInWrite(Object item, Throwable t) {
        log.warn("Skipped during write: {} | Error: {}", item, t.getMessage());
    }
}
```

---

## 12. Scheduling with @Scheduled

```java
package com.example.batch.scheduler;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;

@Configuration
@EnableScheduling
public class BatchScheduler {

    private static final Logger log = LoggerFactory.getLogger(BatchScheduler.class);

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job customerImportJob;

    // Run every night at 2:00 AM
    @Scheduled(cron = "0 0 2 * * ?")
    public void runNightlyImport() {
        log.info("Starting scheduled nightly import");
        try {
            JobParameters params = new JobParametersBuilder()
                .addLong("runTime", System.currentTimeMillis())
                .addString("source", "scheduled")
                .toJobParameters();

            jobLauncher.run(customerImportJob, params);
        } catch (Exception e) {
            log.error("Scheduled job failed", e);
        }
    }

    // Check for new files every 5 minutes
    @Scheduled(fixedDelay = 300_000) // 5 minutes
    public void checkForNewFiles() {
        // Check import directory for new files
        log.debug("Checking for new import files...");
    }
}
```

---

## 13. Real Example: ETL Pipeline Processing 1 Million CSV Rows

```java
package com.example.batch.etl;

import org.springframework.batch.core.*;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.database.JdbcBatchItemWriter;
import org.springframework.batch.item.database.builder.JdbcBatchItemWriterBuilder;
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.builder.FlatFileItemReaderBuilder;
import org.springframework.batch.item.file.mapping.DefaultLineMapper;
import org.springframework.batch.item.file.transform.DelimitedLineTokenizer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.FileSystemResource;
import org.springframework.core.task.TaskExecutor;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

@Configuration
public class LargeScaleEtlConfig {

    // Performance tuning constants
    private static final int CHUNK_SIZE = 5_000;   // Process 5k records per transaction
    private static final int THREAD_COUNT = 8;     // Parallel partitions
    private static final int FETCH_SIZE = 10_000;  // JDBC fetch size

    @Bean
    public TaskExecutor etlTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(THREAD_COUNT);
        executor.setMaxPoolSize(THREAD_COUNT);
        executor.setQueueCapacity(0); // No queue - fail fast if pool full
        executor.setThreadNamePrefix("etl-");
        executor.initialize();
        return executor;
    }

    @Bean
    @StepScope
    public FlatFileItemReader<CustomerCsvRecord> largeFileReader(
            @Value("#{jobParameters['inputFile']}") String inputFile
    ) {
        DefaultLineMapper<CustomerCsvRecord> lineMapper = new DefaultLineMapper<>();

        DelimitedLineTokenizer tokenizer = new DelimitedLineTokenizer();
        tokenizer.setNames("customerId", "firstName", "lastName", "email",
                          "phone", "country", "city", "segment",
                          "totalPurchases", "joinDate");

        lineMapper.setLineTokenizer(tokenizer);
        lineMapper.setFieldSetMapper(fieldSet -> new CustomerCsvRecord(
            fieldSet.readString("customerId"),
            fieldSet.readString("firstName"),
            fieldSet.readString("lastName"),
            fieldSet.readString("email"),
            fieldSet.readString("phone"),
            fieldSet.readString("country"),
            fieldSet.readString("city"),
            fieldSet.readString("segment"),
            new BigDecimal(fieldSet.readString("totalPurchases").isEmpty() ? "0"
                : fieldSet.readString("totalPurchases")),
            fieldSet.readString("joinDate").isEmpty() ? null
                : LocalDate.parse(fieldSet.readString("joinDate"), DateTimeFormatter.ISO_DATE)
        ));

        return new FlatFileItemReaderBuilder<CustomerCsvRecord>()
            .name("largeFileReader")
            .resource(new FileSystemResource(inputFile))
            .linesToSkip(1)
            .lineMapper(lineMapper)
            .build();
    }

    @Bean
    public JdbcBatchItemWriter<com.example.batch.domain.Customer> bulkCustomerWriter(
            DataSource dataSource
    ) {
        return new JdbcBatchItemWriterBuilder<com.example.batch.domain.Customer>()
            .dataSource(dataSource)
            .sql("""
                INSERT INTO customers
                    (customer_id, first_name, last_name, email, phone,
                     country, city, segment, total_purchases, join_date, tier, processed_at)
                VALUES
                    (:customerId, :firstName, :lastName, :email, :phone,
                     :country, :city, :segment, :totalPurchases, :joinDate,
                     CAST(:tier AS varchar), :processedAt)
                ON CONFLICT (customer_id) DO UPDATE SET
                    first_name = EXCLUDED.first_name,
                    total_purchases = EXCLUDED.total_purchases,
                    tier = EXCLUDED.tier,
                    processed_at = EXCLUDED.processed_at
                """)
            .beanMapped()
            .build();
    }

    @Bean
    public Job largeScaleEtlJob(
            JobRepository jobRepository,
            Step largeScaleEtlStep
    ) {
        return new JobBuilder("largeScaleEtlJob", jobRepository)
            .incrementer(new RunIdIncrementer())
            .start(largeScaleEtlStep)
            .build();
    }

    @Bean
    public Step largeScaleEtlStep(
            JobRepository jobRepository,
            PlatformTransactionManager transactionManager,
            FlatFileItemReader<CustomerCsvRecord> largeFileReader,
            com.example.batch.processor.CustomerEnrichmentProcessor processor,
            JdbcBatchItemWriter<com.example.batch.domain.Customer> bulkCustomerWriter,
            com.example.batch.listener.CustomerStepListener stepListener
    ) {
        return new StepBuilder("largeScaleEtlStep", jobRepository)
            .<CustomerCsvRecord, com.example.batch.domain.Customer>chunk(
                CHUNK_SIZE, transactionManager
            )
            .reader(largeFileReader)
            .processor(processor)
            .writer(bulkCustomerWriter)
            .listener(stepListener)
            .taskExecutor(etlTaskExecutor())     // Multi-threaded step
            .throttleLimit(THREAD_COUNT)         // Match thread pool size
            .faultTolerant()
            .skipLimit(10_000)                   // Allow up to 1% bad records (of 1M)
            .skip(Exception.class)
            .build();
    }
}
```

### 13.1 Expected Performance with 1 Million Rows

```java
/*
 * Performance benchmark: 1,000,000 customer records from CSV
 *
 * Hardware: 4-core server, PostgreSQL on same machine
 * File size: ~200 MB CSV
 *
 * Single-threaded (chunk=1000):
 *   - Total time: ~8-12 minutes
 *   - Throughput: ~1,400 records/second
 *   - DB inserts: 1000 batch inserts = 1000 INSERT statements (batched)
 *
 * 8-thread partitioned (chunk=5000):
 *   - Total time: ~90-120 seconds
 *   - Throughput: ~8,000-10,000 records/second
 *   - DB inserts: 200 batch transactions of 5000 records each
 *
 * Optimizations applied:
 *   1. Chunk size 5000 (reduces transaction overhead)
 *   2. PostgreSQL COPY alternative (even faster, see below)
 *   3. Disable foreign key checks during import
 *   4. Create indexes AFTER import
 *   5. Use UNLOGGED tables during import, then alter to LOGGED
 */

// Fastest PostgreSQL import: COPY command
// 1 million rows in ~15-20 seconds
// COPY customers FROM '/path/to/customers.csv' CSV HEADER;
```

---

## Summary

| Component | Role | Key Config |
|-----------|------|------------|
| `Job` | Top-level batch unit | `JobBuilder`, `incrementer`, `listener` |
| `Step` | Logical processing unit | `StepBuilder`, chunk size, fault tolerance |
| `ItemReader` | Read items one by one | Cursor, paging, file readers |
| `ItemProcessor` | Transform/filter items | Return `null` to skip |
| `ItemWriter` | Write a chunk at once | JDBC batch, JPA, file writers |
| `JobRepository` | Persists metadata | Spring-managed batch tables |
| `JobLauncher` | Launches jobs with params | Async or sync launch |
| `Partitioner` | Splits data for parallel steps | Range or list-based |
| Skip Policy | Handles bad records | `skipLimit`, `skip()`, `SkipListener` |
| Retry Policy | Retries transient failures | `retryLimit`, `retry()` |

---

## Next Part Preview

**Part 044: Spring Integration** — We'll explore Enterprise Integration Patterns (EIP), Spring Integration flows, adapters, transformers, routers, and build a file processing pipeline that routes, transforms, and routes messages through multiple channels.
