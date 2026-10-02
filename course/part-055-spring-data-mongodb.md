# Part 055: Spring Data MongoDB

## Overview

MongoDB is a document-oriented NoSQL database that stores data as flexible, JSON-like BSON documents. Unlike relational databases, MongoDB has no fixed schema — documents in the same collection can have different fields. Spring Data MongoDB provides a rich abstraction: repositories, template operations, and POJO mapping that make MongoDB feel natural in a Spring application.

By the end of this part you will be able to:
- Set up Spring Data MongoDB with Testcontainers
- Map Java objects to MongoDB documents
- Use repositories and MongoTemplate for CRUD and queries
- Write aggregation pipelines
- Store and retrieve files with GridFS
- Handle transactions and implement change streams

---

## Table of Contents

1. [MongoDB Core Concepts](#1-mongodb-core-concepts)
2. [Project Setup](#2-project-setup)
3. [Document Annotations](#3-document-annotations)
4. [MongoRepository](#4-mongorepository)
5. [MongoTemplate — Full Control](#5-mongotemplate--full-control)
6. [Aggregation Pipeline](#6-aggregation-pipeline)
7. [GridFS for File Storage](#7-gridfs-for-file-storage)
8. [Transactions in MongoDB](#8-transactions-in-mongodb)
9. [Change Streams (Reactive CDC)](#9-change-streams-reactive-cdc)
10. [Indexing and Performance](#10-indexing-and-performance)
11. [Real Example: E-commerce Catalog with Aggregation](#11-real-example-e-commerce-catalog-with-aggregation)
12. [Summary](#12-summary)

---

## 1. MongoDB Core Concepts

### Terminology

| MongoDB          | SQL Equivalent  | Description                               |
|------------------|-----------------|-------------------------------------------|
| Database         | Database        | Container for collections                 |
| Collection       | Table           | Group of documents                        |
| Document         | Row             | BSON object (key-value pairs)             |
| Field            | Column          | Key in a BSON document                    |
| `_id`            | Primary key     | Unique identifier (ObjectId by default)   |
| Index            | Index           | Improves query performance                |
| Replica Set      | Master-Slave    | Group of mongod processes for HA          |
| Sharding         | Partitioning    | Horizontal scaling across servers         |

### BSON Types

```javascript
// Example MongoDB document (BSON)
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "Laptop Pro 16",
  "price": NumberDecimal("1299.99"),
  "inStock": true,
  "tags": ["electronics", "laptop", "apple"],
  "specs": {
    "ram": 16,
    "storage": "512GB SSD",
    "display": "16 inch Retina"
  },
  "images": [
    { "url": "https://example.com/img1.jpg", "alt": "Front view" }
  ],
  "createdAt": ISODate("2024-01-15T10:30:00Z")
}
```

---

## 2. Project Setup

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Data MongoDB -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>

    <!-- Spring Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Validation -->
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

    <!-- Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- Testcontainers for MongoDB -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>mongodb</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/ecommerce
      # For authenticated connection:
      # uri: mongodb://user:password@localhost:27017/ecommerce?authSource=admin
      auto-index-creation: true  # auto create @Indexed annotations

logging:
  level:
    org.springframework.data.mongodb.core: DEBUG
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  mongodb:
    image: mongo:7.0
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
      MONGO_INITDB_DATABASE: ecommerce
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
      - ./mongo-init.js:/docker-entrypoint-initdb.d/init.js

  mongo-express:
    image: mongo-express:latest
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: password
      ME_CONFIG_MONGODB_URL: mongodb://admin:password@mongodb:27017/
    depends_on:
      - mongodb

volumes:
  mongo_data:
```

### MongoDB Configuration

```java
package com.example.catalog.config;

import com.mongodb.ReadPreference;
import com.mongodb.WriteConcern;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.mongodb.MongoDatabaseFactory;
import org.springframework.data.mongodb.MongoTransactionManager;
import org.springframework.data.mongodb.config.AbstractMongoClientConfiguration;
import org.springframework.data.mongodb.core.convert.MongoCustomConversions;

import java.util.Arrays;
import java.util.concurrent.TimeUnit;

@Configuration
public class MongoConfig {

    // Enable transactions (requires a replica set)
    @Bean
    public MongoTransactionManager transactionManager(MongoDatabaseFactory dbFactory) {
        return new MongoTransactionManager(dbFactory);
    }
}
```

---

## 3. Document Annotations

### Main Document Class

```java
package com.example.catalog.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.*;
import org.springframework.data.mongodb.core.index.CompoundIndex;
import org.springframework.data.mongodb.core.index.CompoundIndexes;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.index.TextIndexed;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;

@Document(collection = "products")
@CompoundIndexes({
    @CompoundIndex(name = "category_brand", def = "{'category': 1, 'brand': 1}"),
    @CompoundIndex(name = "category_price", def = "{'category': 1, 'price': 1}"),
    @CompoundIndex(name = "active_created", def = "{'active': 1, 'createdAt': -1}")
})
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Product {

    @Id
    private String id;  // maps to _id (ObjectId)

    @TextIndexed(weight = 3)  // full-text index with weight
    @Field("name")
    private String name;

    @TextIndexed(weight = 1)
    @Field("description")
    private String description;

    @Indexed
    @Field("category")
    private String category;

    @Indexed
    @Field("brand")
    private String brand;

    @Field("price")
    private BigDecimal price;

    @Field("stock_quantity")
    private int stockQuantity;

    @Field("average_rating")
    private Double averageRating;

    @Field("review_count")
    private int reviewCount;

    @Indexed
    @Field("tags")
    private List<String> tags;

    @Field("images")
    private List<ProductImage> images;

    @Field("specs")
    private Map<String, Object> specs;  // flexible key-value specs

    @Field("variants")
    private List<ProductVariant> variants;  // embedded documents

    @Indexed
    @Field("active")
    private boolean active;

    @Field("slug")
    @Indexed(unique = true, sparse = true)
    private String slug;

    @CreatedDate
    @Field("created_at")
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Field("updated_at")
    private LocalDateTime updatedAt;

    @CreatedBy
    @Field("created_by")
    private String createdBy;

    @Version
    private Long version;  // optimistic locking
}
```

### Embedded Documents

```java
package com.example.catalog.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.mongodb.core.mapping.Field;

import java.math.BigDecimal;
import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductVariant {

    @Field("sku")
    private String sku;

    @Field("name")
    private String name;

    @Field("price")
    private BigDecimal price;

    @Field("stock")
    private int stock;

    @Field("attributes")
    private List<VariantAttribute> attributes;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
class VariantAttribute {
    @Field("key")
    private String key;
    @Field("value")
    private String value;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
class ProductImage {
    @Field("url")
    private String url;
    @Field("alt")
    private String alt;
    @Field("primary")
    private boolean primary;
}
```

### Enabling Auditing

```java
package com.example.catalog.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.AuditorAware;
import org.springframework.data.mongodb.config.EnableMongoAuditing;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;

import java.util.Optional;

@Configuration
@EnableMongoAuditing
public class AuditingConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName);
    }
}
```

---

## 4. MongoRepository

### Repository Interface

```java
package com.example.catalog.repository;

import com.example.catalog.model.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Slice;
import org.springframework.data.mongodb.repository.Aggregation;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

public interface ProductRepository extends MongoRepository<Product, String> {

    // --- Derived query methods ---

    List<Product> findByCategory(String category);

    Page<Product> findByCategory(String category, Pageable pageable);

    List<Product> findByBrandAndActiveIsTrue(String brand);

    List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);

    List<Product> findByTagsContaining(String tag);

    Optional<Product> findBySlug(String slug);

    boolean existsBySlug(String slug);

    long countByCategory(String category);

    Page<Product> findByActiveIsTrueAndCategoryOrderByPriceAsc(
        String category, Pageable pageable
    );

    List<Product> findByAverageRatingGreaterThanEqualAndActiveIsTrue(
        double minRating
    );

    List<Product> findByCategoryInAndActiveIsTrue(List<String> categories);

    // Find by nested field
    List<Product> findByVariantsSku(String sku);

    // Find created after
    List<Product> findByCreatedAtAfter(LocalDateTime date);

    // --- @Query with MongoDB query ---

    @Query("{ 'category': ?0, 'price': { '$lte': ?1 }, 'active': true }")
    List<Product> findByCategoryAndMaxPrice(String category, BigDecimal maxPrice);

    // Field projection — only return name, price, category
    @Query(value = "{ 'category': ?0 }",
           fields = "{ 'name': 1, 'price': 1, 'category': 1, 'brand': 1 }")
    List<Product> findByCategory_Projection(String category);

    // Full-text search using $text
    @Query("{ '$text': { '$search': ?0 } }")
    Page<Product> fullTextSearch(String searchText, Pageable pageable);

    // Delete query
    @Query(value = "{ 'active': false, 'updated_at': { '$lt': ?0 } }",
           delete = true)
    long deleteInactiveOlderThan(LocalDateTime cutoff);

    // --- Aggregation in repository ---

    @Aggregation(pipeline = {
        "{ '$match': { 'active': true } }",
        "{ '$group': { '_id': '$category', 'count': { '$sum': 1 }, " +
        "  'avg_price': { '$avg': '$price' }, 'total_stock': { '$sum': '$stock_quantity' } } }",
        "{ '$sort': { 'count': -1 } }"
    })
    List<CategorySummary> getCategorySummaries();

    @Aggregation(pipeline = {
        "{ '$match': { 'category': ?0 } }",
        "{ '$group': { '_id': '$brand', 'count': { '$sum': 1 }, " +
        "  'min_price': { '$min': '$price' }, 'max_price': { '$max': '$price' } } }",
        "{ '$sort': { 'count': -1 } }",
        "{ '$limit': 10 }"
    })
    List<BrandSummary> getBrandSummaryByCategory(String category);
}
```

### Aggregation Result DTOs

```java
package com.example.catalog.dto;

import lombok.Data;
import org.springframework.data.mongodb.core.mapping.Field;

import java.math.BigDecimal;

@Data
public class CategorySummary {
    @Field("_id")
    private String category;
    private long count;
    @Field("avg_price")
    private double avgPrice;
    @Field("total_stock")
    private long totalStock;
}

@Data
class BrandSummary {
    @Field("_id")
    private String brand;
    private long count;
    @Field("min_price")
    private BigDecimal minPrice;
    @Field("max_price")
    private BigDecimal maxPrice;
}
```

---

## 5. MongoTemplate — Full Control

### CRUD and Queries

```java
package com.example.catalog.service;

import com.example.catalog.model.Product;
import com.example.catalog.dto.ProductUpdateRequest;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.*;
import org.springframework.data.mongodb.core.FindAndModifyOptions;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.*;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.util.List;

@Service
@RequiredArgsConstructor
public class ProductTemplateService {

    private final MongoTemplate mongoTemplate;

    // --- Insert / Save ---

    public Product insert(Product product) {
        return mongoTemplate.insert(product);
    }

    public List<Product> insertAll(List<Product> products) {
        return (List<Product>) mongoTemplate.insertAll(products);
    }

    // --- Find ---

    public Product findById(String id) {
        return mongoTemplate.findById(id, Product.class);
    }

    public List<Product> findByCategory(String category) {
        Query query = new Query(Criteria.where("category").is(category));
        return mongoTemplate.find(query, Product.class);
    }

    public Page<Product> findWithPagination(
            String category, int page, int size, String sortField, Sort.Direction dir) {

        Pageable pageable = PageRequest.of(page, size, Sort.by(dir, sortField));

        Query query = new Query(
            Criteria.where("active").is(true)
                .and("category").is(category)
        ).with(pageable);

        List<Product> products = mongoTemplate.find(query, Product.class);
        long total = mongoTemplate.count(new Query(
            Criteria.where("active").is(true).and("category").is(category)
        ), Product.class);

        return new PageImpl<>(products, pageable, total);
    }

    // --- Complex Criteria ---

    public List<Product> searchProducts(
            String keyword, String category,
            BigDecimal minPrice, BigDecimal maxPrice,
            Double minRating, List<String> tags) {

        Criteria criteria = new Criteria();

        // AND conditions
        if (keyword != null && !keyword.isBlank()) {
            criteria.and("$text").is(new BasicDBObject("$search", keyword));
            // OR use regex:
            // criteria.orOperator(
            //     Criteria.where("name").regex(keyword, "i"),
            //     Criteria.where("description").regex(keyword, "i")
            // );
        }
        if (category != null) {
            criteria.and("category").is(category);
        }
        if (minPrice != null && maxPrice != null) {
            criteria.and("price").gte(minPrice).lte(maxPrice);
        } else if (minPrice != null) {
            criteria.and("price").gte(minPrice);
        } else if (maxPrice != null) {
            criteria.and("price").lte(maxPrice);
        }
        if (minRating != null) {
            criteria.and("average_rating").gte(minRating);
        }
        if (tags != null && !tags.isEmpty()) {
            criteria.and("tags").in(tags);
        }
        criteria.and("active").is(true);

        Query query = new Query(criteria)
            .with(Sort.by(Sort.Direction.DESC, "average_rating"));

        return mongoTemplate.find(query, Product.class);
    }

    // --- Update ---

    // Update single field
    public void updatePrice(String id, BigDecimal newPrice) {
        Query query = new Query(Criteria.where("_id").is(id));
        Update update = new Update()
            .set("price", newPrice)
            .currentDate("updated_at");
        mongoTemplate.updateFirst(query, update, Product.class);
    }

    // Bulk update
    public long activateByCategory(String category) {
        Query query = new Query(Criteria.where("category").is(category));
        Update update = new Update()
            .set("active", true)
            .currentDate("updated_at");
        return mongoTemplate.updateMulti(query, update, Product.class)
            .getModifiedCount();
    }

    // Increment field
    public void incrementReviewCount(String id, double newRating) {
        Query query = new Query(Criteria.where("_id").is(id));
        Update update = new Update()
            .inc("review_count", 1);
        // Recalculate average — for real use, compute in aggregation
        mongoTemplate.updateFirst(query, update, Product.class);
    }

    // Push to array
    public void addTag(String id, String tag) {
        Query query = new Query(Criteria.where("_id").is(id));
        Update update = new Update()
            .addToSet("tags", tag);  // addToSet = no duplicates
        mongoTemplate.updateFirst(query, update, Product.class);
    }

    // Pull from array
    public void removeTag(String id, String tag) {
        Query query = new Query(Criteria.where("_id").is(id));
        Update update = new Update()
            .pull("tags", tag);
        mongoTemplate.updateFirst(query, update, Product.class);
    }

    // Find and modify (atomic, returns old or new doc)
    public Product findAndDecrementStock(String id, int quantity) {
        Query query = new Query(
            Criteria.where("_id").is(id)
                .and("stock_quantity").gte(quantity)
        );
        Update update = new Update()
            .inc("stock_quantity", -quantity)
            .currentDate("updated_at");
        FindAndModifyOptions options = FindAndModifyOptions.options().returnNew(true);
        return mongoTemplate.findAndModify(query, update, options, Product.class);
    }

    // Upsert — insert if not found
    public void upsertBySlug(Product product) {
        Query query = new Query(Criteria.where("slug").is(product.getSlug()));
        Update update = new Update()
            .setOnInsert("created_at", product.getCreatedAt())
            .set("name", product.getName())
            .set("price", product.getPrice())
            .set("updated_at", java.time.LocalDateTime.now());
        mongoTemplate.upsert(query, update, Product.class);
    }

    // --- Delete ---

    public void deleteById(String id) {
        Query query = new Query(Criteria.where("_id").is(id));
        mongoTemplate.remove(query, Product.class);
    }

    // --- Distinct values ---

    public List<String> getAllCategories() {
        return mongoTemplate.findDistinct("category", Product.class, String.class);
    }

    // --- Check existence ---

    public boolean existsBySlug(String slug) {
        Query query = new Query(Criteria.where("slug").is(slug));
        return mongoTemplate.exists(query, Product.class);
    }
}
```

---

## 6. Aggregation Pipeline

### Spring Data Aggregation API

```java
package com.example.catalog.service;

import com.example.catalog.dto.*;
import com.example.catalog.model.Product;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Sort;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.aggregation.*;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.stereotype.Service;

import java.util.List;

import static org.springframework.data.mongodb.core.aggregation.Aggregation.*;

@Service
@RequiredArgsConstructor
public class ProductAggregationService {

    private final MongoTemplate mongoTemplate;

    // --- Category statistics ---

    public List<CategoryStatsDto> getCategoryStatistics() {
        Aggregation agg = newAggregation(
            match(Criteria.where("active").is(true)),
            group("category")
                .count().as("totalProducts")
                .sum("stock_quantity").as("totalStock")
                .avg("price").as("averagePrice")
                .min("price").as("minPrice")
                .max("price").as("maxPrice")
                .avg("average_rating").as("avgRating"),
            project("totalProducts", "totalStock", "averagePrice", "minPrice", "maxPrice", "avgRating")
                .and("_id").as("category"),
            sort(Sort.Direction.DESC, "totalProducts"),
            limit(20)
        );

        AggregationResults<CategoryStatsDto> results =
            mongoTemplate.aggregate(agg, "products", CategoryStatsDto.class);

        return results.getMappedResults();
    }

    // --- Price bucket aggregation ---

    public List<PriceBucketDto> getPriceBuckets() {
        BucketOperation bucket = Aggregation.bucket("price")
            .withBoundaries(0, 50, 100, 250, 500, 1000)
            .withDefaultBucket("1000+")
            .andOutputCount().as("count")
            .andOutputExpression("$price").apply(AccumulatorOperators.Sum.sumOf("price")).as("totalValue");

        Aggregation agg = newAggregation(
            match(Criteria.where("active").is(true)),
            bucket
        );

        return mongoTemplate.aggregate(agg, "products", PriceBucketDto.class)
            .getMappedResults();
    }

    // --- Top products by rating per category ---

    public List<TopProductsDto> getTopProductsByCategory(int topN) {
        Aggregation agg = newAggregation(
            match(Criteria.where("active").is(true)
                .and("average_rating").gte(3.0)
                .and("review_count").gte(5)),
            sort(Sort.Direction.DESC, "average_rating"),
            group("category")
                .push(
                    new BasicDBObject("id", "$_id")
                        .append("name", "$name")
                        .append("price", "$price")
                        .append("rating", "$average_rating")
                        .append("reviews", "$review_count")
                ).as("products"),
            project()
                .and("_id").as("category")
                .and("products").slice(topN).as("topProducts")
        );

        return mongoTemplate.aggregate(agg, "products", TopProductsDto.class)
            .getMappedResults();
    }

    // --- Monthly sales/product trends ---

    public List<MonthlyTrendDto> getMonthlyProductTrend(int year) {
        Aggregation agg = newAggregation(
            match(Criteria.where("created_at").gte(
                java.time.LocalDateTime.of(year, 1, 1, 0, 0)
            ).lt(
                java.time.LocalDateTime.of(year + 1, 1, 1, 0, 0)
            )),
            project()
                .andExpression("year(created_at)").as("year")
                .andExpression("month(created_at)").as("month")
                .and("price").as("price"),
            group("year", "month")
                .count().as("productCount")
                .sum("price").as("totalValue"),
            sort(Sort.Direction.ASC, "_id.year", "_id.month")
        );

        return mongoTemplate.aggregate(agg, "products", MonthlyTrendDto.class)
            .getMappedResults();
    }

    // --- Unwind and group (variants analysis) ---

    public List<VariantStockDto> getLowStockVariants(int threshold) {
        Aggregation agg = newAggregation(
            match(Criteria.where("active").is(true)),
            unwind("variants"),
            match(Criteria.where("variants.stock").lte(threshold)),
            project()
                .and("_id").as("productId")
                .and("name").as("productName")
                .and("variants.sku").as("sku")
                .and("variants.name").as("variantName")
                .and("variants.stock").as("stock"),
            sort(Sort.Direction.ASC, "stock")
        );

        return mongoTemplate.aggregate(agg, "products", VariantStockDto.class)
            .getMappedResults();
    }

    // --- Lookup (join) ---

    public List<ProductWithReviewsDto> getProductsWithReviews(String category) {
        Aggregation agg = newAggregation(
            match(Criteria.where("category").is(category).and("active").is(true)),
            LookupOperation.newLookup()
                .from("reviews")
                .localField("_id")
                .foreignField("product_id")
                .as("reviews"),
            project()
                .and("name").as("name")
                .and("price").as("price")
                .and("reviews").as("reviews")
                .and("reviews").size().as("reviewCount"),
            sort(Sort.Direction.DESC, "reviewCount")
        );

        return mongoTemplate.aggregate(agg, "products", ProductWithReviewsDto.class)
            .getMappedResults();
    }

    // --- Faceted search aggregation ---

    public FacetedResultDto getFacetedResults(String keyword, String category) {
        MatchOperation baseMatch = keyword != null
            ? match(Criteria.where("$text").is(new BasicDBObject("$search", keyword)))
            : match(Criteria.where("active").is(true));

        Aggregation agg = newAggregation(
            baseMatch,
            facet()
                .and(
                    match(Criteria.where("active").is(true)),
                    group("category").count().as("count"),
                    sort(Sort.Direction.DESC, "count")
                ).as("categoryFacets")
                .and(
                    match(Criteria.where("active").is(true)),
                    group("brand").count().as("count"),
                    sort(Sort.Direction.DESC, "count"),
                    limit(10)
                ).as("brandFacets")
                .and(
                    bucket("price")
                        .withBoundaries(0, 50, 200, 1000)
                        .withDefaultBucket("over_1000")
                        .andOutputCount().as("count")
                ).as("priceFacets")
        );

        AggregationResults<FacetedResultDto> results =
            mongoTemplate.aggregate(agg, "products", FacetedResultDto.class);

        return results.getUniqueMappedResult();
    }
}
```

---

## 7. GridFS for File Storage

### Configuration and Service

```java
package com.example.catalog.service;

import com.mongodb.client.gridfs.model.GridFSFile;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.bson.types.ObjectId;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.data.mongodb.gridfs.GridFsOperations;
import org.springframework.data.mongodb.gridfs.GridFsTemplate;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class GridFsService {

    private final GridFsTemplate gridFsTemplate;
    private final GridFsOperations gridFsOperations;

    public String storeFile(MultipartFile file, String productId) throws IOException {
        // Build metadata
        org.bson.Document metadata = new org.bson.Document()
            .append("productId", productId)
            .append("contentType", file.getContentType())
            .append("originalName", file.getOriginalFilename());

        ObjectId id = gridFsTemplate.store(
            file.getInputStream(),
            file.getOriginalFilename(),
            file.getContentType(),
            metadata
        );

        log.info("Stored file {} with id {}", file.getOriginalFilename(), id);
        return id.toString();
    }

    public GridFsResource loadFile(String fileId) {
        GridFSFile file = gridFsTemplate.findOne(
            new Query(Criteria.where("_id").is(new ObjectId(fileId)))
        );

        if (file == null) {
            throw new RuntimeException("File not found: " + fileId);
        }

        return new GridFsResource(file, gridFsOperations.getResource(file));
    }

    public List<GridFSFile> listFilesForProduct(String productId) {
        List<GridFSFile> files = new ArrayList<>();
        gridFsTemplate.find(
            new Query(Criteria.where("metadata.productId").is(productId))
        ).forEach(files::add);
        return files;
    }

    public void deleteFile(String fileId) {
        gridFsTemplate.delete(
            new Query(Criteria.where("_id").is(new ObjectId(fileId)))
        );
        log.info("Deleted file {}", fileId);
    }

    public void deleteAllForProduct(String productId) {
        gridFsTemplate.delete(
            new Query(Criteria.where("metadata.productId").is(productId))
        );
    }

    public record GridFsResource(
        GridFSFile file,
        org.springframework.data.mongodb.gridfs.GridFsResource resource
    ) {}
}
```

### File Upload Controller

```java
package com.example.catalog.controller;

import com.example.catalog.service.GridFsService;
import lombok.RequiredArgsConstructor;
import org.springframework.core.io.InputStreamResource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.Map;

@RestController
@RequestMapping("/api/v1/products/{productId}/images")
@RequiredArgsConstructor
public class ProductImageController {

    private final GridFsService gridFsService;

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<Map<String, String>> upload(
            @PathVariable String productId,
            @RequestParam("file") MultipartFile file) throws IOException {

        String fileId = gridFsService.storeFile(file, productId);
        return ResponseEntity.ok(Map.of("fileId", fileId));
    }

    @GetMapping("/{fileId}")
    public ResponseEntity<InputStreamResource> download(
            @PathVariable String productId,
            @PathVariable String fileId) throws IOException {

        GridFsService.GridFsResource resource = gridFsService.loadFile(fileId);
        String contentType = (String) resource.file().getMetadata().get("contentType");

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"" + resource.file().getFilename() + "\"")
            .contentType(MediaType.parseMediaType(contentType != null ? contentType : "application/octet-stream"))
            .contentLength(resource.file().getLength())
            .body(new InputStreamResource(resource.resource().getInputStream()));
    }

    @DeleteMapping("/{fileId}")
    public ResponseEntity<Void> delete(
            @PathVariable String productId,
            @PathVariable String fileId) {

        gridFsService.deleteFile(fileId);
        return ResponseEntity.noContent().build();
    }
}
```

---

## 8. Transactions in MongoDB

MongoDB supports multi-document ACID transactions on replica sets (and sharded clusters in 4.2+).

### Transaction Service

```java
package com.example.catalog.service;

import com.example.catalog.model.Product;
import com.example.catalog.model.Order;
import com.example.catalog.repository.ProductRepository;
import com.example.catalog.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.mongodb.MongoTransactionManager;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.support.TransactionTemplate;

import java.math.BigDecimal;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderService {

    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;
    private final MongoTemplate mongoTemplate;

    // Declarative transaction
    @Transactional
    public Order createOrder(String customerId, List<OrderItem> items) {
        BigDecimal total = BigDecimal.ZERO;

        for (OrderItem item : items) {
            // Check and decrement stock atomically
            Product product = mongoTemplate.findAndModify(
                new Query(
                    Criteria.where("_id").is(item.getProductId())
                        .and("stock_quantity").gte(item.getQuantity())
                ),
                new Update()
                    .inc("stock_quantity", -item.getQuantity())
                    .currentDate("updated_at"),
                FindAndModifyOptions.options().returnNew(true),
                Product.class
            );

            if (product == null) {
                throw new InsufficientStockException(
                    "Insufficient stock for product: " + item.getProductId()
                );
            }

            total = total.add(product.getPrice().multiply(new BigDecimal(item.getQuantity())));
        }

        Order order = Order.builder()
            .customerId(customerId)
            .items(items)
            .totalAmount(total)
            .status("CREATED")
            .build();

        return orderRepository.save(order);
    }

    // Programmatic transaction (for more control)
    public Order cancelOrder(String orderId) {
        TransactionTemplate txTemplate = new TransactionTemplate(transactionManager);

        return txTemplate.execute(status -> {
            try {
                Order order = orderRepository.findById(orderId)
                    .orElseThrow(() -> new RuntimeException("Order not found: " + orderId));

                if (!"CREATED".equals(order.getStatus())) {
                    throw new IllegalStateException("Cannot cancel order in status: " + order.getStatus());
                }

                // Restore stock
                for (OrderItem item : order.getItems()) {
                    Query q = new Query(Criteria.where("_id").is(item.getProductId()));
                    Update u = new Update().inc("stock_quantity", item.getQuantity());
                    mongoTemplate.updateFirst(q, u, Product.class);
                }

                order.setStatus("CANCELLED");
                return orderRepository.save(order);

            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }

    private final MongoTransactionManager transactionManager;
}
```

---

## 9. Change Streams (Reactive CDC)

Change streams let you subscribe to real-time changes in a MongoDB collection.

### Change Stream Listener

```java
package com.example.catalog.listener;

import com.example.catalog.model.Product;
import com.mongodb.client.model.changestream.ChangeStreamDocument;
import com.mongodb.client.model.changestream.OperationType;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.bson.Document;
import org.springframework.context.event.EventListener;
import org.springframework.data.mongodb.core.ChangeStreamOptions;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.messaging.ChangeStreamRequest;
import org.springframework.data.mongodb.core.messaging.DefaultMessageListenerContainer;
import org.springframework.data.mongodb.core.messaging.MessageListener;
import org.springframework.data.mongodb.core.messaging.MessageListenerContainer;
import org.springframework.stereotype.Component;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

@Slf4j
@Component
@RequiredArgsConstructor
public class ProductChangeStreamListener {

    private final MongoTemplate mongoTemplate;
    private MessageListenerContainer container;

    @PostConstruct
    public void startListening() {
        container = new DefaultMessageListenerContainer(mongoTemplate);
        container.start();

        MessageListener<ChangeStreamDocument<Document>, Product> listener = message -> {
            ChangeStreamDocument<Document> raw = message.getRaw();
            if (raw == null) return;

            OperationType operationType = raw.getOperationType();
            Product product = message.getBody();

            switch (operationType) {
                case INSERT -> {
                    log.info("Product inserted: {}", product != null ? product.getId() : null);
                    // e.g., notify search index
                    onProductInsert(product);
                }
                case UPDATE -> {
                    log.info("Product updated: {}", product != null ? product.getId() : null);
                    onProductUpdate(product);
                }
                case DELETE -> {
                    String deletedId = raw.getDocumentKey() != null
                        ? raw.getDocumentKey().getObjectId("_id").getValue().toString()
                        : null;
                    log.info("Product deleted: {}", deletedId);
                    onProductDelete(deletedId);
                }
                default -> log.debug("Unhandled change: {}", operationType);
            }
        };

        ChangeStreamRequest<Product> request = ChangeStreamRequest.builder(listener)
            .collection("products")
            .options(ChangeStreamOptions.builder()
                .returnFullDocumentOnUpdate()
                .build())
            .build();

        container.register(request, Product.class);
        log.info("Product change stream listener started");
    }

    @PreDestroy
    public void stopListening() {
        if (container != null && container.isRunning()) {
            container.stop();
        }
    }

    private void onProductInsert(Product product) {
        // Sync to Elasticsearch, invalidate cache, etc.
    }

    private void onProductUpdate(Product product) {
        // Update search index
    }

    private void onProductDelete(String productId) {
        // Remove from search index, cascade delete
    }
}
```

### Reactive Change Stream

```java
package com.example.catalog.service;

import com.example.catalog.model.Product;
import lombok.RequiredArgsConstructor;
import org.springframework.data.mongodb.core.ReactiveMongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;

import java.time.Duration;

@Service
@RequiredArgsConstructor
public class ReactiveChangeStreamService {

    private final ReactiveMongoTemplate reactiveMongoTemplate;

    public Flux<Product> watchPriceChanges() {
        return reactiveMongoTemplate
            .changeStream("products",
                org.springframework.data.mongodb.core.ChangeStreamOptions.builder()
                    .filter(Aggregation.newAggregation(
                        Aggregation.match(Criteria.where("operationType").in("insert", "update"))
                    ))
                    .returnFullDocumentOnUpdate()
                    .build(),
                Product.class
            )
            .map(org.springframework.data.mongodb.core.ChangeStreamEvent::getBody)
            .filter(p -> p != null);
    }
}
```

---

## 10. Indexing and Performance

### Programmatic Index Creation

```java
package com.example.catalog.config;

import com.example.catalog.model.Product;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.Sort;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.index.Index;
import org.springframework.data.mongodb.core.index.IndexOperations;
import org.springframework.data.mongodb.core.index.TextIndexDefinition;

@Slf4j
@Configuration
@RequiredArgsConstructor
public class MongoIndexConfig {

    private final MongoTemplate mongoTemplate;

    @Bean
    public ApplicationRunner createIndexes() {
        return args -> {
            IndexOperations indexOps = mongoTemplate.indexOps(Product.class);

            // Single field index
            indexOps.ensureIndex(new Index()
                .on("slug", Sort.Direction.ASC)
                .unique()
                .sparse()
            );

            // Compound index
            indexOps.ensureIndex(new Index()
                .on("category", Sort.Direction.ASC)
                .on("price", Sort.Direction.ASC)
                .named("category_price_idx")
            );

            // Partial index (index only active products)
            indexOps.ensureIndex(new org.springframework.data.mongodb.core.index.PartialIndexFilter(
                new org.springframework.data.mongodb.core.query.Criteria("active").is(true)
            ));

            // TTL index (auto-delete documents after 30 days)
            indexOps.ensureIndex(new Index()
                .on("created_at", Sort.Direction.ASC)
                .expire(30, java.util.concurrent.TimeUnit.DAYS)
                .named("ttl_created_at")
            );

            // Text index for full-text search
            indexOps.ensureIndex(
                new TextIndexDefinition.TextIndexDefinitionBuilder()
                    .onField("name", 3.0f)
                    .onField("description", 1.0f)
                    .onField("tags", 2.0f)
                    .withDefaultLanguage("english")
                    .named("product_text_idx")
                    .build()
            );

            log.info("MongoDB indexes created/verified");
        };
    }
}
```

---

## 11. Real Example: E-commerce Catalog with Aggregation

### Complete REST Controller

```java
package com.example.catalog.controller;

import com.example.catalog.dto.*;
import com.example.catalog.model.Product;
import com.example.catalog.service.*;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;

@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductCatalogController {

    private final ProductCatalogService catalogService;
    private final ProductAggregationService aggregationService;

    @GetMapping
    public ResponseEntity<Page<Product>> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) String brand,
            @RequestParam(required = false) String keyword,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @RequestParam(defaultValue = "createdAt") String sortBy,
            @RequestParam(defaultValue = "DESC") String sortDir) {

        return ResponseEntity.ok(
            catalogService.search(keyword, category, brand, minPrice, maxPrice,
                page, size, sortBy, sortDir)
        );
    }

    @GetMapping("/{id}")
    public ResponseEntity<Product> getById(@PathVariable String id) {
        return catalogService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/slug/{slug}")
    public ResponseEntity<Product> getBySlug(@PathVariable String slug) {
        return catalogService.findBySlug(slug)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<Product> create(@Valid @RequestBody CreateProductRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(catalogService.create(request));
    }

    @PutMapping("/{id}")
    public ResponseEntity<Product> update(
            @PathVariable String id,
            @Valid @RequestBody UpdateProductRequest request) {
        return ResponseEntity.ok(catalogService.update(id, request));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable String id) {
        catalogService.delete(id);
        return ResponseEntity.noContent().build();
    }

    // Aggregation endpoints
    @GetMapping("/analytics/categories")
    public ResponseEntity<List<CategoryStatsDto>> getCategoryStats() {
        return ResponseEntity.ok(aggregationService.getCategoryStatistics());
    }

    @GetMapping("/analytics/top-rated")
    public ResponseEntity<List<TopProductsDto>> getTopRated(
            @RequestParam(defaultValue = "5") int topN) {
        return ResponseEntity.ok(aggregationService.getTopProductsByCategory(topN));
    }

    @GetMapping("/analytics/price-buckets")
    public ResponseEntity<List<PriceBucketDto>> getPriceBuckets() {
        return ResponseEntity.ok(aggregationService.getPriceBuckets());
    }

    @GetMapping("/analytics/low-stock")
    public ResponseEntity<List<VariantStockDto>> getLowStock(
            @RequestParam(defaultValue = "10") int threshold) {
        return ResponseEntity.ok(aggregationService.getLowStockVariants(threshold));
    }
}
```

### Integration Test with Testcontainers

```java
package com.example.catalog;

import com.example.catalog.model.Product;
import com.example.catalog.repository.ProductRepository;
import com.example.catalog.service.ProductAggregationService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@SpringBootTest
class ProductCatalogIntegrationTest {

    @Container
    static MongoDBContainer mongoDBContainer =
        new MongoDBContainer("mongo:7.0");

    @DynamicPropertySource
    static void mongoProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongoDBContainer::getReplicaSetUrl);
    }

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private ProductAggregationService aggregationService;

    @BeforeEach
    void setUp() {
        productRepository.deleteAll();

        List<Product> products = List.of(
            buildProduct("Laptop Pro 16", "Electronics", "Apple", new BigDecimal("1299.99"), 4.5),
            buildProduct("MacBook Air", "Electronics", "Apple", new BigDecimal("999.99"), 4.3),
            buildProduct("Surface Pro", "Electronics", "Microsoft", new BigDecimal("1099.99"), 4.1),
            buildProduct("Java Book", "Books", "O'Reilly", new BigDecimal("49.99"), 4.7),
            buildProduct("Spring in Action", "Books", "Manning", new BigDecimal("44.99"), 4.6)
        );

        productRepository.saveAll(products);
    }

    private Product buildProduct(String name, String category, String brand,
                                  BigDecimal price, double rating) {
        return Product.builder()
            .name(name)
            .category(category)
            .brand(brand)
            .price(price)
            .averageRating(rating)
            .active(true)
            .stockQuantity(50)
            .createdAt(LocalDateTime.now())
            .build();
    }

    @Test
    void shouldFindByCategory() {
        List<Product> books = productRepository.findByCategory("Books");
        assertThat(books).hasSize(2);
        assertThat(books).allMatch(p -> "Books".equals(p.getCategory()));
    }

    @Test
    void shouldGetCategoryStatistics() {
        var stats = aggregationService.getCategoryStatistics();
        assertThat(stats).hasSize(2);
        assertThat(stats).anyMatch(s -> "Electronics".equals(s.getCategory()));
        assertThat(stats).anyMatch(s -> "Books".equals(s.getCategory()));
    }

    @Test
    void shouldFindByPriceRange() {
        List<Product> affordable = productRepository.findByPriceBetween(
            new BigDecimal("40"), new BigDecimal("60")
        );
        assertThat(affordable).hasSize(2);
        assertThat(affordable).allMatch(p ->
            p.getPrice().compareTo(new BigDecimal("40")) >= 0 &&
            p.getPrice().compareTo(new BigDecimal("60")) <= 0
        );
    }

    @Test
    void shouldPageResults() {
        var page = productRepository.findAll(
            org.springframework.data.domain.PageRequest.of(0, 3)
        );
        assertThat(page.getTotalElements()).isEqualTo(5);
        assertThat(page.getContent()).hasSize(3);
        assertThat(page.getTotalPages()).isEqualTo(2);
    }
}
```

---

## 12. Summary

| Feature                     | Spring Data MongoDB API                                       |
|-----------------------------|---------------------------------------------------------------|
| Document mapping            | `@Document`, `@Id`, `@Field`, `@Indexed`, `@TextIndexed`     |
| Embedded documents          | Nested POJO classes                                          |
| Simple CRUD                 | `MongoRepository` methods                                    |
| Derived queries             | `findByCategory()`, `findByPriceBetween()`, etc.             |
| Custom query                | `@Query("{ 'field': ?0 }")`                                  |
| Complex operations          | `MongoTemplate` with `Criteria`, `Query`, `Update`           |
| Aggregation pipeline        | `Aggregation.newAggregation(match, group, sort, project)`    |
| Full-text search            | `@TextIndexed` + `$text` query                               |
| File storage                | `GridFsTemplate` / `GridFsOperations`                        |
| Transactions                | `@Transactional` with `MongoTransactionManager`              |
| Change streams              | `MessageListenerContainer` / reactive `changeStream()`       |
| Auditing                    | `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`            |
| Optimistic locking          | `@Version`                                                   |

### Key Takeaways

1. Use embedded documents for data that is always accessed together (images, specs, variants).
2. Use references (`@DBRef` or manual ID) for data accessed independently and at scale.
3. The aggregation pipeline is MongoDB's SQL GROUP BY + JOIN — master it.
4. `@Transactional` requires a replica set (even a single-node replica set in dev).
5. Change streams replace polling — use them for real-time sync between services.

---

## Next Part Preview

**Part 056: Reactive Database Access with R2DBC** — Non-blocking relational database access with PostgreSQL, reactive repositories, DatabaseClient, and combining R2DBC with Spring WebFlux.
