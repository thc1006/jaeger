# Storage Backend Implementation Comparison

## Quick Reference Matrix

### Code Reuse & Sharing

```
┌─────────────┬──────────┬──────────────┬─────────────┐
│ Backend     │ Lines    │ Code Reuse   │ Pattern     │
├─────────────┼──────────┼──────────────┼─────────────┤
│ ClickHouse  │ ~1500    │ 0% (native)  │ v2 only     │
│ Elasticsearch│ ~3000    │ 70% (shared) │ Hybrid      │
│ Cassandra   │ ~400     │ 90% (wrap)   │ v1adapter   │
│ Badger      │ ~150     │ 95% (wrap)   │ v1adapter   │
│ Memory      │ ~150     │ 100% (same)  │ v1 native   │
└─────────────┴──────────┴──────────────┴─────────────┘
```

### Backend Capability Matrix

```
                       │ Tracing │ Dependencies │ Sampling │ v1 │ v2 │
───────────────────────┼─────────┼──────────────┼──────────┼────┼────┤
ClickHouse (v2)        │   ✅    │      ❌      │    ❌    │ ❌ │ ✅ │
Elasticsearch (hybrid) │   ✅    │      ✅      │    ✅    │ ✅ │ ✅ │
Cassandra (v2 wrapper) │   ✅    │      ✅      │    ✅    │ ✅ │ ✅ │
Badger (v2 wrapper)    │   ✅    │      ✅      │    ✅    │ ✅ │ ✅ │
Memory (both)          │   ✅    │      ✅      │    ✅    │ ✅ │ ✅ │
Blackhole (v1)         │   ✅    │      ❌      │    ❌    │ ✅ │ ❌ │
```

---

## Implementation Archetypes

### Archetype 1: Adapter Pattern (Wrap v1 in v2)

**Backends:** Cassandra, Badger

**When to use:** Backend already has mature v1 implementation

**Code Structure:**
```
v2/cassandra/factory.go (30 lines)
  └── wraps v1/cassandra/factory.go
      └── uses v1adapter converters

v2/v1adapter/ (shared for all adapters)
  ├── tracereader.go (iterator bridge)
  ├── tracewriter.go (batch to span conversion)
  ├── depreader.go (parameter mapping)
  └── translator.go (ptrace ↔ model conversions)
```

**Factory Code:**
```go
// v2/cassandra/factory.go
type Factory struct {
    v1Factory *cassandra.Factory
}

func (f *Factory) CreateTraceReader() (tracestore.Reader, error) {
    reader, err := f.v1Factory.CreateSpanReader()
    if err != nil {
        return nil, err
    }
    return v1adapter.NewTraceReader(reader), nil
}
```

**Cost/Benefit:**
- Code: ~50 lines
- Reuse: 95% (only factory wrapper)
- Performance: ⚠️ Overhead from conversion
- Maintenance: ✅ Easy (delegates to v1)

---

### Archetype 2: Hybrid Pattern (Shared Base + Implementations)

**Backends:** Elasticsearch

**When to use:** Need both v1 and v2 simultaneously with shared infrastructure

**Code Structure:**
```
elasticsearch/ (shared)
  ├── config/
  ├── client/
  ├── dbmodel/

v1/elasticsearch/ (v1 implementation)
  ├── factory.go (creates v1 readers/writers)
  └── spanstore/
      ├── core_span_reader.go (interface, DB level)
      ├── readerv1.go (wraps core)
      └── writerv1.go (wraps core)

v2/elasticsearch/ (v2 implementation)
  ├── factory.go (creates v2 readers/writers)
  └── tracestore/
      ├── reader.go (wraps core, converts ptrace)
      └── writer.go (wraps core, converts ptrace)
```

**Factory Pattern:**
```go
// v1/elasticsearch/factory.go
type FactoryBase struct {
    client   es.Client
    config   Configuration
}

func (f *FactoryBase) GetSpanReaderParams() SpanReaderParams {
    return SpanReaderParams{
        Client: f.getClient,
        // ... 10+ shared params
    }
}

// v2/elasticsearch/factory.go
type Factory struct {
    coreFactory *elasticsearch.FactoryBase
}

func (f *Factory) CreateTraceReader() (tracestore.Reader, error) {
    params := f.coreFactory.GetSpanReaderParams()
    return v2tracestore.NewTraceReader(params), nil
}
```

**Cost/Benefit:**
- Code: ~300 lines
- Reuse: 70% (shared FactoryBase + client)
- Performance: ✅ Native, no translation
- Maintenance: ⚠️ Complex (3 layers)

---

### Archetype 3: Native v2 Pattern (Direct Implementation)

**Backends:** ClickHouse

**When to use:** New backend, optimized for v2 APIs

**Code Structure:**
```
v2/clickhouse/ (complete, no v1)
  ├── factory.go
  ├── tracestore/
  │   ├── reader.go (ptrace.Traces directly)
  │   ├── writer.go (batch inserts)
  │   └── dbmodel/ (row structures)
  ├── depstore/
  │   └── reader.go (not implemented yet)
  └── sql/ (queries)
```

**Factory Code:**
```go
// v2/clickhouse/factory.go
type Factory struct {
    conn driver.Conn
}

func (f *Factory) CreateTraceReader() (tracestore.Reader, error) {
    return chtracestore.NewReader(f.conn), nil
}

func (f *Factory) CreateTraceWriter() (tracestore.Writer, error) {
    return chtracestore.NewWriter(f.conn), nil
}
```

**Cost/Benefit:**
- Code: ~1500 lines
- Reuse: 0% (new backend)
- Performance: ✅ Optimal (no layers)
- Maintenance: ✅ Simple (single pattern)

---

## Deep Dive: Elasticsearch Hybrid Architecture

### Layer Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│  (Uses v1 or v2 interfaces transparently)                   │
└────────────────┬──────────────────────────────┬─────────────┘
                 │                              │
        ┌────────▼─────────┐         ┌──────────▼──────┐
        │  v1 Factory      │         │  v2 Factory     │
        │  ├─ NewFactory   │         │  ├─ NewFactory  │
        │  └─ Init methods │         │  └─ Init methods│
        └────────┬─────────┘         └────────┬────────┘
                 │                           │
        ┌────────▼──────────────────────────▼──┐
        │  FactoryBase (shared infrastructure) │
        │  ├─ newClientFn (client factory)     │
        │  ├─ config (ES config)               │
        │  ├─ client (singleton)               │
        │  └─ GetSpanReaderParams()            │
        │     GetSpanWriterParams()            │
        │     GetDependencyStoreParams()       │
        └────────────┬───────────────────────┘
                     │
        ┌────────────▼────────────────┐
        │    ES Client Layer          │
        │  (es.Client interface)      │
        │  ├─ Search                  │
        │  ├─ Index                   │
        │  ├─ CreateTemplate          │
        │  └─ DeleteIndex             │
        └────────────┬────────────────┘
                     │
        ┌────────────▼────────────────┐
        │  Core Reader/Writer Layer   │
        │  ├─ CoreSpanReader          │
        │  │  ├─ FindTraces           │
        │  │  ├─ GetOperations        │
        │  │  └─ GetServices          │
        │  ├─ CoreSpanWriter          │
        │  └─ CoreDependencyStore     │
        └────────────┬────────────────┘
                     │
        ┌────────────▼────────────────┐
        │  Database Model Layer       │
        │  ├─ dbmodel.Span            │
        │  ├─ dbmodel.Trace           │
        │  └─ dbmodel.DependencyLink  │
        └────────────┬────────────────┘
                     │
        ┌────────────▼────────────────┐
        │  Elasticsearch JSON         │
        │  (Physical storage)         │
        └────────────────────────────┘
```

### Conversion Flow: v1 Path

```
v1/elasticsearch/spanstore/Reader
  │
  ├─ GetTrace(ctx, GetTraceParameters)
  │   │
  │   └─ Delegates to: CoreSpanReader.GetTraces()
  │       │
  │       ├─ Query elasticsearch
  │       └─ Returns: []dbmodel.Trace
  │
  └─ Convert dbmodel.Trace → model.Trace
      └─ Using: to_domain.go functions
```

### Conversion Flow: v2 Path

```
v2/elasticsearch/tracestore/Reader
  │
  ├─ GetTraces(ctx, GetTraceParams...)
  │   iter.Seq2[[]ptrace.Traces, error]
  │   │
  │   └─ Delegates to: CoreSpanReader.GetTraces()
  │       │
  │       ├─ Query elasticsearch
  │       └─ Returns: []dbmodel.Trace
  │
  └─ Convert dbmodel.Trace → ptrace.Traces
      └─ Using: from_dbmodel.go functions
      └─ Yield in iterator
```

### Key Insight: Three Conversion Layers

```
Model Conversion Chain:
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│ ptrace.Traces│  →   │ model.Batch  │  →   │dbmodel.Span  │
│   (OTLP)     │      │ (Jaeger IDL) │      │  (ES native) │
└──────────────┘      └──────────────┘      └──────────────┘
      ▲                                            ↑
      └────────────────────────────────────────────┘
           (Handled by translator.go)

Each conversion at different layer:
1. ptrace.Traces ← → model.Batch (v1adapter)
2. model.Batch ← → model.Span (spanstore.Reader)
3. model.Span ← → dbmodel.Span (v1/elasticsearch layer)
4. dbmodel.Span ← → ES JSON (es.Client layer)
```

---

## Performance Characteristics

### Read Path Performance

```
ClickHouse (Native v2):
Query → Scan rows → Convert to ptrace.Traces → Yield
└─ ~5ms overhead (conversion only)

Elasticsearch (Hybrid):
Query → dbmodel → model.Trace → ptrace.Traces → Yield
└─ ~15ms overhead (3 conversions)

Cassandra (Adapter):
Query → model.Span → model.Trace → ptrace.Traces → Yield
└─ ~20ms overhead (3 conversions + v1 layer)
```

### Write Path Performance

```
ClickHouse (Native v2):
ptrace.Traces → batch.Append() → batch.Send()
└─ Native batch optimization

Elasticsearch (Hybrid):
ptrace.Traces → model.Span → dbmodel.Span → Index
└─ Loop for each span (overhead for large batches)

Cassandra (Adapter):
ptrace.Traces → model.Span (loop) → WriteSpan()
└─ N calls per trace (no batching in v1adapter)
```

---

## File Structure Comparison

### ClickHouse (Simplest - Native v2)

```
internal/storage/v2/clickhouse/
├── factory.go               [128 lines]
├── tracestore/
│   ├── reader.go            [150 lines, iterator pattern]
│   ├── writer.go            [60 lines, batch insert]
│   ├── dbmodel/
│   │   ├── spanrow.go       [row structure]
│   │   ├── from.go          [ptrace → row]
│   │   └── to.go            [row → ptrace]
│   └── driver_test.go
├── depstore/
│   └── reader.go            [not implemented]
└── sql/
    ├── spans.sql
    ├── services.sql
    └── operations.sql

Total: ~1500 lines (production code)
Single responsibility: Convert ptrace ↔ ClickHouse rows
```

### Elasticsearch (Complex - Hybrid)

```
internal/storage/
├── elasticsearch/           [shared infrastructure]
│   ├── config/              [700 lines, ES config parser]
│   ├── client/              [400 lines, ES client wrapper]
│   ├── dbmodel/             [150 lines, ES span model]
│   ├── query/               [200 lines, query builders]
│   └── filter/              [200 lines, filter builders]
│
├── v1/elasticsearch/        [v1 implementation]
│   ├── factory.go           [250 lines]
│   ├── factory_v1.go        [100 lines]
│   └── spanstore/
│       ├── core_span_reader.go  [interface definition]
│       ├── readerv1.go          [100 lines, wraps core]
│       ├── writerv1.go          [100 lines, wraps core]
│       ├── from_domain.go       [ptrace → model]
│       └── to_domain.go         [model → ptrace]
│
└── v2/elasticsearch/        [v2 implementation]
    ├── factory.go           [100 lines]
    ├── tracestore/
    │   ├── reader.go        [100 lines]
    │   ├── writer.go        [60 lines]
    │   ├── from_dbmodel.go  [dbmodel → ptrace]
    │   └── to_dbmodel.go    [ptrace → dbmodel]
    └── depstore/
        ├── storagev2.go     [v2 wrapper]
        └── storage.go       [core dependency store]

Total: ~3000 lines
Responsibility: Shared ES client + v1/v2 wrapping
```

### Cassandra (Minimal - Adapter)

```
internal/storage/
├── cassandra/               [shared infrastructure]
│   ├── config/              [cassandra specific]
│   ├── gocql/               [driver wrapper]
│   └── session.go           [session interface]
│
├── v1/cassandra/            [v1 implementation]
│   ├── factory.go           [250 lines, FULL implementation]
│   ├── spanstore/
│   │   ├── reader.go        [complex query logic]
│   │   ├── writer.go        [write options, index]
│   │   └── dbmodel/         [CQL types, filters]
│   ├── dependencystore/
│   └── samplingstore/
│
└── v2/cassandra/            [v2 adapter wrapper]
    ├── factory.go           [110 lines, thin wrapper]
    └── (rest delegated to v1adapter)

Total: ~400 lines (v2 specific)
Responsibility: Wrap v1 with v2 interfaces
```

---

## Decision Tree for New Backend

```
┌─ Is this a NEW backend from scratch?
│  ├─ YES: Go to Native v2 Pattern
│  │   └─ Implement v2/mybackend/factory.go
│  │   └─ Implement tracestore.Reader/Writer
│  │   └─ Implement depstore.Reader (optional)
│  │
│  └─ NO: Already has v1 implementation?
│     ├─ YES: Use Adapter Pattern
│     │   └─ v2/mybackend/factory.go (40 lines)
│     │   └─ Wrap with v1adapter
│     │   └─ Done (reuse 90%)
│     │
│     └─ NO: Need both v1 and v2?
│        ├─ YES: Use Hybrid Pattern
│        │   └─ Create FactoryBase
│        │   └─ Implement v1 and v2 factories
│        │   └─ Share infrastructure (60%)
│        │
│        └─ NO: v2 only?
│           └─ Use Native v2 Pattern
│
└─ Is this Elasticsearch → OpenSearch migration?
   ├─ Drop-in replacement: Use Adapter Pattern
   │  └─ Copy v2/elasticsearch/ → v2/opensearch/
   │  └─ Change client creation only
   │  └─ 95% code reuse
   │
   └─ Different API: Use Hybrid or Native
      └─ Depends on API compatibility
```

