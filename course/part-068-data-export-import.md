# Part 068: Data Import and Export

## Overview

Bulk data operations are essential for every real application — importing product catalogs from
suppliers, exporting reports for accounting, and seeding databases for testing. This part covers
CSV with OpenCSV, Excel with Apache POI, Spring Batch for large-file processing, streaming for
memory efficiency, async progress tracking, and a full product import pipeline with a validation
error report.

---

## 1. Project Setup

### Maven dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- Spring Batch -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>

    <!-- OpenCSV -->
    <dependency>
        <groupId>com.opencsv</groupId>
        <artifactId>opencsv</artifactId>
        <version>5.9</version>
    </dependency>

    <!-- Apache POI (Excel .xlsx) -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml</artifactId>
        <version>5.2.5</version>
    </dependency>

    <!-- Jakarta Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  batch:
    job:
      enabled: false   # Don't auto-run jobs on startup
    jdbc:
      initialize-schema: always

  servlet:
    multipart:
      max-file-size: 50MB
      max-request-size: 52MB

app:
  import:
    upload-dir: /tmp/imports
    chunk-size: 500
    max-errors: 100   # Stop if more than 100 rows fail
```

---

## 2. Domain Model

```java
package com.example.dataimport.domain;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(name = "products")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    @NotBlank
    private String sku;

    @Column(nullable = false)
    @NotBlank
    @Size(min = 2, max = 255)
    private String name;

    @Column(length = 4000)
    private String description;

    @Column(precision = 12, scale = 2)
    @NotNull
    @DecimalMin(value = "0.0", inclusive = false)
    private BigDecimal price;

    @Column(nullable = false)
    @NotNull
    @Min(0)
    private Integer stock;

    @Column
    @NotBlank
    private String category;

    @Column
    private String brand;

    @Column
    private Boolean active = true;

    @Column
    private Instant createdAt;

    @PrePersist
    void onCreate() { this.createdAt = Instant.now(); }
}
```

---

## 3. CSV Import with OpenCSV

### CSV row DTO with header binding

```java
package com.example.dataimport.dto;

import com.opencsv.bean.CsvBindByName;
import com.opencsv.bean.CsvBindByPosition;
import jakarta.validation.constraints.*;
import lombok.*;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class ProductCsvRow {

    @CsvBindByName(column = "SKU", required = true)
    @NotBlank(message = "SKU is required")
    private String sku;

    @CsvBindByName(column = "NAME", required = true)
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 255)
    private String name;

    @CsvBindByName(column = "DESCRIPTION")
    private String description;

    @CsvBindByName(column = "PRICE", required = true)
    @NotNull(message = "Price is required")
    private String price;   // keep as String for custom parsing/error message

    @CsvBindByName(column = "STOCK", required = true)
    @NotNull
    private String stock;

    @CsvBindByName(column = "CATEGORY", required = true)
    @NotBlank(message = "Category is required")
    private String category;

    @CsvBindByName(column = "BRAND")
    private String brand;

    @CsvBindByName(column = "ACTIVE")
    private String active;  // "true"/"false"/"1"/"0"/"yes"/"no"
}
```

### CSV import service

```java
package com.example.dataimport.service;

import com.example.dataimport.domain.Product;
import com.example.dataimport.dto.ProductCsvRow;
import com.example.dataimport.repository.ProductRepository;
import com.opencsv.bean.*;
import com.opencsv.exceptions.CsvDataTypeMismatchException;
import com.opencsv.exceptions.CsvRequiredFieldEmptyException;
import jakarta.validation.*;
import lombok.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.*;
import java.math.BigDecimal;
import java.nio.charset.StandardCharsets;
import java.util.*;

@Slf4j
@Service
@RequiredArgsConstructor
public class CsvImportService {

    private final ProductRepository productRepository;
    private final Validator validator;

    @Transactional
    public ImportResult importProducts(InputStream csvStream) throws IOException {

        List<ImportError> errors  = new ArrayList<>();
        int successCount = 0;
        int rowNumber    = 1; // 1 = header

        try (Reader reader = new InputStreamReader(csvStream, StandardCharsets.UTF_8)) {
            CsvToBean<ProductCsvRow> csvToBean = new CsvToBeanBuilder<ProductCsvRow>(reader)
                .withType(ProductCsvRow.class)
                .withIgnoreLeadingWhiteSpace(true)
                .withIgnoreEmptyLine(true)
                .build();

            Iterator<ProductCsvRow> rowIter = csvToBean.iterator();
            while (rowIter.hasNext()) {
                rowNumber++;
                ProductCsvRow row;
                try {
                    row = rowIter.next();
                } catch (Exception e) {
                    errors.add(new ImportError(rowNumber, "PARSE_ERROR", e.getMessage()));
                    continue;
                }

                // Bean Validation
                Set<ConstraintViolation<ProductCsvRow>> violations = validator.validate(row);
                if (!violations.isEmpty()) {
                    violations.forEach(v -> errors.add(new ImportError(
                        rowNumber, v.getPropertyPath().toString(), v.getMessage())));
                    continue;
                }

                // Business validation
                List<String> bizErrors = validateBusiness(row, rowNumber);
                if (!bizErrors.isEmpty()) {
                    bizErrors.forEach(msg -> errors.add(new ImportError(rowNumber, "BUSINESS", msg)));
                    continue;
                }

                // Convert and persist
                try {
                    Product product = toProduct(row);
                    productRepository.save(product);
                    successCount++;
                } catch (Exception e) {
                    errors.add(new ImportError(rowNumber, "DB_ERROR", e.getMessage()));
                }
            }
        }

        return ImportResult.builder()
            .totalRows(rowNumber - 1)
            .successCount(successCount)
            .errorCount(errors.size())
            .errors(errors)
            .build();
    }

    private List<String> validateBusiness(ProductCsvRow row, int rowNum) {
        List<String> errs = new ArrayList<>();

        try {
            BigDecimal price = new BigDecimal(row.getPrice());
            if (price.compareTo(BigDecimal.ZERO) <= 0)
                errs.add("Price must be positive");
        } catch (NumberFormatException e) {
            errs.add("Price is not a valid number: " + row.getPrice());
        }

        try {
            int stock = Integer.parseInt(row.getStock());
            if (stock < 0) errs.add("Stock cannot be negative");
        } catch (NumberFormatException e) {
            errs.add("Stock is not a valid integer: " + row.getStock());
        }

        if (productRepository.existsBySku(row.getSku())) {
            errs.add("SKU already exists: " + row.getSku());
        }

        return errs;
    }

    private Product toProduct(ProductCsvRow row) {
        return Product.builder()
            .sku(row.getSku().trim())
            .name(row.getName().trim())
            .description(row.getDescription())
            .price(new BigDecimal(row.getPrice()))
            .stock(Integer.parseInt(row.getStock()))
            .category(row.getCategory())
            .brand(row.getBrand())
            .active(parseBoolean(row.getActive(), true))
            .build();
    }

    private boolean parseBoolean(String value, boolean defaultValue) {
        if (value == null || value.isBlank()) return defaultValue;
        return "true".equalsIgnoreCase(value) || "1".equals(value) || "yes".equalsIgnoreCase(value);
    }
}
```

### Import result and error records

```java
package com.example.dataimport.service;

import lombok.*;
import java.util.List;

@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class ImportResult {
    private int totalRows;
    private int successCount;
    private int errorCount;
    private List<ImportError> errors;
    public boolean isSuccess() { return errorCount == 0; }
}

@Data @AllArgsConstructor
class ImportError {
    private int rowNumber;
    private String field;
    private String message;
}
```

---

## 4. CSV Export with OpenCSV

```java
package com.example.dataimport.service;

import com.example.dataimport.domain.Product;
import com.example.dataimport.repository.ProductRepository;
import com.opencsv.CSVWriter;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Service;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class CsvExportService {

    private static final String[] HEADERS = {
        "SKU", "NAME", "DESCRIPTION", "PRICE", "STOCK", "CATEGORY", "BRAND", "ACTIVE"
    };

    private final ProductRepository productRepository;

    /**
     * Write all products to the given OutputStream as CSV.
     * Uses streaming to avoid loading everything into memory at once.
     */
    public void exportProductsToStream(OutputStream out) throws IOException {
        try (Writer writer = new OutputStreamWriter(out, StandardCharsets.UTF_8);
             CSVWriter csvWriter = new CSVWriter(writer)) {

            csvWriter.writeNext(HEADERS);

            int page = 0;
            int size = 1000;
            while (true) {
                List<Product> products =
                    productRepository.findAll(PageRequest.of(page++, size)).getContent();
                if (products.isEmpty()) break;

                for (Product p : products) {
                    csvWriter.writeNext(new String[]{
                        p.getSku(),
                        p.getName(),
                        p.getDescription() == null ? "" : p.getDescription(),
                        p.getPrice().toPlainString(),
                        String.valueOf(p.getStock()),
                        p.getCategory(),
                        p.getBrand() == null ? "" : p.getBrand(),
                        String.valueOf(p.getActive())
                    });
                }
                csvWriter.flush();
            }
        }
    }
}
```

---

## 5. Excel Import/Export with Apache POI

### Excel import

```java
package com.example.dataimport.service;

import com.example.dataimport.domain.Product;
import com.example.dataimport.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.InputStream;
import java.math.BigDecimal;
import java.util.*;

@Slf4j
@Service
@RequiredArgsConstructor
public class ExcelImportService {

    private final ProductRepository productRepository;

    @Transactional
    public ImportResult importProductsFromExcel(InputStream excelStream) throws Exception {

        List<ImportError> errors = new ArrayList<>();
        int successCount = 0;

        try (Workbook workbook = new XSSFWorkbook(excelStream)) {
            Sheet sheet = workbook.getSheet("Products");
            if (sheet == null) sheet = workbook.getSheetAt(0);

            // Read header row to build column index map
            Row header = sheet.getRow(0);
            Map<String, Integer> colIndex = buildColumnIndex(header);

            for (int i = 1; i <= sheet.getLastRowNum(); i++) {
                Row row = sheet.getRow(i);
                if (row == null || isRowEmpty(row)) continue;

                try {
                    Product p = readProductRow(row, colIndex, i + 1);
                    productRepository.save(p);
                    successCount++;
                } catch (ImportRowException e) {
                    errors.add(new ImportError(i + 1, e.getField(), e.getMessage()));
                } catch (Exception e) {
                    errors.add(new ImportError(i + 1, "UNKNOWN", e.getMessage()));
                }
            }
        }

        return ImportResult.builder()
            .successCount(successCount)
            .errorCount(errors.size())
            .errors(errors)
            .build();
    }

    private Map<String, Integer> buildColumnIndex(Row header) {
        Map<String, Integer> map = new HashMap<>();
        if (header == null) return map;
        for (Cell cell : header) {
            map.put(cell.getStringCellValue().trim().toUpperCase(), cell.getColumnIndex());
        }
        return map;
    }

    private Product readProductRow(Row row, Map<String, Integer> idx, int rowNum) {
        String sku  = getString(row, idx, "SKU");
        String name = getString(row, idx, "NAME");
        if (sku.isBlank())  throw new ImportRowException("SKU",  "SKU is required",  rowNum);
        if (name.isBlank()) throw new ImportRowException("NAME", "Name is required", rowNum);

        BigDecimal price;
        try {
            price = BigDecimal.valueOf(getNumeric(row, idx, "PRICE"));
        } catch (Exception e) {
            throw new ImportRowException("PRICE", "Invalid price value", rowNum);
        }

        int stock;
        try {
            stock = (int) getNumeric(row, idx, "STOCK");
        } catch (Exception e) {
            throw new ImportRowException("STOCK", "Invalid stock value", rowNum);
        }

        return Product.builder()
            .sku(sku.trim())
            .name(name.trim())
            .description(getString(row, idx, "DESCRIPTION"))
            .price(price)
            .stock(stock)
            .category(getString(row, idx, "CATEGORY"))
            .brand(getString(row, idx, "BRAND"))
            .active(true)
            .build();
    }

    private String getString(Row row, Map<String, Integer> idx, String col) {
        Integer i = idx.get(col);
        if (i == null) return "";
        Cell cell = row.getCell(i);
        if (cell == null) return "";
        return switch (cell.getCellType()) {
            case STRING  -> cell.getStringCellValue();
            case NUMERIC -> String.valueOf((long) cell.getNumericCellValue());
            case BOOLEAN -> String.valueOf(cell.getBooleanCellValue());
            default      -> "";
        };
    }

    private double getNumeric(Row row, Map<String, Integer> idx, String col) {
        Integer i = idx.get(col);
        if (i == null) throw new IllegalArgumentException("Column not found: " + col);
        Cell cell = row.getCell(i);
        if (cell == null) throw new IllegalArgumentException("Cell is null");
        return cell.getCellType() == CellType.NUMERIC
            ? cell.getNumericCellValue()
            : Double.parseDouble(cell.getStringCellValue());
    }

    private boolean isRowEmpty(Row row) {
        for (Cell c : row) {
            if (c.getCellType() != CellType.BLANK) return false;
        }
        return true;
    }
}
```

```java
package com.example.dataimport.service;

import lombok.*;

@Data @AllArgsConstructor
public class ImportRowException extends RuntimeException {
    private final String field;
    private final int rowNumber;
    public ImportRowException(String field, String message, int rowNumber) {
        super(message);
        this.field = field;
        this.rowNumber = rowNumber;
    }
}
```

### Excel export with styling

```java
package com.example.dataimport.service;

import com.example.dataimport.domain.Product;
import com.example.dataimport.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.*;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Service;

import java.io.*;
import java.util.List;

@Service
@RequiredArgsConstructor
public class ExcelExportService {

    private final ProductRepository productRepository;

    public void exportProductsToExcel(OutputStream out) throws IOException {
        try (XSSFWorkbook workbook = new XSSFWorkbook()) {
            XSSFSheet sheet = workbook.createSheet("Products");

            // Header style
            XSSFCellStyle headerStyle = workbook.createCellStyle();
            XSSFFont headerFont = workbook.createFont();
            headerFont.setBold(true);
            headerFont.setFontHeight(12);
            headerStyle.setFont(headerFont);
            headerStyle.setFillForegroundColor(
                new org.apache.poi.xssf.usermodel.XSSFColor(
                    new byte[]{(byte) 0x1a, (byte) 0x73, (byte) 0xe8}, null));
            headerStyle.setFillPattern(FillPatternType.SOLID_FOREGROUND);

            // Currency style
            XSSFCellStyle currencyStyle = workbook.createCellStyle();
            DataFormat format = workbook.createDataFormat();
            currencyStyle.setDataFormat(format.getFormat("$#,##0.00"));

            // Write header row
            String[] headers = {"ID","SKU","Name","Description","Price","Stock","Category","Brand","Active","Created At"};
            Row headerRow = sheet.createRow(0);
            for (int c = 0; c < headers.length; c++) {
                Cell cell = headerRow.createCell(c);
                cell.setCellValue(headers[c]);
                cell.setCellStyle(headerStyle);
            }

            // Write data rows (paginated to manage memory)
            int rowIdx = 1;
            int page = 0;
            while (true) {
                List<Product> products =
                    productRepository.findAll(PageRequest.of(page++, 500)).getContent();
                if (products.isEmpty()) break;

                for (Product p : products) {
                    Row row = sheet.createRow(rowIdx++);
                    row.createCell(0).setCellValue(p.getId());
                    row.createCell(1).setCellValue(p.getSku());
                    row.createCell(2).setCellValue(p.getName());
                    row.createCell(3).setCellValue(p.getDescription() == null ? "" : p.getDescription());

                    Cell priceCell = row.createCell(4);
                    priceCell.setCellValue(p.getPrice().doubleValue());
                    priceCell.setCellStyle(currencyStyle);

                    row.createCell(5).setCellValue(p.getStock());
                    row.createCell(6).setCellValue(p.getCategory());
                    row.createCell(7).setCellValue(p.getBrand() == null ? "" : p.getBrand());
                    row.createCell(8).setCellValue(p.getActive());
                    row.createCell(9).setCellValue(
                        p.getCreatedAt() != null ? p.getCreatedAt().toString() : "");
                }
            }

            // Auto-size columns
            for (int c = 0; c < headers.length; c++) sheet.autoSizeColumn(c);

            workbook.write(out);
        }
    }
}
```

---

## 6. Spring Batch for Large File Processing

```java
package com.example.dataimport.batch;

import com.example.dataimport.domain.Product;
import com.example.dataimport.dto.ProductCsvRow;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.*;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.*;
import org.springframework.batch.item.file.*;
import org.springframework.batch.item.file.mapping.*;
import org.springframework.batch.item.file.transform.DelimitedLineTokenizer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.FileSystemResource;
import org.springframework.transaction.PlatformTransactionManager;

import java.math.BigDecimal;

@Slf4j
@Configuration
@RequiredArgsConstructor
public class ProductImportBatchConfig {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final ProductItemWriter productItemWriter;

    @Bean
    public Job productImportJob(Step productImportStep) {
        return new JobBuilder("productImportJob", jobRepository)
            .start(productImportStep)
            .build();
    }

    @Bean
    public Step productImportStep(
            @Value("${app.import.chunk-size:500}") int chunkSize,
            ItemReader<ProductCsvRow>     reader,
            ItemProcessor<ProductCsvRow, Product> processor) {

        return new StepBuilder("productImportStep", jobRepository)
            .<ProductCsvRow, Product>chunk(chunkSize, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(productItemWriter)
            .faultTolerant()
            .skip(Exception.class)
            .skipLimit(100)
            .listener(new SkipLogListener())
            .build();
    }

    @Bean
    @StepScope
    public FlatFileItemReader<ProductCsvRow> productCsvReader(
            @Value("#{jobParameters['filePath']}") String filePath) {

        FlatFileItemReader<ProductCsvRow> reader = new FlatFileItemReader<>();
        reader.setResource(new FileSystemResource(filePath));
        reader.setLinesToSkip(1); // skip header

        DefaultLineMapper<ProductCsvRow> lineMapper = new DefaultLineMapper<>();

        DelimitedLineTokenizer tokenizer = new DelimitedLineTokenizer();
        tokenizer.setNames("sku","name","description","price","stock","category","brand","active");
        tokenizer.setDelimiter(",");
        tokenizer.setQuoteCharacter('"');

        BeanWrapperFieldSetMapper<ProductCsvRow> fieldSetMapper = new BeanWrapperFieldSetMapper<>();
        fieldSetMapper.setTargetType(ProductCsvRow.class);

        lineMapper.setLineTokenizer(tokenizer);
        lineMapper.setFieldSetMapper(fieldSetMapper);
        reader.setLineMapper(lineMapper);
        reader.setEncoding("UTF-8");

        return reader;
    }

    @Bean
    public ItemProcessor<ProductCsvRow, Product> productProcessor() {
        return row -> {
            if (row.getSku() == null || row.getSku().isBlank()) return null; // skip

            return Product.builder()
                .sku(row.getSku().trim())
                .name(row.getName().trim())
                .description(row.getDescription())
                .price(new BigDecimal(row.getPrice()))
                .stock(Integer.parseInt(row.getStock()))
                .category(row.getCategory())
                .brand(row.getBrand())
                .active(!"false".equalsIgnoreCase(row.getActive()))
                .build();
        };
    }
}
```

```java
package com.example.dataimport.batch;

import com.example.dataimport.domain.Product;
import com.example.dataimport.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.item.*;
import org.springframework.stereotype.Component;

import java.util.List;

@Slf4j
@Component
@RequiredArgsConstructor
public class ProductItemWriter implements ItemWriter<Product> {

    private final ProductRepository productRepository;

    @Override
    public void write(Chunk<? extends Product> chunk) {
        List<? extends Product> products = chunk.getItems();
        productRepository.saveAll(products);
        log.debug("Wrote {} products", products.size());
    }
}
```

```java
package com.example.dataimport.batch;

import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.SkipListener;
import org.springframework.batch.item.file.FlatFileParseException;

@Slf4j
public class SkipLogListener implements SkipListener<Object, Object> {

    @Override
    public void onSkipInRead(Throwable t) {
        if (t instanceof FlatFileParseException e) {
            log.warn("Skipped malformed CSV row {}: {}", e.getLineNumber(), e.getInput());
        } else {
            log.warn("Skipped on read: {}", t.getMessage());
        }
    }

    @Override
    public void onSkipInProcess(Object item, Throwable t) {
        log.warn("Skipped item during processing: {} — {}", item, t.getMessage());
    }

    @Override
    public void onSkipInWrite(Object item, Throwable t) {
        log.warn("Skipped item during write: {} — {}", item, t.getMessage());
    }
}
```

---

## 7. Async Import with Progress Tracking

```java
package com.example.dataimport.domain;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;

@Entity
@Table(name = "import_jobs")
@Data @NoArgsConstructor @AllArgsConstructor @Builder
public class ImportJob {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String filename;

    @Enumerated(EnumType.STRING)
    private ImportJobStatus status;

    private int totalRows;
    private int processedRows;
    private int successRows;
    private int failedRows;

    @Column(length = 2000)
    private String errorSummary;

    @Column(nullable = false)
    private Instant createdAt;

    private Instant completedAt;

    @PrePersist
    void onCreate() { this.createdAt = Instant.now(); }
}
```

```java
package com.example.dataimport.domain;

public enum ImportJobStatus {
    QUEUED, RUNNING, COMPLETED, FAILED, COMPLETED_WITH_ERRORS
}
```

```java
package com.example.dataimport.service;

import com.example.dataimport.batch.*;
import com.example.dataimport.domain.*;
import com.example.dataimport.repository.ImportJobRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.*;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.*;
import java.nio.file.*;
import java.time.Instant;

@Slf4j
@Service
@RequiredArgsConstructor
public class AsyncImportService {

    private final JobLauncher jobLauncher;
    private final Job productImportJob;
    private final ImportJobRepository importJobRepository;

    @Value("${app.import.upload-dir:/tmp/imports}")
    private String uploadDir;

    /**
     * Save file, create job record, and return job ID immediately.
     * Actual processing happens asynchronously.
     */
    public String submitImport(MultipartFile file) throws IOException {
        // Save file to disk
        Path dir = Paths.get(uploadDir);
        Files.createDirectories(dir);
        String filename = System.currentTimeMillis() + "_" + file.getOriginalFilename();
        Path filePath = dir.resolve(filename);
        file.transferTo(filePath);

        // Create job record
        ImportJob job = ImportJob.builder()
            .filename(file.getOriginalFilename())
            .status(ImportJobStatus.QUEUED)
            .build();
        importJobRepository.save(job);

        // Launch asynchronously
        runImport(job.getId(), filePath.toAbsolutePath().toString());
        return job.getId();
    }

    @Async("importExecutor")
    public void runImport(String jobId, String filePath) {
        ImportJob job = importJobRepository.findById(jobId).orElseThrow();
        job.setStatus(ImportJobStatus.RUNNING);
        importJobRepository.save(job);

        try {
            JobParameters params = new JobParametersBuilder()
                .addString("filePath", filePath)
                .addString("jobId",    jobId)
                .addLong("timestamp",  System.currentTimeMillis())
                .toJobParameters();

            JobExecution execution = jobLauncher.run(productImportJob, params);

            long writeCount = execution.getStepExecutions().stream()
                .mapToLong(StepExecution::getWriteCount).sum();
            long skipCount = execution.getStepExecutions().stream()
                .mapToLong(StepExecution::getSkipCount).sum();

            job.setSuccessRows((int) writeCount);
            job.setFailedRows((int) skipCount);
            job.setCompletedAt(Instant.now());
            job.setStatus(skipCount > 0
                ? ImportJobStatus.COMPLETED_WITH_ERRORS
                : ImportJobStatus.COMPLETED);

        } catch (Exception e) {
            log.error("Import job {} failed", jobId, e);
            job.setStatus(ImportJobStatus.FAILED);
            job.setErrorSummary(e.getMessage());
            job.setCompletedAt(Instant.now());
        }

        importJobRepository.save(job);
    }
}
```

```java
package com.example.dataimport.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class ImportAsyncConfig {

    @Bean(name = "importExecutor")
    public Executor importExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(4);
        executor.setQueueCapacity(20);
        executor.setThreadNamePrefix("import-");
        executor.initialize();
        return executor;
    }
}
```

---

## 8. Import REST Controller

```java
package com.example.dataimport.controller;

import com.example.dataimport.domain.ImportJob;
import com.example.dataimport.repository.ImportJobRepository;
import com.example.dataimport.service.*;
import lombok.RequiredArgsConstructor;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.*;
import java.net.URI;

@RestController
@RequestMapping("/api/v1/import")
@RequiredArgsConstructor
public class ImportController {

    private final AsyncImportService asyncImportService;
    private final ImportJobRepository importJobRepository;
    private final CsvImportService csvImportService;

    /**
     * Async upload — returns 202 Accepted with job status URL.
     */
    @PostMapping(value = "/products/async", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<Void> importAsync(@RequestParam("file") MultipartFile file)
            throws IOException {
        String jobId = asyncImportService.submitImport(file);
        return ResponseEntity.accepted()
            .location(URI.create("/api/v1/import/jobs/" + jobId))
            .build();
    }

    /**
     * Sync upload — returns full result (suitable for small files < 5 MB).
     */
    @PostMapping(value = "/products/sync", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<ImportResult> importSync(@RequestParam("file") MultipartFile file)
            throws IOException {
        ImportResult result = csvImportService.importProducts(file.getInputStream());
        HttpStatus status = result.isSuccess() ? HttpStatus.OK : HttpStatus.MULTI_STATUS;
        return ResponseEntity.status(status).body(result);
    }

    @GetMapping("/jobs/{id}")
    public ResponseEntity<ImportJob> getJobStatus(@PathVariable String id) {
        return importJobRepository.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

```java
package com.example.dataimport.controller;

import com.example.dataimport.service.*;
import lombok.RequiredArgsConstructor;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

import jakarta.servlet.http.HttpServletResponse;

@RestController
@RequestMapping("/api/v1/export")
@RequiredArgsConstructor
public class ExportController {

    private final CsvExportService csvExportService;
    private final ExcelExportService excelExportService;

    @GetMapping("/products/csv")
    public void exportCsv(HttpServletResponse response) throws Exception {
        response.setContentType("text/csv;charset=UTF-8");
        response.setHeader(HttpHeaders.CONTENT_DISPOSITION,
                           "attachment; filename=\"products.csv\"");
        // Flush headers before streaming
        response.flushBuffer();
        csvExportService.exportProductsToStream(response.getOutputStream());
    }

    @GetMapping("/products/excel")
    public void exportExcel(HttpServletResponse response) throws Exception {
        response.setContentType(
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        response.setHeader(HttpHeaders.CONTENT_DISPOSITION,
                           "attachment; filename=\"products.xlsx\"");
        response.flushBuffer();
        excelExportService.exportProductsToExcel(response.getOutputStream());
    }
}
```

---

## 9. Error Report Generation

```java
package com.example.dataimport.service;

import com.example.dataimport.service.ImportError;
import com.opencsv.CSVWriter;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.*;
import org.springframework.stereotype.Service;

import java.io.*;
import java.util.List;

@Service
public class ImportErrorReportService {

    /**
     * Generate a CSV error report for failed import rows.
     */
    public byte[] generateCsvErrorReport(ImportResult result) throws IOException {
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        try (Writer writer = new java.io.OutputStreamWriter(bos);
             CSVWriter csv = new CSVWriter(writer)) {

            csv.writeNext(new String[]{"ROW", "FIELD", "ERROR_MESSAGE"});
            for (ImportError err : result.getErrors()) {
                csv.writeNext(new String[]{
                    String.valueOf(err.getRowNumber()),
                    err.getField(),
                    err.getMessage()
                });
            }
        }
        return bos.toByteArray();
    }

    /**
     * Generate an Excel error report with styling.
     */
    public byte[] generateExcelErrorReport(ImportResult result) throws IOException {
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        try (XSSFWorkbook wb = new XSSFWorkbook()) {
            XSSFSheet sheet = wb.createSheet("Import Errors");

            // Summary row
            Row summary = sheet.createRow(0);
            summary.createCell(0).setCellValue(
                "Total: " + result.getTotalRows() +
                "  Success: " + result.getSuccessCount() +
                "  Errors: " + result.getErrorCount()
            );

            // Header
            Row header = sheet.createRow(2);
            XSSFCellStyle headerStyle = wb.createCellStyle();
            headerStyle.setFillForegroundColor(
                IndexedColors.RED.getIndex());
            headerStyle.setFillPattern(FillPatternType.SOLID_FOREGROUND);

            String[] cols = {"Row #", "Field", "Error Message"};
            for (int c = 0; c < cols.length; c++) {
                Cell cell = header.createCell(c);
                cell.setCellValue(cols[c]);
                cell.setCellStyle(headerStyle);
            }

            // Error rows
            int rowIdx = 3;
            for (ImportError err : result.getErrors()) {
                Row row = sheet.createRow(rowIdx++);
                row.createCell(0).setCellValue(err.getRowNumber());
                row.createCell(1).setCellValue(err.getField());
                row.createCell(2).setCellValue(err.getMessage());
            }

            for (int c = 0; c < cols.length; c++) sheet.autoSizeColumn(c);
            wb.write(bos);
        }
        return bos.toByteArray();
    }
}
```

---

## 10. Database Seed Data Management

```java
package com.example.dataimport.seed;

import com.example.dataimport.domain.Product;
import com.example.dataimport.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Profile;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.util.List;

@Slf4j
@Component
@Order(1)
@Profile({"dev", "test"})   // Only runs in dev and test environments
@RequiredArgsConstructor
public class DatabaseSeeder implements ApplicationRunner {

    private final ProductRepository productRepository;

    @Override
    public void run(ApplicationArguments args) {
        if (productRepository.count() > 0) {
            log.info("Database already seeded — skipping");
            return;
        }

        log.info("Seeding database with sample products...");
        productRepository.saveAll(buildSampleProducts());
        log.info("Database seeding complete");
    }

    private List<Product> buildSampleProducts() {
        return List.of(
            Product.builder().sku("LAPTOP-001").name("ProBook 15").category("Electronics")
                .brand("TechCo").price(BigDecimal.valueOf(999.99)).stock(50).active(true).build(),
            Product.builder().sku("LAPTOP-002").name("UltraBook 13").category("Electronics")
                .brand("TechCo").price(BigDecimal.valueOf(1299.99)).stock(30).active(true).build(),
            Product.builder().sku("MOUSE-001").name("Wireless Mouse").category("Accessories")
                .brand("ClickMaster").price(BigDecimal.valueOf(29.99)).stock(200).active(true).build(),
            Product.builder().sku("KB-001").name("Mechanical Keyboard").category("Accessories")
                .brand("TypeRight").price(BigDecimal.valueOf(79.99)).stock(100).active(true).build(),
            Product.builder().sku("MONITOR-001").name("27\" 4K Monitor").category("Monitors")
                .brand("ViewClear").price(BigDecimal.valueOf(449.99)).stock(25).active(true).build()
        );
    }
}
```

---

## 11. Data Transformation Pipeline

```java
package com.example.dataimport.pipeline;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.List;
import java.util.function.Function;

/**
 * Generic data transformation pipeline.
 * Each transformer is a function that can transform or filter (return null to drop).
 */
@Slf4j
@Component
public class DataPipeline<I, O> {

    private final List<Function<Object, Object>> steps = new ArrayList<>();

    @SuppressWarnings("unchecked")
    public DataPipeline<I, O> addStep(Function<?, ?> step) {
        steps.add((Function<Object, Object>) step);
        return this;
    }

    @SuppressWarnings("unchecked")
    public O process(I input) {
        Object current = input;
        for (Function<Object, Object> step : steps) {
            if (current == null) return null;
            current = step.apply(current);
        }
        return (O) current;
    }

    public List<O> processAll(List<I> inputs) {
        List<O> results = new ArrayList<>();
        for (I input : inputs) {
            O result = process(input);
            if (result != null) results.add(result);
        }
        return results;
    }
}
```

### Example pipeline usage

```java
package com.example.dataimport.service;

import com.example.dataimport.domain.Product;
import com.example.dataimport.dto.ProductCsvRow;
import com.example.dataimport.pipeline.DataPipeline;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;

@Service
public class ProductTransformationService {

    public DataPipeline<ProductCsvRow, Product> buildPipeline() {
        return new DataPipeline<ProductCsvRow, Product>()
            // Step 1: Trim all strings
            .addStep((ProductCsvRow row) -> {
                row.setSku(row.getSku() == null ? null : row.getSku().trim().toUpperCase());
                row.setName(row.getName() == null ? null : row.getName().trim());
                return row;
            })
            // Step 2: Validate required fields (return null to drop)
            .addStep((ProductCsvRow row) -> {
                if (row.getSku() == null || row.getSku().isBlank()) return null;
                if (row.getName() == null || row.getName().isBlank()) return null;
                return row;
            })
            // Step 3: Convert to domain object
            .addStep((ProductCsvRow row) -> Product.builder()
                .sku(row.getSku())
                .name(row.getName())
                .description(row.getDescription())
                .price(new BigDecimal(row.getPrice()))
                .stock(Integer.parseInt(row.getStock()))
                .category(row.getCategory())
                .brand(row.getBrand())
                .active(true)
                .build());
    }
}
```

---

## 12. Integration Tests

```java
package com.example.dataimport.service;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.mock.web.MockMultipartFile;

import java.nio.charset.StandardCharsets;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class CsvImportServiceTest {

    @Autowired CsvImportService csvImportService;

    private static final String CSV_HEADER = "SKU,NAME,DESCRIPTION,PRICE,STOCK,CATEGORY,BRAND,ACTIVE\n";

    @Test
    void validCsvImportsSuccessfully() throws Exception {
        String csv = CSV_HEADER +
            "SKU-001,Widget A,A great widget,9.99,100,Widgets,Acme,true\n" +
            "SKU-002,Widget B,Another widget,19.99,50,Widgets,Acme,true\n";

        ImportResult result = csvImportService.importProducts(
            new java.io.ByteArrayInputStream(csv.getBytes(StandardCharsets.UTF_8)));

        assertThat(result.getSuccessCount()).isEqualTo(2);
        assertThat(result.getErrorCount()).isEqualTo(0);
    }

    @Test
    void invalidPriceCausesRowError() throws Exception {
        String csv = CSV_HEADER +
            "SKU-003,Widget C,Desc,NOT_A_PRICE,100,Widgets,Acme,true\n";

        ImportResult result = csvImportService.importProducts(
            new java.io.ByteArrayInputStream(csv.getBytes(StandardCharsets.UTF_8)));

        assertThat(result.getSuccessCount()).isEqualTo(0);
        assertThat(result.getErrorCount()).isGreaterThan(0);
        assertThat(result.getErrors().get(0).getMessage()).containsIgnoringCase("price");
    }

    @Test
    void emptyCsvImportsZeroRows() throws Exception {
        String csv = CSV_HEADER;

        ImportResult result = csvImportService.importProducts(
            new java.io.ByteArrayInputStream(csv.getBytes(StandardCharsets.UTF_8)));

        assertThat(result.getSuccessCount()).isEqualTo(0);
        assertThat(result.getErrorCount()).isEqualTo(0);
    }
}
```

---

## Summary Table

| Feature | Library | Key Class |
|---|---|---|
| CSV import | OpenCSV | `CsvImportService` |
| CSV export | OpenCSV | `CsvExportService` |
| Excel import | Apache POI | `ExcelImportService` |
| Excel export | Apache POI | `ExcelExportService` |
| Large-file batch | Spring Batch | `ProductImportBatchConfig` |
| Skip/fault-tolerant | Spring Batch `FaultTolerant` | `StepBuilder.faultTolerant()` |
| Async import | `@Async` + `JobLauncher` | `AsyncImportService` |
| Progress tracking | `ImportJob` entity | `ImportJobRepository` |
| Validation | Bean Validation + custom | `CsvImportService.validateBusiness()` |
| Error reports | OpenCSV / POI | `ImportErrorReportService` |
| Data pipeline | Generic `DataPipeline<I,O>` | `DataPipeline` |
| Database seeding | `ApplicationRunner` | `DatabaseSeeder` |

---

## Next Part Preview

**Part 069: Advanced Caching Strategies** — dive deep into Redis data structures, Lua scripting
for atomic inventory operations, multi-level L1 + L2 caching with Caffeine and Redis, cache
stampede prevention, and a complete flash-sale system that handles 10,000+ concurrent users
without overselling.
