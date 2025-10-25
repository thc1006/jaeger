# Issue #7612: Investigate the path to replace olivere/elastic driver

**Author:** Yuri Shkuro (@yurishkuro)  
**Created:** 2025-10-23T14:01:58Z  
**Status:** Open  
**Labels:** enhancement, help wanted, good first issue, area/storage, storage/elasticsearch

## Issue Body

The `olivere/elastic` library is deprecated, but our Elasticsearch/OpenSearch storage implementation is heavily dependent on it. Bugs like this https://github.com/jaegertracing/jaeger/issues/2192#issuecomment-3435539949 cannot be resolved.

Objective: Investigate available options for Go drivers for ES/OS and compile a report so that we can build a roadmap around it.
  * Which libraries are available for use
  * What are their version compatibility profiles for the ES/OS versions we currently support
  * Is there one library that can continue to work across both ES and OS
  * What are the differences in how that library works compared to `olivere/elastic`
  * How much changes would we need to make internally

Worth noting that our use of `olivere/elastic` is indirect, we have a shim layer `internal/storage/elasticsearch/client/interfaces.go` that attempts (only partially) to abstract away the underlying driver.

## Comments

(No comments as of fetch date)

## Referenced Issues/PRs

### Issue #2192: Related Bug Context

**URL:** https://github.com/jaegertracing/jaeger/issues/2192#issuecomment-3435539949

**Key comment from @jstasiak (2025-10-23):**

We started experiencing this problem with Jaeger 2.8 and AWS OpenSearch yesterday. The OpenSearch machine has a 10 MB HTTP request size limit if I understand correctly (following a link from above, https://docs.aws.amazon.com/opensearch-service/latest/developerguide/limits.html#network-limits) and we have this in our Jaeger configuration:

```yaml
opensearch:
  bulk_processing:
    max_bytes: 5000000
```

and `max_bytes` is used like so (https://github.com/jaegertracing/jaeger/blob/2c8ca4b7f39706825f3faae64686ae6ba845bc40/internal/storage/elasticsearch/config/config.go#L291):

```go
bulkProc, err := rawClient.BulkProcessor().
    Before(func(id int64, _ /* requests */ []elastic.BulkableRequest) {
        bcb.startTimes.Store(id, time.Now())
    }).
    After(bcb.invoke).
    BulkSize(c.BulkProcessing.MaxBytes).
    Workers(c.BulkProcessing.Workers).
    BulkActions(c.BulkProcessing.MaxActions).
    FlushInterval(c.BulkProcessing.FlushInterval).
    Do(ctx)
```

but I don't think it actually limits the size of the request in any way but rather detects when the data should be committed (https://github.com/olivere/elastic/blob/4cdb89f6e627228e7cb3b53e1b1ef8630cc71a0a/bulk_processor.go#L50)?

```go
type BulkProcessorService struct {
...
    bulkSize             int           // # of bytes after which to commit
```

So enough data may accumulate quickly to reach the AWS limit and then it keeps failing forever until a restart.

There is an `olivere/elastic` PR that works around that, https://github.com/olivere/elastic/pull/1661, but in the comments there:

> > Hey @olivere , as this repository is deprecated, is it not accepting patches anymore?
>
> Yes. This repository is deprecated and basically dormant. You are strongly encouraged to switch to the official client.

**Impact:** This bug demonstrates that the deprecated library cannot be patched, forcing migration.

