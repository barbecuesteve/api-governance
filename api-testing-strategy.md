<script type="module">
	import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
	mermaid.initialize({
		startOnLoad: true,
		theme: 'dark'
	});
</script>

[Back to API Governance Framework](README.md)

# API Testing Strategy & Performance Governance

This document extends the API governance framework to include comprehensive testing capabilities. Testing is not separate from governance—it's how we verify that governance actually works. The platform's existing components (Registry, Gateway, Auditor) provide natural integration points for automated testing throughout the API lifecycle.

---

## Table of Contents

1. [Testing Philosophy](#testing-philosophy)
2. [Testing in the API Lifecycle](#testing-in-the-api-lifecycle)
3. [Core Testing Capabilities](#core-testing-capabilities)
   - [Contract Testing](#contract-testing)
   - [Performance & Load Testing](#performance--load-testing)
   - [Chaos Engineering](#chaos-engineering)
4. [Automated Governance & Quality Gates](#automated-governance--quality-gates)
5. [CI/CD Integration](#cicd-integration)
6. [Testing Infrastructure](#testing-infrastructure)

---

## Testing Philosophy

### Testing as Governance

Traditional API testing happens in isolation—teams run tests in their own environments, results live in CI logs nobody reads, and production issues still surprise everyone. **Testing integrated with governance changes this.**

When testing data flows through the same platform that manages API lifecycle:
- **Test results block publication** — APIs cannot reach "Published" state without passing quality gates
- **Performance baselines are enforced** — New versions must meet or exceed previous version's latency
- **Consumer impact is visible** — Before deploying, see how changes affect actual consumers
- **Historical trends inform decisions** — Auditor tracks performance over time

### Shift Left, But Also Shift Right

**Shift Left:** Catch problems early through contract testing, schema validation, and mock-based integration tests in development.

**Shift Right:** Production is the ultimate test environment. The Auditor provides continuous performance monitoring that validates what synthetic tests can only estimate.

The platform supports both: rigorous pre-production gates *and* continuous production validation.

---

## Testing in the API Lifecycle

Testing integrates at every lifecycle stage:

| Stage | Testing Activity | Platform Integration |
|-------|------------------|---------------------|
| **Design** | Schema validation, mock generation | Registry validates specs (OpenAPI, GraphQL SDL, AsyncAPI); generates mocks automatically |
| **Review** | Contract compatibility check | Automated breaking change detection before approval |
| **Build** | Unit tests, contract tests | CI/CD integration via Registry webhooks |
| **Pre-Production** | Performance baseline, load testing | Gateway routes to test environments; Auditor captures baselines |
| **Publication** | Quality gate validation | Registry blocks publication if tests fail |
| **Production** | Continuous performance monitoring | Auditor tracks SLO compliance in real-time |
| **Deprecation** | Consumer migration verification | Auditor confirms consumers moved to new version |

<pre class="mermaid">
flowchart LR
    subgraph Design
        D1[Schema Validation]
        D2[Mock Gen]
    end
    subgraph Review
        R1[Breaking Change Detection]
    end
    subgraph Build
        B1[Contract Tests]
        B2[Unit Tests]
    end
    subgraph Staging
        S1[Performance Baseline]
        S2[Load Tests]
    end
    subgraph Prod
        P1[Continuous Monitoring]
        P2[SLO Tracking]
    end
    
    Design --> Review --> Build --> Staging --> Prod
</pre>

---

## Core Testing Capabilities

This section covers the fundamental testing approaches that validate API behavior, performance, and resilience. These capabilities work together to ensure APIs meet their contracts under normal and adverse conditions.

### Contract Testing

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

<pre class="mermaid">
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e8f4f8','primaryTextColor':'#000','primaryBorderColor':'#000','lineColor':'#333'}}}%%
sequenceDiagram
    participant Consumer as Consumer Team
    participant Registry as Registry
    participant Producer as Producer Team

    Consumer->>Registry: 1. Define contract
    Registry->>Producer: 2. Notify producer
    Producer->>Registry: 3. Fetch contracts
    Note over Producer: 4. Run contract tests
    Producer->>Registry: 5. Report results
    Registry->>Consumer: 5. Contract verified
</pre>

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
```

<table>
<tr><th colspan="2">Change: Field 'customer_name' removed from GET /orders/{id}</th></tr>
<tr><td><strong>Consumers expecting this field:</strong></td><td></td></tr>
<tr><td>checkout-service</td><td>12,000 calls/day</td></tr>
<tr><td>reporting-dashboard</td><td>500 calls/day</td></tr>
<tr><td>mobile-app-backend</td><td>8,000 calls/day</td></tr>
<tr><td colspan="2"><strong>Action Required:</strong> Bump to major version (v2 → v3)</td></tr>
</table>

### Protocol-Specific Contract Testing

Contract testing varies by API protocol. The platform supports all three first-class protocols with tailored approaches.

#### REST (OpenAPI) Contracts

The example above shows REST contract testing. Additional considerations:

- **HTTP semantics matter** — Status codes, headers, and content types are part of the contract
- **Path parameters** — Contracts specify expected path patterns and parameter types
- **Pagination contracts** — Consumers can specify expected pagination behavior

#### GraphQL Contracts

GraphQL contracts focus on **operations and fragments**, not endpoints:

```json
{
  "consumer": "mobile-app",
  "provider": "orders-graphql-api",
  "protocol": "graphql",
  "interactions": [
    {
      "description": "fetch order with line items",
      "operation": "query",
      "query": "query GetOrder($id: ID!) { order(id: $id) { id status lineItems { sku quantity } } }",
      "variables": { "id": "order-123" },
      "expectedResponse": {
        "data": {
          "order": {
            "id": "order-123",
            "status": "string",
            "lineItems": [{ "sku": "string", "quantity": "number" }]
          }
        }
      }
    }
  ]
}
```

**GraphQL-Specific Breaking Changes:**

| Change | Breaking? | Notes |
|--------|-----------|-------|
| Remove field from type | ✅ Yes | Consumers may query this field |
| Change field type | ✅ Yes | Response shape changes |
| Make nullable field non-nullable | ✅ Yes | Consumers may not handle null |
| Add required argument | ✅ Yes | Existing queries will fail |
| Remove type from union | ✅ Yes | Consumers may expect this type |
| Deprecate field | ❌ No | Field still works, just warned |
| Add optional field | ❌ No | Consumers ignore unknown fields |
| Add optional argument | ❌ No | Existing queries still work |

**GraphQL Contract Testing Tools:**
- **Apollo Studio** — Schema change detection and field usage tracking
- **GraphQL Inspector** — CLI for schema diffing and breaking change detection
- **Stellate** — Schema registry with compatibility checking

#### AsyncAPI (Event-Driven) Contracts

Event-driven contracts define **message schemas and channel bindings**:

```json
{
  "consumer": "notification-service",
  "provider": "orders-events",
  "protocol": "asyncapi",
  "interactions": [
    {
      "description": "order created event",
      "channel": "orders.created",
      "operation": "subscribe",
      "message": {
        "headers": {
          "correlationId": "string",
          "timestamp": "string"
        },
        "payload": {
          "orderId": "string",
          "customerId": "string",
          "totalAmount": "number",
          "currency": "string"
        }
      }
    },
    {
      "description": "order status changed event",
      "channel": "orders.status.changed",
      "operation": "subscribe",
      "message": {
        "payload": {
          "orderId": "string",
          "previousStatus": "string",
          "newStatus": "string",
          "changedAt": "string"
        }
      }
    }
  ]
}
```

**AsyncAPI-Specific Breaking Changes:**

| Change | Breaking? | Notes |
|--------|-----------|-------|
| Remove field from payload | ✅ Yes | Consumers may depend on field |
| Change field type | ✅ Yes | Deserialization will fail |
| Rename channel | ✅ Yes | Consumers subscribed to old name |
| Change message format (JSON→Avro) | ✅ Yes | Consumers can't deserialize |
| Add required field without default | ✅ Yes | Old messages won't validate |
| Add optional field | ❌ No | Consumers ignore unknown fields |
| Add new channel | ❌ No | Consumers don't auto-subscribe |

**AsyncAPI Contract Testing Tools:**
- **AsyncAPI Diff** — Schema comparison and breaking change detection
- **Specmatic** — Contract testing for async APIs
- **Schema Registry** (Confluent/Apicurio) — Compatibility checking for Avro/JSON Schema

#### Multi-Protocol Contract Verification

When an API exposes multiple protocols, contracts must be verified across all:

<table style="border: 2px solid #333; border-collapse: collapse; width: 100%; font-family: system-ui, sans-serif;">
<tr style="background-color: #2c5aa0; color: white;">
  <th colspan="3" style="padding: 12px; text-align: center; font-size: 1.1em;">Orders API Contract Status</th>
</tr>
<tr style="background-color: #e8f4f8;">
  <td style="padding: 8px; font-weight: bold;">REST (OpenAPI)</td>
  <td style="padding: 8px;">checkout-service</td>
  <td style="padding: 8px;">✅ 12 interactions verified</td>
</tr>
<tr>
  <td style="padding: 8px;"></td>
  <td style="padding: 8px;">admin-dashboard</td>
  <td style="padding: 8px;">✅ 8 interactions verified</td>
</tr>
<tr>
  <td style="padding: 8px;"></td>
  <td style="padding: 8px;">mobile-app</td>
  <td style="padding: 8px;">✅ 15 interactions verified</td>
</tr>
<tr style="background-color: #e8f4f8;">
  <td style="padding: 8px; font-weight: bold;">GraphQL</td>
  <td style="padding: 8px;">mobile-app</td>
  <td style="padding: 8px;">✅ 6 operations verified</td>
</tr>
<tr>
  <td style="padding: 8px;"></td>
  <td style="padding: 8px;">analytics-service</td>
  <td style="padding: 8px;">⚠️ 2/4 operations verified (2 deprecated)</td>
</tr>
<tr style="background-color: #e8f4f8;">
  <td style="padding: 8px; font-weight: bold;">AsyncAPI (Events)</td>
  <td style="padding: 8px;">notification-service</td>
  <td style="padding: 8px;">✅ 3 channels verified</td>
</tr>
<tr>
  <td style="padding: 8px;"></td>
  <td style="padding: 8px;">audit-logger</td>
  <td style="padding: 8px;">✅ 5 channels verified</td>
</tr>
<tr>
  <td style="padding: 8px;"></td>
  <td style="padding: 8px;">inventory-sync</td>
  <td style="padding: 8px;">❌ 1 channel FAILED<br><small style="color: #666;">└─ orders.created: Missing 'warehouseId' field</small></td>
</tr>
<tr style="background-color: #ffebee; border-top: 2px solid #333;">
  <td colspan="3" style="padding: 12px; text-align: center;">
    <strong>Overall Status: ❌ BLOCKED</strong><br>
    <span style="color: #666;">Fix required before publication</span>
  </td>
</tr>
</table>

### Performance & Load Testing

#### Why Performance Testing Belongs in Governance

Performance is a feature. An API that returns correct data in 5 seconds isn't meeting its contract if consumers expect 200ms. The governance platform makes performance:

- **Visible** — Every API has documented latency expectations
- **Measured** — Auditor tracks actual performance continuously
- **Enforced** — Quality gates prevent slow APIs from reaching production
- **Comparable** — New versions benchmarked against previous versions

#### Performance Dimensions

| Dimension | Definition | Measurement |
|-----------|------------|-------------|
| **Latency** | Time from request to response | p50, p90, p95, p99 percentiles |
| **Throughput** | Requests handled per second | RPS at various concurrency levels |
| **Error Rate** | Percentage of failed requests | 4xx, 5xx rates under load |
| **Saturation** | Resource utilization | CPU, memory, connection pool usage |
| **Scalability** | Performance change with load | Latency curve as RPS increases |

#### Performance SLOs in API Specifications

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

#### Platform-Native Load Testing

The Gateway and Auditor provide natural infrastructure for load testing:

<pre class="mermaid">
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e8f4f8','primaryTextColor':'#000','primaryBorderColor':'#000','lineColor':'#333'}}}%%
flowchart TB
    subgraph Orchestration
        Scheduler[Test Scheduler<br/>Registry]
    end
    
    subgraph Workers
        W1[Worker Node 1]
        W2[Worker Node 2]
        W3[Worker Node 3]
    end
    
    subgraph Gateway Layer
        GW[Gateway - Staging<br/>Test traffic tagged with test_run_id]
    end
    
    subgraph APIs
        A[API A]
        B[API B]
        C[API C]
    end
    
    subgraph Metrics
        Auditor[Auditor<br/>Aggregates metrics by test_run_id]
    end
    
    Scheduler --> W1
    Scheduler --> W2
    Scheduler --> W3
    W1 --> GW
    W2 --> GW
    W3 --> GW
    GW --> A
    GW --> B
    GW --> C
    A --> Auditor
    B --> Auditor
    C --> Auditor
</pre>

#### Load Test Specification

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

#### Real Traffic Replay

The Auditor captures production traffic patterns that can be replayed for realistic load testing:

<pre class="mermaid">
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e8f4f8','primaryTextColor':'#000','primaryBorderColor':'#000','lineColor':'#333'}}}%%
flowchart LR
    subgraph Production
        Logs[Auditor Logs]
    end
    
    subgraph Sanitization
        Redactor[Redactor]
        Redacts[Redacts:<br/>PII, Tokens, Keys]
    end
    
    subgraph Staging
        Replay[Replay Engine]
    end
    
    Logs -->|Raw traffic| Redactor
    Redactor -->|Sanitized| Replay
    Redactor --> Redacts
</pre>
**Benefits of Traffic Replay:**
- **Realistic workload distribution** — Actual endpoint usage patterns, not guesses
- **Edge cases included** — Real requests include unusual parameters synthetic tests miss
- **Seasonal patterns** — Replay Monday morning traffic, month-end spikes, etc.
- **Consumer behavior** — See how actual consumers call the API

#### Shadow Traffic Testing

For high-confidence pre-production validation, mirror production traffic to the new version:

<pre class="mermaid">
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e8f4f8','primaryTextColor':'#000','primaryBorderColor':'#000','lineColor':'#333'}}}%%
flowchart TB
    Request[Production Request]
    Gateway[Gateway - Production]
    V21[API v2.1 - Current]
    V22[API v2.2 - Shadow]
    Response[Response returned to client]
    Compare[Response compared but discarded]
    Comparator[Comparator:<br/>Latency Δ, Response Δ, Error Δ]
    
    Request --> Gateway
    Gateway --> V21
    Gateway -->|async copy| V22
    V21 --> Response
    V22 --> Compare
    Compare --> Comparator
</pre>

**Shadow Traffic Comparison Report:**

<table>
<tr><th colspan="4">Shadow Traffic Comparison: v2.1 vs v2.2</th></tr>
<tr><td colspan="4"><strong>Duration:</strong> 24 hours | <strong>Requests:</strong> 1,247,832</td></tr>
<tr><th colspan="4">Latency Comparison</th></tr>
<tr><th>Percentile</th><th>v2.1 (prod)</th><th>v2.2 (shadow)</th><th>Difference</th></tr>
<tr><td>p50</td><td>43ms</td><td>41ms</td><td>-4.6% ✅</td></tr>
<tr><td>p95</td><td>112ms</td><td>108ms</td><td>-3.5% ✅</td></tr>
<tr><td>p99</td><td>198ms</td><td>187ms</td><td>-5.5% ✅</td></tr>
<tr><th colspan="4">Response Comparison</th></tr>
<tr><td>Identical responses</td><td colspan="2">1,245,219</td><td>99.79%</td></tr>
<tr><td>Expected differences</td><td colspan="2">2,401</td><td>0.19% ⚠️</td></tr>
<tr><td>Unexpected differences</td><td colspan="2">212</td><td>0.02% ⚠️</td></tr>
<tr><th colspan="4">Unexpected Differences (sample)</th></tr>
<tr><td colspan="4"><code>GET /orders/98234</code><br/>v2.1: <code>{"status": "pending", "items": [...]}</code><br/>v2.2: <code>{"status": "PENDING", "items": [...]}</code><br/><em>Issue: Enum case change (pending → PENDING)</em></td></tr>
<tr><td colspan="4"><code>GET /orders?customer_id=12345&limit=100</code><br/>v2.1: 100 items returned<br/>v2.2: 50 items returned<br/><em>Issue: Default limit changed from 100 to 50</em></td></tr>
<tr><th colspan="4">⚠️ VERDICT: REVIEW REQUIRED</th></tr>
<tr><td colspan="4">2 potential breaking changes detected in shadow comparison</td></tr>
</table>

### Platform-Native Load Testing

The Gateway and Auditor provide natural infrastructure for load testing:

<pre class="mermaid">
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e8f4f8','primaryTextColor':'#000','primaryBorderColor':'#000','lineColor':'#333'}}}%%
flowchart TB
    subgraph Orchestration
        Scheduler[Test Scheduler<br/>Registry]
    end
    
    subgraph Workers
        W1[Worker Node 1]
        W2[Worker Node 2]
        W3[Worker Node 3]
    end
    
    subgraph Gateway Layer
        GW[Gateway - Staging<br/>Test traffic tagged with test_run_id]
    end
    
    subgraph APIs
        A[API A]
        B[API B]
        C[API C]
    end
    
    subgraph Metrics
        Auditor[Auditor<br/>Aggregates metrics by test_run_id]
    end
    
    Scheduler --> W1
    Scheduler --> W2
    Scheduler --> W3
    W1 --> GW
    W2 --> GW
    W3 --> GW
    GW --> A
    GW --> B
    GW --> C
    A --> Auditor
    B --> Auditor
    C --> Auditor
</pre>

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

<pre class="mermaid">
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e8f4f8','primaryTextColor':'#000','primaryBorderColor':'#000','lineColor':'#333'}}}%%
flowchart LR
    subgraph Production
        Logs[Auditor Logs]
    end
    
    subgraph Sanitization
        Redactor[Redactor]
        Redacts[Redacts:<br/>PII, Tokens, Keys]
    end
    
    subgraph Staging
        Replay[Replay Engine]
    end
    
    Logs -->|Raw traffic| Redactor
    Redactor -->|Sanitized| Replay
    Redactor --> Redacts
</pre>
**Benefits of Traffic Replay:**
- **Realistic workload distribution** — Actual endpoint usage patterns, not guesses
- **Edge cases included** — Real requests include unusual parameters synthetic tests miss
- **Seasonal patterns** — Replay Monday morning traffic, month-end spikes, etc.
- **Consumer behavior** — See how actual consumers call the API

### Shadow Traffic Testing

For high-confidence pre-production validation, mirror production traffic to the new version:

<pre class="mermaid">
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e8f4f8','primaryTextColor':'#000','primaryBorderColor':'#000','lineColor':'#333'}}}%%
flowchart TB
    Request[Production Request]
    Gateway[Gateway - Production]
    V21[API v2.1 - Current]
    V22[API v2.2 - Shadow]
    Response[Response returned to client]
    Compare[Response compared but discarded]
    Comparator[Comparator:<br/>Latency Δ, Response Δ, Error Δ]
    
    Request --> Gateway
    Gateway --> V21
    Gateway -->|async copy| V22
    V21 --> Response
    V22 --> Compare
    Compare --> Comparator
</pre>

**Shadow Traffic Comparison Report:**

<table>
<tr><th colspan="4">Shadow Traffic Comparison: v2.1 vs v2.2</th></tr>
<tr><td colspan="4"><strong>Duration:</strong> 24 hours | <strong>Requests:</strong> 1,247,832</td></tr>
<tr><th colspan="4">Latency Comparison</th></tr>
<tr><th>Percentile</th><th>v2.1 (prod)</th><th>v2.2 (shadow)</th><th>Difference</th></tr>
<tr><td>p50</td><td>43ms</td><td>41ms</td><td>-4.6% ✅</td></tr>
<tr><td>p95</td><td>112ms</td><td>108ms</td><td>-3.5% ✅</td></tr>
<tr><td>p99</td><td>198ms</td><td>187ms</td><td>-5.5% ✅</td></tr>
<tr><th colspan="4">Response Comparison</th></tr>
<tr><td>Identical responses</td><td colspan="2">1,245,219</td><td>99.79%</td></tr>
<tr><td>Expected differences</td><td colspan="2">2,401</td><td>0.19% ⚠️</td></tr>
<tr><td>Unexpected differences</td><td colspan="2">212</td><td>0.02% ⚠️</td></tr>
<tr><th colspan="4">Unexpected Differences (sample)</th></tr>
<tr><td colspan="4"><code>GET /orders/98234</code><br/>v2.1: <code>{"status": "pending", "items": [...]}</code><br/>v2.2: <code>{"status": "PENDING", "items": [...]}</code><br/><em>Issue: Enum case change (pending → PENDING)</em></td></tr>
<tr><td colspan="4"><code>GET /orders?customer_id=12345&limit=100</code><br/>v2.1: 100 items returned<br/>v2.2: 50 items returned<br/><em>Issue: Default limit changed from 100 to 50</em></td></tr>
<tr><th colspan="4">⚠️ VERDICT: REVIEW REQUIRED</th></tr>
<tr><td colspan="4">2 potential breaking changes detected in shadow comparison</td></tr>
</table>

### Chaos Engineering

#### Why Chaos Engineering?

Load tests verify performance under expected conditions. Chaos engineering verifies **resilience under failure conditions**:
- What happens when a downstream dependency is slow?
- How does the API behave when the database connection pool is exhausted?
- Do circuit breakers actually trip?
- Is graceful degradation working?

#### Gateway-Injected Faults

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

#### Chaos Experiment Types

| Experiment | Fault Injected | Validates |
|------------|---------------|-----------|
| **Latency injection** | Add 100-5000ms delay | Timeout handling, async patterns |
| **Error injection** | Return 500/503 errors | Retry logic, circuit breakers |
| **Partial failure** | Fail specific endpoints | Graceful degradation |
| **Resource exhaustion** | Limit connections | Connection pool sizing |
| **Data corruption** | Malformed responses | Input validation on consumers |
| **Clock skew** | Offset timestamps | Token validation, caching |
| **Network partition** | Block specific routes | Failover, redundancy |

#### Automated Resilience Scoring

The Auditor calculates a resilience score based on chaos experiment results:

<table>
<tr><th colspan="3">Resilience Score: orders-api v2.1</th></tr>
<tr><td colspan="3"><strong>Overall Score: 78/100 ⚠️</strong></td></tr>
<tr><th>Category</th><th>Score</th><th>Status</th></tr>
<tr><td>Timeout Handling</td><td>92/100</td><td>✅ Excellent</td></tr>
<tr><td>Circuit Breaker</td><td>85/100</td><td>✅ Good</td></tr>
<tr><td>Retry Logic</td><td>80/100</td><td>✅ Good</td></tr>
<tr><td>Graceful Degradation</td><td>65/100</td><td>⚠️ Needs Work</td></tr>
<tr><td>Error Response Quality</td><td>70/100</td><td>⚠️ Needs Work</td></tr>
<tr><td>Failover Speed</td><td>68/100</td><td>⚠️ Needs Work</td></tr>
<tr><th colspan="3">Recommendations</th></tr>
<tr><td colspan="3">• Graceful degradation: Return cached data when inventory-api is unavailable instead of 503</td></tr>
<tr><td colspan="3">• Error responses: Include retry-after header on 503s</td></tr>
<tr><td colspan="3">• Failover: Reduce circuit breaker threshold from 10 to 5 consecutive failures</td></tr>
</table>

---

## Automated Governance & Quality Gates

This section centralizes all concepts related to enforcing quality rules through automated governance. It combines performance baselines, regression detection, quality gate configuration, and metrics dashboards to create a cohesive system for maintaining API quality standards.

### Performance Baselines

Every API version has a performance baseline established during pre-production testing:

<table>
<tr><th colspan="3">Performance Baseline: orders-api v2.1</th></tr>
<tr><th colspan="3">Endpoint: GET /orders/{id}</th></tr>
<tr><th>Percentile</th><th>Latency</th><th>Status</th></tr>
<tr><td>p50</td><td>42ms</td><td></td></tr>
<tr><td>p90</td><td>89ms</td><td></td></tr>
<tr><td>p95</td><td>112ms</td><td>SLO Target: 150ms ✅ PASS</td></tr>
<tr><td>p99</td><td>198ms</td><td></td></tr>
<tr><th colspan="3">Summary</th></tr>
<tr><td>Throughput</td><td colspan="2">3,200 RPS @ 100 concurrent connections</td></tr>
<tr><td>Error Rate</td><td colspan="2">0.02% under load</td></tr>
<tr><td>Baseline Date</td><td colspan="2">2025-11-20</td></tr>
<tr><td>Test Duration</td><td colspan="2">15 minutes</td></tr>
<tr><td>Test Environment</td><td colspan="2">staging-us-east-1</td></tr>
</table>

### Regression Detection

When a new version is submitted, automated performance tests compare against the baseline:

<table>
<tr><th colspan="4">Performance Comparison: v2.1.0 → v2.2.0</th></tr>
<tr><th colspan="4">GET /orders/{id}</th></tr>
<tr><th>Percentile</th><th>Baseline (v2.1)</th><th>New (v2.2)</th><th>Change</th></tr>
<tr><td>p50</td><td>42ms</td><td>45ms</td><td>+7% ⚠️</td></tr>
<tr><td>p90</td><td>89ms</td><td>94ms</td><td>+6% ⚠️</td></tr>
<tr><td>p95</td><td>112ms</td><td>156ms</td><td>+39% ❌</td></tr>
<tr><td>p99</td><td>198ms</td><td>312ms</td><td>+58% ❌</td></tr>
<tr><th colspan="4">❌ REGRESSION DETECTED</th></tr>
<tr><td colspan="4">p95 latency exceeds SLO (150ms) and baseline by >20%</td></tr>
<tr><th colspan="4">Possible Causes</th></tr>
<tr><td colspan="4">• New database query in OrderService.getOrder()</td></tr>
<tr><td colspan="4">• N+1 query pattern detected in order line items</td></tr>
<tr><td colspan="4">• Missing index on orders.customer_id</td></tr>
<tr><th colspan="4">Action: Publication blocked until regression resolved</th></tr>
</table>

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

### Test Metrics Dashboard

The Auditor provides unified visibility into all test results:

<table>
<tr><th colspan="4">API Testing Dashboard</th></tr>
<tr><th colspan="4">Test Execution Summary (Last 7 Days)</th></tr>
<tr><td colspan="2"><strong>Total Tests:</strong> 1,247</td><td colspan="2"><strong>Pass Rate:</strong> 94.2%</td></tr>
<tr><th>Test Type</th><th>Count</th><th>Pass Rate</th><th></th></tr>
<tr><td>Contract Tests</td><td>847</td><td>96%</td><td>████████████████░</td></tr>
<tr><td>Performance Tests</td><td>234</td><td>91%</td><td>██████████████░░░</td></tr>
<tr><td>Load Tests</td><td>98</td><td>89%</td><td>█████████████░░░░</td></tr>
<tr><td>Chaos Tests</td><td>68</td><td>94%</td><td>███████████████░░</td></tr>
<tr><th colspan="4">Quality Gate Status</th></tr>
<tr><td colspan="4"><strong>APIs Ready for Production:</strong> 42/47</td></tr>
<tr><th colspan="4">Blocked APIs</th></tr>
<tr><td>inventory-api v3.2</td><td colspan="3">Performance regression</td></tr>
<tr><td>shipping-api v2.0</td><td colspan="3">Contract test failures</td></tr>
<tr><td>auth-api v4.1</td><td colspan="3">Pending load test</td></tr>
<tr><td>pricing-api v2.3</td><td colspan="3">Chaos test failures</td></tr>
<tr><td>notifications-api v1.5</td><td colspan="3">Missing baseline</td></tr>
<tr><th colspan="4">Performance Trends</th></tr>
<tr><td colspan="4"><strong>p95 Latency (All APIs, 30 days)</strong><br/>Trend: Stable (σ = 12ms)</td></tr>
</table>

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

## CI/CD Integration

The CI/CD integration is a crucial piece of the implementation puzzle, showing how testing and governance integrate seamlessly into development workflows.

### CI/CD Pipeline Integration

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

## Testing Infrastructure

### Test Environment Management

The platform provides isolated test environments that mirror production:

<pre class="mermaid">
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e8f4f8','primaryTextColor':'#000','primaryBorderColor':'#000','lineColor':'#333'}}}%%
flowchart TB
    subgraph Production
        PGW[Gateway - prod]
        PRG[Registry - prod]
        PAU[Auditor - prod]
        PAP[APIs - prod]
    end
    
    subgraph Staging
        SGW[Gateway - staging]
        SRG[Registry - staging]
        SAU[Auditor - staging]
        SAP[APIs - staging]
        SN1[Production config synced daily]
        SN2[Synthetic test data]
        SN3[Full load testing capability]
    end
    
    subgraph Ephemeral["Ephemeral Test Environments"]
        PR1[PR #1234<br/>orders-api v2.2-pr1234<br/>TTL: 24h]
        PR2[PR #1235<br/>users-api v3.1-pr1235<br/>TTL: 24h]
        PR3[PR #1236<br/>orders-api v2.2-pr1236<br/>TTL: 24h]
        EN1[Spun up automatically on PR]
        EN2[Destroyed after merge/close]
        EN3[Contract tests only]
    end
    
    Production -->|Config Mirror| Staging
    Staging -->|On-Demand Envs| Ephemeral
</pre>

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
