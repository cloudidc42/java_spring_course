# Part 054: Spring Data Elasticsearch

## Overview

Elasticsearch is a distributed, RESTful search and analytics engine built on Apache Lucene. Spring Data Elasticsearch provides a high-level abstraction over the Elasticsearch Java client, letting you use familiar Spring Data patterns (repositories, templates, annotations) while leveraging the full power of Elasticsearch's search capabilities.

By the end of this part you will be able to:
- Set up Spring Data Elasticsearch with Spring Boot
- Map Java objects to Elasticsearch documents
- Perform full-text search, filtering, and faceted search
- Write aggregations and geospatial queries
- Manage indices and handle mapping migrations
- Highlight matching terms in search results

---

## Table of Contents

1. [Elasticsearch Core Concepts](#1-elasticsearch-core-concepts)
2. [Project Setup](#2-project-setup)
3. [Document Mapping with Annotations](#3-document-mapping-with-annotations)
4. [ElasticsearchRepository](#4-elasticsearchrepository)
5. [ElasticsearchOperations and NativeQuery](#5-elasticsearchoperations-and-nativequery)
6. [Full-Text Search Queries](#6-full-text-search-queries)
7. [Aggregations](#7-aggregations)
8. [Highlighting Search Results](#8-highlighting-search-results)
9. [Geospatial Search](#9-geospatial-search)
10. [Index Management and Mapping Migration](#10-index-management-and-mapping-migration)
11. [Real Example: Product Search Engine with Facets](#11-real-example-product-search-engine-with-facets)
12. [Summary](#12-summary)

---

## 1. Elasticsearch Core Concepts

### Fundamental Terms

| Elasticsearch Term | SQL Analogy       | Description                                          |
|-------------------|-------------------|------------------------------------------------------|
| Index             | Table             | Container for documents of a similar type            |
| Document          | Row               | JSON object stored in an index                       |
| Field             | Column            | Key-value pair within a document                     |
| Mapping           | Schema            | Defines field types and indexing rules               |
| Shard             | Partition         | Physical slice of an index for horizontal scaling    |
| Replica           | Backup            | Copy of a shard for fault tolerance                  |

### Inverted Index

Elasticsearch uses an **inverted index** — instead of scanning documents for terms, it maps terms to document IDs:

```
Term "spring" → [doc1, doc3, doc5]
Term "boot"   → [doc1, doc2]
Term "java"   → [doc1, doc3, doc4]
```

This is what makes full-text search extremely fast.

### Query DSL

Elasticsearch queries are expressed as JSON. The two major categories:

- **Leaf queries**: match on a single field (`match`, `term`, `range`)
- **Compound queries**: combine multiple queries (`bool` with `must`, `should`, `must_not`, `filter`)

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "spring boot" } }
      ],
      "filter": [
        { "term": { "category": "framework" } },
        { "range": { "price": { "lte": 100 } } }
      ]
    }
  }
}
```

---

## 2. Project Setup

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Data Elasticsearch -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    </dependency>

    <!-- Spring Web (for REST controllers) -->
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
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>elasticsearch</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: changeme
    connection-timeout: 5s
    socket-timeout: 30s

logging:
  level:
    org.springframework.data.elasticsearch: DEBUG
    tracer: TRACE  # logs raw ES requests/responses
```

### Docker Compose for Local Development

```yaml
# docker-compose.yml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  es_data:
```

---

## 3. Document Mapping with Annotations

### @Document

The `@Document` annotation marks a class as an Elasticsearch document and configures the index.

```java
package com.example.search.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Document(
    indexName = "products",
    createIndex = true,
    shards = 1,
    replicas = 0
)
@Setting(settingPath = "elasticsearch/product-settings.json")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductDocument {

    @Id
    private String id;

    @MultiField(
        mainField = @Field(type = FieldType.Text, analyzer = "english"),
        otherFields = {
            @InnerField(suffix = "keyword", type = FieldType.Keyword),
            @InnerField(suffix = "suggest", type = FieldType.Completion)
        }
    )
    private String name;

    @Field(type = FieldType.Text, analyzer = "english")
    private String description;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Keyword)
    private String brand;

    @Field(type = FieldType.Double)
    private BigDecimal price;

    @Field(type = FieldType.Integer)
    private Integer stockQuantity;

    @Field(type = FieldType.Float)
    private Float averageRating;

    @Field(type = FieldType.Integer)
    private Integer reviewCount;

    @Field(type = FieldType.Keyword)
    private List<String> tags;

    @Field(type = FieldType.Nested)
    private List<AttributeDocument> attributes;

    @GeoPointField
    private GeoPoint location;

    @Field(type = FieldType.Boolean)
    private boolean active;

    @Field(
        type = FieldType.Date,
        format = DateFormat.date_hour_minute_second
    )
    private LocalDateTime createdAt;

    @Field(
        type = FieldType.Date,
        format = DateFormat.date_hour_minute_second
    )
    private LocalDateTime updatedAt;
}
```

### Nested Document

```java
package com.example.search.model;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class AttributeDocument {

    @Field(type = FieldType.Keyword)
    private String name;

    @Field(type = FieldType.Keyword)
    private String value;
}
```

### Custom Index Settings

```json
// src/main/resources/elasticsearch/product-settings.json
{
  "analysis": {
    "analyzer": {
      "english_custom": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": [
          "lowercase",
          "english_stop",
          "english_stemmer",
          "asciifolding"
        ]
      }
    },
    "filter": {
      "english_stop": {
        "type": "stop",
        "stopwords": "_english_"
      },
      "english_stemmer": {
        "type": "stemmer",
        "language": "english"
      }
    }
  }
}
```

### Field Types Reference

```java
// Common @Field type configurations
@Field(type = FieldType.Text)                    // full-text search
@Field(type = FieldType.Keyword)                 // exact match, aggregations
@Field(type = FieldType.Long)                    // integer numbers
@Field(type = FieldType.Double)                  // decimal numbers
@Field(type = FieldType.Boolean)                 // true/false
@Field(type = FieldType.Date, format = DateFormat.date_time)  // dates
@Field(type = FieldType.Object)                  // nested object (flat)
@Field(type = FieldType.Nested)                  // nested object (with sub-queries)
@Field(type = FieldType.Ip)                      // IP addresses
@Field(type = FieldType.Dense_Vector, dims = 768) // vector embeddings
```

---

## 4. ElasticsearchRepository

### Basic Repository

```java
package com.example.search.repository;

import com.example.search.model.ProductDocument;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.annotations.Highlight;
import org.springframework.data.elasticsearch.annotations.HighlightField;
import org.springframework.data.elasticsearch.annotations.Query;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;

import java.math.BigDecimal;
import java.util.List;

public interface ProductSearchRepository
        extends ElasticsearchRepository<ProductDocument, String> {

    // --- Derived queries from method names ---

    // Find by exact category
    List<ProductDocument> findByCategory(String category);

    // Find by category with pagination
    Page<ProductDocument> findByCategory(String category, Pageable pageable);

    // Find active products in price range
    List<ProductDocument> findByActiveIsTrueAndPriceBetween(
        BigDecimal minPrice, BigDecimal maxPrice
    );

    // Find by brand, sorted by price
    List<ProductDocument> findByBrandOrderByPriceAsc(String brand);

    // Full-text search by name
    Page<ProductDocument> findByName(String name, Pageable pageable);

    // Find top rated products
    List<ProductDocument> findByAverageRatingGreaterThanEqual(Float minRating);

    // Find by multiple categories
    List<ProductDocument> findByCategoryIn(List<String> categories);

    // Count by category
    long countByCategory(String category);

    // Delete by category
    void deleteByCategory(String category);

    // --- @Query with Elasticsearch JSON ---

    @Query("""
        {
          "bool": {
            "must": [
              { "match": { "name": "?0" } }
            ],
            "filter": [
              { "term": { "active": true } }
            ]
          }
        }
        """)
    Page<ProductDocument> searchByNameAndActive(String name, Pageable pageable);

    @Query("""
        {
          "multi_match": {
            "query": "?0",
            "fields": ["name^3", "description", "tags"],
            "type": "best_fields",
            "fuzziness": "AUTO"
          }
        }
        """)
    Page<ProductDocument> fullTextSearch(String query, Pageable pageable);

    @Highlight(fields = {
        @HighlightField(name = "name"),
        @HighlightField(name = "description")
    })
    @Query("""
        {
          "match": {
            "name": {
              "query": "?0",
              "fuzziness": "AUTO"
            }
          }
        }
        """)
    SearchHits<ProductDocument> searchWithHighlight(String name);
}
```

### Repository Usage in Service

```java
package com.example.search.service;

import com.example.search.model.ProductDocument;
import com.example.search.repository.ProductSearchRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ProductSearchRepository repository;

    public ProductDocument save(ProductDocument product) {
        return repository.save(product);
    }

    public void saveAll(List<ProductDocument> products) {
        repository.saveAll(products);  // bulk index
    }

    public Optional<ProductDocument> findById(String id) {
        return repository.findById(id);
    }

    public Page<ProductDocument> findByCategory(String category, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("price").ascending());
        return repository.findByCategory(category, pageable);
    }

    public List<ProductDocument> findByPriceRange(BigDecimal min, BigDecimal max) {
        return repository.findByActiveIsTrueAndPriceBetween(min, max);
    }

    public Page<ProductDocument> search(String query, int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return repository.fullTextSearch(query, pageable);
    }

    public void delete(String id) {
        repository.deleteById(id);
    }
}
```

---

## 5. ElasticsearchOperations and NativeQuery

`ElasticsearchOperations` (backed by `ElasticsearchTemplate`) gives you full control over queries.

### Injecting Operations

```java
package com.example.search.service;

import co.elastic.clients.elasticsearch._types.query_dsl.*;
import co.elastic.clients.elasticsearch.core.search.Highlight;
import co.elastic.clients.elasticsearch.core.search.HighlightField;
import com.example.search.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import org.springframework.data.elasticsearch.client.elc.NativeQuery;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHit;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.query.Query;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
@RequiredArgsConstructor
public class AdvancedSearchService {

    private final ElasticsearchOperations elasticsearchOperations;

    // --- Native Query using Elasticsearch Java Client builder ---

    public SearchHits<ProductDocument> searchWithBoolQuery(
            String keyword, String category, Double minPrice, Double maxPrice) {

        Query query = NativeQuery.builder()
            .withQuery(q -> q
                .bool(b -> {
                    // must: affects score
                    b.must(m -> m
                        .multiMatch(mm -> mm
                            .query(keyword)
                            .fields("name^3", "description", "tags")
                            .type(TextQueryType.BestFields)
                            .fuzziness("AUTO")
                        )
                    );

                    // filter: does not affect score, faster
                    if (category != null) {
                        b.filter(f -> f
                            .term(t -> t.field("category").value(category))
                        );
                    }
                    if (minPrice != null || maxPrice != null) {
                        b.filter(f -> f
                            .range(r -> {
                                r.field("price");
                                if (minPrice != null) r.gte(co.elastic.clients.json.JsonData.of(minPrice));
                                if (maxPrice != null) r.lte(co.elastic.clients.json.JsonData.of(maxPrice));
                                return r;
                            })
                        );
                    }

                    // boost active products
                    b.should(s -> s
                        .term(t -> t.field("active").value(true).boost(1.5f))
                    );

                    return b;
                })
            )
            .withHighlightQuery(new org.springframework.data.elasticsearch.core.query.highlight.Highlight(
                List.of(
                    new org.springframework.data.elasticsearch.core.query.highlight.HighlightField("name"),
                    new org.springframework.data.elasticsearch.core.query.highlight.HighlightField("description")
                )
            ))
            .withPageable(org.springframework.data.domain.PageRequest.of(0, 10))
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class);
    }

    // --- Match all with sorting ---

    public SearchHits<ProductDocument> findAll(int page, int size) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q.matchAll(m -> m))
            .withSort(s -> s.field(f -> f.field("createdAt").order(co.elastic.clients.elasticsearch._types.SortOrder.Desc)))
            .withPageable(org.springframework.data.domain.PageRequest.of(page, size))
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class);
    }

    // --- IDs query ---

    public SearchHits<ProductDocument> findByIds(List<String> ids) {
        Query query = NativeQuery.builder()
            .withIds(ids)
            .build();

        return elasticsearchOperations.search(query, ProductDocument.class);
    }

    // --- Extract content from SearchHits ---

    public List<ProductDocument> extractContent(SearchHits<ProductDocument> hits) {
        return hits.getSearchHits().stream()
            .map(SearchHit::getContent)
            .toList();
    }

    public Map<String, List<String>> extractHighlights(SearchHit<ProductDocument> hit) {
        return hit.getHighlightFields();
    }
}
```

---

## 6. Full-Text Search Queries

### Match Query

```java
package com.example.search.service;

import co.elastic.clients.elasticsearch._types.query_dsl.TextQueryType;
import com.example.search.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import org.springframework.data.elasticsearch.client.elc.NativeQuery;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class FullTextSearchService {

    private final ElasticsearchOperations ops;

    // 1. match: single field, analyzed text
    public SearchHits<ProductDocument> matchQuery(String field, String text) {
        var query = NativeQuery.builder()
            .withQuery(q -> q
                .match(m -> m
                    .field(field)
                    .query(text)
                    .analyzer("english")
                    .fuzziness("AUTO")
                    .minimumShouldMatch("75%")
                )
            )
            .build();
        return ops.search(query, ProductDocument.class);
    }

    // 2. multi_match: search across multiple fields
    public SearchHits<ProductDocument> multiMatchQuery(String text) {
        var query = NativeQuery.builder()
            .withQuery(q -> q
                .multiMatch(m -> m
                    .query(text)
                    .fields("name^3", "description^1", "brand^2", "tags^1.5")
                    .type(TextQueryType.BestFields)
                    .tieBreaker(0.3)
                )
            )
            .build();
        return ops.search(query, ProductDocument.class);
    }

    // 3. match_phrase: exact phrase in order
    public SearchHits<ProductDocument> matchPhraseQuery(String field, String phrase) {
        var query = NativeQuery.builder()
            .withQuery(q -> q
                .matchPhrase(m -> m
                    .field(field)
                    .query(phrase)
                    .slop(2)  // allow 2 words between phrase words
                )
            )
            .build();
        return ops.search(query, ProductDocument.class);
    }

    // 4. match_phrase_prefix: autocomplete / prefix search
    public SearchHits<ProductDocument> prefixQuery(String field, String prefix) {
        var query = NativeQuery.builder()
            .withQuery(q -> q
                .matchPhrasePrefix(m -> m
                    .field(field)
                    .query(prefix)
                    .maxExpansions(50)
                )
            )
            .build();
        return ops.search(query, ProductDocument.class);
    }

    // 5. query_string: full Lucene syntax support
    public SearchHits<ProductDocument> queryStringQuery(String queryString) {
        var query = NativeQuery.builder()
            .withQuery(q -> q
                .queryString(qs -> qs
                    .query(queryString)
                    .fields("name", "description", "tags")
                    .defaultOperator(co.elastic.clients.elasticsearch._types.query_dsl.Operator.And)
                    .analyzeWildcard(true)
                )
            )
            .build();
        return ops.search(query, ProductDocument.class);
    }

    // 6. fuzzy: typo-tolerant search
    public SearchHits<ProductDocument> fuzzyQuery(String field, String value) {
        var query = NativeQuery.builder()
            .withQuery(q -> q
                .fuzzy(f -> f
                    .field(field)
                    .value(value)
                    .fuzziness("AUTO")
                    .maxExpansions(50)
                    .prefixLength(1)
                )
            )
            .build();
        return ops.search(query, ProductDocument.class);
    }

    // 7. bool: compound query
    public SearchHits<ProductDocument> boolQuery(String must, String category) {
        var query = NativeQuery.builder()
            .withQuery(q -> q
                .bool(b -> b
                    .must(m -> m.match(mm -> mm.field("name").query(must)))
                    .filter(f -> f.term(t -> t.field("category").value(category)))
                    .filter(f -> f.term(t -> t.field("active").value(true)))
                    .mustNot(mn -> mn.range(r -> r
                        .field("stockQuantity")
                        .lte(co.elastic.clients.json.JsonData.of(0))
                    ))
                    .should(s -> s.range(r -> r
                        .field("averageRating")
                        .gte(co.elastic.clients.json.JsonData.of(4.0))
                        .boost(2.0f)
                    ))
                    .minimumShouldMatch("0")
                )
            )
            .build();
        return ops.search(query, ProductDocument.class);
    }
}
```

---

## 7. Aggregations

### Terms, Date Histogram, and Metric Aggregations

```java
package com.example.search.service;

import co.elastic.clients.elasticsearch._types.aggregations.*;
import com.example.search.model.ProductDocument;
import com.example.search.dto.SearchFacets;
import lombok.RequiredArgsConstructor;
import org.springframework.data.elasticsearch.client.elc.ElasticsearchAggregation;
import org.springframework.data.elasticsearch.client.elc.NativeQuery;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.stereotype.Service;

import java.util.*;

@Service
@RequiredArgsConstructor
public class AggregationService {

    private final ElasticsearchOperations ops;

    // --- Terms Aggregation (faceted search) ---

    public SearchFacets getCategoryFacets(String searchQuery) {
        var query = NativeQuery.builder()
            .withQuery(q -> searchQuery != null
                ? q.multiMatch(m -> m.query(searchQuery).fields("name", "description"))
                : q.matchAll(m -> m)
            )
            // Limit results to 0 since we only need aggregations
            .withMaxResults(0)
            .withAggregation("by_category",
                Aggregation.of(a -> a
                    .terms(t -> t
                        .field("category")
                        .size(20)
                        .minDocCount(1)
                        .order(List.of(NamedValue.of("_count", SortOrder.Desc)))
                    )
                )
            )
            .withAggregation("by_brand",
                Aggregation.of(a -> a
                    .terms(t -> t
                        .field("brand")
                        .size(20)
                    )
                )
            )
            .withAggregation("price_range",
                Aggregation.of(a -> a
                    .range(r -> r
                        .field("price")
                        .ranges(
                            AggregationRange.of(ar -> ar.to("50").key("under_50")),
                            AggregationRange.of(ar -> ar.from("50").to("100").key("50_to_100")),
                            AggregationRange.of(ar -> ar.from("100").to("500").key("100_to_500")),
                            AggregationRange.of(ar -> ar.from("500").key("over_500"))
                        )
                    )
                )
            )
            .withAggregation("avg_price",
                Aggregation.of(a -> a.avg(avg -> avg.field("price")))
            )
            .withAggregation("min_price",
                Aggregation.of(a -> a.min(min -> min.field("price")))
            )
            .withAggregation("max_price",
                Aggregation.of(a -> a.max(max -> max.field("price")))
            )
            .withAggregation("rating_histogram",
                Aggregation.of(a -> a
                    .histogram(h -> h
                        .field("averageRating")
                        .interval(1.0)
                        .minDocCount(1)
                        .extendedBounds(b -> b.min(1.0).max(5.0))
                    )
                )
            )
            .build();

        SearchHits<ProductDocument> hits = ops.search(query, ProductDocument.class);
        return parseFacets(hits);
    }

    // --- Date Histogram Aggregation ---

    public Map<String, Long> getProductCreationTimeline() {
        var query = NativeQuery.builder()
            .withQuery(q -> q.matchAll(m -> m))
            .withMaxResults(0)
            .withAggregation("creation_over_time",
                Aggregation.of(a -> a
                    .dateHistogram(dh -> dh
                        .field("createdAt")
                        .calendarInterval(CalendarInterval.Month)
                        .format("yyyy-MM")
                        .minDocCount(0)
                    )
                )
            )
            .build();

        SearchHits<ProductDocument> hits = ops.search(query, ProductDocument.class);
        Map<String, Long> timeline = new LinkedHashMap<>();

        if (hits.hasAggregations()) {
            var aggs = (ElasticsearchAggregation) hits.getAggregations()
                .get("creation_over_time");
            aggs.aggregation().getAggregate().dateHistogram().buckets().array()
                .forEach(bucket ->
                    timeline.put(bucket.keyAsString(), bucket.docCount())
                );
        }

        return timeline;
    }

    // --- Nested Aggregation ---

    public Map<String, Long> getAttributeValueCounts(String attributeName) {
        var query = NativeQuery.builder()
            .withQuery(q -> q.matchAll(m -> m))
            .withMaxResults(0)
            .withAggregation("attributes_agg",
                Aggregation.of(a -> a
                    .nested(n -> n.path("attributes"))
                    .aggregations("by_name",
                        Aggregation.of(inner -> inner
                            .filter(f -> f
                                .term(t -> t.field("attributes.name").value(attributeName))
                            )
                            .aggregations("by_value",
                                Aggregation.of(innerInner -> innerInner
                                    .terms(t -> t.field("attributes.value").size(50))
                                )
                            )
                        )
                    )
                )
            )
            .build();

        SearchHits<ProductDocument> hits = ops.search(query, ProductDocument.class);
        Map<String, Long> valueCounts = new LinkedHashMap<>();

        if (hits.hasAggregations()) {
            var nestedAgg = (ElasticsearchAggregation) hits.getAggregations().get("attributes_agg");
            var filterAgg = nestedAgg.aggregation().getAggregate().nested()
                .aggregations().get("by_name");
            filterAgg.filter().aggregations().get("by_value")
                .sterms().buckets().array()
                .forEach(b -> valueCounts.put(b.key().stringValue(), b.docCount()));
        }

        return valueCounts;
    }

    private SearchFacets parseFacets(SearchHits<ProductDocument> hits) {
        SearchFacets facets = new SearchFacets();
        facets.setTotalHits(hits.getTotalHits());

        if (!hits.hasAggregations()) return facets;

        // Parse category facets
        var categoryAgg = (ElasticsearchAggregation) hits.getAggregations().get("by_category");
        Map<String, Long> categories = new LinkedHashMap<>();
        categoryAgg.aggregation().getAggregate().sterms().buckets().array()
            .forEach(b -> categories.put(b.key().stringValue(), b.docCount()));
        facets.setCategories(categories);

        // Parse brand facets
        var brandAgg = (ElasticsearchAggregation) hits.getAggregations().get("by_brand");
        Map<String, Long> brands = new LinkedHashMap<>();
        brandAgg.aggregation().getAggregate().sterms().buckets().array()
            .forEach(b -> brands.put(b.key().stringValue(), b.docCount()));
        facets.setBrands(brands);

        // Parse avg price
        var avgPriceAgg = (ElasticsearchAggregation) hits.getAggregations().get("avg_price");
        facets.setAveragePrice(avgPriceAgg.aggregation().getAggregate().avg().value());

        return facets;
    }
}
```

### Facets DTO

```java
package com.example.search.dto;

import lombok.Data;
import java.util.Map;

@Data
public class SearchFacets {
    private long totalHits;
    private Map<String, Long> categories;
    private Map<String, Long> brands;
    private Map<String, Long> priceRanges;
    private double averagePrice;
    private double minPrice;
    private double maxPrice;
}
```

---

## 8. Highlighting Search Results

### Highlight Configuration

```java
package com.example.search.service;

import com.example.search.model.ProductDocument;
import com.example.search.dto.ProductSearchResult;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.elasticsearch.client.elc.NativeQuery;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHit;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.query.highlight.Highlight;
import org.springframework.data.elasticsearch.core.query.highlight.HighlightField;
import org.springframework.data.elasticsearch.core.query.highlight.HighlightFieldParameters;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class HighlightSearchService {

    private final ElasticsearchOperations ops;

    public List<ProductSearchResult> searchWithHighlights(String keyword, int page, int size) {

        // Configure highlight fields
        HighlightFieldParameters nameParams = HighlightFieldParameters.builder()
            .withPreTags("<em class='highlight'>")
            .withPostTags("</em>")
            .withNumberOfFragments(1)
            .withFragmentSize(150)
            .build();

        HighlightFieldParameters descParams = HighlightFieldParameters.builder()
            .withPreTags("<em class='highlight'>")
            .withPostTags("</em>")
            .withNumberOfFragments(3)
            .withFragmentSize(200)
            .withNoMatchSize(100)  // return 100 chars even if no match
            .build();

        Highlight highlight = new Highlight(List.of(
            new HighlightField("name", nameParams),
            new HighlightField("description", descParams),
            new HighlightField("tags", HighlightFieldParameters.builder()
                .withNumberOfFragments(0)  // entire field
                .build()
            )
        ));

        var query = NativeQuery.builder()
            .withQuery(q -> q
                .multiMatch(m -> m
                    .query(keyword)
                    .fields("name^3", "description", "tags^2")
                    .fuzziness("AUTO")
                )
            )
            .withHighlightQuery(highlight)
            .withPageable(PageRequest.of(page, size))
            .build();

        SearchHits<ProductDocument> hits = ops.search(query, ProductDocument.class);

        return hits.getSearchHits().stream()
            .map(this::toSearchResult)
            .collect(Collectors.toList());
    }

    private ProductSearchResult toSearchResult(SearchHit<ProductDocument> hit) {
        ProductDocument product = hit.getContent();
        Map<String, List<String>> highlights = hit.getHighlightFields();

        return ProductSearchResult.builder()
            .product(product)
            .score(hit.getScore())
            .highlightedName(
                highlights.getOrDefault("name", List.of())
                    .stream().findFirst().orElse(product.getName())
            )
            .highlightedDescription(
                highlights.getOrDefault("description", List.of())
            )
            .highlightedTags(
                highlights.getOrDefault("tags", List.of())
            )
            .build();
    }
}
```

### Search Result DTO

```java
package com.example.search.dto;

import com.example.search.model.ProductDocument;
import lombok.Builder;
import lombok.Data;
import java.util.List;

@Data
@Builder
public class ProductSearchResult {
    private ProductDocument product;
    private float score;
    private String highlightedName;
    private List<String> highlightedDescription;
    private List<String> highlightedTags;
}
```

---

## 9. Geospatial Search

### Geo Point Field and Queries

```java
package com.example.search.service;

import com.example.search.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import org.springframework.data.elasticsearch.client.elc.NativeQuery;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.geo.GeoPoint;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class GeoSearchService {

    private final ElasticsearchOperations ops;

    // Find products within a distance from a location
    public SearchHits<ProductDocument> findNearby(
            double lat, double lon, String distance) {

        var query = NativeQuery.builder()
            .withQuery(q -> q
                .bool(b -> b
                    .filter(f -> f
                        .geoDistance(gd -> gd
                            .field("location")
                            .location(l -> l.latlon(ll -> ll.lat(lat).lon(lon)))
                            .distance(distance)  // e.g. "10km", "5mi"
                        )
                    )
                    .filter(af -> af.term(t -> t.field("active").value(true)))
                )
            )
            .withSort(s -> s
                .geoDistance(gd -> gd
                    .field("location")
                    .location(l -> l.latlon(ll -> ll.lat(lat).lon(lon)))
                    .order(co.elastic.clients.elasticsearch._types.SortOrder.Asc)
                    .unit(co.elastic.clients.elasticsearch._types.DistanceUnit.Kilometers)
                )
            )
            .build();

        return ops.search(query, ProductDocument.class);
    }

    // Find within a bounding box
    public SearchHits<ProductDocument> findInBoundingBox(
            double topLeftLat, double topLeftLon,
            double bottomRightLat, double bottomRightLon) {

        var query = NativeQuery.builder()
            .withQuery(q -> q
                .geoBoundingBox(gbb -> gbb
                    .field("location")
                    .boundingBox(bb -> bb
                        .coords(c -> c
                            .tlbr(tlbr -> tlbr
                                .topLeft(tl -> tl.latlon(ll -> ll.lat(topLeftLat).lon(topLeftLon)))
                                .bottomRight(br -> br.latlon(ll -> ll.lat(bottomRightLat).lon(bottomRightLon)))
                            )
                        )
                    )
                )
            )
            .build();

        return ops.search(query, ProductDocument.class);
    }

    // Find within a polygon
    public SearchHits<ProductDocument> findInPolygon(List<GeoPoint> polygon) {
        var geoPoints = polygon.stream()
            .map(p -> co.elastic.clients.elasticsearch._types.GeoLocation.of(
                l -> l.latlon(ll -> ll.lat(p.getLat()).lon(p.getLon()))
            ))
            .toList();

        var query = NativeQuery.builder()
            .withQuery(q -> q
                .geoPolygon(gp -> gp
                    .field("location")
                    .polygon(poly -> poly.points(geoPoints))
                )
            )
            .build();

        return ops.search(query, ProductDocument.class);
    }
}
```

---

## 10. Index Management and Mapping Migration

### IndexOperations

```java
package com.example.search.config;

import com.example.search.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.IndexOperations;
import org.springframework.data.elasticsearch.core.document.Document;
import org.springframework.data.elasticsearch.core.index.AliasAction;
import org.springframework.data.elasticsearch.core.index.AliasActionParameters;
import org.springframework.data.elasticsearch.core.index.AliasActions;

@Slf4j
@Configuration
@RequiredArgsConstructor
public class ElasticsearchIndexConfig {

    private final ElasticsearchOperations elasticsearchOperations;

    @Bean
    public ApplicationRunner indexSetupRunner() {
        return args -> {
            IndexOperations indexOps = elasticsearchOperations
                .indexOps(ProductDocument.class);

            if (!indexOps.exists()) {
                log.info("Creating products index...");
                indexOps.createWithMapping();
                log.info("Products index created successfully");
            } else {
                log.info("Products index already exists");
                // Optionally update mapping for new fields (additive only)
                // indexOps.putMapping();
            }
        };
    }

    // Zero-downtime index migration using aliases
    public void migrateIndex(String newIndexName) {
        IndexOperations indexOps = elasticsearchOperations
            .indexOps(ProductDocument.class);

        // 1. Create new index with new mapping
        Document newMapping = indexOps.createMapping(ProductDocument.class);
        // (create with new index name via custom settings)

        // 2. Reindex old data to new index
        // This typically uses the Reindex API or a data migration job

        // 3. Update alias to point to new index
        AliasActions aliasActions = new AliasActions(
            AliasAction.add(AliasActionParameters.builder()
                .withIndices(newIndexName)
                .withAliases("products")
                .build()),
            AliasAction.remove(AliasActionParameters.builder()
                .withIndices("products_v1")
                .withAliases("products")
                .build())
        );
        indexOps.alias(aliasActions);

        log.info("Index migration to {} completed", newIndexName);
    }

    public void deleteIndex() {
        IndexOperations indexOps = elasticsearchOperations
            .indexOps(ProductDocument.class);
        if (indexOps.exists()) {
            indexOps.delete();
        }
    }

    public void refreshIndex() {
        IndexOperations indexOps = elasticsearchOperations
            .indexOps(ProductDocument.class);
        indexOps.refresh();
    }
}
```

### Bulk Operations

```java
package com.example.search.service;

import com.example.search.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.query.BulkOptions;
import org.springframework.data.elasticsearch.core.query.IndexQuery;
import org.springframework.data.elasticsearch.core.query.IndexQueryBuilder;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class BulkIndexService {

    private final ElasticsearchOperations ops;

    public void bulkIndex(List<ProductDocument> products) {
        List<IndexQuery> queries = products.stream()
            .map(p -> new IndexQueryBuilder()
                .withId(p.getId())
                .withObject(p)
                .build()
            )
            .collect(Collectors.toList());

        BulkOptions bulkOptions = BulkOptions.builder()
            .withRefreshPolicy(
                org.springframework.data.elasticsearch.core.RefreshPolicy.IMMEDIATE
            )
            .build();

        List<String> ids = ops.bulkIndex(queries, bulkOptions, ProductDocument.class);
        log.info("Bulk indexed {} products", ids.size());
    }
}
```

---

## 11. Real Example: Product Search Engine with Facets

### Complete Implementation

#### REST Controller

```java
package com.example.search.controller;

import com.example.search.dto.*;
import com.example.search.service.ProductFacetSearchService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/products/search")
@RequiredArgsConstructor
public class ProductSearchController {

    private final ProductFacetSearchService searchService;

    @GetMapping
    public ResponseEntity<FacetedSearchResponse> search(
            @Valid @ModelAttribute ProductSearchRequest request) {
        return ResponseEntity.ok(searchService.search(request));
    }

    @GetMapping("/suggest")
    public ResponseEntity<List<String>> suggest(
            @RequestParam String prefix,
            @RequestParam(defaultValue = "5") int size) {
        return ResponseEntity.ok(searchService.suggest(prefix, size));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductDocument> getById(@PathVariable String id) {
        return searchService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

#### Search Request DTO

```java
package com.example.search.dto;

import lombok.Data;
import org.springframework.format.annotation.DateTimeFormat;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Data
public class ProductSearchRequest {

    private String query;            // full-text search
    private String category;         // exact filter
    private String brand;            // exact filter
    private List<String> tags;       // any match

    private BigDecimal minPrice;
    private BigDecimal maxPrice;

    private Float minRating;

    private Boolean inStock;         // stockQuantity > 0

    private Double lat;              // geo search
    private Double lon;
    private String distance;         // e.g. "10km"

    @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
    private LocalDateTime createdAfter;

    private String sortBy = "relevance"; // relevance, price_asc, price_desc, rating, newest
    private int page = 0;
    private int size = 20;
}
```

#### Faceted Search Response DTO

```java
package com.example.search.dto;

import com.example.search.model.ProductDocument;
import lombok.Builder;
import lombok.Data;

import java.util.List;
import java.util.Map;

@Data
@Builder
public class FacetedSearchResponse {
    private long totalHits;
    private int page;
    private int size;
    private int totalPages;
    private List<ProductSearchResult> results;

    // Facets
    private Map<String, Long> categoryFacets;
    private Map<String, Long> brandFacets;
    private Map<String, Long> priceRangeFacets;
    private double minPrice;
    private double maxPrice;
    private double avgPrice;

    // Meta
    private long queryTimeMs;
}
```

#### Core Faceted Search Service

```java
package com.example.search.service;

import co.elastic.clients.elasticsearch._types.SortOrder;
import co.elastic.clients.elasticsearch._types.aggregations.*;
import co.elastic.clients.json.JsonData;
import com.example.search.dto.*;
import com.example.search.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.elasticsearch.client.elc.ElasticsearchAggregation;
import org.springframework.data.elasticsearch.client.elc.NativeQuery;
import org.springframework.data.elasticsearch.client.elc.NativeQueryBuilder;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHit;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.query.highlight.Highlight;
import org.springframework.data.elasticsearch.core.query.highlight.HighlightField;
import org.springframework.data.elasticsearch.core.query.highlight.HighlightFieldParameters;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductFacetSearchService {

    private final ElasticsearchOperations ops;
    private final ProductSearchRepository repository;

    public FacetedSearchResponse search(ProductSearchRequest req) {
        long start = System.currentTimeMillis();

        NativeQueryBuilder queryBuilder = NativeQuery.builder()
            .withPageable(PageRequest.of(req.getPage(), req.getSize()));

        // --- Build main query ---
        queryBuilder.withQuery(q -> q
            .bool(b -> {
                // Full-text must clause
                if (req.getQuery() != null && !req.getQuery().isBlank()) {
                    b.must(m -> m
                        .multiMatch(mm -> mm
                            .query(req.getQuery())
                            .fields("name^4", "description^1", "brand^2", "tags^2")
                            .type(co.elastic.clients.elasticsearch._types.query_dsl.TextQueryType.BestFields)
                            .fuzziness("AUTO")
                            .minimumShouldMatch("50%")
                        )
                    );
                } else {
                    b.must(m -> m.matchAll(ma -> ma));
                }

                // Filters (don't affect relevance score)
                if (req.getCategory() != null) {
                    b.filter(f -> f.term(t -> t.field("category").value(req.getCategory())));
                }
                if (req.getBrand() != null) {
                    b.filter(f -> f.term(t -> t.field("brand").value(req.getBrand())));
                }
                if (req.getTags() != null && !req.getTags().isEmpty()) {
                    b.filter(f -> f.terms(t -> t
                        .field("tags")
                        .terms(tv -> tv.value(req.getTags().stream()
                            .map(co.elastic.clients.elasticsearch._types.FieldValue::of)
                            .toList()))
                    ));
                }
                if (req.getMinPrice() != null || req.getMaxPrice() != null) {
                    b.filter(f -> f.range(r -> {
                        r.field("price");
                        if (req.getMinPrice() != null) r.gte(JsonData.of(req.getMinPrice()));
                        if (req.getMaxPrice() != null) r.lte(JsonData.of(req.getMaxPrice()));
                        return r;
                    }));
                }
                if (req.getMinRating() != null) {
                    b.filter(f -> f.range(r -> r
                        .field("averageRating")
                        .gte(JsonData.of(req.getMinRating()))
                    ));
                }
                if (Boolean.TRUE.equals(req.getInStock())) {
                    b.filter(f -> f.range(r -> r
                        .field("stockQuantity")
                        .gt(JsonData.of(0))
                    ));
                }
                if (req.getCreatedAfter() != null) {
                    b.filter(f -> f.range(r -> r
                        .field("createdAt")
                        .gte(JsonData.of(req.getCreatedAfter().toString()))
                    ));
                }
                // Geo filter
                if (req.getLat() != null && req.getLon() != null && req.getDistance() != null) {
                    b.filter(f -> f
                        .geoDistance(gd -> gd
                            .field("location")
                            .location(l -> l.latlon(ll -> ll.lat(req.getLat()).lon(req.getLon())))
                            .distance(req.getDistance())
                        )
                    );
                }

                b.filter(f -> f.term(t -> t.field("active").value(true)));
                return b;
            })
        );

        // --- Sorting ---
        switch (req.getSortBy()) {
            case "price_asc" -> queryBuilder.withSort(s -> s.field(f -> f.field("price").order(SortOrder.Asc)));
            case "price_desc" -> queryBuilder.withSort(s -> s.field(f -> f.field("price").order(SortOrder.Desc)));
            case "rating" -> queryBuilder.withSort(s -> s.field(f -> f.field("averageRating").order(SortOrder.Desc)));
            case "newest" -> queryBuilder.withSort(s -> s.field(f -> f.field("createdAt").order(SortOrder.Desc)));
            // "relevance" → default score-based sorting
        }

        // --- Highlighting ---
        if (req.getQuery() != null && !req.getQuery().isBlank()) {
            HighlightFieldParameters params = HighlightFieldParameters.builder()
                .withPreTags("<mark>")
                .withPostTags("</mark>")
                .withFragmentSize(150)
                .withNumberOfFragments(2)
                .build();
            queryBuilder.withHighlightQuery(new Highlight(List.of(
                new HighlightField("name", params),
                new HighlightField("description", params)
            )));
        }

        // --- Aggregations ---
        queryBuilder
            .withAggregation("by_category", Aggregation.of(a -> a.terms(t -> t.field("category").size(30))))
            .withAggregation("by_brand", Aggregation.of(a -> a.terms(t -> t.field("brand").size(30))))
            .withAggregation("price_ranges", Aggregation.of(a -> a
                .range(r -> r.field("price").ranges(
                    AggregationRange.of(ar -> ar.to("50").key("Under $50")),
                    AggregationRange.of(ar -> ar.from("50").to("200").key("$50 - $200")),
                    AggregationRange.of(ar -> ar.from("200").to("1000").key("$200 - $1000")),
                    AggregationRange.of(ar -> ar.from("1000").key("Over $1000"))
                ))
            ))
            .withAggregation("avg_price", Aggregation.of(a -> a.avg(avg -> avg.field("price"))))
            .withAggregation("min_price", Aggregation.of(a -> a.min(min -> min.field("price"))))
            .withAggregation("max_price", Aggregation.of(a -> a.max(max -> max.field("price"))));

        // --- Execute ---
        SearchHits<ProductDocument> hits = ops.search(queryBuilder.build(), ProductDocument.class);

        // --- Map results ---
        List<ProductSearchResult> results = hits.getSearchHits().stream()
            .map(this::toResult)
            .collect(Collectors.toList());

        // --- Parse aggregations ---
        Map<String, Long> categoryFacets = parseStermsAgg(hits, "by_category");
        Map<String, Long> brandFacets = parseStermsAgg(hits, "by_brand");
        Map<String, Long> priceRangeFacets = parseRangeAgg(hits, "price_ranges");
        double avgPrice = parseMetricAgg(hits, "avg_price");
        double minPrice = parseMetricAgg(hits, "min_price");
        double maxPrice = parseMetricAgg(hits, "max_price");

        long queryTime = System.currentTimeMillis() - start;
        int totalPages = (int) Math.ceil((double) hits.getTotalHits() / req.getSize());

        return FacetedSearchResponse.builder()
            .totalHits(hits.getTotalHits())
            .page(req.getPage())
            .size(req.getSize())
            .totalPages(totalPages)
            .results(results)
            .categoryFacets(categoryFacets)
            .brandFacets(brandFacets)
            .priceRangeFacets(priceRangeFacets)
            .avgPrice(avgPrice)
            .minPrice(minPrice)
            .maxPrice(maxPrice)
            .queryTimeMs(queryTime)
            .build();
    }

    public List<String> suggest(String prefix, int size) {
        var query = NativeQuery.builder()
            .withQuery(q -> q
                .matchPhrasePrefix(mpp -> mpp
                    .field("name")
                    .query(prefix)
                    .maxExpansions(size)
                )
            )
            .withPageable(PageRequest.of(0, size))
            .build();

        return ops.search(query, ProductDocument.class)
            .getSearchHits().stream()
            .map(h -> h.getContent().getName())
            .distinct()
            .toList();
    }

    public Optional<ProductDocument> findById(String id) {
        return Optional.ofNullable(ops.get(id, ProductDocument.class));
    }

    // --- Helper methods ---

    private ProductSearchResult toResult(SearchHit<ProductDocument> hit) {
        Map<String, List<String>> highlights = hit.getHighlightFields();
        return ProductSearchResult.builder()
            .product(hit.getContent())
            .score(hit.getScore())
            .highlightedName(
                highlights.getOrDefault("name", List.of())
                    .stream().findFirst().orElse(hit.getContent().getName())
            )
            .highlightedDescription(highlights.getOrDefault("description", List.of()))
            .build();
    }

    private Map<String, Long> parseStermsAgg(SearchHits<?> hits, String aggName) {
        if (!hits.hasAggregations()) return Map.of();
        var agg = (ElasticsearchAggregation) hits.getAggregations().get(aggName);
        if (agg == null) return Map.of();
        Map<String, Long> result = new LinkedHashMap<>();
        agg.aggregation().getAggregate().sterms().buckets().array()
            .forEach(b -> result.put(b.key().stringValue(), b.docCount()));
        return result;
    }

    private Map<String, Long> parseRangeAgg(SearchHits<?> hits, String aggName) {
        if (!hits.hasAggregations()) return Map.of();
        var agg = (ElasticsearchAggregation) hits.getAggregations().get(aggName);
        if (agg == null) return Map.of();
        Map<String, Long> result = new LinkedHashMap<>();
        agg.aggregation().getAggregate().range().buckets().array()
            .forEach(b -> result.put(b.key(), b.docCount()));
        return result;
    }

    private double parseMetricAgg(SearchHits<?> hits, String aggName) {
        if (!hits.hasAggregations()) return 0.0;
        var agg = (ElasticsearchAggregation) hits.getAggregations().get(aggName);
        if (agg == null) return 0.0;
        var aggregate = agg.aggregation().getAggregate();
        if (aggregate.isAvg()) return aggregate.avg().value();
        if (aggregate.isMin()) return aggregate.min().value();
        if (aggregate.isMax()) return aggregate.max().value();
        return 0.0;
    }
}
```

### Integration Test with Testcontainers

```java
package com.example.search;

import com.example.search.model.ProductDocument;
import com.example.search.dto.ProductSearchRequest;
import com.example.search.service.ProductFacetSearchService;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.IndexOperations;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.elasticsearch.ElasticsearchContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@SpringBootTest
class ProductSearchIntegrationTest {

    @Container
    static ElasticsearchContainer elasticsearch =
        new ElasticsearchContainer("docker.elastic.co/elasticsearch/elasticsearch:8.11.0")
            .withEnv("xpack.security.enabled", "false");

    @DynamicPropertySource
    static void elasticsearchProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.elasticsearch.uris",
            elasticsearch::getHttpHostAddress);
    }

    @Autowired
    private ElasticsearchOperations ops;

    @Autowired
    private ProductFacetSearchService searchService;

    @BeforeAll
    static void setup(@Autowired ElasticsearchOperations ops,
                      @Autowired ProductSearchRepository repo) {
        IndexOperations indexOps = ops.indexOps(ProductDocument.class);
        if (!indexOps.exists()) indexOps.createWithMapping();

        // Seed data
        List<ProductDocument> products = List.of(
            ProductDocument.builder()
                .id("1")
                .name("Spring Boot in Action")
                .description("A comprehensive guide to Spring Boot")
                .category("Books")
                .brand("Manning")
                .price(new BigDecimal("49.99"))
                .averageRating(4.5f)
                .stockQuantity(100)
                .active(true)
                .tags(List.of("java", "spring", "tutorial"))
                .createdAt(LocalDateTime.now().minusDays(30))
                .build(),
            ProductDocument.builder()
                .id("2")
                .name("Elasticsearch: The Definitive Guide")
                .description("Master Elasticsearch for search and analytics")
                .category("Books")
                .brand("O'Reilly")
                .price(new BigDecimal("59.99"))
                .averageRating(4.8f)
                .stockQuantity(50)
                .active(true)
                .tags(List.of("elasticsearch", "search", "analytics"))
                .createdAt(LocalDateTime.now().minusDays(10))
                .build()
        );

        repo.saveAll(products);
        ops.indexOps(ProductDocument.class).refresh();
    }

    @Test
    void shouldReturnResultsForFullTextSearch() {
        var request = new ProductSearchRequest();
        request.setQuery("spring boot");
        request.setPage(0);
        request.setSize(10);

        var response = searchService.search(request);

        assertThat(response.getTotalHits()).isGreaterThan(0);
        assertThat(response.getResults()).isNotEmpty();
        assertThat(response.getCategoryFacets()).containsKey("Books");
    }

    @Test
    void shouldFilterByCategory() {
        var request = new ProductSearchRequest();
        request.setCategory("Books");
        request.setPage(0);
        request.setSize(10);

        var response = searchService.search(request);

        assertThat(response.getResults())
            .allMatch(r -> r.getProduct().getCategory().equals("Books"));
    }

    @Test
    void shouldFilterByPriceRange() {
        var request = new ProductSearchRequest();
        request.setMinPrice(new BigDecimal("40"));
        request.setMaxPrice(new BigDecimal("55"));
        request.setPage(0);
        request.setSize(10);

        var response = searchService.search(request);

        assertThat(response.getResults())
            .allMatch(r -> {
                BigDecimal price = r.getProduct().getPrice();
                return price.compareTo(new BigDecimal("40")) >= 0
                    && price.compareTo(new BigDecimal("55")) <= 0;
            });
    }

    @Test
    void shouldReturnHighlights() {
        var request = new ProductSearchRequest();
        request.setQuery("elasticsearch");
        request.setPage(0);
        request.setSize(10);

        var response = searchService.search(request);

        assertThat(response.getResults())
            .anyMatch(r -> r.getHighlightedName().contains("<mark>"));
    }

    @Test
    void shouldReturnSuggestions() {
        List<String> suggestions = searchService.suggest("Spri", 5);
        assertThat(suggestions).isNotEmpty();
        assertThat(suggestions.get(0)).containsIgnoringCase("spring");
    }
}
```

---

## 12. Summary

| Feature                     | Spring Data Elasticsearch API                                  |
|-----------------------------|----------------------------------------------------------------|
| Document mapping            | `@Document`, `@Field`, `@MultiField`, `@GeoPointField`        |
| Simple CRUD                 | `ElasticsearchRepository` methods                             |
| Derived queries             | `findByCategory()`, `findByPriceBetween()`, etc.              |
| Custom JSON query           | `@Query("{ ... }")`                                           |
| Complex queries             | `ElasticsearchOperations` + `NativeQuery`                     |
| Full-text                   | `match`, `multi_match`, `bool`, `fuzzy`, `query_string`       |
| Aggregations                | `terms`, `date_histogram`, `range`, `avg`, `min`, `max`       |
| Highlighting                | `HighlightQuery` + `SearchHit.getHighlightFields()`           |
| Geospatial                  | `geo_distance`, `geo_bounding_box`, `geo_polygon`             |
| Bulk indexing               | `ElasticsearchOperations.bulkIndex()`                         |
| Index management            | `IndexOperations.createWithMapping()`, `putMapping()`         |

### Key Takeaways

1. Use `@Field(type = FieldType.Keyword)` for filtering and aggregations, `FieldType.Text` for full-text search.
2. Use `bool` query with `filter` for non-scoring filters — faster because results are cached.
3. Aggregations are your "facets" — always add them alongside search queries.
4. Use `NativeQuery` with the Elasticsearch Java client builder for full control.
5. In production, always use index aliases for zero-downtime migrations.

---

## Next Part Preview

**Part 055: Spring Data MongoDB** — Document databases, nested documents, aggregation pipelines, GridFS, change streams, and transactions in MongoDB with Spring Data.
