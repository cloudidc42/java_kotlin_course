# Part 63: Elasticsearch & Full-Text Search
## ขั้นตอนที่ 4311-4380: Advanced Search, Indexing, Aggregations

---

## 63.1 Elasticsearch Concepts

```
Elasticsearch คืออะไร:
  - Distributed search engine บน Apache Lucene
  - ใช้สำหรับ: full-text search, log analysis, analytics
  - Stores data as JSON documents in Indexes
  - Near real-time: documents searchable within ~1 second after index

เปรียบเทียบกับ Database:
  SQL          | Elasticsearch
  -------------|----------------
  Database     | Index
  Table        | (Index = Table)
  Row          | Document
  Column       | Field
  SQL Query    | Query DSL (JSON)
  JOIN         | Nested/denormalize

Use Cases:
  ✓ E-commerce product search (typo tolerance, facets)
  ✓ Log analysis (ELK Stack)
  ✓ Autocomplete / suggestions
  ✓ Geospatial search
  ✓ Analytics dashboards
```

---

## 63.2 Spring Data Elasticsearch Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

```yaml
# application.yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: ${ELASTIC_PASSWORD}
```

```java
import org.springframework.data.elasticsearch.annotations.*;
import org.springframework.data.annotation.*;

// Document mapping
@Document(indexName = "products")
@Setting(settingPath = "/elasticsearch/product-settings.json")
public class ProductDocument {
    
    @Id
    private String id;
    
    @Field(type = FieldType.Text, analyzer = "thai_analyzer")
    private String name;
    
    @Field(type = FieldType.Text, analyzer = "thai_analyzer")
    private String description;
    
    @Field(type = FieldType.Keyword)
    private String category;
    
    @Field(type = FieldType.Double)
    private Double price;
    
    @Field(type = FieldType.Integer)
    private Integer stockQuantity;
    
    @Field(type = FieldType.Boolean)
    private Boolean active;
    
    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private java.time.LocalDateTime createdAt;
    
    // Nested tags
    @Field(type = FieldType.Keyword)
    private java.util.List<String> tags;
    
    // getters/setters/constructors omitted
}
```

---

## 63.3 Repository & Basic Search

```java
import org.springframework.data.elasticsearch.repository.*;
import org.springframework.data.domain.*;

public interface ProductSearchRepository extends ElasticsearchRepository<ProductDocument, String> {
    
    // Auto-generated queries from method names
    Page<ProductDocument> findByCategory(String category, Pageable pageable);
    
    java.util.List<ProductDocument> findByPriceBetween(double minPrice, double maxPrice);
    
    java.util.List<ProductDocument> findByNameContaining(String keyword);
    
    java.util.List<ProductDocument> findByCategoryAndActiveTrue(String category);
    
    long countByCategory(String category);
}

// Service
@Service
class ProductSearchService {
    
    private final ProductSearchRepository searchRepository;
    private final ElasticsearchOperations elasticOps;
    
    ProductSearchService(ProductSearchRepository searchRepository,
                         ElasticsearchOperations elasticOps) {
        this.searchRepository = searchRepository;
        this.elasticOps = elasticOps;
    }
    
    // ====== Index a product ======
    public void indexProduct(Product product) {
        var doc = new ProductDocument();
        doc.setId(product.getId());
        doc.setName(product.getName());
        doc.setDescription(product.getDescription());
        doc.setCategory(product.getCategory());
        doc.setPrice(product.getPrice());
        doc.setActive(product.isActive());
        doc.setCreatedAt(product.getCreatedAt());
        searchRepository.save(doc);
    }
    
    // ====== Full-text search ======
    public org.springframework.data.domain.Page<ProductDocument> search(
            String keyword, String category, Double minPrice, Double maxPrice,
            int page, int size) {
        
        var query = new org.springframework.data.elasticsearch.core.query.NativeQueryBuilder()
            .withQuery(buildSearchQuery(keyword, category, minPrice, maxPrice))
            .withSort(org.springframework.data.elasticsearch.core.query.Query.DEFAULT_SORT)
            .withPageable(PageRequest.of(page, size))
            .build();
        
        var hits = elasticOps.search(query, ProductDocument.class);
        return new org.springframework.data.support.PageableExecutionUtils.Page<>(
            hits.getSearchHits().stream()
                .map(org.springframework.data.elasticsearch.core.SearchHit::getContent)
                .toList(),
            PageRequest.of(page, size),
            hits::getTotalHits
        );
    }
    
    private org.springframework.data.elasticsearch.client.elc.NativeQuery buildSearchQuery(
            String keyword, String category, Double minPrice, Double maxPrice) {
        
        var boolQuery = co.elastic.clients.elasticsearch._types.query_dsl.BoolQuery.of(b -> {
            // Must: keyword search on name and description
            if (keyword != null && !keyword.isBlank()) {
                b.must(m -> m.multiMatch(mm -> mm
                    .query(keyword)
                    .fields("name^3", "description^1")  // name has 3x boost
                    .fuzziness("AUTO")  // typo tolerance
                    .type(co.elastic.clients.elasticsearch._types.query_dsl.TextQueryType.BestFields)
                ));
            }
            
            // Filter: category
            if (category != null) {
                b.filter(f -> f.term(t -> t.field("category").value(category)));
            }
            
            // Filter: price range
            if (minPrice != null || maxPrice != null) {
                b.filter(f -> f.range(r -> {
                    r.field("price");
                    if (minPrice != null) r.gte(co.elastic.clients.json.JsonData.of(minPrice));
                    if (maxPrice != null) r.lte(co.elastic.clients.json.JsonData.of(maxPrice));
                    return r;
                }));
            }
            
            // Filter: only active products
            b.filter(f -> f.term(t -> t.field("active").value(true)));
            
            return b;
        });
        
        return org.springframework.data.elasticsearch.client.elc.NativeQuery.builder()
            .withQuery(q -> q.bool(boolQuery))
            .build();
    }
}
```

---

## 63.4 Aggregations (Faceted Search)

```java
import co.elastic.clients.elasticsearch._types.aggregations.*;

@Service
class SearchFacetService {
    
    private final ElasticsearchOperations elasticOps;
    
    SearchFacetService(ElasticsearchOperations elasticOps) {
        this.elasticOps = elasticOps;
    }
    
    // Faceted search: category counts, price ranges, brand counts
    public SearchWithFacets searchWithFacets(String keyword) {
        
        var query = org.springframework.data.elasticsearch.client.elc.NativeQuery.builder()
            .withQuery(q -> q.multiMatch(mm -> mm
                .query(keyword)
                .fields("name", "description")
                .fuzziness("AUTO")
            ))
            // Aggregation: count by category
            .withAggregation("categories", Aggregation.of(a -> a
                .terms(t -> t.field("category").size(20))
            ))
            // Aggregation: price ranges
            .withAggregation("price_ranges", Aggregation.of(a -> a
                .range(r -> r
                    .field("price")
                    .ranges(
                        rv -> rv.to("100"),
                        rv -> rv.from("100").to("500"),
                        rv -> rv.from("500").to("1000"),
                        rv -> rv.from("1000")
                    )
                )
            ))
            // Aggregation: average price
            .withAggregation("avg_price", Aggregation.of(a -> a
                .avg(avg -> avg.field("price"))
            ))
            .withPageable(PageRequest.of(0, 20))
            .build();
        
        var result = elasticOps.search(query, ProductDocument.class);
        
        // Extract aggregations
        var aggregations = result.getAggregations();
        var categoryFacets = new java.util.ArrayList<FacetItem>();
        
        if (aggregations != null) {
            var categoryAgg = aggregations.get("categories");
            // Process terms buckets
        }
        
        var products = result.getSearchHits().stream()
            .map(org.springframework.data.elasticsearch.core.SearchHit::getContent)
            .toList();
        
        return new SearchWithFacets(products, categoryFacets, result.getTotalHits());
    }
    
    record FacetItem(String key, long count) {}
    record SearchWithFacets(
        java.util.List<ProductDocument> products,
        java.util.List<FacetItem> categories,
        long totalHits
    ) {}
}
```

---

## 63.5 Autocomplete & Suggestions

```java
@Service
class AutocompleteService {
    
    private final ElasticsearchOperations elasticOps;
    
    AutocompleteService(ElasticsearchOperations elasticOps) {
        this.elasticOps = elasticOps;
    }
    
    // Prefix search for autocomplete
    public java.util.List<String> suggest(String prefix) {
        var query = org.springframework.data.elasticsearch.client.elc.NativeQuery.builder()
            .withQuery(q -> q.bool(b -> b
                .should(s -> s.prefix(p -> p.field("name.suggest").value(prefix.toLowerCase())))
                .should(s -> s.matchPhrasePrefix(mpp -> mpp.field("name").query(prefix)))
            ))
            .withSourceFilter(new org.springframework.data.elasticsearch.core.query.FetchSourceFilter(
                new String[]{"name"}, null))
            .withPageable(PageRequest.of(0, 10))
            .build();
        
        return elasticOps.search(query, ProductDocument.class)
            .getSearchHits().stream()
            .map(hit -> hit.getContent().getName())
            .distinct()
            .toList();
    }
    
    // "Did you mean?" correction
    public java.util.List<String> didYouMean(String misspelled) {
        var query = org.springframework.data.elasticsearch.client.elc.NativeQuery.builder()
            .withQuery(q -> q.match(m -> m
                .field("name")
                .query(misspelled)
                .fuzziness("2")
            ))
            .withPageable(PageRequest.of(0, 5))
            .build();
        
        return elasticOps.search(query, ProductDocument.class)
            .getSearchHits().stream()
            .map(hit -> hit.getContent().getName())
            .toList();
    }
}

// Autocomplete REST endpoint
@RestController
@RequestMapping("/api/search")
class SearchController {
    
    private final ProductSearchService searchService;
    private final AutocompleteService autocompleteService;
    
    SearchController(ProductSearchService searchService, AutocompleteService autocompleteService) {
        this.searchService = searchService;
        this.autocompleteService = autocompleteService;
    }
    
    @GetMapping
    SearchResult search(
            @RequestParam(required = false) String q,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) Double minPrice,
            @RequestParam(required = false) Double maxPrice,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        var results = searchService.search(q, category, minPrice, maxPrice, page, size);
        return new SearchResult(
            results.getContent(),
            results.getTotalElements(),
            results.getTotalPages()
        );
    }
    
    @GetMapping("/suggest")
    java.util.List<String> suggest(@RequestParam String q) {
        return autocompleteService.suggest(q);
    }
    
    record SearchResult(
        java.util.List<ProductDocument> items,
        long total,
        int totalPages
    ) {}
}
```

---

## สรุป Part 63

```
Elasticsearch Key Points:

Setup:
  spring-boot-starter-data-elasticsearch
  @Document(indexName = "products")
  @Field(type = FieldType.Text, analyzer = "...")

Search Types:
  term      = exact match (keyword fields)
  match     = full-text search (analyzed)
  multiMatch = search across multiple fields
  fuzzy     = typo tolerance (fuzziness: "AUTO")
  range     = price/date range
  bool      = combine must/should/filter/must_not

Performance Tips:
  - Use "keyword" type for exact match, filtering, sorting, aggregations
  - Use "text" type for full-text search only
  - field^3 = boost specific field in search score
  - filter context = no scoring (faster than query context)
  - Use scroll API for large result sets (not deep pagination)

Aggregations (Facets):
  terms = count by category
  range = bucket by price range
  avg, sum, min, max = numeric stats
  date_histogram = events over time

When to use Elasticsearch:
  ✓ Full-text search with relevance scoring
  ✓ Autocomplete / prefix search
  ✓ Log analytics (ELK stack)
  ✗ Transactions / consistency (use PostgreSQL)
  ✗ Primary datastore (use as search layer alongside SQL)
```

➡️ [Part 64: WebSocket & Real-time Communication](./Part-64-WebSocket.md)
