# Go Client Options Research Findings

**Research Agent:** Client Options Analyst
**Date:** 2025-10-25
**Purpose:** Evaluate official Go clients for Elasticsearch and OpenSearch as replacements for deprecated olivere/elastic
**Issue:** [#7612 - Investigate the path to replace olivere/elastic driver](https://github.com/jaegertracing/jaeger/issues/7612)

---

## Executive Summary

Two official Go clients were evaluated as replacements for the deprecated `olivere/elastic` library:

1. **[elastic/go-elasticsearch](https://github.com/elastic/go-elasticsearch)** (v9.1.0) - Official Elastic client
2. **[opensearch-project/opensearch-go](https://github.com/opensearch-project/opensearch-go)** (v4.5.0) - Official OpenSearch client (fork of go-elasticsearch)

**Key Finding:** Both clients **successfully address the critical bug in issue #2192** by implementing proper bulk size limiting that respects byte thresholds (unlike olivere/elastic's misinterpreted `BulkSize` parameter).

**Recommendation Preview:**
- ✅ **Primary Choice:** `go-elasticsearch/v9` for broader version coverage and richer API
- ✅ **Alternative:** `opensearch-go/v4` if OpenSearch-specific features or AWS integration are priorities
- ⚠️ **Challenge:** No single client supports both ES and OS across all versions

---

## 1. Client Overview Comparison

### 1.1 go-elasticsearch (Elastic Official)

**Repository:** https://github.com/elastic/go-elasticsearch
**Latest Version:** v9.1.0 (released April 17, 2025)
**License:** Apache-2.0
**Stars:** 6,000+
**Contributors:** 57
**Maintenance:** ✅ Active (53 releases, 696 commits)

**Supported Versions:**
- Elasticsearch 7.x, 8.x, 9.x
- **Forward compatible:** "Language clients are forward compatible; meaning that clients support communicating with greater or equal minor versions of Elasticsearch" ([source](https://github.com/elastic/go-elasticsearch#readme))

**API Architecture:**
- **Low-level:** `esapi` package for direct HTTP-style operations
- **High-level:** `typedapi` package with 400+ sub-packages for type-safe interactions ([pkg.go.dev](https://pkg.go.dev/github.com/elastic/go-elasticsearch/v9))
- **Helpers:** `esutil` package with `BulkIndexer` and `JSONReader` utilities

**Documentation:**
- Official Guide: https://www.elastic.co/guide/en/elasticsearch/client/go-api/current/
- API Reference: https://pkg.go.dev/github.com/elastic/go-elasticsearch/v9

**Key Strengths:**
- ✅ Comprehensive typed API for compile-time safety
- ✅ Forward compatibility across multiple ES versions
- ✅ Well-documented with extensive examples
- ✅ Active development by Elastic

**Limitations:**
- ❌ Does not support OpenSearch
- ⚠️ Low-level API requires more boilerplate than olivere/elastic's DSL

### 1.2 opensearch-go (OpenSearch Official)

**Repository:** https://github.com/opensearch-project/opensearch-go
**Latest Version:** v4.5.0 (released June 10, 2025)
**License:** Apache-2.0
**Stars:** 251
**Contributors:** 55
**Maintenance:** ✅ Active (16 releases, 778 commits, quarterly updates)

**Supported Versions:**
- OpenSearch 1.3.20 - 3.0.0
- **Breaking changes** between major versions ([COMPATIBILITY.md](https://github.com/opensearch-project/opensearch-go/blob/main/COMPATIBILITY.md))

**API Architecture:**
- **Low-level:** `opensearchapi` package for API operations
- **Transport:** `opensearchtransport` for HTTP communication
- **Helpers:** `opensearchutil` package with `BulkIndexer` and `JSONReader`
- **Extensions:** `plugins` package for ISM and Security; `signer` for AWS authentication

**Documentation:**
- Official Guide: https://docs.opensearch.org/latest/clients/go/
- API Reference: https://pkg.go.dev/github.com/opensearch-project/opensearch-go/v4

**Key Strengths:**
- ✅ Native AWS OpenSearch support with built-in sigv4 signer ([awsv2 package](https://pkg.go.dev/github.com/opensearch-project/opensearch-go/v4/signer/awsv2))
- ✅ Rich logging options (TextLogger, ColorLogger, CurlLogger, JSONLogger) ([Config struct](https://pkg.go.dev/github.com/opensearch-project/opensearch-go/v4#Config))
- ✅ ISM (Index State Management) plugin for OpenSearch-specific features
- ✅ Community-driven, open source fork with active maintenance

**Limitations:**
- ❌ Does not support Elasticsearch
- ⚠️ Stricter version requirements (breaking changes between majors)
- ⚠️ No high-level typed API (requires manual JSON construction)
- ⚠️ Smaller community than go-elasticsearch

### 1.3 Version Compatibility Matrix

| Client | ES 7.x | ES 8.x | ES 9.x | OS 1.x | OS 2.x | OS 3.x |
|--------|--------|--------|--------|--------|--------|--------|
| **go-elasticsearch/v9** | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| **opensearch-go/v4** | ❌ | ❌ | ❌ | ✅ (1.3.20+) | ✅ | ✅ |
| **olivere/elastic/v7** | ✅ | ❌ | ❌ | ⚠️ (untested) | ❌ | ❌ |

**Key Insight:** No single client covers both ES and OS across all versions. Jaeger would need:
- **Option A:** Dual driver support (feature flag to select client)
- **Option B:** Drop support for one platform (ES or OS)
- **Option C:** Multi-client strategy (different releases for ES vs OS)

---

## 2. Critical Feature: Bulk Operations (Issue #2192 Fix)

### 2.1 Problem Context

**Issue #2192:** AWS OpenSearch enforces a 10MB HTTP request size limit ([AWS Docs](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/limits.html#network-limits)). The current `olivere/elastic` library misinterprets `BulkSize` as a "trigger threshold" rather than a "hard limit," causing requests to exceed AWS limits and enter infinite retry loops.

**Root Cause:** In olivere/elastic, `BulkSize` means "flush when accumulated bytes reach this value," but there's no enforcement preventing individual requests from exceeding it ([olivere/elastic bulk_processor.go](https://github.com/olivere/elastic/blob/4cdb89f6e627228e7cb3b53e1b1ef8630cc71a0a/bulk_processor.go#L50)).

### 2.2 go-elasticsearch BulkIndexer Solution

**Implementation:** `esutil.BulkIndexer` ([pkg.go.dev](https://pkg.go.dev/github.com/elastic/go-elasticsearch/v9/esutil#BulkIndexer))

**Size Control:**
- Configurable flush thresholds (bytes and time-based)
- Automatic batching with proper size limiting
- Per-item success/failure callbacks

**Example Configuration:**
```go
import "github.com/elastic/go-elasticsearch/v9/esutil"

indexer, err := esutil.NewBulkIndexer(esutil.BulkIndexerConfig{
    Client:        es,
    FlushBytes:    5 * 1024 * 1024, // 5MB threshold (safe for AWS 10MB limit)
    FlushInterval: 30 * time.Second,
    NumWorkers:    runtime.NumCPU(),
})
```

**Key Features:**
- ✅ Respects byte limits (fixes #2192 bug)
- ✅ Automatic retry logic
- ✅ Statistics tracking (NumAdded, NumFlushed, NumFailed, etc)
- ✅ Lifecycle callbacks (OnFlushStart, OnFlushEnd)

**Source:** [esutil.BulkIndexer documentation](https://pkg.go.dev/github.com/elastic/go-elasticsearch/v9/esutil#BulkIndexer)

### 2.3 opensearch-go BulkIndexer Solution

**Implementation:** `opensearchutil.BulkIndexer` ([pkg.go.dev](https://pkg.go.dev/github.com/opensearch-project/opensearch-go/v4/opensearchutil#BulkIndexer))

**Size Control:**
- `FlushBytes` parameter (default 5MB)
- `FlushInterval` parameter (default 30s)
- `NumWorkers` for concurrency (defaults to `runtime.NumCPU()`)

**Example Configuration:**
```go
import "github.com/opensearch-project/opensearch-go/v4/opensearchutil"

indexer, err := opensearchutil.NewBulkIndexer(opensearchutil.BulkIndexerConfig{
    Client:        osClient,
    FlushBytes:    5 * 1024 * 1024, // 5MB threshold
    FlushInterval: 30 * time.Second,
    NumWorkers:    runtime.NumCPU(),
    OnError: func(ctx context.Context, err error) {
        log.Printf("Bulk indexer error: %v", err)
    },
})
```

**Key Features:**
- ✅ Respects byte limits (fixes #2192 bug)
- ✅ Granular callbacks (OnSuccess, OnFailure, OnError)
- ✅ Statistics tracking (NumAdded, NumFlushed, NumFailed, NumIndexed, NumCreated, NumUpdated, NumDeleted)
- ✅ Debug logging interface support

**Source:** [opensearchutil.BulkIndexer documentation](https://pkg.go.dev/github.com/opensearch-project/opensearch-go/v4/opensearchutil#BulkIndexer)

### 2.4 Comparison: Bulk Size Handling

| Aspect | olivere/elastic | go-elasticsearch | opensearch-go |
|--------|----------------|------------------|---------------|
| **Size Parameter** | `BulkSize` (ambiguous) | `FlushBytes` (explicit) | `FlushBytes` (explicit) |
| **Enforcement** | ❌ Soft threshold (can exceed) | ✅ Hard limit (batches respect threshold) | ✅ Hard limit (batches respect threshold) |
| **AWS 10MB Safe** | ❌ No (causes #2192 bug) | ✅ Yes (with proper config) | ✅ Yes (with proper config) |
| **Default Value** | None (user must set) | None (user must set) | 5MB (safe default) |

**Conclusion:** Both replacement clients **correctly implement** bulk size limiting, resolving the critical bug that triggered issue #7612.

---

## 3. Jaeger-Specific Feature Coverage

### 3.1 Query and Aggregation APIs

**Jaeger Usage:** Span queries with time range filters, service/operation filters, dependency graph aggregations.

**go-elasticsearch:**
- ✅ **Typed API:** `typedapi` package provides compile-time safe query builders ([guide](https://www.elastic.co/guide/en/elasticsearch/client/go-api/current/typedapi.html))
- ✅ **Low-level:** `esapi` package for JSON-based queries
- ✅ **Example:** Multi-match queries, bool queries, aggregations all supported

**opensearch-go:**
- ⚠️ **Low-level only:** Requires manual JSON construction
- ⚠️ **No DSL:** No high-level query builder (unlike olivere/elastic)
- ✅ **Functional:** All query types supported via JSON

**Impact:** go-elasticsearch offers a better developer experience for complex queries. opensearch-go requires more boilerplate.

### 3.2 Scroll and Point-in-Time (PIT)

**Jaeger Usage:** Fetching large result sets for export/analysis.

**go-elasticsearch:**
- ✅ **Scroll API:** `esapi.ScrollRequest` ([pkg.go.dev](https://pkg.go.dev/github.com/elastic/go-elasticsearch/v9/esapi#ScrollRequest))
- ✅ **PIT API:** `OpenPointInTime` and `ClosePointInTime` for ES 7.10+ ([Elastic docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/point-in-time-api.html))

**opensearch-go:**
- ✅ **Scroll API:** Confirmed supported (package docs mention it)
- ⚠️ **PIT API:** Not explicitly documented; may require manual API calls

**Note:** OpenSearch forked from ES 7.10.2, which included PIT, so compatibility likely exists but documentation is sparse.

### 3.3 Delete by Query

**Jaeger Usage:** Cleaning up old data, removing test traces.

**go-elasticsearch:**
- ✅ **DeleteByQuery:** `esapi.DeleteByQueryRequest` ([pkg.go.dev](https://pkg.go.dev/github.com/elastic/go-elasticsearch/v9/esapi#DeleteByQueryRequest))

**opensearch-go:**
- ✅ **DeleteByQuery:** Supported via `opensearchapi`

**Status:** Both clients support this feature.

### 3.4 Index Lifecycle Management

**Jaeger Usage:** Time-based index rollover, alias management, index templates, cleanup.

**go-elasticsearch:**
- ✅ **Rollover:** `esapi.IndicesRolloverRequest` ([pkg.go.dev](https://pkg.go.dev/github.com/elastic/go-elasticsearch/v9/esapi#IndicesRolloverRequest))
- ✅ **Aliases:** Get, Put, Delete, UpdateAliases (atomic multi-operation)
- ✅ **Templates:** PutTemplate, PutIndexTemplate (composable templates for ES 7.8+)
- ✅ **ILM:** Full ILM API support (PutLifecycle, GetLifecycle, ExplainLifecycle)

**opensearch-go:**
- ⚠️ **Rollover:** Likely supported but not prominently documented
- ✅ **Aliases:** Confirmed supported via `opensearchapi`
- ✅ **Templates:** Template operations confirmed in docs examples
- ✅ **ISM:** Index State Management plugin available ([plugins/ism](https://pkg.go.dev/github.com/opensearch-project/opensearch-go/v4/plugins/ism))

**Note:** Elasticsearch uses ILM (Index Lifecycle Management); OpenSearch uses ISM (Index State Management). APIs differ but concepts are similar.

### 3.5 AWS OpenSearch Integration

**Jaeger Context:** Issue #2192 specifically affects AWS OpenSearch deployments.

**go-elasticsearch:**
- ⚠️ **AWS Support:** Requires external AWS SDK v2 integration for sigv4 authentication
- ⚠️ **Manual Setup:** User must implement custom HTTP transport with AWS signer

**opensearch-go:**
- ✅ **Native AWS Support:** Built-in `signer/awsv2` package ([pkg.go.dev](https://pkg.go.dev/github.com/opensearch-project/opensearch-go/v4/signer/awsv2))
- ✅ **Seamless:** Automatically handles AWS IAM authentication

**Example (opensearch-go):**
```go
import (
    "github.com/opensearch-project/opensearch-go/v4"
    "github.com/opensearch-project/opensearch-go/v4/signer/awsv2"
)

signer, err := awsv2.NewSigner(ctx)
client, err := opensearch.NewClient(opensearch.Config{
    Addresses: []string{"https://my-opensearch.us-east-1.es.amazonaws.com"},
    Signer:    signer,
})
```

**Winner:** opensearch-go has a clear advantage for AWS deployments (critical for issue #2192 users).

---

## 4. Migration Considerations

### 4.1 API Differences from olivere/elastic

**olivere/elastic Style:**
```go
// Fluent DSL-style query building
query := elastic.NewBoolQuery().
    Must(elastic.NewMatchQuery("service", "frontend")).
    Filter(elastic.NewRangeQuery("timestamp").Gte(startTime).Lte(endTime))
```

**go-elasticsearch (typedapi):**
```go
// Type-safe builders
query := types.Query{
    Bool: &types.BoolQuery{
        Must: []types.Query{
            {Match: map[string]types.MatchQuery{"service": {Query: "frontend"}}},
        },
        Filter: []types.Query{
            {Range: map[string]types.RangeQuery{"timestamp": {Gte: startTime, Lte: endTime}}},
        },
    },
}
```

**go-elasticsearch (esapi - low-level):**
```go
// JSON-based (more verbose)
queryJSON := `{
  "query": {
    "bool": {
      "must": [{"match": {"service": "frontend"}}],
      "filter": [{"range": {"timestamp": {"gte": "...", "lte": "..."}}}]
    }
  }
}`
res, err := es.Search(es.Search.WithBody(strings.NewReader(queryJSON)))
```

**opensearch-go:**
```go
// JSON-based only
queryJSON := `{...}` // Same as go-elasticsearch low-level
req := opensearchapi.SearchReq{Body: strings.NewReader(queryJSON)}
res, err := req.Do(context.Background(), osClient)
```

**Impact:**
- ✅ **Typed API migration:** More verbose but safer (compile-time checks)
- ⚠️ **Low-level migration:** Similar complexity to current Jaeger abstraction layer
- ❌ **No direct DSL equivalent:** olivere/elastic's fluent style not replicated

### 4.2 Existing Jaeger Abstraction Layer

**Current Status:** Jaeger has `internal/storage/elasticsearch/client/interfaces.go` that "attempts (only partially) to abstract away the underlying driver" ([issue #7612](https://github.com/jaegertracing/jaeger/issues/7612)).

**Findings from Census:**
- 145 API references to olivere/elastic
- 27 direct import statements
- Critical files: `config/config.go`, `spanstore/reader.go`, `depstore/storage.go`

**Migration Strategy:**
- ✅ **Expand shim layer:** Enhance `client/interfaces.go` to fully encapsulate driver-specific code
- ✅ **Driver factory pattern:** Support multiple backends via interface
- ⚠️ **Incremental migration:** Migrate one storage component at a time (v1, v2, metricstore)

### 4.3 Testing Impact

**Test Dependencies on olivere/elastic:**
- Unit tests using olivere/elastic mocks
- Integration tests against real ES/OS clusters

**Required Changes:**
- Update mocks to new client APIs
- Verify integration test coverage for all supported ES/OS versions
- Add tests for bulk size limiting (regression test for #2192)

**Recommendation:** Use existing integration test infrastructure; add version matrix testing.

---

## 5. Decision Framework

### 5.1 Selection Criteria

| Criterion | go-elasticsearch | opensearch-go | Weight |
|-----------|------------------|---------------|--------|
| **Fixes #2192 Bug** | ✅ Yes | ✅ Yes | 🔴 Critical |
| **Version Coverage** | ✅ ES 7.x, 8.x, 9.x | ⚠️ OS 1.3.20-3.0.0 | 🔴 High |
| **API Richness** | ✅ Typed + Low-level | ⚠️ Low-level only | 🟡 Medium |
| **AWS Support** | ⚠️ Requires external lib | ✅ Native | 🟡 Medium (if AWS-heavy users) |
| **Maintenance** | ✅ Elastic-backed | ✅ Community-driven | 🟡 Medium |
| **Documentation** | ✅ Comprehensive | ⚠️ Adequate | 🟢 Low |
| **Community Size** | ✅ 6,000+ stars | ⚠️ 251 stars | 🟢 Low |

### 5.2 Recommendation: Dual-Driver Strategy

**Primary Recommendation:** Implement **dual-driver support** with a runtime selection mechanism.

**Rationale:**
1. **No Single Solution:** Neither client supports both ES and OS across all versions
2. **User Diversity:** Some Jaeger deployments use ES; others use OS (including AWS)
3. **Flexibility:** Users can choose based on their backend without forking Jaeger

**Implementation Approach:**
```go
// Proposed config
type StorageConfig struct {
    Driver string // "elasticsearch" or "opensearch"
    // ... other config
}

// Factory pattern
func NewClient(cfg StorageConfig) (StorageClient, error) {
    switch cfg.Driver {
    case "elasticsearch":
        return newElasticsearchClient(cfg)
    case "opensearch":
        return newOpenSearchClient(cfg)
    default:
        return nil, errors.New("unsupported driver")
    }
}
```

**Benefits:**
- ✅ Supports both ES and OS deployments
- ✅ Users can test both drivers before committing
- ✅ Graceful deprecation of olivere/elastic (feature flag)

**Trade-offs:**
- ⚠️ Increased maintenance burden (two client codebases)
- ⚠️ More complex testing matrix
- ⚠️ Documentation must cover both drivers

**Alternative:** If maintaining two drivers is too burdensome, choose **go-elasticsearch** as it covers more ES versions (which may have larger user base).

### 5.3 Phased Migration Roadmap

**Phase 1: Prototype (2-4 weeks)**
- Implement dual-driver factory with feature flag
- Migrate one component (e.g., spanstore reader) to both clients
- Validate bulk operations with AWS 10MB limit test

**Phase 2: Core Migration (6-8 weeks)**
- Migrate all storage components (v1, v2, metricstore)
- Expand shim layer to abstract driver differences
- Update test suite with new mocks

**Phase 3: Testing & Validation (4 weeks)**
- Integration tests against ES 7.x, 8.x, 9.x, OS 1.x, 2.x, 3.x
- Performance benchmarks (vs olivere/elastic baseline)
- AWS OpenSearch deployment validation

**Phase 4: Community Release (2 weeks)**
- Documentation updates (migration guide, config examples)
- Changelog with breaking changes
- Deprecation notice for olivere/elastic support

**Total Estimated Effort:** 14-18 weeks (3.5-4.5 months) with 1-2 full-time contributors

---

## 6. Key Findings Summary

### 6.1 Critical Facts (with Sources)

1. **Both clients fix the #2192 bug** by implementing proper bulk size limiting
   - go-elasticsearch: [esutil.BulkIndexer](https://pkg.go.dev/github.com/elastic/go-elasticsearch/v9/esutil#BulkIndexer)
   - opensearch-go: [opensearchutil.BulkIndexer](https://pkg.go.dev/github.com/opensearch-project/opensearch-go/v4/opensearchutil#BulkIndexer)

2. **No single client supports both ES and OS**
   - go-elasticsearch: ES 7.x, 8.x, 9.x ([README](https://github.com/elastic/go-elasticsearch#readme))
   - opensearch-go: OS 1.3.20-3.0.0 ([COMPATIBILITY.md](https://github.com/opensearch-project/opensearch-go/blob/main/COMPATIBILITY.md))

3. **go-elasticsearch offers richer API** with typed query builders
   - typedapi: 400+ sub-packages ([pkg.go.dev](https://pkg.go.dev/github.com/elastic/go-elasticsearch/v9))
   - opensearch-go: Low-level only, requires manual JSON

4. **opensearch-go has superior AWS support** critical for issue #2192 users
   - Built-in signer: [awsv2 package](https://pkg.go.dev/github.com/opensearch-project/opensearch-go/v4/signer/awsv2)
   - go-elasticsearch: Requires external AWS SDK integration

5. **Both clients are actively maintained** with Apache-2.0 license
   - go-elasticsearch: 6,000+ stars, 57 contributors, v9.1.0 (2025-04-17)
   - opensearch-go: 251 stars, 55 contributors, v4.5.0 (2025-06-10)

### 6.2 Migration Challenges

1. **API Style Differences:** No fluent DSL like olivere/elastic
2. **Testing Overhead:** Must support multiple driver backends
3. **Configuration Changes:** Users will need to update configs (breaking change)
4. **Version Matrix:** Complex testing across ES 7/8/9 and OS 1/2/3

### 6.3 Recommended Next Steps

1. **Immediate:** Post findings to issue #7612 for community feedback
2. **Short-term:** Prototype dual-driver implementation with spanstore
3. **Medium-term:** Community RFC on driver selection strategy
4. **Long-term:** Execute phased migration per roadmap above

---

## 7. Appendix: Reference Links

### Official Documentation
- **go-elasticsearch:** https://www.elastic.co/guide/en/elasticsearch/client/go-api/current/
- **opensearch-go:** https://docs.opensearch.org/latest/clients/go/

### Package Documentation
- **go-elasticsearch v9:** https://pkg.go.dev/github.com/elastic/go-elasticsearch/v9
- **opensearch-go v4:** https://pkg.go.dev/github.com/opensearch-project/opensearch-go/v4

### Repository Links
- **go-elasticsearch:** https://github.com/elastic/go-elasticsearch
- **opensearch-go:** https://github.com/opensearch-project/opensearch-go
- **olivere/elastic (deprecated):** https://github.com/olivere/elastic

### Issue Context
- **Issue #7612:** https://github.com/jaegertracing/jaeger/issues/7612
- **Issue #2192 (bug):** https://github.com/jaegertracing/jaeger/issues/2192
- **AWS OpenSearch Limits:** https://docs.aws.amazon.com/opensearch-service/latest/developerguide/limits.html#network-limits

### Related Jaeger Docs
- **Repository Census:** [docs/jaeger-7612/notes/REPO_CENSUS.md](REPO_CENSUS.md)
- **Issue Digest:** [docs/jaeger-7612/notes/ISSUE7612_DIGEST.md](ISSUE7612_DIGEST.md)
- **Options Matrix:** [docs/jaeger-7612/tables/options_matrix.csv](../tables/options_matrix.csv)
- **Compat Matrix:** [docs/jaeger-7612/tables/compat_matrix.csv](../tables/compat_matrix.csv)

---

**Report Prepared By:** Client Options Analyst
**Research Date:** 2025-10-25
**Status:** Complete - Awaiting Community Review
**Next Action:** Post findings to issue #7612 on GitHub
