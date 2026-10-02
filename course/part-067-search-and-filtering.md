# Part 067: Advanced Search and Filtering

## Overview

Real-world applications demand flexible, high-performance search. This part walks from simple
JPA queries to production-grade Elasticsearch with faceted search, autocomplete, and dynamic
RSQL filtering — all tied together in a product search service that handles millions of SKUs.

---

## 1. Project Setup

### Maven dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- QueryDSL -->
    <dependency>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-jpa</artifactId>
        <classifier>jakarta</classifier>
        <version>5.1.0</version>
    </dependency>
    <dependency>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-apt</artifactId>
        <classifier>jakarta</classifier>
        <version>5.1.0</version>
        <scope>provided</scope>
    </dependency>

    <!-- RSQL parser -->
    <dependency>
        <groupId>io.github.perplexhub</groupId>
        <artifactId>rsql-jpa-spring-boot-starter</artifactId>
        <version>6.0.20</version>
    </dependency>

    <!-- Spring Data Elasticsearch -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>

<!-- QueryDSL annotation processor -->
<build>
  <plugins>
    <plugin>
      <groupId>com.mysema.maven</groupId>
      <artifactId>apt-maven-plugin</artifactId>
      <version>1.1.3</version>
      <executions>
        <execution>
          <goals><goal>process</goal></goals>
          <configuration>
            <outputDirectory>target/generated-sources/java</outputDirectory>
            <processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

### application.yml

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: ${ES_USER:elastic}
    password: ${ES_PASSWORD:changeme}
    connection-timeout: 3s
    socket-timeout: 30s
  jpa:
    hibernate:
      ddl-auto: validate

app:
  search:
    max-page-size: 100
    autocomplete-size: 10
```

---

## 2. Domain Model

```java
package com.example.search.domain;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.NaturalId;

import java.math.BigDecimal;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "products")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NaturalId
    @Column(nullable = false, unique = true, length = 50)
    private String sku;

    @Column(nullable = false)
    private String name;

    @Column(length = 4000)
    private String description;

    @Column(precision = 12, scale = 2)
    private BigDecimal price;

    @Column(precision = 12, scale = 2)
    private BigDecimal salePrice;

    @Column(nullable = false)
    private Integer stock;

    @Column(nullable = false)
    private Boolean active;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "brand_id")
    private Brand brand;

    @ManyToMany
    @JoinTable(
        name = "product_tags",
        joinColumns = @JoinColumn(name = "product_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private Set<Tag> tags = new HashSet<>();

    @Column
    private Double rating;

    @Column
    private Integer reviewCount;
}
```

---

## 3. JPA Criteria API

```java
package com.example.search.criteria;

import com.example.search.domain.Product;
import jakarta.persistence.EntityManager;
import jakarta.persistence.TypedQuery;
import jakarta.persistence.criteria.*;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.*;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

@Repository
@RequiredArgsConstructor
public class ProductCriteriaRepository {

    private final EntityManager em;

    public Page<Product> search(ProductSearchCriteria criteria, Pageable pageable) {
        CriteriaBuilder cb = em.getCriteriaBuilder();

        // --- Count query ---
        CriteriaQuery<Long> countQuery = cb.createQuery(Long.class);
        Root<Product> countRoot = countQuery.from(Product.class);
        countQuery.select(cb.count(countRoot));
        countQuery.where(buildPredicates(criteria, cb, countRoot));

        long total = em.createQuery(countQuery).getSingleResult();

        // --- Data query ---
        CriteriaQuery<Product> dataQuery = cb.createQuery(Product.class);
        Root<Product> root = dataQuery.from(Product.class);
        root.fetch("category", JoinType.LEFT);
        root.fetch("brand",    JoinType.LEFT);

        dataQuery.select(root);
        dataQuery.where(buildPredicates(criteria, cb, root));
        dataQuery.orderBy(buildOrders(criteria.getSort(), cb, root));

        TypedQuery<Product> typedQuery = em.createQuery(dataQuery);
        typedQuery.setFirstResult((int) pageable.getOffset());
        typedQuery.setMaxResults(pageable.getPageSize());

        List<Product> results = typedQuery.getResultList();
        return new PageImpl<>(results, pageable, total);
    }

    private Predicate[] buildPredicates(ProductSearchCriteria c,
                                         CriteriaBuilder cb, Root<Product> root) {
        List<Predicate> predicates = new ArrayList<>();

        // Always filter active products
        predicates.add(cb.isTrue(root.get("active")));

        if (c.getKeyword() != null && !c.getKeyword().isBlank()) {
            String like = "%" + c.getKeyword().toLowerCase() + "%";
            predicates.add(cb.or(
                cb.like(cb.lower(root.get("name")),        like),
                cb.like(cb.lower(root.get("description")), like),
                cb.like(cb.lower(root.get("sku")),         like)
            ));
        }

        if (c.getCategoryId() != null) {
            predicates.add(cb.equal(root.get("category").get("id"), c.getCategoryId()));
        }

        if (c.getBrandId() != null) {
            predicates.add(cb.equal(root.get("brand").get("id"), c.getBrandId()));
        }

        if (c.getMinPrice() != null) {
            Expression<BigDecimal> effectivePrice = cb.<BigDecimal>selectCase()
                .when(cb.isNotNull(root.get("salePrice")), root.get("salePrice"))
                .otherwise(root.get("price"));
            predicates.add(cb.greaterThanOrEqualTo(effectivePrice, c.getMinPrice()));
        }

        if (c.getMaxPrice() != null) {
            Expression<BigDecimal> effectivePrice = cb.<BigDecimal>selectCase()
                .when(cb.isNotNull(root.get("salePrice")), root.get("salePrice"))
                .otherwise(root.get("price"));
            predicates.add(cb.lessThanOrEqualTo(effectivePrice, c.getMaxPrice()));
        }

        if (c.getInStock() != null && c.getInStock()) {
            predicates.add(cb.greaterThan(root.get("stock"), 0));
        }

        if (c.getMinRating() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("rating"), c.getMinRating()));
        }

        return predicates.toArray(Predicate[]::new);
    }

    private List<Order> buildOrders(String sort, CriteriaBuilder cb, Root<Product> root) {
        return switch (sort == null ? "relevance" : sort) {
            case "price_asc"  -> List.of(cb.asc(root.get("price")));
            case "price_desc" -> List.of(cb.desc(root.get("price")));
            case "rating"     -> List.of(cb.desc(root.get("rating")));
            case "newest"     -> List.of(cb.desc(root.get("id")));
            default           -> List.of(cb.desc(root.get("rating")),
                                         cb.desc(root.get("reviewCount")));
        };
    }
}
```

```java
package com.example.search.criteria;

import lombok.*;
import java.math.BigDecimal;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductSearchCriteria {
    private String keyword;
    private Long   categoryId;
    private Long   brandId;
    private BigDecimal minPrice;
    private BigDecimal maxPrice;
    private Boolean    inStock;
    private Double     minRating;
    private String     sort;       // relevance | price_asc | price_desc | rating | newest
}
```

---

## 4. Spring Data Specifications

```java
package com.example.search.specification;

import com.example.search.domain.Product;
import jakarta.persistence.criteria.*;
import org.springframework.data.jpa.domain.Specification;

import java.math.BigDecimal;

public class ProductSpecifications {

    private ProductSpecifications() {}

    public static Specification<Product> isActive() {
        return (root, q, cb) -> cb.isTrue(root.get("active"));
    }

    public static Specification<Product> keywordMatches(String keyword) {
        return (root, q, cb) -> {
            if (keyword == null || keyword.isBlank()) return cb.conjunction();
            String like = "%" + keyword.toLowerCase() + "%";
            return cb.or(
                cb.like(cb.lower(root.get("name")),        like),
                cb.like(cb.lower(root.get("description")), like)
            );
        };
    }

    public static Specification<Product> inCategory(Long categoryId) {
        return (root, q, cb) ->
            categoryId == null ? cb.conjunction()
                               : cb.equal(root.get("category").get("id"), categoryId);
    }

    public static Specification<Product> byBrand(Long brandId) {
        return (root, q, cb) ->
            brandId == null ? cb.conjunction()
                            : cb.equal(root.get("brand").get("id"), brandId);
    }

    public static Specification<Product> priceBetween(BigDecimal min, BigDecimal max) {
        return (root, q, cb) -> {
            List<Predicate> predicates = new java.util.ArrayList<>();
            if (min != null) predicates.add(cb.greaterThanOrEqualTo(root.get("price"), min));
            if (max != null) predicates.add(cb.lessThanOrEqualTo(root.get("price"), max));
            return cb.and(predicates.toArray(Predicate[]::new));
        };
    }

    public static Specification<Product> inStock() {
        return (root, q, cb) -> cb.greaterThan(root.get("stock"), 0);
    }

    public static Specification<Product> ratingAtLeast(Double min) {
        return (root, q, cb) ->
            min == null ? cb.conjunction()
                        : cb.greaterThanOrEqualTo(root.get("rating"), min);
    }
}
```

```java
package com.example.search.repository;

import com.example.search.domain.Product;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;

public interface ProductRepository
        extends JpaRepository<Product, Long>,
                JpaSpecificationExecutor<Product> {

    @Query("SELECT p FROM Product p JOIN FETCH p.category JOIN FETCH p.brand " +
           "WHERE p.id = :id")
    java.util.Optional<Product> findByIdFull(@Param("id") Long id);
}
```

```java
package com.example.search.service;

import com.example.search.criteria.ProductSearchCriteria;
import com.example.search.domain.Product;
import com.example.search.repository.ProductRepository;
import com.example.search.specification.ProductSpecifications;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.*;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ProductRepository productRepository;

    @Transactional(readOnly = true)
    public Page<Product> search(ProductSearchCriteria criteria, Pageable pageable) {
        Specification<Product> spec = Specification.where(ProductSpecifications.isActive())
            .and(ProductSpecifications.keywordMatches(criteria.getKeyword()))
            .and(ProductSpecifications.inCategory(criteria.getCategoryId()))
            .and(ProductSpecifications.byBrand(criteria.getBrandId()))
            .and(ProductSpecifications.priceBetween(criteria.getMinPrice(), criteria.getMaxPrice()))
            .and(criteria.getInStock() != null && criteria.getInStock()
                     ? ProductSpecifications.inStock() : null)
            .and(ProductSpecifications.ratingAtLeast(criteria.getMinRating()));

        return productRepository.findAll(spec, pageable);
    }
}
```

---

## 5. QueryDSL Integration

QueryDSL generates `QProduct`, `QCategory`, etc. at compile time via the APT plugin.

```java
package com.example.search.querydsl;

import com.example.search.domain.*;
import com.querydsl.core.BooleanBuilder;
import com.querydsl.jpa.impl.JPAQueryFactory;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.*;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.util.List;

@Repository
@RequiredArgsConstructor
public class ProductQueryDslRepository {

    private final JPAQueryFactory queryFactory;

    public Page<Product> search(String keyword,
                                Long categoryId,
                                BigDecimal minPrice,
                                BigDecimal maxPrice,
                                Boolean inStock,
                                Pageable pageable) {

        QProduct product  = QProduct.product;
        QCategory category = QCategory.category;

        BooleanBuilder where = new BooleanBuilder();
        where.and(product.active.isTrue());

        if (keyword != null && !keyword.isBlank()) {
            where.and(product.name.containsIgnoreCase(keyword)
                .or(product.description.containsIgnoreCase(keyword)));
        }

        if (categoryId != null) {
            where.and(product.category.id.eq(categoryId));
        }

        if (minPrice != null) {
            where.and(product.price.goe(minPrice));
        }

        if (maxPrice != null) {
            where.and(product.price.loe(maxPrice));
        }

        if (Boolean.TRUE.equals(inStock)) {
            where.and(product.stock.gt(0));
        }

        long total = queryFactory.selectFrom(product)
                                 .where(where)
                                 .fetchCount();

        List<Product> results = queryFactory
            .selectFrom(product)
            .leftJoin(product.category, category).fetchJoin()
            .leftJoin(product.brand, QBrand.brand).fetchJoin()
            .where(where)
            .orderBy(product.rating.desc().nullsLast())
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .fetch();

        return new PageImpl<>(results, pageable, total);
    }
}
```

```java
package com.example.search.config;

import com.querydsl.jpa.impl.JPAQueryFactory;
import jakarta.persistence.EntityManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class QueryDslConfig {
    @Bean
    JPAQueryFactory jpaQueryFactory(EntityManager em) {
        return new JPAQueryFactory(em);
    }
}
```

---

## 6. RSQL Query Language for REST APIs

RSQL lets API clients send structured filter expressions as query strings:
`GET /api/products?filter=price=lt=100;category.name==Electronics`

```java
package com.example.search.controller;

import com.example.search.domain.Product;
import com.example.search.repository.ProductRepository;
import io.github.perplexhub.rsql.RSQLJPASupport;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.*;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductRsqlController {

    private final ProductRepository productRepository;

    /**
     * Example filter expressions:
     *   price=lt=100
     *   category.name==Electronics
     *   rating=ge=4;stock=gt=0
     *   name=like=*laptop*,name=like=*notebook*
     *
     * Sort: sort=price,asc or sort=rating,desc
     */
    @GetMapping("/search")
    public ResponseEntity<Page<Product>> search(
            @RequestParam(required = false) String filter,
            @RequestParam(defaultValue = "0")  int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "id,asc") String sort) {

        Specification<Product> spec = Specification
            .where(RSQLJPASupport.<Product>toSpecification(filter))
            .and((root, q, cb) -> cb.isTrue(root.get("active")));

        Sort sortOrder = RSQLJPASupport.toSort(sort);
        Pageable pageable = PageRequest.of(page, size, sortOrder);

        return ResponseEntity.ok(productRepository.findAll(spec, pageable));
    }
}
```

### Custom RSQL operators

```java
package com.example.search.rsql;

import io.github.perplexhub.rsql.RSQLCustomPredicateInput;
import io.github.perplexhub.rsql.RSQLJPASupport;
import jakarta.annotation.PostConstruct;
import jakarta.persistence.criteria.*;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.util.List;

@Component
public class RsqlCustomOperators {

    @PostConstruct
    public void registerCustomOperators() {
        // =sale= operator: matches products currently on sale (salePrice != null)
        RSQLJPASupport.addPredicateConverter(new io.github.perplexhub.rsql.RSQLCustomPredicateConverter() {
            @Override
            public boolean supportsOperator(String operator) {
                return "=sale=".equals(operator);
            }

            @Override
            public Predicate convert(RSQLCustomPredicateInput input) {
                Path<?> path = input.getRoot().get("salePrice");
                return "true".equalsIgnoreCase(input.getArguments().get(0))
                    ? path.isNotNull()
                    : path.isNull();
            }
        });
    }
}
```

---

## 7. Elasticsearch Document

```java
package com.example.search.elasticsearch;

import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.*;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;

@Document(indexName = "products")
@Setting(settingPath = "es/product-settings.json")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ProductDocument {

    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "standard")
    private String name;

    @Field(type = FieldType.Text, analyzer = "standard")
    private String description;

    @Field(type = FieldType.Keyword)
    private String sku;

    @Field(type = FieldType.Scaled_Float, scalingFactor = 100)
    private BigDecimal price;

    @Field(type = FieldType.Scaled_Float, scalingFactor = 100)
    private BigDecimal salePrice;

    @Field(type = FieldType.Keyword)
    private String categoryName;

    @Field(type = FieldType.Long)
    private Long categoryId;

    @Field(type = FieldType.Keyword)
    private String brandName;

    @Field(type = FieldType.Long)
    private Long brandId;

    @Field(type = FieldType.Double)
    private Double rating;

    @Field(type = FieldType.Integer)
    private Integer reviewCount;

    @Field(type = FieldType.Integer)
    private Integer stock;

    @Field(type = FieldType.Boolean)
    private Boolean active;

    @Field(type = FieldType.Keyword)
    private List<String> tags;

    @Field(type = FieldType.Date)
    private Instant createdAt;

    // Completion suggester field for autocomplete
    @CompletionField(maxInputLength = 100)
    private Completion nameSuggest;
}
```

### src/main/resources/es/product-settings.json

```json
{
  "analysis": {
    "analyzer": {
      "product_analyzer": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["lowercase", "asciifolding", "stop"]
      },
      "autocomplete_analyzer": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["lowercase", "asciifolding", "edge_ngram_filter"]
      },
      "autocomplete_search_analyzer": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["lowercase", "asciifolding"]
      }
    },
    "filter": {
      "edge_ngram_filter": {
        "type": "edge_ngram",
        "min_gram": 2,
        "max_gram": 20
      }
    }
  }
}
```

---

## 8. Elasticsearch Repository and Queries

```java
package com.example.search.elasticsearch;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;

import java.util.List;

public interface ProductEsRepository
        extends ElasticsearchRepository<ProductDocument, String> {

    Page<ProductDocument> findByActiveTrue(Pageable pageable);

    List<ProductDocument> findByNameContainingIgnoreCase(String name);
}
```

```java
package com.example.search.elasticsearch;

import co.elastic.clients.elasticsearch._types.query_dsl.*;
import co.elastic.clients.elasticsearch._types.*;
import co.elastic.clients.elasticsearch.core.SearchRequest;
import co.elastic.clients.elasticsearch.core.SearchResponse;
import co.elastic.clients.elasticsearch.core.search.*;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.*;
import org.springframework.data.elasticsearch.client.elc.ElasticsearchTemplate;
import org.springframework.data.elasticsearch.core.*;
import org.springframework.data.elasticsearch.core.query.*;
import org.springframework.data.elasticsearch.core.mapping.IndexCoordinates;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.util.*;

@Repository
@RequiredArgsConstructor
public class ProductEsQueryRepository {

    private final ElasticsearchOperations esOperations;

    /**
     * Full bool query: keyword on name/description + filters for category, brand,
     * price range, stock, rating. Returns page of hits with score.
     */
    public SearchHits<ProductDocument> search(EsProductSearchRequest req, Pageable pageable) {

        var boolQuery = new BoolQuery.Builder();

        // --- Must (affects score) ---
        if (req.getKeyword() != null && !req.getKeyword().isBlank()) {
            boolQuery.must(m -> m.multiMatch(mm -> mm
                .query(req.getKeyword())
                .fields("name^3", "description^1", "brandName^2", "tags^2")
                .type(TextQueryType.BestFields)
                .fuzziness("AUTO")
            ));
        } else {
            boolQuery.must(m -> m.matchAll(ma -> ma));
        }

        // --- Filter (does NOT affect score) ---
        boolQuery.filter(f -> f.term(t -> t.field("active").value(true)));

        if (req.getCategoryId() != null) {
            boolQuery.filter(f -> f.term(t -> t
                .field("categoryId")
                .value(req.getCategoryId())));
        }

        if (req.getBrandId() != null) {
            boolQuery.filter(f -> f.term(t -> t
                .field("brandId")
                .value(req.getBrandId())));
        }

        if (req.getMinPrice() != null || req.getMaxPrice() != null) {
            boolQuery.filter(f -> f.range(r -> {
                r.field("price");
                if (req.getMinPrice() != null) r.gte(JsonData.of(req.getMinPrice()));
                if (req.getMaxPrice() != null) r.lte(JsonData.of(req.getMaxPrice()));
                return r;
            }));
        }

        if (Boolean.TRUE.equals(req.getInStock())) {
            boolQuery.filter(f -> f.range(r -> r.field("stock").gt(JsonData.of(0))));
        }

        if (req.getMinRating() != null) {
            boolQuery.filter(f -> f.range(r -> r
                .field("rating")
                .gte(JsonData.of(req.getMinRating()))));
        }

        if (req.getTags() != null && !req.getTags().isEmpty()) {
            boolQuery.filter(f -> f.terms(t -> t
                .field("tags")
                .terms(tv -> tv.value(req.getTags().stream()
                    .map(FieldValue::of)
                    .toList()))));
        }

        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q.bool(boolQuery.build()))
            .withPageable(pageable)
            .build();

        return esOperations.search(query, ProductDocument.class);
    }

    /**
     * Faceted search: return aggregation counts for category, brand, price ranges.
     */
    public FacetedSearchResult facetedSearch(EsProductSearchRequest req, Pageable pageable) {

        var boolQuery = buildBaseQuery(req);

        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q.bool(boolQuery))
            .withPageable(pageable)
            .withAggregation("categories", a -> a.terms(t -> t.field("categoryName").size(20)))
            .withAggregation("brands",     a -> a.terms(t -> t.field("brandName").size(20)))
            .withAggregation("price_ranges", a -> a.range(r -> r
                .field("price")
                .ranges(
                    rv -> rv.key("under_50").to(50.0),
                    rv -> rv.key("50_100").from(50.0).to(100.0),
                    rv -> rv.key("100_250").from(100.0).to(250.0),
                    rv -> rv.key("250_500").from(250.0).to(500.0),
                    rv -> rv.key("over_500").from(500.0)
                )
            ))
            .withAggregation("avg_rating", a -> a.avg(av -> av.field("rating")))
            .build();

        SearchHits<ProductDocument> hits = esOperations.search(query, ProductDocument.class);

        return FacetedSearchResult.builder()
            .hits(hits.getSearchHits())
            .totalHits(hits.getTotalHits())
            .facets(extractFacets(hits.getAggregations()))
            .build();
    }

    private BoolQuery buildBaseQuery(EsProductSearchRequest req) {
        var builder = new BoolQuery.Builder();
        if (req.getKeyword() != null && !req.getKeyword().isBlank()) {
            builder.must(m -> m.multiMatch(mm -> mm
                .query(req.getKeyword())
                .fields("name^3", "description", "brandName^2")
                .fuzziness("AUTO")));
        } else {
            builder.must(m -> m.matchAll(x -> x));
        }
        builder.filter(f -> f.term(t -> t.field("active").value(true)));
        return builder.build();
    }

    private Map<String, List<FacetEntry>> extractFacets(
            org.springframework.data.elasticsearch.core.AggregationsContainer<?> aggs) {
        // Simplified extraction — real impl uses ElasticsearchAggregations
        return Map.of(); // implement per Elasticsearch client version
    }
}
```

```java
package com.example.search.elasticsearch;

import lombok.*;
import java.math.BigDecimal;
import java.util.List;

@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class EsProductSearchRequest {
    private String      keyword;
    private Long        categoryId;
    private Long        brandId;
    private BigDecimal  minPrice;
    private BigDecimal  maxPrice;
    private Boolean     inStock;
    private Double      minRating;
    private List<String> tags;
}
```

```java
package com.example.search.elasticsearch;

import lombok.*;
import org.springframework.data.elasticsearch.core.SearchHit;
import java.util.List;
import java.util.Map;

@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class FacetedSearchResult {
    private List<SearchHit<ProductDocument>> hits;
    private long totalHits;
    private Map<String, List<FacetEntry>> facets;
}

@Data @AllArgsConstructor
class FacetEntry {
    private String key;
    private long count;
}
```

---

## 9. Autocomplete with Elasticsearch

```java
package com.example.search.elasticsearch;

import lombok.RequiredArgsConstructor;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.query.NativeQuery;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class AutocompleteService {

    private final ElasticsearchOperations esOperations;

    /**
     * Prefix-based autocomplete using the completion suggester.
     */
    public List<String> suggest(String prefix, int size) {
        NativeQuery query = NativeQuery.builder()
            .withSuggester(s -> s
                .text(prefix)
                .suggesters("name_suggest", ss -> ss
                    .completion(c -> c
                        .field("nameSuggest")
                        .size(size)
                        .skipDuplicates(true)
                    )
                )
            )
            .build();

        var results = esOperations.suggest(query, ProductDocument.class);
        return results.getSuggestion("name_suggest")
                      .getEntries().stream()
                      .flatMap(e -> e.getOptions().stream())
                      .map(o -> o.getText().toString())
                      .toList();
    }

    /**
     * Edge n-gram based autocomplete (works on regular text field).
     */
    public List<ProductDocument> searchAsYouType(String prefix, int size) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q.matchPhrasePrefix(m -> m
                .field("name")
                .query(prefix)
                .maxExpansions(10)
            ))
            .withMaxResults(size)
            .build();

        return esOperations.search(query, ProductDocument.class)
                           .getSearchHits().stream()
                           .map(h -> h.getContent())
                           .toList();
    }
}
```

---

## 10. Index Sync Service

```java
package com.example.search.elasticsearch;

import com.example.search.domain.Product;
import com.example.search.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.elasticsearch.core.suggest.Completion;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.event.TransactionalEventListener;
import org.springframework.transaction.event.TransactionPhase;

import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductIndexService {

    private final ProductEsRepository esRepository;
    private final ProductRepository productRepository;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onProductSaved(ProductSavedEvent event) {
        productRepository.findByIdFull(event.getProductId()).ifPresent(p -> {
            ProductDocument doc = toDocument(p);
            esRepository.save(doc);
            log.info("Indexed productId={}", p.getId());
        });
    }

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onProductDeleted(ProductDeletedEvent event) {
        esRepository.deleteById(event.getProductId().toString());
    }

    /** Full reindex — run from admin endpoint or on startup. */
    public void reindexAll() {
        log.info("Starting full reindex...");
        esRepository.deleteAll();
        int page = 0;
        int size = 500;
        long total = 0;

        while (true) {
            var chunk = productRepository.findAll(
                org.springframework.data.domain.PageRequest.of(page++, size));
            if (chunk.isEmpty()) break;

            List<ProductDocument> docs = chunk.getContent().stream()
                .map(this::toDocument).toList();
            esRepository.saveAll(docs);
            total += docs.size();
            log.info("Reindexed {} products so far...", total);
        }
        log.info("Full reindex complete. Total indexed: {}", total);
    }

    private ProductDocument toDocument(Product p) {
        Completion suggest = new Completion(new String[]{p.getName()});

        return ProductDocument.builder()
            .id(p.getId().toString())
            .name(p.getName())
            .description(p.getDescription())
            .sku(p.getSku())
            .price(p.getPrice())
            .salePrice(p.getSalePrice())
            .categoryName(p.getCategory() != null ? p.getCategory().getName() : null)
            .categoryId(p.getCategory() != null ? p.getCategory().getId() : null)
            .brandName(p.getBrand() != null ? p.getBrand().getName() : null)
            .brandId(p.getBrand() != null ? p.getBrand().getId() : null)
            .rating(p.getRating())
            .reviewCount(p.getReviewCount())
            .stock(p.getStock())
            .active(p.getActive())
            .tags(p.getTags().stream().map(t -> t.getName()).toList())
            .nameSuggest(suggest)
            .build();
    }
}
```

---

## 11. Search REST Controller

```java
package com.example.search.controller;

import com.example.search.elasticsearch.*;
import com.example.search.service.ProductSearchService;
import com.example.search.criteria.ProductSearchCriteria;
import lombok.*;
import org.springframework.data.domain.*;
import org.springframework.data.elasticsearch.core.SearchHit;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;

@RestController
@RequestMapping("/api/v1/search")
@RequiredArgsConstructor
public class SearchController {

    private final ProductEsQueryRepository esQueryRepository;
    private final AutocompleteService autocompleteService;
    private final ProductSearchService jpaSearchService;

    /**
     * Primary search endpoint (Elasticsearch).
     *
     * GET /api/v1/search/products?q=laptop&category=1&minPrice=100&maxPrice=500
     *                             &inStock=true&sort=price_asc&page=0&size=20
     */
    @GetMapping("/products")
    public ResponseEntity<FacetedSearchResult> search(
            @RequestParam(required = false) String q,
            @RequestParam(required = false) Long category,
            @RequestParam(required = false) Long brand,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @RequestParam(required = false) Boolean inStock,
            @RequestParam(required = false) Double minRating,
            @RequestParam(required = false) List<String> tags,
            @RequestParam(defaultValue = "0")  int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "relevance") String sort) {

        EsProductSearchRequest req = EsProductSearchRequest.builder()
            .keyword(q)
            .categoryId(category)
            .brandId(brand)
            .minPrice(minPrice)
            .maxPrice(maxPrice)
            .inStock(inStock)
            .minRating(minRating)
            .tags(tags)
            .build();

        Sort springSort = switch (sort) {
            case "price_asc"  -> Sort.by("price").ascending();
            case "price_desc" -> Sort.by("price").descending();
            case "rating"     -> Sort.by("rating").descending();
            case "newest"     -> Sort.by("createdAt").descending();
            default           -> Sort.by("_score").descending();
        };

        Pageable pageable = PageRequest.of(page, Math.min(size, 100), springSort);
        return ResponseEntity.ok(esQueryRepository.facetedSearch(req, pageable));
    }

    /**
     * Autocomplete suggestions.
     *
     * GET /api/v1/search/autocomplete?q=lapt
     */
    @GetMapping("/autocomplete")
    public ResponseEntity<List<String>> autocomplete(
            @RequestParam String q,
            @RequestParam(defaultValue = "10") int size) {
        return ResponseEntity.ok(autocompleteService.suggest(q, size));
    }

    /**
     * Search-as-you-type (full document results, not just names).
     */
    @GetMapping("/sayt")
    public ResponseEntity<List<ProductDocument>> searchAsYouType(
            @RequestParam String q,
            @RequestParam(defaultValue = "5") int size) {
        return ResponseEntity.ok(autocompleteService.searchAsYouType(q, size));
    }

    /**
     * JPA fallback search (used when Elasticsearch is unavailable).
     */
    @GetMapping("/products/jpa")
    public ResponseEntity<Page<com.example.search.domain.Product>> jpaSearch(
            @RequestParam(required = false) String q,
            @RequestParam(required = false) Long category,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @RequestParam(defaultValue = "0")  int page,
            @RequestParam(defaultValue = "20") int size) {

        ProductSearchCriteria criteria = ProductSearchCriteria.builder()
            .keyword(q).categoryId(category)
            .minPrice(minPrice).maxPrice(maxPrice)
            .build();

        return ResponseEntity.ok(
            jpaSearchService.search(criteria, PageRequest.of(page, size))
        );
    }
}
```

---

## 12. Search Result Ranking

```java
package com.example.search.elasticsearch;

import org.springframework.data.elasticsearch.core.query.NativeQuery;
import org.springframework.stereotype.Component;

/**
 * Builds queries with custom scoring (Function Score Query).
 * Boosts popular, well-rated, in-stock, and recently created products.
 */
@Component
public class ScoringQueryBuilder {

    public NativeQuery buildScoredQuery(String keyword, Pageable pageable) {
        return NativeQuery.builder()
            .withQuery(q -> q.functionScore(fs -> fs
                .query(inner -> inner.multiMatch(mm -> mm
                    .query(keyword)
                    .fields("name^3", "description^1", "brandName^2")
                    .fuzziness("AUTO")
                ))
                .functions(
                    // Boost highly rated products
                    fn -> fn.filter(f -> f.range(r -> r.field("rating").gte(JsonData.of(4))))
                            .weight(1.5),

                    // Boost in-stock products
                    fn -> fn.filter(f -> f.range(r -> r.field("stock").gt(JsonData.of(0))))
                            .weight(1.3),

                    // Boost products with many reviews (popularity)
                    fn -> fn.fieldValueFactor(fvf -> fvf
                        .field("reviewCount")
                        .factor(0.1)
                        .modifier(FieldValueFactorModifier.Log1p)
                        .missing(0.0)
                    ),

                    // Decay score by age (recent products score higher)
                    fn -> fn.gauss(g -> g
                        .field("createdAt")
                        .placement(p -> p
                            .origin(java.time.Instant.now().toString())
                            .scale("30d")
                            .decay(0.5)
                        )
                    )
                )
                .scoreMode(FunctionScoreMode.Sum)
                .boostMode(FunctionBoostMode.Multiply)
            ))
            .withPageable(pageable)
            .build();
    }
}
```

---

## 13. Testing

```java
package com.example.search.specification;

import com.example.search.domain.Product;
import com.example.search.repository.ProductRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.data.jpa.domain.Specification;

import java.math.BigDecimal;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
class ProductSpecificationsTest {

    @Autowired ProductRepository repository;

    @Test
    void keywordMatchesNameOrDescription() {
        Specification<Product> spec =
            ProductSpecifications.isActive()
                .and(ProductSpecifications.keywordMatches("laptop"));

        List<Product> results = repository.findAll(spec);
        assertThat(results).allMatch(p ->
            p.getName().toLowerCase().contains("laptop") ||
            p.getDescription().toLowerCase().contains("laptop")
        );
    }

    @Test
    void priceBetweenFiltersCorrectly() {
        Specification<Product> spec =
            ProductSpecifications.isActive()
                .and(ProductSpecifications.priceBetween(
                    BigDecimal.valueOf(50), BigDecimal.valueOf(200)));

        List<Product> results = repository.findAll(spec);
        assertThat(results).allMatch(p ->
            p.getPrice().compareTo(BigDecimal.valueOf(50))  >= 0 &&
            p.getPrice().compareTo(BigDecimal.valueOf(200)) <= 0
        );
    }
}
```

---

## Summary Table

| Technique | Library/API | Key Class |
|---|---|---|
| Dynamic SQL queries | JPA Criteria API | `ProductCriteriaRepository` |
| Composable predicates | Spring Data `Specification` | `ProductSpecifications` |
| Type-safe queries | QueryDSL | `ProductQueryDslRepository` |
| REST filter expressions | RSQL (`rsql-jpa`) | `ProductRsqlController` |
| Full-text search | Elasticsearch bool query | `ProductEsQueryRepository` |
| Faceted search | ES aggregations | `facetedSearch()` |
| Autocomplete | Completion suggester | `AutocompleteService` |
| Search-as-you-type | `match_phrase_prefix` | `AutocompleteService` |
| Result scoring | Function Score Query | `ScoringQueryBuilder` |
| Index sync | `@TransactionalEventListener` | `ProductIndexService` |
| Full reindex | Paginated batch insert | `ProductIndexService.reindexAll()` |

---

## Next Part Preview

**Part 068: Data Import and Export** — build a production-ready bulk import pipeline using Spring
Batch for large CSVs, Apache POI for Excel, streaming for memory-efficient processing, async
progress tracking, and a validation-with-error-report pattern for the product catalog import
feature.
