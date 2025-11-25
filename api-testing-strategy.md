# API Testing Strategy & Performance Governance

This document extends the API governance framework to include comprehensive testing capabilities. Testing is not separate from governance—it's how we verify that governance actually works. The platform's existing components (Registry, Gateway, Auditor) provide natural integration points for automated testing throughout the API lifecycle.

---

## Table of Contents

1. [Testing Philosophy](#testing-philosophy)
2. [Testing in the API Lifecycle](#testing-in-the-api-lifecycle)
3. [Contract Testing](#contract-testing)
4. [Performance Testing](#performance-testing)
5. [Automated Performance Governance](#automated-performance-governance)
6. [Load Testing Integration](#load-testing-integration)
7. [Chaos Engineering for APIs](#chaos-engineering-for-apis)
8. [Testing Infrastructure](#testing-infrastructure)
9. [Metrics & Quality Gates](#metrics--quality-gates)

---

## Testing Philosophy

### Testing as Governance

Traditional API testing happens in isolation—teams run tests in their own environments, results live in CI logs nobody reads, and production issues still surprise everyone. **Testing integrated with governance changes this.**

When testing data flows through the same platform that manages API lifecycle:
- **Test results block publication** — APIs cannot reach "Published" state without passing quality gates
- **Performance baselines are enforced** — New versions must meet or exceed previous version's latency
- **Consumer impact is visible** — Before deploying, see how changes affect actual consumers
- **Historical trends inform decisions** — Auditor tracks performance over time, not just point-in-time snapshots

### Shift Left, But Also Shift Right

**Shift Left:** Catch problems early through contract testing, schema validation, and mock-based integration tests in development.

**Shift Right:** Production is the ultimate test environment. The Auditor provides continuous performance monitoring that validates what synthetic tests can only estimate.

The platform supports both: rigorous pre-production gates *and* continuous production validation.

---

## Testing in the API Lifecycle

Testing integrates at every lifecycle stage:

| Stage | Testing Activity | Platform Integration |
|-------|------------------|---------------------|
| **Design** | Schema validation, mock generation | Registry validates OpenAPI specs; generates mocks automatically |
| **Review** | Contract compatibility check | Automated breaking change detection before approval |
| **Build** | Unit tests, contract tests | CI/CD integration via Registry webhooks |
| **Pre-Production** | Performance baseline, load testing | Gateway routes to test environments; Auditor captures baselines |
| **Publication** | Quality gate validation | Registry blocks publication if tests fail |
| **Production** | Continuous performance monitoring | Auditor tracks SLO compliance in real-time |
| **Deprecation** | Consumer migration verification | Auditor confirms consumers moved to new version |

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Design  │───▶│ Review  │───▶│  Build  │───▶│ Staging │───▶│  Prod   │
└────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘
     │              │              │              │              │
     ▼              ▼              ▼              ▼              ▼
  Schema         Breaking       Contract      Performance   Continuous
  Validation     Change         Tests         Baseline      Monitoring
  Mock Gen       Detection      Unit Tests    Load Tests    SLO Tracking
```

---

## Contract Testing

### The Problem with Integration Testing

Traditional integration tests are:
- **Slow** — Require all services running together
- **Flaky** — Fail due to environment issues, not code problems
- **Incomplete** — Can't cover all consumer scenarios
- **Late** — Run after code is written, when changes are expensive

### Consumer-Driven Contract Testing

Contract testing inverts the model: **consumers define their expectations, producers verify they meet them.**

#### How It Works with the Platform

1. **Consumers publish contracts to Registry**
   ```json
   {
     "consumer": "checkout-service",
     "provider": "inventory-api",
     "interactions": [
       {
         "description": "get item stock level",
         "request": {
           "method": "GET",
           "path": "/items/12345/stock"
         },
         "response": {
           "status": 200,
           "body": {
             "item_id": "12345",
             "quantity": 42,
             "warehouse": "string"
           }
         }
       }
     ]
   }
   ```

2. **Registry stores contracts alongside subscriptions**
   - Each subscription can have associated contract expectations
   - Contracts versioned with API versions
   - Breaking contract = breaking consumer

3. **Producer CI verifies contracts**
   - On every build, producer fetches all consumer contracts from Registry
   - Tests run against actual implementation
   - Failures block deployment

4. **Registry tracks contract coverage**
   - Which consumers have contracts?
   - Which API endpoints are covered?
   - Which interactions are untested?

#### Contract Testing Workflow

```
Consumer Team                    Registry                      Producer Team
     │                              │                               │
     │  1. Define contract          │                               │
     │─────────────────────────────▶│                               │
     │                              │                               │
     │                              │  2. Notify producer           │
     │                              │──────────────────────────────▶│
     │                              │                               │
     │                              │  3. Fetch contracts           │
     │                              │◀──────────────────────────────│
     │                              │                               │
     │                              │  4. Run contract tests        │
     │                              │                               │
     │  5. Contract verified        │  5. Report results            │
     │◀─────────────────────────────│◀──────────────────────────────│
```

#### Breaking Change Detection

The Registry automatically detects breaking changes by comparing new API versions against existing contracts:

**Breaking Changes Detected:**
- Required field removed from response
- Field type changed (string → integer)
- Endpoint path changed
- Required request parameter added
- Response status code changed for same scenario

**Non-Breaking Changes (Safe):**
- New optional field in response
- New optional endpoint added
- New optional request parameter
- Expanded enum values (additive)

```
Registry Analysis on PR:

  ⚠️  Breaking Changes Detected
  
  Affected Consumers: 3
  
  ┌─────────────────────────────────────────────────────────────┐
  │ Change: Field 'customer_name' removed from GET /orders/{id} │
  │                                                             │
  │ Consumers expecting this field:                             │
  │   • checkout-service (12,000 calls/day)                     │
  │   • reporting-dashboard (500 calls/day)                     │
  │   • mobile-app-backend (8,000 calls/day)                    │
  │                                                             │
  │ Action Required: Bump to major version (v2 → v3)            │
  └─────────────────────────────────────────────────────────────┘
```

---

## Performance Testing

### Why Performance Testing Belongs in Governance

Performance is a feature. An API that returns correct data in 5 seconds isn't meeting its contract if consumers expect 200ms. The governance platform makes performance:

- **Visible** — Every API has documented latency expectations
- **Measured** — Auditor tracks actual performance continuously
- **Enforced** — Quality gates prevent slow APIs from reaching production
- **Comparable** — New versions benchmarked against previous versions

### Performance Dimensions

| Dimension | Definition | Measurement |
|-----------|------------|-------------|
| **Latency** | Time from request to response | p50, p90, p95, p99 percentiles |
| **Throughput** | Requests handled per second | RPS at various concurrency levels |
| **Error Rate** | Percentage of failed requests | 4xx, 5xx rates under load |
| **Saturation** | Resource utilization | CPU, memory, connection pool usage |
| **Scalability** | Performance change with load | Latency curve as RPS increases |

### Performance SLOs in API Specifications

API specifications include performance expectations:

```yaml
openapi: 3.0.0
info:
  title: Order API
  version: 2.1.0
  x-performance-slo:
    latency:
      p50: 50ms
      p95: 150ms
      p99: 300ms
    throughput:
      minimum_rps: 1000
      target_rps: 5000
    availability: 99.9%
    error_budget:
      monthly_downtime: 43m
      error_rate: 0.1%
```

These SLOs are:
- **Stored in Registry** alongside API metadata
- **Displayed in Developer Portal** so consumers know what to expect
- **Validated by Auditor** against actual production metrics
- **Enforced by Quality Gates** during publication

---

## Automated Performance Governance

### Performance Baselines

Every API version has a performance baseline established during pre-production testing:

```
┌──────────────────────────────────────────────────────────────────┐
│                    Performance Baseline: orders-api v2.1         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Endpoint: GET /orders/{id}                                      │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ Latency (ms)                                               │  │
│  │                                                            │  │
│  │  p50:  ████████░░░░░░░░░░░░  42ms                         │  │
│  │  p90:  █████████████░░░░░░░  89ms                         │  │
│  │  p95:  ██████████████░░░░░░  112ms                        │  │
│  │  p99:  ███████████████████░  198ms                        │  │
│  │                                                            │  │
│  │  SLO Target p95: 150ms  ✅ PASS                           │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Throughput: 3,200 RPS @ 100 concurrent connections              │
│  Error Rate: 0.02% under load                                    │
│  Baseline Date: 2025-11-20                                       │
│  Test Duration: 15 minutes                                       │
│  Test Environment: staging-us-east-1                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Regression Detection

When a new version is submitted, automated performance tests compare against the baseline:

```
┌──────────────────────────────────────────────────────────────────┐
│         Performance Comparison: v2.1.0 → v2.2.0                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  GET /orders/{id}                                                │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │            Baseline (v2.1)    New (v2.2)     Change        │  │
│  │  p50:      42ms               45ms           +7%   ⚠️      │  │
│  │  p90:      89ms               94ms           +6%   ⚠️      │  │
│  │  p95:      112ms              156ms          +39%  ❌      │  │
│  │  p99:      198ms              312ms          +58%  ❌      │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ❌ REGRESSION DETECTED                                          │
│                                                                  │
│  p95 latency exceeds SLO (150ms) and baseline by >20%           │
│                                                                  │
│  Possible causes:                                                │
│  • New database query in OrderService.getOrder()                 │
│  • N+1 query pattern detected in order line items                │
│  • Missing index on orders.customer_id                           │
│                                                                  │
│  Action: Publication blocked until regression resolved           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Performance Quality Gates

Quality gates define pass/fail criteria for performance tests:

```yaml
# Registry quality gate configuration
quality_gates:
  performance:
    enabled: true
    
    # Absolute thresholds (must meet SLO)
    slo_compliance:
      required: true
      metrics:
        - latency_p95
        - latency_p99
        - error_rate
    
    # Relative thresholds (vs. previous version)
    regression_limits:
      latency_p50: +10%    # Allow up to 10% slower
      latency_p95: +15%    # Allow up to 15% slower
      latency_p99: +20%    # Allow up to 20% slower
      throughput: -10%     # Allow up to 10% less throughput
      error_rate: +0.05%   # Allow up to 0.05% more errors
    
    # Minimum test requirements
    test_requirements:
      min_duration: 10m
      min_requests: 10000
      required_percentiles: [p50, p90, p95, p99]
      required_load_levels: [baseline, 2x, 5x]
    
    # Failure behavior
    on_failure:
      block_publication: true
      notify:
        - producer_team
        - platform_team
      require_override_approval: governance_group
```

### Automated Performance Test Triggers

Performance tests run automatically at key lifecycle points:

| Trigger | Test Type | Duration | Load Level |
|---------|-----------|----------|------------|
| PR opened | Smoke test | 2 min | 10% baseline |
| PR merged to main | Baseline test | 10 min | 100% baseline |
| Staging deployment | Load test | 15 min | 100%, 200%, 500% |
| Pre-production approval | Soak test | 1 hour | 100% baseline |
| Canary deployment | Shadow traffic | Continuous | Production mirror |
| Weekly scheduled | Regression check | 30 min | Full suite |

---

## Load Testing Integration

### Platform-Native Load Testing

The Gateway and Auditor provide natural infrastructure for load testing:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Load Testing Architecture                    │
└─────────────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │  Test Scheduler │
                    │   (Registry)    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
       ┌──────────┐   ┌──────────┐   ┌──────────┐
       │  Worker  │   │  Worker  │   │  Worker  │
       │  Node 1  │   │  Node 2  │   │  Node 3  │
       └────┬─────┘   └────┬─────┘   └────┬─────┘
            │              │              │
            └──────────────┼──────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Gateway   │◄─── Test traffic tagged
                    │  (Staging)  │     with test_run_id
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
         ┌────────┐  ┌────────┐  ┌────────┐
         │ API A  │  │ API B  │  │ API C  │
         └────────┘  └────────┘  └────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Auditor   │◄─── Aggregates metrics
                    │             │     by test_run_id
                    └─────────────┘
```

### Load Test Specification

Load tests are defined declaratively and stored in Registry:

```yaml
load_test:
  name: orders-api-load-test
  api: orders-api
  version: "2.1"
  environment: staging
  
  scenarios:
    - name: baseline
      description: Normal production-like load
      duration: 10m
      ramp_up: 2m
      virtual_users: 100
      requests_per_second: 500
      
    - name: peak
      description: 2x expected peak load
      duration: 10m
      ramp_up: 3m
      virtual_users: 200
      requests_per_second: 1000
      
    - name: stress
      description: Find breaking point
      duration: 15m
      ramp_up: 5m
      virtual_users: 500
      requests_per_second: 2500
      
    - name: soak
      description: Extended duration stability
      duration: 2h
      ramp_up: 5m
      virtual_users: 100
      requests_per_second: 500

  workload:
    # Weighted distribution of API calls
    - endpoint: GET /orders/{id}
      weight: 50
      parameters:
        id: ${random_order_id}
        
    - endpoint: GET /orders?customer_id={id}
      weight: 30
      parameters:
        id: ${random_customer_id}
        
    - endpoint: POST /orders
      weight: 15
      body: ${order_template}
      
    - endpoint: DELETE /orders/{id}
      weight: 5
      parameters:
        id: ${deletable_order_id}

  data_sources:
    random_order_id:
      type: csv
      file: test-data/order-ids.csv
      
    random_customer_id:
      type: range
      min: 1000
      max: 99999
      
    order_template:
      type: json_template
      file: test-data/order-template.json
      
    deletable_order_id:
      type: api
      endpoint: POST /test/orders  # Creates test order
      extract: $.order_id

  assertions:
    - metric: latency_p95
      condition: "<"
      threshold: 150ms
      
    - metric: latency_p99
      condition: "<"
      threshold: 300ms
      
    - metric: error_rate
      condition: "<"
      threshold: 0.1%
      
    - metric: throughput
      condition: ">="
      threshold: 500rps

  notifications:
    on_start:
      - slack: #api-platform
    on_complete:
      - slack: #api-platform
      - email: api-owners@company.com
    on_failure:
      - pagerduty: api-platform-oncall
```

### Real Traffic Replay

The Auditor captures production traffic patterns that can be replayed for realistic load testing:

```
Production Traffic Capture → Sanitization → Replay in Staging

┌─────────────────────────────────────────────────────────────────┐
│                    Traffic Replay Pipeline                       │
└─────────────────────────────────────────────────────────────────┘

  Production                Sanitization              Staging
  ┌────────┐               ┌────────────┐           ┌────────┐
  │ Auditor│──────────────▶│  Redactor  │──────────▶│ Replay │
  │  Logs  │  Raw traffic  │            │ Sanitized │ Engine │
  └────────┘               └────────────┘           └────────┘
                                 │
                                 ▼
                           ┌──────────┐
                           │ Redacts: │
                           │ • PII    │
                           │ • Tokens │
                           │ • Keys   │
                           └──────────┘
```

**Benefits of Traffic Replay:**
- **Realistic workload distribution** — Actual endpoint usage patterns, not guesses
- **Edge cases included** — Real requests include unusual parameters synthetic tests miss
- **Seasonal patterns** — Replay Monday morning traffic, month-end spikes, etc.
- **Consumer behavior** — See how actual consumers call the API

### Shadow Traffic Testing

For high-confidence pre-production validation, mirror production traffic to the new version:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Shadow Traffic Architecture                   │
└─────────────────────────────────────────────────────────────────┘

                         Production Request
                                │
                                ▼
                    ┌───────────────────────┐
                    │       Gateway         │
                    │    (Production)       │
                    └───────────┬───────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼ (async copy)
            ┌───────────────┐       ┌───────────────┐
            │  API v2.1     │       │  API v2.2     │
            │  (Current)    │       │  (Shadow)     │
            └───────┬───────┘       └───────┬───────┘
                    │                       │
                    ▼                       ▼
            Response returned        Response compared
            to client                but discarded
                                            │
                                            ▼
                                    ┌───────────────┐
                                    │   Comparator  │
                                    │               │
                                    │ • Latency Δ   │
                                    │ • Response Δ  │
                                    │ • Error Δ     │
                                    └───────────────┘
```

**Shadow Traffic Comparison Report:**

```
┌──────────────────────────────────────────────────────────────────┐
│           Shadow Traffic Comparison: v2.1 vs v2.2                │
│           Duration: 24 hours | Requests: 1,247,832               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  LATENCY COMPARISON                                              │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │        v2.1 (prod)     v2.2 (shadow)    Difference         │  │
│  │  p50:  43ms            41ms             -4.6% ✅           │  │
│  │  p95:  112ms           108ms            -3.5% ✅           │  │
│  │  p99:  198ms           187ms            -5.5% ✅           │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  RESPONSE COMPARISON                                             │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Identical responses:     1,245,219  (99.79%)              │  │
│  │  Expected differences:        2,401  (0.19%)  ⚠️           │  │
│  │  Unexpected differences:        212  (0.02%)  ⚠️           │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  UNEXPECTED DIFFERENCES (sample):                                │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  GET /orders/98234                                         │  │
│  │    v2.1: {"status": "pending", "items": [...]}            │  │
│  │    v2.2: {"status": "PENDING", "items": [...]}            │  │
│  │    Issue: Enum case change (pending → PENDING)             │  │
│  │                                                            │  │
│  │  GET /orders?customer_id=12345&limit=100                   │  │
│  │    v2.1: 100 items returned                                │  │
│  │    v2.2: 50 items returned                                 │  │
│  │    Issue: Default limit changed from 100 to 50             │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  VERDICT: ⚠️ REVIEW REQUIRED                                    │
│  2 potential breaking changes detected in shadow comparison      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Chaos Engineering for APIs

### Why Chaos Engineering?

Load tests verify performance under expected conditions. Chaos engineering verifies **resilience under failure conditions**:
- What happens when a downstream dependency is slow?
- How does the API behave when the database connection pool is exhausted?
- Do circuit breakers actually trip?
- Is graceful degradation working?

### Gateway-Injected Faults

The Gateway can inject faults without modifying backend services:

```yaml
chaos_experiment:
  name: downstream-latency-injection
  api: orders-api
  environment: staging
  
  # Target specific traffic
  traffic_selector:
    percentage: 10%  # Only affect 10% of requests
    # OR target specific consumers
    subscription_ids:
      - "test-consumer-123"
  
  faults:
    - type: latency
      delay: 500ms
      probability: 0.3  # 30% of selected traffic
      
    - type: error
      status_code: 503
      probability: 0.1  # 10% of selected traffic
      body: '{"error": "service_unavailable"}'
      
    - type: timeout
      duration: 30s
      probability: 0.05  # 5% of selected traffic

  duration: 15m
  
  abort_conditions:
    - metric: error_rate
      threshold: "> 5%"
      action: abort_and_rollback
      
  success_criteria:
    - description: "Circuit breaker trips within 30s"
      metric: circuit_breaker_open
      condition: "== true"
      within: 30s
      
    - description: "Error rate stabilizes after circuit opens"
      metric: error_rate
      condition: "< 1%"
      after: circuit_breaker_open
```

### Chaos Experiment Types

| Experiment | Fault Injected | Validates |
|------------|---------------|-----------|
| **Latency injection** | Add 100-5000ms delay | Timeout handling, async patterns |
| **Error injection** | Return 500/503 errors | Retry logic, circuit breakers |
| **Partial failure** | Fail specific endpoints | Graceful degradation |
| **Resource exhaustion** | Limit connections | Connection pool sizing |
| **Data corruption** | Malformed responses | Input validation on consumers |
| **Clock skew** | Offset timestamps | Token validation, caching |
| **Network partition** | Block specific routes | Failover, redundancy |

### Automated Resilience Scoring

The Auditor calculates a resilience score based on chaos experiment results:

```
┌──────────────────────────────────────────────────────────────────┐
│                 Resilience Score: orders-api v2.1                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Overall Score: 78/100  ⚠️                                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ Category                    Score    Status                │  │
│  │                                                            │  │
│  │ Timeout Handling            92/100   ✅ Excellent          │  │
│  │ Circuit Breaker             85/100   ✅ Good               │  │
│  │ Retry Logic                 80/100   ✅ Good               │  │
│  │ Graceful Degradation        65/100   ⚠️ Needs Work        │  │
│  │ Error Response Quality      70/100   ⚠️ Needs Work        │  │
│  │ Failover Speed              68/100   ⚠️ Needs Work        │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Recommendations:                                                │
│  • Graceful degradation: Return cached data when inventory-api   │
│    is unavailable instead of 503                                 │
│  • Error responses: Include retry-after header on 503s           │
│  • Failover: Reduce circuit breaker threshold from 10 to 5       │
│    consecutive failures                                          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Testing Infrastructure

### Test Environment Management

The platform provides isolated test environments that mirror production:

```
┌──────────────────────────────────────────────────────────────────┐
│                    Test Environment Topology                      │
└──────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │                        Production                            │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
  │  │ Gateway  │  │ Registry │  │ Auditor  │  │  APIs    │    │
  │  │ (prod)   │  │ (prod)   │  │ (prod)   │  │  (prod)  │    │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
  └─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │   Config Mirror   │
                    └─────────┬─────────┘
                              │
  ┌─────────────────────────────────────────────────────────────┐
  │                        Staging                               │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
  │  │ Gateway  │  │ Registry │  │ Auditor  │  │  APIs    │    │
  │  │ (staging)│  │ (staging)│  │ (staging)│  │ (staging)│    │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
  │                                                              │
  │  • Production config synced daily                            │
  │  • Synthetic test data                                       │
  │  • Isolated network                                          │
  │  • Full load testing capability                              │
  └─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │  On-Demand Envs   │
                    └─────────┬─────────┘
                              │
  ┌─────────────────────────────────────────────────────────────┐
  │                   Ephemeral Test Environments                │
  │                                                              │
  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐   │
  │  │  PR #1234     │  │  PR #1235     │  │  PR #1236     │   │
  │  │  ───────────  │  │  ───────────  │  │  ───────────  │   │
  │  │  Gateway ✓    │  │  Gateway ✓    │  │  Gateway ✓    │   │
  │  │  orders-api   │  │  users-api    │  │  orders-api   │   │
  │  │  v2.2-pr1234  │  │  v3.1-pr1235  │  │  v2.2-pr1236  │   │
  │  │               │  │               │  │               │   │
  │  │  TTL: 24h     │  │  TTL: 24h     │  │  TTL: 24h     │   │
  │  └───────────────┘  └───────────────┘  └───────────────┘   │
  │                                                              │
  │  • Spun up automatically on PR                               │
  │  • Destroyed after merge/close                               │
  │  • Minimal test data                                         │
  │  • Contract tests only                                       │
  └─────────────────────────────────────────────────────────────┘
```

### Test Data Management

```yaml
test_data_strategy:
  production_data:
    # Never use real production data
    policy: prohibited
    
  synthetic_data:
    # Generated to match production patterns
    generators:
      - type: faker
        locale: en_US
        seed: 12345  # Reproducible
        
      - type: production_statistics
        # Generate data matching production distributions
        source: auditor_metrics
        
  anonymized_data:
    # Production data with PII removed
    pipeline:
      - extract: production_snapshot
      - transform: anonymize_pii
      - transform: scramble_ids
      - load: staging_database
    schedule: weekly
    
  test_fixtures:
    # Known data for deterministic tests
    location: test-data/fixtures/
    version_controlled: true
```

### Load Generator Infrastructure

Dedicated infrastructure for generating load:

```yaml
load_generator:
  # Kubernetes-based distributed load generation
  infrastructure:
    type: kubernetes
    namespace: load-testing
    
  workers:
    image: company/load-generator:latest
    replicas:
      min: 3
      max: 50
      auto_scale: true
    resources:
      cpu: 2
      memory: 4Gi
      
  distribution:
    # Spread workers across regions for realistic latency
    regions:
      - us-east-1: 40%
      - us-west-2: 30%
      - eu-west-1: 30%
      
  tools:
    primary: k6  # Modern, scriptable
    alternatives:
      - locust  # Python-based
      - gatling  # Scala-based
      - vegeta  # Simple HTTP load
```

---

## Metrics & Quality Gates

### Test Metrics Dashboard

The Auditor provides unified visibility into all test results:

```
┌──────────────────────────────────────────────────────────────────┐
│                    API Testing Dashboard                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Test Execution Summary (Last 7 Days)                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                                                            │  │
│  │  Total Tests: 1,247        Pass Rate: 94.2%               │  │
│  │                                                            │  │
│  │  By Type:                                                  │  │
│  │    Contract Tests:    847  (96% pass)  ████████████████░  │  │
│  │    Performance Tests: 234  (91% pass)  ██████████████░░░  │  │
│  │    Load Tests:         98  (89% pass)  █████████████░░░░  │  │
│  │    Chaos Tests:        68  (94% pass)  ███████████████░░  │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Quality Gate Status                                             │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                                                            │  │
│  │  APIs Ready for Production:     42/47                      │  │
│  │                                                            │  │
│  │  Blocked APIs:                                             │  │
│  │    • inventory-api v3.2    - Performance regression       │  │
│  │    • shipping-api v2.0     - Contract test failures       │  │
│  │    • auth-api v4.1         - Pending load test            │  │
│  │    • pricing-api v2.3      - Chaos test failures          │  │
│  │    • notifications-api v1.5 - Missing baseline            │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Performance Trends                                              │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  p95 Latency (All APIs, 30 days)                          │  │
│  │                                                            │  │
│  │  150ms ┤                                                   │  │
│  │        │      ╭─╮                                          │  │
│  │  100ms ┤  ╭───╯ ╰──╮    ╭──╮                              │  │
│  │        │ ─╯        ╰────╯  ╰─────────────────────────     │  │
│  │   50ms ┤                                                   │  │
│  │        │                                                   │  │
│  │    0ms ┼────────────────────────────────────────────────  │  │
│  │        Oct 25    Nov 1    Nov 8    Nov 15    Nov 22       │  │
│  │                                                            │  │
│  │  Trend: Stable (σ = 12ms)                                 │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Quality Gate Configuration

Define organization-wide quality standards:

```yaml
# Organization-wide quality gate policy
quality_gates:
  # Tier 1: Critical APIs (payments, auth, core data)
  tier_1:
    contract_tests:
      required: true
      coverage: ">= 90%"
      
    performance:
      baseline_required: true
      slo_compliance: required
      regression_tolerance: 10%
      
    load_test:
      required: true
      scenarios: [baseline, peak, stress]
      min_duration: 15m
      
    chaos:
      required: true
      experiments: [latency, errors, timeout]
      resilience_score: ">= 80"
      
    security:
      owasp_scan: required
      dependency_scan: required
      
  # Tier 2: Important APIs (internal services)
  tier_2:
    contract_tests:
      required: true
      coverage: ">= 70%"
      
    performance:
      baseline_required: true
      slo_compliance: required
      regression_tolerance: 15%
      
    load_test:
      required: true
      scenarios: [baseline, peak]
      min_duration: 10m
      
    chaos:
      required: false
      recommended: true
      
  # Tier 3: Low-risk APIs (internal tools, non-critical)
  tier_3:
    contract_tests:
      required: true
      coverage: ">= 50%"
      
    performance:
      baseline_required: recommended
      slo_compliance: recommended
      
    load_test:
      required: false
      recommended: true
```

### CI/CD Integration

```yaml
# .github/workflows/api-quality.yml
name: API Quality Gates

on:
  pull_request:
    paths:
      - 'apis/**'
      - 'openapi/**'

jobs:
  contract-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Fetch consumer contracts
        run: |
          curl -H "Authorization: Bearer ${{ secrets.REGISTRY_TOKEN }}" \
            "${{ vars.REGISTRY_URL }}/apis/${{ env.API_ID }}/contracts" \
            -o contracts.json
            
      - name: Run contract tests
        run: |
          npm run test:contracts -- --contracts contracts.json
          
      - name: Upload results to Registry
        run: |
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.REGISTRY_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d @test-results.json \
            "${{ vars.REGISTRY_URL }}/apis/${{ env.API_ID }}/test-results"

  performance-baseline:
    runs-on: ubuntu-latest
    needs: contract-tests
    steps:
      - name: Deploy to ephemeral environment
        run: |
          kubectl apply -f k8s/ephemeral-env.yaml
          
      - name: Run performance baseline
        run: |
          k6 run \
            --out json=results.json \
            --tag testid=${{ github.sha }} \
            performance/baseline.js
            
      - name: Compare with previous baseline
        run: |
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.REGISTRY_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d @results.json \
            "${{ vars.REGISTRY_URL }}/apis/${{ env.API_ID }}/performance/compare"
            
      - name: Check quality gate
        run: |
          RESULT=$(curl -s "${{ vars.REGISTRY_URL }}/apis/${{ env.API_ID }}/quality-gate")
          if [ "$(echo $RESULT | jq -r '.passed')" != "true" ]; then
            echo "Quality gate failed:"
            echo $RESULT | jq '.failures'
            exit 1
          fi
```

---

## Summary

Testing integrated with API governance transforms quality from an afterthought to a core capability:

| Traditional Testing | Governance-Integrated Testing |
|--------------------|------------------------------|
| Tests run in isolation | Tests inform lifecycle decisions |
| Results in CI logs | Results in Auditor dashboards |
| Pass/fail binary | Trend analysis and regression detection |
| Manual performance benchmarks | Automated baseline comparison |
| Load tests on request | Scheduled, triggered, continuous |
| Chaos as special project | Chaos as routine validation |
| Quality gates per team | Organization-wide standards |

The platform's existing components provide natural integration points:
- **Registry** stores test specifications, contracts, and quality gate configurations
- **Gateway** enables fault injection, traffic mirroring, and test traffic routing
- **Auditor** aggregates results, tracks trends, and enforces quality gates

This approach ensures APIs meet quality standards before they impact consumers—and continues validating them in production.

---

[Back to Technical Design](technical-design.md) | [Back to Main README](README.md)
