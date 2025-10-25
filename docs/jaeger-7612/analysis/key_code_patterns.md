# Jaeger Storage: Key Code Patterns and Examples

## Pattern 1: v1adapter Bridge (Cassandra/Badger)

### The Wrapper Pattern

```go
// File: internal/storage/v2/cassandra/factory.go
type Factory struct {
    v1Factory *cassandra.Factory  // The wrapped v1 implementation
}

func NewFactory(opts cassandra.Options, ...) (*Factory, error) {
    v1f, err := newFactoryWithConfig(opts, ...)
    return &Factory{v1Factory: v1f}, nil
}

// Minimal delegation
func (f *Factory) CreateTraceReader() (tracestore.Reader, error) {
    reader, err := f.v1Factory.CreateSpanReader()
    if err != nil {
        return nil, err
    }
    return v1adapter.NewTraceReader(reader), nil  // ADAPTER BRIDGE
}

func (f *Factory) CreateTraceWriter() (tracestore.Writer, error) {
    writer, err := f.v1Factory.CreateSpanWriter()
    if err != nil {
        return nil, err
    }
    return v1adapter.NewTraceWriter(writer), nil  // ADAPTER BRIDGE
}
```

### The v1adapter Bridge

```go
// File: internal/storage/v2/v1adapter/tracereader.go
type TraceReader struct {
    spanReader spanstore.Reader  // v1 interface
}

// Converts v2 iterator call to v1 synchronous call
func (tr *TraceReader) GetTraces(
    ctx context.Context,
    traceIDs ...tracestore.GetTraceParams,
) iter.Seq2[[]ptrace.Traces, error] {
    return func(yield func([]ptrace.Traces, error) bool) {
        for _, idParams := range traceIDs {
            // Convert v2 param to v1 param
            query := spanstore.GetTraceParameters{
                TraceID:   ToV1TraceID(idParams.TraceID),
                StartTime: idParams.Start,
                EndTime:   idParams.End,
            }
            // Call v1 reader
            t, err := tr.spanReader.GetTrace(ctx, query)
            if err != nil {
                if errors.Is(err, spanstore.ErrTraceNotFound) {
                    continue  // Skip missing traces
                }
                yield(nil, err)
                return
            }
            // Convert v1 Trace to v2 ptrace.Traces
            batch := &model.Batch{Spans: t.GetSpans()}
            trace := V1BatchesToTraces([]*model.Batch{batch})
            if !yield([]ptrace.Traces{trace}, nil) {
                return
            }
        }
    }
}
```

### Key Conversion Functions

```go
// File: internal/storage/v2/v1adapter/translator.go

// ID conversion (bidirectional)
func ToV1TraceID(id pcommon.TraceID) model.TraceID {
    return model.TraceID(id[:])  // OTEL binary → Jaeger binary
}

func FromV1TraceID(id model.TraceID) pcommon.TraceID {
    var traceID pcommon.TraceID
    copy(traceID[:], id)  // Jaeger binary → OTEL binary
    return traceID
}

// Trace conversion
func V1BatchesToTraces(batches []*model.Batch) ptrace.Traces {
    traces := ptrace.NewTraces()
    for _, batch := range batches {
        rs := traces.ResourceSpans().AppendEmpty()
        // Set resource attributes from batch.Process
        // ... process attributes
        
        ss := rs.ScopeSpans().AppendEmpty()
        for _, span := range batch.Spans {
            otelSpan := ss.Spans().AppendEmpty()
            // Convert model.Span → ptrace.Span
            // ... lots of field mapping
        }
    }
    return traces
}
```

---

## Pattern 2: Hybrid Base Factory (Elasticsearch)

### The Shared FactoryBase

```go
// File: internal/storage/v1/elasticsearch/factory.go
type FactoryBase struct {
    metricsFactory metrics.Factory
    logger         *zap.Logger
    tracer         trace.TracerProvider

    newClientFn func(ctx context.Context, c *config.Configuration, ...) 
        (es.Client, error)

    config *config.Configuration
    client atomic.Pointer[es.Client]  // Shared, atomic access
    
    pwdFileWatcher *fswatcher.FSWatcher  // Password watching
    templateBuilder es.TemplateBuilder
}

func NewFactoryBase(
    ctx context.Context,
    cfg config.Configuration,
    metricsFactory metrics.Factory,
    logger *zap.Logger,
) (*FactoryBase, error) {
    f := &FactoryBase{
        config:      &cfg,
        newClientFn: config.NewClient,  // Configurable client factory
    }
    
    // Create client once, share with v1 and v2
    client, err := f.newClientFn(ctx, f.config, logger, metricsFactory)
    if err != nil {
        return nil, err
    }
    f.client.Store(&client)
    
    return f, nil
}

// Return parameters, not readers/writers
// This allows both v1 and v2 to construct their own wrappers
func (f *FactoryBase) GetSpanReaderParams() esspanstore.SpanReaderParams {
    return esspanstore.SpanReaderParams{
        Client:              f.getClient,  // Closure to get current client
        MaxDocCount:         f.config.MaxDocCount,
        MaxSpanAge:          f.config.MaxSpanAge,
        IndexPrefix:         f.config.Indices.IndexPrefix,
        SpanIndex:           f.config.Indices.Spans,
        ServiceIndex:        f.config.Indices.Services,
        TagDotReplacement:   f.config.Tags.DotReplacement,
        UseReadWriteAliases: f.config.UseReadWriteAliases,
        ReadAliasSuffix:     f.config.ReadAliasSuffix,
        RemoteReadClusters:  f.config.RemoteReadClusters,
        Logger:              f.logger,
        Tracer:              f.tracer.Tracer("esspanstore.SpanReader"),
    }
}

// Helper to safely get current client
func (f *FactoryBase) getClient() es.Client {
    if c := f.client.Load(); c != nil {
        return *c
    }
    return nil
}
```

### v1 Factory Uses Shared Base

```go
// File: internal/storage/v1/elasticsearch/factory.go
type Factory struct {
    *FactoryBase
}

func (f *Factory) CreateSpanReader() (spanstore.Reader, error) {
    params := f.GetSpanReaderParams()  // Get from base
    // Create core reader using shared params
    sr, err := cspanstore.NewSpanReader(params)
    if err != nil {
        return nil, err
    }
    // Add metrics decorator
    return spanstoremetrics.NewReaderDecorator(sr, f.metricsFactory), nil
}

func (f *Factory) CreateSpanWriter() (spanstore.Writer, error) {
    params := f.GetSpanWriterParams()  // Get from base
    return cspanstore.NewSpanWriter(params), nil
}
```

### v2 Factory Uses Shared Base

```go
// File: internal/storage/v2/elasticsearch/factory.go
type Factory struct {
    coreFactory    *elasticsearch.FactoryBase  // Reuse shared base
    config         escfg.Configuration
    metricsFactory metrics.Factory
}

func NewFactory(ctx context.Context, cfg escfg.Configuration, telset telemetry.Settings) 
    (*Factory, error) {
    // Create shared base with v1 infrastructure
    coreFactory, err := elasticsearch.NewFactoryBase(ctx, cfg, telset.Metrics, telset.Logger)
    if err != nil {
        return nil, err
    }
    
    return &Factory{
        coreFactory:    coreFactory,
        config:         cfg,
        metricsFactory: telset.Metrics,
    }, nil
}

// Reuse shared parameters, but wrap with v2 interface
func (f *Factory) CreateTraceReader() (tracestore.Reader, error) {
    params := f.coreFactory.GetSpanReaderParams()  // REUSE from base
    return tracestoremetrics.NewReaderDecorator(
        v2tracestore.NewTraceReader(params),  // Wrap with v2
        f.metricsFactory,
    ), nil
}
```

### v2 TraceReader (Thin Wrapper)

```go
// File: internal/storage/v2/elasticsearch/tracestore/reader.go
type TraceReader struct {
    spanReader spanstore.CoreSpanReader  // Use core, not full reader
}

func NewTraceReader(p spanstore.SpanReaderParams) *TraceReader {
    return &TraceReader{
        spanReader: spanstore.NewSpanReader(p),
    }
}

// Convert v2 calls to v1 core calls
func (t *TraceReader) GetTraces(ctx context.Context, params ...tracestore.GetTraceParams) 
    iter.Seq2[[]ptrace.Traces, error] {
    return func(yield func([]ptrace.Traces, error) bool) {
        // Convert param types
        dbTraceIds := make([]dbmodel.TraceID, 0, len(params))
        for _, id := range params {
            dbTraceIds = append(dbTraceIds, dbmodel.TraceID(id.TraceID.String()))
        }
        
        // Call core reader (same as v1)
        dbTraces, err := t.spanReader.GetTraces(ctx, dbTraceIds)
        if err != nil {
            yield(nil, err)
            return
        }
        
        // Convert each trace
        for _, trace := range dbTraces {
            // Convert dbmodel.Trace → ptrace.Traces
            td, err := FromDBModel(trace.Spans)
            if err != nil {
                yield(nil, err)
                return
            }
            if !yield([]ptrace.Traces{td}, nil) {
                return
            }
        }
    }
}
```

---

## Pattern 3: Native v2 Implementation (ClickHouse)

### Factory (Direct, No Base Class)

```go
// File: internal/storage/v2/clickhouse/factory.go
type Factory struct {
    config Configuration
    telset telemetry.Settings
    conn   driver.Conn  // Direct database connection
}

func NewFactory(ctx context.Context, cfg Configuration, telset telemetry.Settings) 
    (*Factory, error) {
    // Create connection directly
    conn, err := clickhouse.Open(&clickhouse.Options{
        Protocol:    getProtocol(cfg.Protocol),
        Addr:        cfg.Addresses,
        DialTimeout: cfg.DialTimeout,
        Auth: clickhouse.Auth{
            Database: cfg.Database,
        },
    })
    if err != nil {
        return nil, fmt.Errorf("failed to create ClickHouse connection: %w", err)
    }
    
    // Ping to verify connection
    err = conn.Ping(ctx)
    if err != nil {
        return nil, errors.Join(
            fmt.Errorf("failed to ping ClickHouse: %w", err),
            conn.Close(),
        )
    }
    
    // Create schema if needed
    if cfg.CreateSchema {
        // ... execute CREATE TABLE statements
    }
    
    return &Factory{
        config: cfg,
        telset: telset,
        conn:   conn,
    }, nil
}

// Minimal factory methods
func (f *Factory) CreateTraceReader() (tracestore.Reader, error) {
    return chtracestore.NewReader(f.conn), nil  // Direct, no wrapper
}

func (f *Factory) CreateTraceWriter() (tracestore.Writer, error) {
    return chtracestore.NewWriter(f.conn), nil  // Direct, no wrapper
}
```

### Reader (Iterator-Based)

```go
// File: internal/storage/v2/clickhouse/tracestore/reader.go
type Reader struct {
    conn driver.Conn
}

// Return iterator for memory efficiency
func (r *Reader) GetTraces(
    ctx context.Context,
    traceIDs ...tracestore.GetTraceParams,
) iter.Seq2[[]ptrace.Traces, error] {
    return func(yield func([]ptrace.Traces, error) bool) {
        for _, traceID := range traceIDs {
            // Query database
            rows, err := r.conn.Query(ctx, sql.SelectSpansByTraceID, traceID.TraceID)
            if err != nil {
                yield(nil, fmt.Errorf("failed to query trace: %w", err))
                return
            }
            
            // Stream results
            for rows.Next() {
                span, err := dbmodel.ScanRow(rows)
                if err != nil {
                    if !yield(nil, fmt.Errorf("failed to scan span row: %w", err)) {
                        return
                    }
                    continue
                }
                
                // Convert to ptrace and yield
                trace := dbmodel.FromRow(span)
                if !yield([]ptrace.Traces{trace}, nil) {
                    return
                }
            }
            
            if err := rows.Close(); err != nil {
                yield(nil, fmt.Errorf("failed to close rows: %w", err))
                return
            }
        }
    }
}

// Direct implementation without wrapper
func (r *Reader) GetServices(ctx context.Context) ([]string, error) {
    rows, err := r.conn.Query(ctx, sql.SelectServices)
    if err != nil {
        return nil, fmt.Errorf("failed to query services: %w", err)
    }
    defer rows.Close()
    
    var services []string
    for rows.Next() {
        var service dbmodel.Service
        if err := rows.ScanStruct(&service); err != nil {
            return nil, fmt.Errorf("failed to scan row: %w", err)
        }
        services = append(services, service.Name)
    }
    return services, nil
}
```

### Writer (Batch Optimization)

```go
// File: internal/storage/v2/clickhouse/tracestore/writer.go
type Writer struct {
    conn driver.Conn
}

func (w *Writer) WriteTraces(ctx context.Context, td ptrace.Traces) error {
    // Prepare batch for entire trace
    batch, err := w.conn.PrepareBatch(ctx, sql.InsertSpan)
    if err != nil {
        return fmt.Errorf("failed to prepare batch: %w", err)
    }
    defer batch.Close()
    
    // Iterate through all spans
    for _, rs := range td.ResourceSpans().All() {
        for _, ss := range rs.ScopeSpans().All() {
            for _, span := range ss.Spans().All() {
                // Convert to database row
                sr := dbmodel.ToRow(rs.Resource(), ss.Scope(), span)
                
                // Append to batch
                err = batch.Append(
                    sr.ID,
                    sr.TraceID,
                    sr.ParentSpanID,
                    sr.Name,
                    sr.Kind,
                    sr.StartTime,
                    sr.Duration,
                    // ... more fields
                )
                if err != nil {
                    return fmt.Errorf("failed to append span to batch: %w", err)
                }
            }
        }
    }
    
    // Send entire batch atomically
    return batch.Send()
}
```

---

## Pattern 4: Core Interface (Database Abstraction)

### CoreSpanReader (Database Level)

```go
// File: internal/storage/v1/elasticsearch/spanstore/core_span_reader.go
type CoreSpanReader interface {
    // Find by query (database-level, not domain-level)
    FindTraceIDs(ctx context.Context, traceQuery dbmodel.TraceQueryParameters) 
        ([]dbmodel.TraceID, error)
    
    // Find full traces
    FindTraces(ctx context.Context, traceQuery dbmodel.TraceQueryParameters) 
        ([]dbmodel.Trace, error)
    
    // Get operations
    GetOperations(ctx context.Context, query dbmodel.OperationQueryParameters) 
        ([]dbmodel.Operation, error)
    
    // Get services
    GetServices(ctx context.Context) ([]string, error)
    
    // Get specific traces by ID
    GetTraces(ctx context.Context, query []dbmodel.TraceID) 
        ([]dbmodel.Trace, error)
}

// Both v1 and v2 can wrap this
var (
    _ spanstore.CoreSpanReader = (*Reader)(nil)
)

type Reader struct {
    client              es.Client
    maxDocCount         int
    indexPrefix         string
    // ... parameters
}

// Implementation delegates to Elasticsearch
func (r *Reader) GetTraces(ctx context.Context, traceIDs []dbmodel.TraceID) 
    ([]dbmodel.Trace, error) {
    // Query Elasticsearch with trace IDs
    // Return dbmodel.Trace structs
}
```

---

## Pattern 5: Model Conversion Layers

### Level 1: Domain Model (v1)

```go
// File: jaeger-idl/model/span.go
type Span struct {
    TraceID       TraceID
    SpanID        SpanID
    ParentSpanID  SpanID
    OperationName string
    References    []SpanRef
    StartTime     time.Time
    Duration      time.Duration
    Tags          []KeyValue
    Logs          []Log
    ProcessID     string
    Process       *Process
}
```

### Level 2: Database Model (Backend-Specific)

```go
// File: internal/storage/elasticsearch/dbmodel/span.go
type Span struct {
    TraceID       string           `json:"traceID"`
    SpanID        string           `json:"spanID"`
    ParentSpanID  string           `json:"parentSpanID"`
    OperationName string           `json:"operationName"`
    References    []SpanRef        `json:"references"`
    StartTime     uint64           `json:"startTime"`  // microseconds
    Duration      uint64           `json:"duration"`   // microseconds
    Tags          map[string]interface{} `json:"tags"`
    Logs          []LogStructure   `json:"logs"`
    ProcessID     string           `json:"processID"`
    Process       Process          `json:"process"`
}
```

### Level 3: OTLP Model (v2)

```go
// File: go.opentelemetry.io/collector/pdata/ptrace/span.go
type Span interface {
    TraceID() pcommon.TraceID
    SpanID() pcommon.SpanID
    ParentSpanID() pcommon.SpanID
    Name() string
    Kind() SpanKind
    StartTimestamp() pcommon.Timestamp
    EndTimestamp() pcommon.Timestamp
    Attributes() pcommon.Map
    Events() SpanEventSlice
    Links() SpanLinkSlice
    Status() Status
}
```

### Conversion Between Levels

```go
// File: internal/storage/v2/elasticsearch/tracestore/to_dbmodel.go
// ptrace.Traces → dbmodel.Span
func ToDBModel(traces ptrace.Traces) []dbmodel.Span {
    var spans []dbmodel.Span
    for _, rs := range traces.ResourceSpans().All() {
        for _, ss := range rs.ScopeSpans().All() {
            for _, span := range ss.Spans().All() {
                dbSpan := &dbmodel.Span{
                    TraceID:       span.TraceID().String(),
                    SpanID:        span.SpanID().String(),
                    ParentSpanID:  span.ParentSpanID().String(),
                    OperationName: span.Name(),
                    StartTime:     uint64(span.StartTimestamp()),
                    Duration:      uint64(span.EndTimestamp() - span.StartTimestamp()),
                    // ... convert attributes, events, etc.
                }
                spans = append(spans, *dbSpan)
            }
        }
    }
    return spans
}

// File: internal/storage/v2/elasticsearch/tracestore/from_dbmodel.go
// dbmodel.Span → ptrace.Traces
func FromDBModel(dbSpans []dbmodel.Span) ptrace.Traces {
    traces := ptrace.NewTraces()
    
    // Group spans by trace ID
    traceMap := make(map[string][]dbmodel.Span)
    for _, span := range dbSpans {
        traceMap[span.TraceID] = append(traceMap[span.TraceID], span)
    }
    
    for _, groupSpans := range traceMap {
        rs := traces.ResourceSpans().AppendEmpty()
        ss := rs.ScopeSpans().AppendEmpty()
        
        for _, dbSpan := range groupSpans {
            span := ss.Spans().AppendEmpty()
            span.SetName(dbSpan.OperationName)
            span.SetTraceID(traceID)
            span.SetSpanID(spanID)
            // ... set other fields
        }
    }
    
    return traces
}
```

---

## Key Takeaways

### Three Architecture Patterns

1. **Adapter (Cassandra/Badger)**
   - Wraps existing v1
   - 40 lines of factory code
   - 95% reuse via v1adapter

2. **Hybrid (Elasticsearch)**
   - Shared FactoryBase
   - Both v1 and v2 use shared params
   - 70% reuse of infrastructure

3. **Native (ClickHouse)**
   - Direct v2 implementation
   - 1500 lines, all new
   - Optimal performance

### Abstraction Layers

```
Application
     ↓
v1/v2 Factory
     ↓
v1adapter (if adapter pattern)
     ↓
CoreReader/Writer (if hybrid)
     ↓
dbmodel (database-specific types)
     ↓
Physical Storage (ES, Cassandra, ClickHouse)
```

### Code Reuse Strategy

- **Shared:** Infrastructure (config, client, metrics)
- **Reused:** Conversion functions (ptrace ↔ domain models)
- **Delegated:** Database operations (via Core interfaces)
- **Minimal:** Factory wrappers (40-110 lines)

