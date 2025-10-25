# Elasticsearch/OpenSearch Client Library Investigation

## TL;DR

- **No single library works for both ES and OS** (Elastic added product detection in Dec 2021)
- **Two official clients available**: `elastic/go-elasticsearch` (ES only) and `opensearch-go` (OS only)
- **Jaeger is already partially migrated** to go-elasticsearch/v9 (templates only, bulk ops still broken)
- **Recommendation**: Dual-client strategy to support both platforms
- **Migration effort**: ~38 files, mainly query DSL changes

---

## What I Investigated

I looked into replacing `olivere/elastic` for Jaeger's ES/OS storage. The maintainer [declared the library dormant](https://github.com/olivere/elastic/pull/1661#issuecomment-1319834556) in Oct 2023, and critical bugs like #2192 (AWS 10MB limit) can't be fixed. I focused on the five questions from the issue:

1. Available Go libraries
2. Version compatibility
3. Single library for ES+OS?
4. API differences vs olivere/elastic
5. Internal code changes needed

---

## Available Libraries

### 1. elastic/go-elasticsearch (Official Elastic)

**Current status:**
- v9.1.0 (April 2025) - Latest
- v8.17.x (Dec 2024) - Stable for ES 8.x
- Apache 2.0 license
- Active development

**Platform support:**
- ✅ Elasticsearch 6.x, 7.x, 8.x, 9.x
- ❌ OpenSearch (rejected since v7.13.2 in Dec 2021)

Since elastic/go-elasticsearch v7.13.2, the client checks for an `X-Elastic-Product: Elasticsearch` header. If connecting to OpenSearch, you get:

```
elasticsearch.UnsupportedProductError: The client noticed that
the server is not Elasticsearch and we do not support this unknown product
```

This is a deliberate choice by Elastic, not a bug. I tested with OpenSearch 2.x and confirmed the error. Last version without detection: v7.13.1 (not recommended - no updates since 2021).

**API style:**
- Low-level: `esapi` package (JSON strings)
- High-level: `typedapi` package (type-safe structs, 400+ packages)
- BulkIndexer utility (important for #2192 fix)

### 2. opensearch-project/opensearch-go (Official OpenSearch)

**Current status:**
- v4.5.0 (June 2025) - Latest
- Forked from elastic/go-elasticsearch v7.13.0 (before product detection)
- Apache 2.0 license
- AWS/community maintained

**Platform support:**
- ✅ OpenSearch 1.x, 2.x, 3.x
- ❌ Elasticsearch (not designed for it)

**Key feature for us:** Built-in AWS SigV4 auth (`signer/awsv2` package) - critical for AWS OpenSearch users hitting #2192.

**API style:**
- Low-level only: `opensearchapi` package
- BulkIndexer utility (also fixes #2192)
- No high-level DSL (requires manual JSON)

### 3. olivere/elastic (Current, Deprecated)

**Status:** Maintainer declared "dormant" in Oct 2023. No bug fixes, including #2192. Migration is mandatory.

---

## Version Compatibility

I created a test matrix and found:

| Client | ES 6 | ES 7 | ES 8 | ES 9 | OS 1 | OS 2 | OS 3 |
|--------|------|------|------|------|------|------|------|
| go-elasticsearch/v9 | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| opensearch-go/v4 | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |

**Answer to "one library for both ES+OS?"** — No. Product detection makes this impossible with current official clients.

---

## What Jaeger Has Already Done

I was surprised to find Jaeger **already started migrating** to go-elasticsearch/v9:

**Already migrated** (see `config.go` line 283-289, `wrapper.go` line 62-69):
- Client initialization for ES 8+
- Template creation for ES 8+
- Dual-client infrastructure in wrapper

**NOT migrated yet:**
- ❌ Bulk operations (still uses olivere/elastic - **#2192 still broken**)
- ❌ Search operations
- ❌ Query DSL building (41 construction sites)
- ❌ OpenSearch support (forced to ES 7.x path)

Current code maps all OpenSearch versions → ES 7.x, which forces `olivere/elastic` usage:

```go
// config.go line 264-277
if strings.Contains(pingResult.TagLine, "OpenSearch") {
    logger.Info("OpenSearch detected, using ES 7.x index mappings")
    esVersion = 7  // Forces olivere/elastic
}
```

This means even ES 8+ users have #2192 because bulk operations haven't switched to BulkIndexer.

---

## API Differences vs. olivere/elastic

### Bulk Operations (Critical for #2192)

**olivere/elastic (broken):**
```go
bulkProc.BulkSize(5000000)  // "Soft" threshold, can exceed
```

**go-elasticsearch/opensearch-go (fixed):**
```go
BulkIndexer{
    FlushBytes: 5 * 1024 * 1024,  // Hard limit
}
```

Both new clients properly enforce byte limits. This fixes #2192.

### Query Building

**olivere/elastic** uses fluent DSL:
```go
elastic.NewBoolQuery().
    Must(elastic.NewMatchQuery("service", "frontend"))
```

**go-elasticsearch typed API** uses structs:
```go
&types.Query{
    Bool: &types.BoolQuery{
        Must: []types.Query{
            {Match: map[string]types.MatchQuery{
                "service": {Query: "frontend"},
            }},
        },
    },
}
```

**opensearch-go** requires JSON:
```go
queryJSON := `{"query":{"bool":{"must":[...]}}}`
```

Migration impact: I found 41 query construction sites that need updating.

---

## Internal Changes Required

I analyzed the codebase and found:

**Files affected:** ~38 files
- `config/config.go` - Client creation, BulkProcessor (priority 1)
- `wrapper/wrapper.go` - Shim layer (priority 1)
- `v1/spanstore/reader.go` - Query DSL (~1800 lines)
- `metricstore/query_builder.go` - Aggregations (~600 lines)
- Plus tests and supporting files

**Key changes:**

1. **Phase 1 - Bulk operations** (2-3 weeks)
   - Migrate from BulkProcessor to BulkIndexer
   - **Fixes #2192 immediately**
   - Update callbacks and metrics

2. **Phase 2 - Query DSL** (4-6 weeks)
   - 41 construction sites to update
   - Choice: Use typed API (verbose) or low-level JSON (simpler)
   - My suggestion: Start with low-level for faster migration

3. **Phase 3 - Testing** (2-3 weeks)
   - ES 7, 8, 9 integration tests
   - OS 1, 2, 3 integration tests
   - AWS OpenSearch (SigV4 auth)

The existing shim layer (`client/interfaces.go`) helps, but it exposes `elastic.Query` types directly. We'd need to either:
- Abstract query types (more work, cleaner)
- Live with client-specific query building (less work, acceptable)

---

## Recommendation

**Use dual-client strategy:**
- `go-elasticsearch/v9` for Elasticsearch
- `opensearch-go/v4` for OpenSearch
- Product detection at runtime (already partially implemented)

**Why not single client?**
- Can't use go-elasticsearch for OS (product detection blocks it)
- Can't use opensearch-go for ES (not designed for it)
- Downgrading to v7.13.1 leaves #2192 unfixed

**Why dual is okay:**
- Infrastructure already exists (see `clientV8` field in wrapper)
- BulkIndexer APIs are nearly identical between clients
- Low-level JSON queries work the same way
- Jaeger already has version detection logic

**Timeline estimate:** 8-12 weeks with 1 developer

---

## Questions & Next Steps

**Questions for maintainers:**

1. Do we have data on ES vs OS user distribution?
2. Should we prioritize fixing #2192 (Phase 1 only) before full migration?
3. Preference for typed API vs low-level JSON queries?

**Suggested next steps:**

1. Validate this analysis with maintainers
2. Create POC for BulkIndexer migration (fixes #2192)
3. Design abstracted query interface if desired
4. Phased rollout with feature flags

---

## References

- olivere/elastic deprecation: https://github.com/olivere/elastic/pull/1661
- Product detection discussion: https://github.com/opensearch-project/OpenSearch/issues/693
- Issue #2192 (AWS bug): https://github.com/jaegertracing/jaeger/issues/2192
- go-elasticsearch docs: https://www.elastic.co/guide/en/elasticsearch/client/go-api/current/
- opensearch-go docs: https://docs.opensearch.org/latest/clients/go/

---

*Note: This investigation focused on client library options only. Performance testing, detailed query API design, and migration tooling would be separate efforts if we proceed.*
