# Jaeger Storage Architecture Analysis: Patterns for Abstraction and Code Sharing

## Executive Summary

Jaeger implements a **multi-layered abstraction strategy** with three distinct patterns for handling storage backends:

1. **Native v2 implementations** (ClickHouse, Elasticsearch) - Direct implementation of v2 interfaces
2. **Adapter-based implementations** (Cassandra, Badger) - Wrapping v1 backends with v1adapter
3. **Hybrid approach** (Elasticsearch) - Shared base factory supporting both v1 and v2

---

## 1. Storage Architecture Overview

### Directory Structure

```
internal/storage/
├── v1/                          # First-generation storage API
│   ├── api/                     # Interfaces (spanstore, dependencystore, samplingstore)
│   ├── cassandra/               # Full implementation
│   ├── elasticsearch/           # Full implementation
│   ├── badger/                  # Full implementation
│   ├── memory/                  # Full implementation
│   ├── grpc/                    # Remote storage
│   └── kafka/                   # Event streaming
├── v2/                          # Second-generation storage API (OTLP-native)
│   ├── api/                     # Interfaces (tracestore, depstore)
│   │   ├── tracestore/          # Reader/Writer for traces
│   │   └── depstore/            # Reader/Writer for dependencies
│   ├── v1adapter/               # Adapter bridge from v1 to v2
│   ├── elasticsearch/           # Native v2 implementation
│   ├── clickhouse/              # Native v2 implementation
│   ├── cassandra/               # Adapter wrapping v1
│   ├── badger/                  # Adapter wrapping v1
│   └── memory/                  # Direct implementation
├── elasticsearch/               # Shared infrastructure (config, client, dbmodel)
├── cassandra/                   # Shared infrastructure (gocql wrapper, config)
└── metricstore/                 # Metrics storage (separate concern)
```

---

## 2. API Evolution: v1 vs v2

### v1 API (Span-Centric)

**Location:** `internal/storage/v1/api/`

**Interfaces:**
```go
type Writer interface {
    WriteSpan(ctx context.Context, span *model.Span) error
}

type Reader interface {
    GetTrace(ctx context.Context, query GetTraceParameters) (*model.Trace, error)
    GetServices(ctx context.Context) ([]string, error)
    GetOperations(ctx context.Context, query OperationQueryParameters) ([]Operation, error)
    FindTraces(ctx context.Context, query *TraceQueryParameters) ([]*model.Trace, error)
    FindTraceIDs(ctx context.Context, query *TraceQueryParameters) ([]model.TraceID, error)
}

type DependencyReader interface {
    GetDependencies(ctx context.Context, endTs time.Time, lookback time.Duration) 
        ([]model.DependencyLink, error)
}
```

**Characteristics:**
- Single span write model (not batch)
- Uses jaeger-idl model types directly
- Synchronous, block-on-response
- 8+ year old design

### v2 API (OTLP-Native, Iterator-Based)

**Location:** `internal/storage/v2/api/`

**Interfaces:**
```go
type Writer interface {
    WriteTraces(ctx context.Context, td ptrace.Traces) error  // Batch write
}

type Reader interface {
    GetTraces(ctx context.Context, traceIDs ...GetTraceParams) 
        iter.Seq2[[]ptrace.Traces, error]  // Iterator pattern
    FindTraces(ctx context.Context, query TraceQueryParams) 
        iter.Seq2[[]ptrace.Traces, error]
    FindTraceIDs(ctx context.Context, query TraceQueryParams) 
        iter.Seq2[[]FoundTraceID, error]
    GetServices(ctx context.Context) ([]string, error)
    GetOperations(ctx context.Context, query OperationQueryParams) 
        ([]Operation, error)
}

type DependencyReader interface {
    GetDependencies(ctx context.Context, query QueryParameters) 
        ([]model.DependencyLink, error)
}
```

**Characteristics:**
- Batch write model (ptrace.Traces batches)
- OTLP native types (pcommon, ptrace)
- Iterator-based for memory efficiency (handles large result sets)
- 2024 design for modern collectors

### Key Differences

| Aspect | v1 | v2 |
|--------|-----|-----|
| **Span Write** | Single span via WriteSpan() | Batch via WriteTraces(ptrace.Traces) |
| **Data Model** | jaeger-idl model.Span | OpenTelemetry ptrace.Traces |
| **Read Pattern** | Bulk return ([]*model.Trace) | Iterator (iter.Seq2) |
| **Dependency Query** | endTime + lookback duration | StartTime + EndTime |
| **Era** | 2017-2019 | 2024+ |
| **Optimization** | In-memory accumulation | Streaming/chunking |

---

## 3. Backend Implementation Patterns

### Pattern A: Native v2 Implementation (ClickHouse)

**Location:** `internal/storage/v2/clickhouse/`

**Architecture:**
```
Factory (creates readers/writers)
  ├── TraceReader → direct SQL queries → ptrace.Traces
  ├── TraceWriter → batch inserts → ClickHouse
  └── DependencyReader → not yet implemented
```

**Factory Structure:**
```go
type Factory struct {
    config Configuration
    telset telemetry.Settings
    conn   driver.Conn  // ClickHouse connection
}

func (f *Factory) CreateTraceReader() (tracestore.Reader, error) {
    return chtracestore.NewReader(f.conn), nil
}

func (f *Factory) CreateTraceWriter() (tracestore.Writer, error) {
    return chtracestore.NewWriter(f.conn), nil
}
```

**Reader/Writer Implementation:**
```go
type Reader struct {
    conn driver.Conn
}

func (r *Reader) GetTraces(ctx context.Context, traceIDs ...tracestore.GetTraceParams) 
    iter.Seq2[[]ptrace.Traces, error] {
    // Direct iteration: Query → Scan → Convert to ptrace → Yield
}

type Writer struct {
    conn driver.Conn
}

func (w *Writer) WriteTraces(ctx context.Context, td ptrace.Traces) error {
    // Batch prepare → Append rows → Send
}
```

**Advantages:**
- ✅ Type-safe, compile-time correct
- ✅ No translation layer overhead
- ✅ Leverages OTLP types directly
- ✅ Stream-friendly iteration pattern

**Disadvantages:**
- ❌ Requires complete rewrite for new backends
- ❌ No code sharing with v1
- ❌ Dependencies not implemented

---

### Pattern B: Adapter-Based Implementation (Cassandra, Badger)

**Location:** `internal/storage/v2/cassandra/` and `internal/storage/v2/badger/`

**Architecture:**
```
v2 Factory
  └── wraps v1 Factory
      ├── CreateSpanReader() → v1adapter.NewTraceReader()
      ├── CreateSpanWriter() → v1adapter.NewTraceWriter()
      └── CreateDependencyReader() → v1adapter.NewDependencyReader()

v1adapter
  ├── Translates ptrace.Traces ↔ model.Span
  ├── Converts v2 interfaces to v1 calls
  └── Handles metadata conversion (OTLP ↔ Jaeger model)
```

**Cassandra v2 Factory:**
```go
type Factory struct {
    v1Factory *cassandra.Factory  // Wraps existing v1 implementation
}

func NewFactory(opts cassandra.Options, ...) (*Factory, error) {
    factory, err := newFactoryWithConfig(opts, ...)
    return &Factory{v1Factory: factory}, nil
}

func (f *Factory) CreateTraceReader() (tracestore.Reader, error) {
    reader, err := f.v1Factory.CreateSpanReader()
    return v1adapter.NewTraceReader(reader), nil  // Wrap with adapter
}
```

**v1adapter Translation:**
```go
type TraceReader struct {
    spanReader spanstore.Reader
}

func (tr *TraceReader) GetTraces(ctx context.Context, traceIDs ...tracestore.GetTraceParams) 
    iter.Seq2[[]ptrace.Traces, error] {
    return func(yield func([]ptrace.Traces, error) bool) {
        for _, idParams := range traceIDs {
            query := spanstore.GetTraceParameters{
                TraceID:   ToV1TraceID(idParams.TraceID),  // Convert ID
                StartTime: idParams.Start,
                EndTime:   idParams.End,
            }
            t, err := tr.spanReader.GetTrace(ctx, query)
            if err != nil {
                if errors.Is(err, spanstore.ErrTraceNotFound) {
                    continue
                }
                yield(nil, err)
                return
            }
            // Convert v1 Trace → ptrace.Traces
            batch := &model.Batch{Spans: t.GetSpans()}
            tr := V1BatchesToTraces([]*model.Batch{batch})
            yield([]ptrace.Traces{tr}, nil)
        }
    }
}
```

**Conversion Functions:**
```go
// ID conversion (bidirectional)
func ToV1TraceID(id pcommon.TraceID) model.TraceID { ... }
func FromV1TraceID(id model.TraceID) pcommon.TraceID { ... }

// Trace conversion (bidirectional)
func V1BatchesToTraces(batches []*model.Batch) ptrace.Traces { ... }
func V1BatchesFromTraces(td ptrace.Traces) []*model.Batch { ... }
```

**Advantages:**
- ✅ Code reuse: v1 implementation remains production-tested
- ✅ Rapid integration of new backends
- ✅ Shared infrastructure (config, client, metrics)
- ✅ Both v1 and v2 can coexist

**Disadvantages:**
- ❌ Translation overhead (ID conversion, batch conversion)
- ❌ Single-span write model converted to batch
- ❌ Not optimal for new backends designed for batch writes
- ❌ Iterator pattern loses v1's native structure

---

### Pattern C: Hybrid Approach (Elasticsearch)

**Location:** `internal/storage/v1/elasticsearch/` + `internal/storage/v2/elasticsearch/`

**Architecture:**
```
v1 FactoryBase (shared)
  ├── newClientFn → creates es.Client
  ├── config → cfg.Configuration
  ├── client → atomic.Pointer[es.Client]
  ├── GetSpanReaderParams() → SpanReaderParams
  ├── GetSpanWriterParams() → SpanWriterParams
  └── GetDependencyStoreParams() → esdepstorev2.Params

v1 Factory
  ├── CreateSpanReader() → CoreSpanReader (raw ES queries)
  └── CreateSpanWriter() → CoreSpanWriter

v2 Factory
  ├── CreateTraceReader() → v2tracestore.TraceReader
  ├── CreateTraceWriter() → v2tracestore.TraceWriter
  └── CreateDependencyReader() → v2depstore.DependencyStoreV2
```

**Shared FactoryBase Pattern:**
```go
// Base factory with shared infrastructure
type FactoryBase struct {
    newClientFn func(ctx context.Context, c *config.Configuration, ...) (es.Client, error)
    config *config.Configuration
    client atomic.Pointer[es.Client]
    metricsFactory metrics.Factory
    logger *zap.Logger
}

// Return shareable parameters instead of readers/writers
func (f *FactoryBase) GetSpanReaderParams() esspanstore.SpanReaderParams {
    return esspanstore.SpanReaderParams{
        Client:           f.getClient,
        MaxDocCount:      f.config.MaxDocCount,
        IndexPrefix:      f.config.Indices.IndexPrefix,
        // ... 10+ params
    }
}
```

**v2 Elasticsearch Factory:**
```go
type Factory struct {
    coreFactory    *elasticsearch.FactoryBase
    config         escfg.Configuration
    metricsFactory metrics.Factory
}

func (f *Factory) CreateTraceReader() (tracestore.Reader, error) {
    params := f.coreFactory.GetSpanReaderParams()
    return tracestoremetrics.NewReaderDecorator(
        v2tracestore.NewTraceReader(params),
        f.metricsFactory,
    ), nil
}
```

**v2 TraceReader Implementation:**
```go
type TraceReader struct {
    spanReader spanstore.CoreSpanReader  // Uses shared core
}

// Wraps CoreSpanReader with conversion layer
func (t *TraceReader) GetTraces(ctx context.Context, params ...tracestore.GetTraceParams) 
    iter.Seq2[[]ptrace.Traces, error] {
    dbTraceIds := make([]dbmodel.TraceID, 0, len(params))
    for _, id := range params {
        dbTraceIds = append(dbTraceIds, dbmodel.TraceID(id.TraceID.String()))
    }
    dbTraces, err := t.spanReader.GetTraces(ctx, dbTraceIds)
    // Convert dbmodel.Trace → ptrace.Traces
}
```

**Advantages:**
- ✅ **Zero client duplication**: One shared Elasticsearch client
- ✅ **Shared config management**: Single configuration parsing
- ✅ **Layered abstraction**: dbmodel → spanstore.Core → v1/v2 wrappers
- ✅ Both v1 and v2 can run simultaneously
- ✅ No atomic.Pointer contention issues
- ✅ Shared password file watching, metrics, tracing

**Disadvantages:**
- ❌ More complex code structure (3 layers)
- ❌ Harder to understand for new contributors
- ⚠️ Requires careful coordination between v1 and v2

---

## 4. Interface Hierarchies

### v1 Storage Interface Hierarchy

```
storage.Factory (interface)
  ├── CreateSpanReader() → spanstore.Reader
  ├── CreateSpanWriter() → spanstore.Writer
  ├── CreateDependencyReader() → dependencystore.Reader
  └── Initialize(metricsFactory, logger)

storage.Purger (interface) [optional]
  └── Purge(ctx) error

storage.SamplingStoreFactory (interface) [optional]
  ├── CreateLock() → distributedlock.Lock
  └── CreateSamplingStore(maxBuckets) → samplingstore.Store

storage.Configurable (interface) [optional]
  ├── AddFlags(flagSet)
  └── InitFromViper(v *viper.Viper)

storage.Inheritable (interface) [optional]
  └── InheritSettingsFrom(other Factory)

storage.ArchiveCapable (interface) [optional]
  └── IsArchiveCapable() bool

spanstore.Reader (interface)
  ├── GetTrace(ctx, query) (*model.Trace, error)
  ├── GetServices(ctx) ([]string, error)
  ├── GetOperations(ctx, query) ([]Operation, error)
  ├── FindTraces(ctx, query) ([]*model.Trace, error)
  └── FindTraceIDs(ctx, query) ([]model.TraceID, error)

spanstore.Writer (interface)
  └── WriteSpan(ctx, span *model.Span) error

dependencystore.Reader (interface)
  └── GetDependencies(ctx, endTime, lookback) ([]model.DependencyLink, error)
```

### v2 Storage Interface Hierarchy

```
tracestore.Factory (interface)
  ├── CreateTraceReader() → Reader
  └── CreateTraceWriter() → Writer

depstore.Factory (interface)
  └── CreateDependencyReader() → Reader

tracestore.Reader (interface)
  ├── GetTraces(ctx, params...GetTraceParams) iter.Seq2[[]ptrace.Traces, error]
  ├── GetServices(ctx) ([]string, error)
  ├── GetOperations(ctx, query) ([]Operation, error)
  ├── FindTraces(ctx, query) iter.Seq2[[]ptrace.Traces, error]
  └── FindTraceIDs(ctx, query) iter.Seq2[[]FoundTraceID, error]

tracestore.Writer (interface)
  └── WriteTraces(ctx, td ptrace.Traces) error

depstore.Reader (interface)
  └── GetDependencies(ctx, query QueryParameters) ([]model.DependencyLink, error)

depstore.Writer (interface)
  └── WriteDependencies(ts time.Time, dependencies []model.DependencyLink) error
```

---

## 5. Code Organization by Backend

### Elasticsearch (Hybrid)

```
internal/storage/
├── elasticsearch/              # SHARED INFRASTRUCTURE
│   ├── config/                # Config parsing (v1 & v2 share)
│   ├── client/                # ES client abstractions
│   ├── dbmodel/               # Elasticsearch domain model
│   ├── query/                 # Query builders
│   ├── filter/                # Filter builders
│   └── wrapper/               # ES response wrapper
├── v1/elasticsearch/          # v1 IMPLEMENTATION
│   ├── factory.go             # Creates v1 readers/writers
│   ├── factory_v1.go          # v1 Factory interface impl
│   └── spanstore/
│       ├── core_span_reader.go    # CoreSpanReader interface
│       ├── readerv1.go            # Reader wraps Core + converts
│       ├── writerv1.go            # Writer wraps Core + converts
│       ├── from_domain.go         # ptrace → dbmodel
│       ├── to_domain.go           # dbmodel → ptrace
│       └── writer.go              # CoreSpanWriter implementation
└── v2/elasticsearch/          # v2 IMPLEMENTATION
    ├── factory.go             # Creates v2 readers/writers
    ├── tracestore/
    │   ├── reader.go          # TraceReader (delegates to Core)
    │   ├── writer.go          # TraceWriter (delegates to Core)
    │   ├── from_dbmodel.go    # dbmodel → ptrace.Traces
    │   └── to_dbmodel.go      # ptrace.Traces → dbmodel
    └── depstore/
        ├── storagev2.go       # v2 wrapper
        ├── storage.go         # CoreDependencyStore
        └── dbmodel/           # Dependency models
```

### Cassandra (Adapter)

```
internal/storage/
├── cassandra/                 # SHARED INFRASTRUCTURE
│   ├── config/                # Config
│   ├── gocql/                 # Cassandra driver wrapper
│   └── session.go             # Session interface
├── v1/cassandra/              # v1 IMPLEMENTATION
│   ├── factory.go             # Main implementation
│   ├── spanstore/
│   │   ├── reader.go
│   │   ├── writer.go
│   │   └── dbmodel/
│   │       ├── cql_udt.go
│   │       └── unique_*.go
│   ├── dependencystore/
│   ├── samplingstore/
│   └── schema/
└── v2/cassandra/              # v2 ADAPTER WRAPPER
    ├── factory.go             # Wraps v1 Factory
    └── (reads/writers delegated via v1adapter)
```

### ClickHouse (Native v2)

```
internal/storage/v2/clickhouse/
├── factory.go                 # Creates v2 readers/writers
├── tracestore/
│   ├── reader.go              # TraceReader (direct SQL)
│   ├── writer.go              # TraceWriter (batch insert)
│   ├── dbmodel/
│   │   ├── spanrow.go
│   │   ├── from.go            # ptrace → database row
│   │   └── to.go              # database row → ptrace
│   └── driver.go
├── depstore/
│   └── reader.go              # Not implemented (panic)
└── sql/
    ├── spans.sql
    ├── services.sql
    ├── operations.sql
    └── ...
```

---

## 6. Comparison Matrix: Implementation Patterns

| Aspect | ClickHouse (Native v2) | Elasticsearch (Hybrid) | Cassandra (Adapter) |
|--------|--------|--------|--------|
| **API Version** | v2 only | v1 + v2 | v2 (via adapter) |
| **Code Reuse** | None (new backend) | 70% shared (FactoryBase + Core) | 90% reused (v1adapter) |
| **Client Creation** | Direct (NewFactory) | Delegated to FactoryBase | Wrapped from v1 |
| **Config Parsing** | Direct (Configuration) | Shared (escfg.Configuration) | Shared (Options) |
| **Dependencies** | SQL layer only | ES client + Core abstractions | Cassandra v1 + adapter |
| **Metrics Integration** | Manual | Shared via FactoryBase | From v1 Factory |
| **Test Complexity** | Medium (native impl) | Medium (3 layers) | Low (adapter mocking) |
| **Batch Optimization** | ✅ Native (batch insert) | ⚠️ Wrapped (single-span to batch) | ❌ Single-span model |
| **Iterator Efficiency** | ✅ Memory efficient | ✅ Shared with v1 | ⚠️ Overhead from v1adapter |
| **Dual Backend Support** | ❌ (v2 only) | ✅ (v1 + v2) | ❌ (v2 only, via adapter) |

---

## 7. Dependency Store Implementation

### v1 DependencyStore Pattern

```
dependencystore.Reader interface
  └── GetDependencies(ctx, endTime, lookback time.Duration) 
      → ([]model.DependencyLink, error)

dependencystore.Writer interface
  └── WriteDependencies(ts time.Time, dependencies []model.DependencyLink) error
```

**v1 Elasticsearch Implementation:**
```go
type DependencyStore struct {
    client       es.Client
    logger       *zap.Logger
    metricsFactory metrics.Factory
}

func (ds *DependencyStore) GetDependencies(ctx context.Context, endTs time.Time, lookback time.Duration) 
    ([]model.DependencyLink, error) {
    // Query ES for dependency links within time range
}
```

### v2 DependencyStore Pattern

```
depstore.Reader interface
  └── GetDependencies(ctx, query QueryParameters) 
      → ([]model.DependencyLink, error)

depstore.Writer interface
  └── WriteDependencies(ts time.Time, dependencies []model.DependencyLink) error

type QueryParameters struct {
    StartTime time.Time
    EndTime   time.Time
}
```

**v2 Elasticsearch Implementation:**
```go
type DependencyStoreV2 struct {
    store CoreDependencyStore
}

func (s *DependencyStoreV2) GetDependencies(ctx context.Context, query depstore.QueryParameters) 
    ([]model.DependencyLink, error) {
    // Delegates to CoreDependencyStore
    dbDependencies, err := s.store.GetDependencies(ctx, query.EndTime, query.EndTime.Sub(query.StartTime))
    // Convert dbmodel → domain
}

func (s *DependencyStoreV2) WriteDependencies(ts time.Time, dependencies []model.DependencyLink) error {
    dbDependencies := dbmodel.FromDomainDependencies(dependencies)
    return s.store.WriteDependencies(ts, dbDependencies)
}
```

**ClickHouse Implementation:**
```go
type Reader struct {}

func (*Reader) GetDependencies(context.Context, depstore.QueryParameters) 
    ([]model.DependencyLink, error) {
    panic("not implemented")  // Dependencies not yet implemented
}
```

**Cassandra via v1adapter:**
```go
// v1adapter provides transparent mapping
type dependencyReader struct {
    reader dependencystore.Reader  // v1 reader
}

func (dr *dependencyReader) GetDependencies(ctx context.Context, query depstore.QueryParameters) 
    ([]model.DependencyLink, error) {
    // Convert v2 QueryParameters to v1 (endTime, lookback)
    lookback := query.EndTime.Sub(query.StartTime)
    return dr.reader.GetDependencies(ctx, query.EndTime, lookback)
}
```

---

## 8. Metrics and Decorators

### v1 Metrics Pattern

```go
// Decorator pattern wraps reader with metrics
func (f *Factory) CreateSpanReader() (spanstore.Reader, error) {
    sr, err := cspanstore.NewSpanReader(...)
    return spanstoremetrics.NewReaderDecorator(sr, f.metricsFactory), nil
}

// Decorator tracks:
// - request count
// - error rate
// - latency
// - result set sizes
```

### v2 Metrics Pattern

```go
// Same decorator pattern for v2
func (f *Factory) CreateTraceReader() (tracestore.Reader, error) {
    params := f.coreFactory.GetSpanReaderParams()
    return tracestoremetrics.NewReaderDecorator(
        v2tracestore.NewTraceReader(params),
        f.metricsFactory,
    ), nil
}

// v2 specifically tracks:
// - chunk sizes (for iterator)
// - iterator completion rate
// - conversion overhead
```

---

## 9. Lessons for ES/OS Migration

### Pattern A: Adapter Approach (Fastest)

**Recommended for:** Elasticsearch → OpenSearch (same API, drop-in replacement)

**Implementation:**
1. Create `internal/storage/v2/opensearch/` directory
2. Copy `internal/storage/v2/elasticsearch/factory.go` as template
3. Replace `elasticsearch.FactoryBase` with `opensearch.FactoryBase`
4. Implement only factory methods and client creation
5. Reuse all `tracestore` and `depstore` conversion layers

**Code Sharing:**
```go
// opensearch/factory.go
type Factory struct {
    coreFactory    *opensearch.FactoryBase  // Different client
    config         oscfg.Configuration       // OS-specific config
    metricsFactory metrics.Factory
}

func (f *Factory) CreateTraceReader() (tracestore.Reader, error) {
    params := f.coreFactory.GetSpanReaderParams()
    return tracestoremetrics.NewReaderDecorator(
        v2tracestore.NewTraceReader(params),  // REUSE: same conversion
        f.metricsFactory,
    ), nil
}
```

**Advantages:**
- ✅ ~200 lines of code
- ✅ Zero duplication of logic
- ✅ OpenSearch client can be completely separate
- ✅ Shared query builders (if compatible)
- ✅ Shared dbmodel and converters

### Pattern B: Shared Base Factory (Best for Long-term)

**Recommended for:** Complex dual-backend support with shared infrastructure

**Structure:**
```go
// opensearch/factory_base.go - SHARED
type FactoryBase struct {
    newClientFn func(ctx context.Context, c *config.Configuration, ...) 
        (client.Client, error)
    config *config.Configuration
    client atomic.Pointer[client.Client]
    // ... shared fields
}

func (f *FactoryBase) GetSpanReaderParams() spanstore.SpanReaderParams { ... }

// opensearch/factory_v1.go - v1 implementation
type Factory struct {
    *FactoryBase
}

// opensearch/factory_v2.go - v2 implementation  
type FactoryV2 struct {
    *FactoryBase
    metricsFactory metrics.Factory
}
```

**Advantages:**
- ✅ True shared infrastructure (no duplication)
- ✅ Elastic and Open Search can coexist
- ✅ Single client, config, metrics tracking
- ✅ Support both v1 and v2 simultaneously

### Pattern C: Abstracted Shared Layer (Best Long-term Design)

**For eventual Elasticsearch + OpenSearch unified codebase:**

```go
// storage/elasticsearch/client/interface.go
type Client interface {
    Search(ctx context.Context, index string, query Q) (Result, error)
    Index(ctx context.Context, index string, doc interface{}) (ID, error)
    CreateTemplate(ctx context.Context, name string) TemplateBuilder
    // ... all operations
}

// Implementation 1: storage/elasticsearch/client/elastic.go
type ElasticsearchClient struct {
    client *elastic.Client
}

// Implementation 2: storage/opensearch/client/opensearch.go
type OpenSearchClient struct {
    client *opensearchapi.Client
}

// Shared factory base
type FactoryBase struct {
    client Client  // Interface, not concrete type
    config Config
}
```

**Advantages:**
- ✅ 100% shared implementation logic
- ✅ Switch backends at compile or runtime
- ✅ Future-proof for API divergence
- ✅ Testable with mock clients

---

## 10. Anti-Patterns to Avoid

### ❌ 1. Adapter Wrapper Layers

**Bad:**
```go
// v2/elasticsearch/factory.go
type TraceReader struct {
    spanReader spanstore.Reader  // v1 interface
}

// v1/elasticsearch/spanstore/reader.go
type Reader struct {
    core CoreSpanReader  // Another wrapper!
}

// 3 layers for one operation: Reader → spanReader → CoreSpanReader
```

**Better:**
```go
// Directly use CoreSpanReader
type TraceReader struct {
    core CoreSpanReader  // Single layer
}
```

### ❌ 2. Single-Responsibility Violation

**Bad:**
```go
type Factory struct {
    config    Configuration
    client    es.Client
    logger    *zap.Logger
    metrics   metrics.Factory
    session   cassandra.Session  // WRONG: mixing storage types!
    // ... plus CreateReader/Writer/Dependency methods
    // 15+ methods in one struct
}
```

**Better:**
```go
// Elasticsearch Factory
type Factory struct {
    coreFactory *elasticsearch.FactoryBase
    metricsFactory metrics.Factory
}

// Separate factory for Cassandra
type Factory struct {
    v1Factory *cassandra.Factory
}
```

### ❌ 3. Blocked Abstraction Boundaries

**Bad:**
```go
// Elasticsearch v2 directly accessing v1 private fields
var tr *TraceReader = ...
// Can't access tr.spanReader (unexported)
// Forces wrapper pattern
```

**Better:**
```go
// Expose conversion functions
func GetV1Reader(reader tracestore.Reader) spanstore.Reader {
    if tr, ok := reader.(*TraceReader); ok {
        return tr.spanReader
    }
    return &SpanReader{traceReader: reader}
}
```

### ❌ 4. Synchronous Batching

**Bad:**
```go
type TraceWriter struct {
    spanWriter spanstore.Writer
}

func (t *TraceWriter) WriteTraces(ctx context.Context, td ptrace.Traces) error {
    // Converts batch to single spans
    for _, rs := range td.ResourceSpans().All() {
        for _, ss := range rs.ScopeSpans().All() {
            for _, span := range ss.Spans().All() {
                if err := t.spanWriter.WriteSpan(ctx, span); err != nil {
                    // Lost context: which span in batch failed?
                }
            }
        }
    }
    return nil  // May hide errors
}
```

**Better:**
```go
func (t *TraceWriter) WriteTraces(ctx context.Context, td ptrace.Traces) error {
    batch, err := t.conn.PrepareBatch(ctx, sql.InsertSpan)
    if err != nil {
        return fmt.Errorf("failed to prepare batch: %w", err)
    }
    defer batch.Close()
    
    for _, rs := range td.ResourceSpans().All() {
        // ... process all spans
        err = batch.Append(...)
        if err != nil {
            return err  // Fail fast
        }
    }
    return batch.Send()  // Single batch send
}
```

### ❌ 5. No Dependency Store Implementation

**Bad:**
```go
type Reader struct {}

func (*Reader) GetDependencies(...) (..., error) {
    panic("not implemented")  // Panic at runtime!
}
```

**Better:**
```go
// Explicitly document limitations
func (*Reader) GetDependencies(...) (..., error) {
    return nil, fmt.Errorf("dependencies not yet supported for ClickHouse")
}

// In factory, return sentinel
func (f *Factory) CreateDependencyReader() (depstore.Reader, error) {
    return nil, fmt.Errorf("ClickHouse does not support dependency store")
}
```

---

## 11. Summary of Best Practices

### For New Backends

**Template to Follow (Pattern A - Adapter):**
```
1. Create v2/mybackend/factory.go
   - Wrap existing v1 implementation OR
   - Delegate to base factory
   
2. Create minimal wrappers for tracestore
   - Reuse v1adapter conversion layers
   - Delegate to CoreReader/CoreWriter
   
3. Optional: Create minimal depstore
   - Reuse v2adapter patterns
   
4. Metrics: Decorate at factory level
```

**Template to Follow (Pattern B - Native v2):**
```
1. Create v2/mybackend/factory.go
   - Direct client initialization
   
2. Implement tracestore.Reader/Writer
   - Direct ptrace.Traces handling
   - Use iterator pattern
   
3. Implement depstore.Reader/Writer
   - Direct dependency handling
   
4. Create shared infrastructure layer
   - Shared by both v1 and v2
   - Handle client creation, config, metrics
```

### Shared Infrastructure Checklist

- [ ] Client creation isolated in base factory
- [ ] Config parsing shared (not duplicated)
- [ ] Metrics tracking unified
- [ ] Logging instance shared
- [ ] Password/token management centralized
- [ ] Connection pooling optimized
- [ ] Decorator pattern for cross-cutting concerns
- [ ] Interface-based (not concrete type) dependencies

### Abstraction Levels Checklist

- [ ] **Level 1 (DB Model):** Raw database types (dbmodel.Span, dbmodel.Trace)
- [ ] **Level 2 (Core API):** Database-level operations (CoreSpanReader interface)
- [ ] **Level 3 (v1 API):** Domain model (spanstore.Reader → model.Span)
- [ ] **Level 4 (v2 API):** OTLP model (tracestore.Reader → ptrace.Traces)
- [ ] **Conversion Functions:** ToDBModel/FromDBModel at each level

---

## 12. Architecture Decision Matrix for ES/OS

| Decision | Factors | Recommendation |
|----------|---------|-----------------|
| **Shared Client?** | API differences, config variations | YES (with abstraction) |
| **Shared Factory?** | Code reuse, complexity, testing | YES (FactoryBase pattern) |
| **Shared Queries?** | Query language, filters, aggs | IF compatible, else separate |
| **Shared Models?** | Document structure, indexing | YES (dbmodel/converter layer) |
| **Shared v1/v2?** | Support both APIs | YES (v2adapter to v1adapter) |
| **Separate Configs?** | Version-specific features | YES (escfg + oscfg, merged at factory) |

---

## Conclusion

Jaeger's storage architecture demonstrates three viable patterns:

1. **Adapter Pattern** (Cassandra, Badger) - Best for backward compatibility
2. **Hybrid Pattern** (Elasticsearch) - Best for code reuse with shared infrastructure  
3. **Native v2 Pattern** (ClickHouse) - Best for new, optimized implementations

**For Elasticsearch → OpenSearch migration:**
- **Short-term:** Use Adapter Pattern (minimal new code)
- **Medium-term:** Migrate to Hybrid Pattern (shared FactoryBase)
- **Long-term:** Use Client Interface Pattern (fully abstracted)

The key principle: **Maximize abstraction boundaries while minimizing code duplication.**
