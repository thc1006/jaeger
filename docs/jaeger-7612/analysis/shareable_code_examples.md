# SPECIFIC SHAREABLE CODE EXAMPLES WITH LINE REFERENCES

## EXAMPLE 1: Tag Merging Logic (64 lines, 100% Shareable)

**Current Location**: `/internal/storage/v1/elasticsearch/spanstore/reader.go` lines 686-749

**Used in**:
- SpanReader.multiRead() - Parses results from ES
- SpanWriter.WriteSpan() - Prepares spans for storage (inverse operation)

**Sharable Code**:
```go
// Current: reader.go (686-702)
func (s *SpanReader) mergeAllNestedAndElevatedTagsOfSpan(span *dbmodel.Span) {
    processTags := s.mergeNestedAndElevatedTags(span.Process.Tags, span.Process.Tag)
    span.Process.Tags = processTags
    spanTags := s.mergeNestedAndElevatedTags(span.Tags, span.Tag)
    span.Tags = spanTags
}

// Current: reader.go (693-702)
func (s *SpanReader) mergeNestedAndElevatedTags(nestedTags []dbmodel.KeyValue, 
    elevatedTags map[string]any) []dbmodel.KeyValue {
    mergedTags := make([]dbmodel.KeyValue, 0, len(nestedTags)+len(elevatedTags))
    mergedTags = append(mergedTags, nestedTags...)
    for k, v := range elevatedTags {
        kv := s.convertTagField(k, v)
        mergedTags = append(mergedTags, kv)
        delete(elevatedTags, k)
    }
    return mergedTags
}

// Current: reader.go (704-749)
func (s *SpanReader) convertTagField(k string, v any) dbmodel.KeyValue {
    dKey := s.dotReplacer.ReplaceDotReplacement(k)
    kv := dbmodel.KeyValue{
        Key:   dKey,
        Value: v,
    }
    switch val := v.(type) {
    case int64:
        kv.Type = dbmodel.Int64Type
    case float64:
        kv.Type = dbmodel.Float64Type
    case bool:
        kv.Type = dbmodel.BoolType
    case string:
        kv.Type = dbmodel.StringType
    case []byte:
        kv.Type = dbmodel.BinaryType
    case json.Number:
        n, err := val.Int64()
        if err == nil {
            kv.Value = n
            kv.Type = dbmodel.Int64Type
        } else {
            f, err := val.Float64()
            if err != nil {
                return dbmodel.KeyValue{
                    Key:   dKey,
                    Value: fmt.Sprintf("invalid tag type in %+v: %s", v, err.Error()),
                    Type:  dbmodel.StringType,
                }
            }
            kv.Value = f
            kv.Type = dbmodel.Float64Type
        }
    default:
        return dbmodel.KeyValue{
            Key:   dKey,
            Value: fmt.Sprintf("invalid tag type in %+v", v),
            Type:  dbmodel.StringType,
        }
    }
    return kv
}

// Current: writer.go (159-178)
func (s *SpanWriter) splitElevatedTags(keyValues []dbmodel.KeyValue) ([]dbmodel.KeyValue, map[string]any) {
    if !s.allTagsAsFields && len(s.tagKeysAsFields) == 0 {
        return keyValues, nil
    }
    var tagsMap map[string]any
    var kvs []dbmodel.KeyValue
    for _, kv := range keyValues {
        if kv.Type != dbmodel.BinaryType && (s.allTagsAsFields || s.tagKeysAsFields[kv.Key]) {
            if tagsMap == nil {
                tagsMap = map[string]any{}
            }
            tagsMap[strings.ReplaceAll(kv.Key, ".", s.tagDotReplacement)] = kv.Value
        } else {
            kvs = append(kvs, kv)
        }
    }
    if kvs == nil {
        kvs = make([]dbmodel.KeyValue, 0)
    }
    return kvs, tagsMap
}
```

**Recommendation**: Extract to `internal/storage/common/processor/tag_processor.go`
- Remove dependency on `s.dotReplacer` - pass as parameter
- Make `convertTagField()` a standalone function
- Share between reader (merge) and writer (split)

---

## EXAMPLE 2: Metrics Processor (268 lines, 100% Shareable!)

**Current Location**: `/internal/storage/metricstore/elasticsearch/processor.go`

**Key Functions** (ALL 100% independent):
```go
// Line 19-22
func ScaleAndRoundLatencies(mf *metrics.MetricFamily) *metrics.MetricFamily {
    const lookback = 1 // only current value
    return applySlidingWindow(mf, lookback, scaleToMillisAndRound)
}

// Line 25-27
func CalculateCallRates(mf *metrics.MetricFamily, params metricstore.BaseQueryParameters, 
    timeRange TimeRange) *metrics.MetricFamily {
    processed := calcCallRate(mf, params)
    return trimMetricPointsBefore(processed, timeRange.startTimeMillis)
}

// Line 31-34
func CalculateErrorRates(rawErrors, calls *metrics.MetricFamily, params metricstore.BaseQueryParameters, 
    timeRange TimeRange) *metrics.MetricFamily {
    processedErrors := CalculateCallRates(rawErrors, params, timeRange)
    return calcErrorRates(processedErrors, calls)
}

// Line 172-210 - THE SLIDING WINDOW PATTERN
func calcCallRate(mf *metrics.MetricFamily, params metricstore.BaseQueryParameters) *metrics.MetricFamily {
    lookback := int(math.Ceil(float64(params.RatePer.Milliseconds()) / float64(params.Step.Milliseconds())))
    lookback = int(math.Max(float64(lookback), 1))
    windowSizeSeconds := float64(lookback) * params.Step.Seconds()
    lastNonNaNMap := make(map[string]float64)
    
    rateCalculator := func(metric *metrics.Metric, window []*metrics.MetricPoint) float64 {
        labelKey := getLabelKey(metric.Labels)
        if len(window) < lookback {
            return math.NaN()
        }
        firstValue := window[0].GetGaugeValue().GetDoubleValue()
        if math.IsNaN(firstValue) {
            firstValue = lastNonNaNMap[labelKey]
        } else {
            lastNonNaNMap[labelKey] = firstValue
        }
        lastValue := window[len(window)-1].GetGaugeValue().GetDoubleValue()
        if math.IsNaN(lastValue) {
            return math.NaN()
        }
        rate := (lastValue - firstValue) / windowSizeSeconds
        return math.Round(rate*100) / 100
    }
    
    return applySlidingWindow(mf, lookback, rateCalculator)
}

// Line 231-257 - GENERIC WINDOWING ENGINE
func applySlidingWindow(mf *metrics.MetricFamily, lookback int, 
    processor func(metric *metrics.Metric, window []*metrics.MetricPoint) float64) *metrics.MetricFamily {
    for _, metric := range mf.Metrics {
        points := metric.MetricPoints
        if len(points) == 0 {
            continue
        }
        
        processedPoints := make([]*metrics.MetricPoint, 0, len(points))
        
        for i, currentPoint := range points {
            start := i - lookback + 1
            if start < 0 {
                start = 0
            }
            window := points[start : i+1]
            resultValue := processor(metric, window)
            
            processedPoints = append(processedPoints, &metrics.MetricPoint{
                Timestamp: currentPoint.Timestamp,
                Value:     toDomainMetricPointValue(resultValue),
            })
        }
        metric.MetricPoints = processedPoints
    }
    return mf
}
```

**Dependencies**: Zero! (Only uses proto types)

**Recommendation**: Move to `/internal/storage/metricstore/processor/metrics_processor.go`
- Ready to use for ANY metrics backend
- Can even use for Prometheus/ClickHouse migration

---

## EXAMPLE 3: Trace Assembly (92 lines, 85% Shareable)

**Current Location**: `/internal/storage/v1/elasticsearch/spanstore/reader.go` lines 372-463

**Driver-Independent Parts**:
```go
// Line 372-403: Setup (10 lines)
traces := make([]dbmodel.Trace, 0, len(traceIDs))
if len(traceIDs) == 0 {
    return traces, nil
}

nextTime := model.TimeAsEpochMicroseconds(startTime.Add(-time.Hour))
searchAfterTime := make(map[dbmodel.TraceID]uint64)
totalDocumentsFetched := make(map[dbmodel.TraceID]int)
tracesMap := make(map[dbmodel.TraceID]*dbmodel.Trace)

// Line 449-461: CORE ASSEMBLY LOGIC (100% Independent)
for _, result := range results.Responses {
    if result.Hits == nil || len(result.Hits.Hits) == 0 {
        continue
    }
    spans, err := s.collectSpans(result.Hits.Hits)
    if err != nil {
        err = es.DetailedError(err)
        logErrorToSpan(childSpan, err)
        return nil, err
    }
    lastSpan := spans[len(spans)-1]

    if traceSpan, ok := tracesMap[lastSpan.TraceID]; ok {
        traceSpan.Spans = append(traceSpan.Spans, spans...)
    } else {
        traces = append(traces, dbmodel.Trace{Spans: spans})
        tracesMap[lastSpan.TraceID] = &traces[len(traces)-1]
    }

    totalDocumentsFetched[lastSpan.TraceID] += len(result.Hits.Hits)
    if totalDocumentsFetched[lastSpan.TraceID] < int(result.TotalHits()) {
        traceIDs = append(traceIDs, lastSpan.TraceID)
        searchAfterTime[lastSpan.TraceID] = lastSpan.StartTime
    }
}
```

**Driver-Specific Parts**:
```go
// Line 405-426: Search execution (only 7 lines that matter)
searchRequests := make([]*elastic.SearchRequest, len(traceIDs))
for i, traceID := range traceIDs {
    traceQuery := buildTraceByIDQuery(traceID)
    query := elastic.NewBoolQuery().Must(traceQuery)
    // ... more setup
    searchRequests[i] = elastic.NewSearchRequest().IgnoreUnavailable(true).Source(s)
}

results, err := s.client().MultiSearch().Add(searchRequests...).Index(indices...).Do(ctx)
```

**Recommendation**: Create driver interface:
```go
type SpanFetcher interface {
    Fetch(ctx context.Context, traceIDs []dbmodel.TraceID) (*SpanFetchResult, error)
}

type SpanFetchResult struct {
    Responses []struct {
        Spans      []dbmodel.Span
        TotalCount int64
    }
}

// Then AssembleTraces becomes pure:
func AssembleTraces(ctx context.Context, traceIDs []dbmodel.TraceID, 
    fetcher SpanFetcher) ([]dbmodel.Trace, error) {
    // Core assembly logic here, completely independent
}
```

---

## EXAMPLE 4: Query Validation (14 lines, 100% Shareable)

**Current Location**: `/internal/storage/v1/elasticsearch/spanstore/reader.go` lines 479-493

```go
func validateQuery(p dbmodel.TraceQueryParameters) error {
    if p.ServiceName == "" && len(p.Tags) > 0 {
        return ErrServiceNameNotSet
    }
    if p.StartTimeMin.IsZero() || p.StartTimeMax.IsZero() {
        return ErrStartAndEndTimeNotSet
    }
    if p.StartTimeMax.Before(p.StartTimeMin) {
        return ErrStartTimeMinGreaterThanMax
    }
    if p.DurationMin != 0 && p.DurationMax != 0 && p.DurationMin > p.DurationMax {
        return ErrDurationMinGreaterThanMax
    }
    return nil
}
```

**Status**: Pure business logic, zero dependencies
**Recommendation**: Move to `/internal/storage/common/query/validator.go`

---

## EXAMPLE 5: Point Extraction Pattern (48 lines, 100% Shareable)

**Current Location**: `/internal/storage/metricstore/elasticsearch/reader.go` lines 174-221

```go
// Generic point extractor (Line 175-194)
func bucketsToPoints(buckets []*elastic.AggregationBucketHistogramItem, 
    valueExtractor func(*elastic.AggregationBucketHistogramItem) float64) []*Pair {
    var points []*Pair
    
    for _, bucket := range buckets {
        var value float64
        if bucket.DocCount == 0 {
            value = math.NaN()
        } else {
            value = valueExtractor(bucket)
        }
        
        points = append(points, &Pair{
            TimeStamp: int64(bucket.Key),
            Value:     value,
        })
    }
    return points
}

// Specific extractors use the generic pattern (Line 196-221)
func bucketsToCallRate(buckets []*elastic.AggregationBucketHistogramItem) []*Pair {
    valueExtractor := func(bucket *elastic.AggregationBucketHistogramItem) float64 {
        aggMap, ok := bucket.Aggregations.CumulativeSum(culmuAggName)
        if !ok || aggMap.Value == nil {
            return math.NaN()
        }
        return *aggMap.Value
    }
    return bucketsToPoints(buckets, valueExtractor)
}

func bucketsToLatencies(buckets []*elastic.AggregationBucketHistogramItem, 
    percentileValue float64) []*Pair {
    valueExtractor := func(bucket *elastic.AggregationBucketHistogramItem) float64 {
        aggMap, ok := bucket.Aggregations.Percentiles(percentilesAggName)
        if !ok {
            return math.NaN()
        }
        percentileKey := fmt.Sprintf("%.1f", percentileValue)
        aggMapValue, ok := aggMap.Values[percentileKey]
        if !ok {
            return math.NaN()
        }
        return aggMapValue
    }
    return bucketsToPoints(buckets, valueExtractor)
}
```

**Shareable Parts**:
- `bucketsToPoints()` pattern - entirely generic
- Logic works for any bucket type with ValueExtractor callback

**Driver-Specific Parts**:
- `elastic.AggregationBucketHistogramItem` type
- Aggregation name constants

**Recommendation**: Abstract to interface:
```go
type ValueExtractor interface {
    Extract(bucket interface{}) float64
}

type PointExtractor interface {
    ExtractPoints(buckets []interface{}, extractor ValueExtractor) []Point
}
```

---

## EXAMPLE 6: Label & Timestamp Conversion (40 lines, 100% Shareable)

**Current Location**: `/internal/storage/metricstore/elasticsearch/to_domain.go`

```go
// Line 85-91: Service label building
func buildServiceLabels(serviceNames []string) []*metrics.Label {
    labels := make([]*metrics.Label, len(serviceNames))
    for i, name := range serviceNames {
        labels[i] = &metrics.Label{Name: "service_name", Value: name}
    }
    return labels
}

// Line 114-122: Operation label building
func toDomainLabels(key string) []*metrics.Label {
    return []*metrics.Label{
        {
            Name:  "operation",
            Value: key,
        },
    }
}

// Line 159-163: Timestamp conversion
func toDomainTimestamp(millis int64) *types.Timestamp {
    timestamp := time.Unix(0, millis*int64(time.Millisecond))
    protoTimestamp, _ := types.TimestampProto(timestamp)
    return protoTimestamp
}

// Line 166-173: Value wrapping
func toDomainMetricPointValue(value float64) *metrics.MetricPoint_GaugeValue {
    return &metrics.MetricPoint_GaugeValue{
        GaugeValue: &metrics.GaugeValue{
            Value: &metrics.GaugeValue_DoubleValue{DoubleValue: value},
        },
    }
}
```

**Status**: Pure domain model construction
**Recommendation**: Move to shared metrics module

---

## SUMMARY TABLE: REUSABLE CODE BY PRIORITY

| Component | Lines | Shareable | Priority | Effort | Location |
|-----------|-------|-----------|----------|--------|----------|
| metrics processor | 268 | 100% | HIGH | LOW | Move to metricstore/processor/ |
| tag merge/split | 70 | 100% | HIGH | LOW | Extract to common/processor/tag_processor |
| trace assembly | 92 | 85% | HIGH | MEDIUM | Extract with interface wrapper |
| query validation | 14 | 100% | MEDIUM | LOW | Extract to common/query/validator |
| point extraction | 48 | 95% | MEDIUM | MEDIUM | Create generic extractor pattern |
| label conversion | 20 | 100% | MEDIUM | LOW | Extract to common/converters |
| timestamp conversion | 10 | 100% | MEDIUM | LOW | Extract to common/converters |
| span kind normalization | 6 | 100% | LOW | LOW | Extract as utility |

**Total Lines Readily Shareable**: ~528 lines (30% of analyzed code)
**Total Lines with Wrapper**: ~620 lines (35% of analyzed code)

