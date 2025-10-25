# Issue #7612 Takeaways & Analysis

## Core Problem Statement

**Deprecated Dependency:** The `olivere/elastic` Go library is deprecated and unmaintained, preventing bug fixes and creating technical debt for Jaeger's ES/OS storage layer.

**Triggering Bug:** Issue #2192 demonstrates a critical bug where AWS OpenSearch 10MB HTTP request limits are exceeded due to misunderstanding of `BulkSize` parameter semantics in olivere/elastic. The library maintainer has confirmed it's dormant and users should migrate to official clients.

## Explicit Requirements

### 1. Driver Inventory
- **Requirement:** Identify all available Go drivers for Elasticsearch and OpenSearch
- **Scope:** Both official and community-maintained libraries
- **Deliverable:** Comprehensive list with brief descriptions

### 2. Version Compatibility Matrix
- **Requirement:** Document version support for each candidate driver
- **Scope:** Cover ES versions and OS versions currently supported by Jaeger
- **Current Support (from census):**
  - olivere/elastic v7.0.32 (ES 7.x)
  - elastic/go-elasticsearch v9.1.0 (ES 8.x, 9.x)
- **Deliverable:** Compatibility matrix table

### 3. Cross-Platform Assessment
- **Requirement:** Determine if a single library can support both ES and OS
- **Context:** Unified support simplifies maintenance
- **Deliverable:** Yes/No per candidate + rationale

### 4. API Differences Analysis
- **Requirement:** Compare how alternative drivers differ from olivere/elastic
- **Focus Areas:**
  - Bulk processing semantics
  - Query DSL structure
  - Error handling patterns
  - Client initialization
  - Connection pooling
- **Deliverable:** API comparison table

### 5. Migration Impact Assessment
- **Requirement:** Quantify internal code changes needed
- **Scope:**
  - Direct API usage points (census found 145 references)
  - Abstraction layer at `internal/storage/elasticsearch/client/interfaces.go`
  - V1 vs V2 storage implementations
  - Test coverage impact
- **Deliverable:** Change estimation by component

## Implicit Acceptance Criteria

### Functional Requirements
- [ ] New driver must support bulk operations with proper size limiting
- [ ] Must correctly interpret and enforce `max_bytes` configuration
- [ ] Should work with AWS OpenSearch constraints (10MB HTTP limit)
- [ ] Must maintain existing query functionality (spans, dependencies, metrics)

### Non-Functional Requirements
- [ ] Performance parity or better vs olivere/elastic
- [ ] Actively maintained library with community/vendor support
- [ ] Go 1.24+ compatibility (current Jaeger requirement)
- [ ] Minimal disruption to existing storage abstraction

### Quality Requirements
- [ ] Migration path should preserve backward compatibility where possible
- [ ] Test suite must validate all storage operations post-migration
- [ ] Documentation for configuration parameter changes

## Risk Assessment

### High Risks
1. **Breaking API Changes**
   - Risk: Alternative drivers may have fundamentally different APIs
   - Impact: Large-scale refactoring across storage layer
   - Mitigation: Leverage existing shim layer at `client/interfaces.go`

2. **Bulk Processing Semantics**
   - Risk: Misunderstanding new driver's bulk behavior (like current bug)
   - Impact: Data loss or AWS limit violations
   - Mitigation: Extensive testing with production-scale data

3. **Version Compatibility Gaps**
   - Risk: No single driver supports all ES/OS versions Jaeger needs
   - Impact: Multiple driver dependencies or dropped version support
   - Mitigation: Early stakeholder communication on version support changes

### Medium Risks
4. **Test Coverage Gaps**
   - Risk: Existing tests may depend on olivere/elastic mocks
   - Impact: Incomplete validation of migration
   - Mitigation: Audit test dependencies before migration

5. **Performance Regression**
   - Risk: New driver may have different performance characteristics
   - Impact: Higher latency or resource usage
   - Mitigation: Benchmark suite before/after migration

### Low Risks
6. **Configuration Breaking Changes**
   - Risk: Parameter names/semantics may change
   - Impact: User configuration updates required
   - Mitigation: Provide migration guide and backward-compatible defaults

## Expected Deliverables

### 1. Driver Comparison Matrix

| Driver | ES Support | OS Support | Unified | Maintenance | License |
|--------|------------|------------|---------|-------------|---------|
| (TBD)  | (TBD)      | (TBD)      | (TBD)   | (TBD)       | (TBD)   |

### 2. Compatibility Matrix

| Driver | ES 7.x | ES 8.x | ES 9.x | OS 1.x | OS 2.x | OS 3.x |
|--------|--------|--------|--------|--------|--------|--------|
| (TBD)  | (TBD)  | (TBD)  | (TBD)  | (TBD)  | (TBD)  | (TBD)  |

### 3. Migration Recommendation

**Format:**
- **Recommended Path:** (Driver choice + rationale)
- **Alternative Options:** (With trade-offs)
- **Implementation Strategy:** (Phased approach vs big-bang)
- **Estimated Effort:** (Person-weeks by component)
- **Breaking Changes:** (List of user-facing changes)

### 4. Decision Checklist for Maintainers

**Selection Criteria:**
- [ ] Active maintenance (commits in last 6 months)
- [ ] Vendor/community backing
- [ ] Production usage evidence
- [ ] Comprehensive documentation
- [ ] Test coverage quality
- [ ] Migration tooling availability

## Key Code Hotspots (From Census)

**Critical files requiring updates (6 refs each):**
- `internal/storage/v2/elasticsearch/depstore/storage.go`
- `internal/storage/v1/elasticsearch/spanstore/reader.go`
- `internal/storage/metricstore/elasticsearch/query_builder.go`
- `internal/storage/elasticsearch/config/config.go`
- `internal/storage/elasticsearch/wrapper/wrapper.go`

**Total API surface:**
- 27 import statements
- 111 `elastic.*` API calls
- 7 `opensearch.*` references (investigation needed)

## Questions for Maintainer Clarification

1. **Version Support Policy:**
   - Q: What is the minimum ES/OS version Jaeger must continue supporting?
   - Impact: Determines viable driver options

2. **Migration Timeline:**
   - Q: Is this a Jaeger 3.0 breaking change candidate or incremental?
   - Impact: Affects strategy (compatibility layer vs clean break)

3. **Multi-Driver Support:**
   - Q: Should Jaeger support multiple drivers simultaneously (feature flag)?
   - Impact: Complexity vs flexibility trade-off

4. **OpenSearch Priority:**
   - Q: Is OpenSearch support equally important as Elasticsearch?
   - Impact: May allow ES-only driver if OS is secondary

5. **Test Infrastructure:**
   - Q: Do integration tests cover all supported ES/OS versions?
   - Impact: Confidence in migration validation

6. **AWS-Specific Requirements:**
   - Q: Are AWS ES/OS services the primary deployment target?
   - Impact: Need to prioritize AWS compatibility quirks

## Next Steps Post-Investigation

1. **Immediate:** Research and populate driver comparison matrices
2. **Short-term:** Prototype with top 2-3 candidates using shim layer
3. **Medium-term:** Community RFC for chosen approach
4. **Long-term:** Phased migration with parallel driver support (optional)

---
**Analysis Date:** 2025-10-25
**Issue Status:** Open, awaiting investigation
**Priority:** High (blocking bug fixes)
