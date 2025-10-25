# Issue #7612 Decision Digest

**Investigation:** Replace `olivere/elastic` Driver
**Priority:** 🔴 High - Blocking bug fixes
**Status:** Open for Research
**Created:** 2025-10-23 by @yurishkuro

---

## Executive Summary

Jaeger's Elasticsearch/OpenSearch storage backend depends on the **deprecated and unmaintained** `olivere/elastic` library. This creates an immediate problem: critical bugs cannot be fixed upstream.

**Triggering Incident:** AWS OpenSearch users are experiencing bulk request failures due to a [10MB HTTP limit](https://github.com/jaegertracing/jaeger/issues/2192#issuecomment-3435539949). The `max_bytes` configuration parameter is misinterpreted by `olivere/elastic`, causing infinite retry loops. A PR fixing this was rejected because the maintainer declared the repository "dormant."

**Business Impact:**
- ❌ Production incidents cannot be resolved
- ❌ Security vulnerabilities cannot be patched
- ❌ New ES/OS versions cannot be supported
- ⚠️ Technical debt accumulating

**Required Action:** Investigate alternative Go drivers and create migration roadmap.

---

## Investigation Objectives

### 5 Core Questions

| # | Question | Deliverable |
|---|----------|-------------|
| 1 | Which Go drivers are available? | Driver inventory table |
| 2 | What are version compatibility profiles? | ES/OS compatibility matrix |
| 3 | Can one library support both ES & OS? | Unified vs split analysis |
| 4 | How do APIs differ from olivere/elastic? | API comparison guide |
| 5 | How much internal change is needed? | Migration impact assessment |

### Additional Context

**Existing Abstraction:** Jaeger has a partial shim layer at `internal/storage/elasticsearch/client/interfaces.go` that isolates *some* driver dependencies. However, from the [repository census](REPO_CENSUS.md):
- **145 ES/OS API references** across codebase
- **27 direct imports** of olivere/elastic
- **Critical files:** spanstore, depstore, metricstore, config

**Current Dependencies (go.mod):**
```
github.com/olivere/elastic/v7 v7.0.32          // Direct, deprecated
github.com/elastic/go-elasticsearch/v9 v9.1.0  // Already in use (somewhere)
```

---

## Requirements Checklist

### Must-Have Investigation Outputs

- [ ] **Driver Comparison Table**
  - Library name, repository, license
  - Active maintenance evidence (commit activity, release cadence)
  - Community/vendor backing (GitHub stars, production users)

- [ ] **Version Compatibility Matrix**
  - ES 7.x, 8.x, 9.x support per driver
  - OpenSearch 1.x, 2.x, 3.x support per driver
  - Note version gaps or limitations

- [ ] **API Differences Analysis**
  - Bulk processing semantics (critical for bug fix)
  - Query DSL construction
  - Client initialization patterns
  - Error handling differences

- [ ] **Migration Impact Estimate**
  - Files requiring changes (by component)
  - Test coverage impact (unit + integration)
  - Configuration breaking changes
  - Estimated person-weeks effort

- [ ] **Recommended Migration Path**
  - Primary driver choice + rationale
  - Alternative options + trade-offs
  - Phased vs big-bang strategy
  - Risk mitigation measures

### Must-Address Technical Questions

- [ ] Does the replacement driver **correctly enforce bulk size limits**?
  - This is the bug we're trying to fix
  - Test against AWS 10MB limit scenarios

- [ ] How does the new driver handle bulk retries on failure?
  - Infinite loops are unacceptable (current problem)
  - Need circuit breaker or max retry logic

- [ ] Can we maintain backward compatibility with existing configs?
  - E.g., `opensearch.bulk_processing.max_bytes`
  - If not, provide migration guide

- [ ] Does the driver support all query types Jaeger uses?
  - Span queries (time range, service, operation)
  - Dependency graph queries
  - Metrics aggregations

- [ ] What is the performance impact?
  - Benchmark bulk write throughput
  - Benchmark query latency (p50, p99)

---

## Risk Assessment

### 🔴 Critical Risks

1. **No Single Driver Supports All Versions**
   - **Risk:** May need multiple drivers or drop version support
   - **Impact:** Community fragmentation, maintenance burden
   - **Mitigation:** Survey users on ES/OS version distribution

2. **Breaking API Changes**
   - **Risk:** Large-scale refactoring required
   - **Impact:** Multiple release cycles, high regression risk
   - **Mitigation:** Leverage existing shim layer, phased rollout

### 🟡 Moderate Risks

3. **Test Infrastructure Gaps**
   - **Risk:** Mocks/fixtures tied to olivere/elastic
   - **Impact:** False confidence in migration
   - **Mitigation:** Audit test dependencies early

4. **Performance Regression**
   - **Risk:** New driver slower than olivere/elastic
   - **Impact:** Increased latency, resource usage
   - **Mitigation:** Benchmark suite before/after

### 🟢 Low Risks

5. **Configuration Migration**
   - **Risk:** Parameter semantics change
   - **Impact:** User disruption (manageable with docs)
   - **Mitigation:** Provide migration guide, backward-compat layer

---

## Questions for Maintainer Confirmation

The following questions **must be answered** before finalizing a migration plan:

### Version Support Policy
- [ ] **Q1:** What is the minimum ES version Jaeger must support going forward?
  - Options: ES 7.x (EOL?), ES 8.x, ES 9.x only
  - Rationale: Determines viable driver options

- [ ] **Q2:** What is the minimum OpenSearch version required?
  - Options: OS 1.x, OS 2.x, OS 3.x only
  - Rationale: Affects whether unified driver is possible

- [ ] **Q3:** Is OpenSearch support equally critical as Elasticsearch?
  - Context: If ES is primary, we have more driver options
  - Impact: May allow ES-only driver with OS as best-effort

### Migration Strategy
- [ ] **Q4:** Should this be a breaking change in Jaeger 3.0 or incremental?
  - Option A: Wait for 3.0, clean break from olivere/elastic
  - Option B: Incremental via feature flag, backward compat
  - Trade-off: Speed vs compatibility

- [ ] **Q5:** Is parallel driver support acceptable (feature flag)?
  - Example: `--storage.driver=olivere|elastic-official`
  - Pros: Gradual migration, rollback capability
  - Cons: Maintenance burden, code complexity

### Deployment Context
- [ ] **Q6:** Are AWS ES/OS services the primary deployment target?
  - Context: AWS has specific quirks (10MB limit, IAM auth)
  - Impact: Need to prioritize AWS compatibility testing

- [ ] **Q7:** Do users deploy against ES/OS clusters with mixed versions?
  - Example: Multi-cluster setups with ES 8 + OS 2
  - Impact: May need multi-driver support

### Testing & Validation
- [ ] **Q8:** Do current integration tests cover all supported ES/OS versions?
  - If not: Gap in migration validation
  - Action: Expand test matrix before migration

- [ ] **Q9:** Is there a canary deployment process for storage changes?
  - Ideal: Gradual rollout with monitoring
  - Fallback: Rollback procedure documented

---

## Suggested Research Approach

### Phase 1: Discovery (1 week)
1. **Driver Inventory**
   - Survey Go ecosystem: official clients, forks, alternatives
   - Check GitHub activity, issue responsiveness, release notes

2. **Initial Compatibility Check**
   - Read driver docs for version support claims
   - Cross-reference with ES/OS compatibility matrices

3. **Stakeholder Survey**
   - Ask community: What ES/OS versions are you using?
   - Prioritize drivers that cover majority use cases

### Phase 2: Deep Dive (2 weeks)
4. **API Comparison**
   - Prototype common operations (bulk write, query) with top 3 drivers
   - Document API differences vs olivere/elastic

5. **Bug Reproduction**
   - Replicate issue #2192 bug with candidate drivers
   - Verify bulk size limiting works correctly

6. **Performance Benchmarking**
   - Use existing Jaeger benchmark suite
   - Measure throughput & latency per driver

### Phase 3: Decision (1 week)
7. **Compile Report**
   - Populate all matrices and tables
   - Draft migration recommendation with rationale

8. **Community RFC**
   - Present findings to maintainers
   - Gather feedback on preferred approach

9. **Finalize Roadmap**
   - Timeline for migration
   - Breaking change policy
   - Rollout strategy

**Total Estimated Effort:** 4 weeks (1 person full-time or distributed)

---

## Success Criteria

This investigation is **complete** when:

- [ ] All 5 core questions are answered with data
- [ ] All 9 maintainer questions have responses
- [ ] A primary driver is recommended with justification
- [ ] Migration effort is estimated (person-weeks, risk level)
- [ ] Roadmap draft is reviewed by at least 2 maintainers

**Output Format:** RFC document for jaegertracing/jaeger repo

---

## Reference Materials

### Related Documents
- [Issue #7612 Raw Transcript](../inputs/issue7612_raw.md)
- [Detailed Takeaways Analysis](issue7612_takeaways.md)
- [Repository Census Report](REPO_CENSUS.md)
- [Link Graph](../tables/issue7612_links.csv)

### Key Code Locations
- Abstraction layer: `internal/storage/elasticsearch/client/interfaces.go`
- Config handling: `internal/storage/elasticsearch/config/config.go`
- V1 storage: `internal/storage/v1/elasticsearch/`
- V2 storage: `internal/storage/v2/elasticsearch/`

### External Links
- [AWS OpenSearch HTTP Limits](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/limits.html#network-limits)
- [olivere/elastic (deprecated)](https://github.com/olivere/elastic)
- [elastic/go-elasticsearch (official)](https://github.com/elastic/go-elasticsearch)

---

**Digest Prepared:** 2025-10-25
**Next Review:** Upon community input on issue #7612
**Owner:** Research volunteer (good first issue)
