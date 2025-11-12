<script type="module">
	import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
	mermaid.initialize({
		startOnLoad: true,
		theme: 'dark'
	});
</script>

# Composition-Based API Governance Platform

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Concepts](#core-concepts)
4. [Component Specifications](#component-specifications)
5. [Integration Contracts](#integration-contracts)
6. [Implementation Guide](#implementation-guide)

---

## 1. Overview

### Design Philosophy

This architecture balances off-the-shelf components with custom governance logic, centered on **subscriptions as the source of truth** for all policy enforcement.

**Key Principles:**
- **Subscriptions drive policy**: All authorization, rate limiting, and auditing decisions derive from subscription metadata
- **Build vs. buy**: Leverage proven tools (Kong, Backstage, Prometheus) while building custom governance logic where needed
- **Single source of truth**: Registry owns all governance data; other components read from it via APIs
- **Policy as code**: Externalize business rules to OPA for flexibility and auditability

### High-Level Architecture

<pre class="mermaid">
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e8f4f8','primaryTextColor':'#000','primaryBorderColor':'#2c5aa0','lineColor':'#2c5aa0','edgeLabelBackground':'#fff','fontSize':'14px'}}}%%
graph TD
    A[Backstage Developer Portal] --> B[API Registry & Governance Core]
    B --> C[Kong Gateway]
    B --> D[Auditor]
    C --> D
    
    style A fill:#e1f5ff,stroke:#0288d1,stroke-width:2px
    style B fill:#fff3e0,stroke:#f57c00,stroke-width:3px
    style C fill:#e1f5ff,stroke:#0288d1,stroke-width:2px
    style D fill:#fff3e0,stroke:#f57c00,stroke-width:3px
</pre>

**Component Roles:**
- **Backstage Developer Portal** (Customize/Extend): Off-the-shelf UI foundation for API catalog, documentation, and developer experience
- **API Registry & Governance Core** (BUILD THIS): Custom service managing subscriptions, approvals, lifecycle states, deprecation workflows, and policy engine
- **Kong Gateway** (Configure): Off-the-shelf API gateway for auth, rate limiting, routing, and observability
- **Auditor** (BUILD THIS): Custom analytics layer built on Prometheus/Grafana for compliance reporting, chargeback, and SLA tracking

### Data Ownership Boundaries

**Backstage Owns (Native Catalog):**
- API discovery metadata: name, description, owner team
- OpenAPI spec file location
- Documentation links (TechDocs)
- Component relationships (which services produce which APIs)

**Registry Owns (Governance Data):**
- API versions, maturity levels, breaking change tracking
- Subscriptions, approvals, scopes, rate limits
- Lifecycle states, deprecation timelines, retirement automation
- Compliance metadata, attestations, data classification
- Usage analytics, chargeback, SLA tracking

**Integration Principle:**

> **Backstage is the UX layer; Registry is the policy engine.**
> 
> Developers discover APIs in Backstage, but all governance actions (request subscription, approve access, deprecate version) flow through Registry APIs. Backstage plugins make Registry data visible and actionable, but never bypass Registry business logic.

---

## 2. Core Concepts

### Subscriptions as First-Class Citizens

**Subscriptions are the source of truth for policy enforcement.** Rather than encoding rules at the API or Gateway level, all authorization and governance decisions derive from subscription metadata.

#### Subscription Schema

```json
{
  "subscription_id": "uuid",
  
  // Natural composite key - UNIQUE constraint on these four fields
  "consumer_app_id": "uuid",     // WHO is consuming
  "api_id": "uuid",               // WHAT they're consuming  
  "api_version": "2.1.0",         // WHICH version
  "environment": "production",    // WHERE (dev/staging/prod)
  
  "status": "approved|pending|revoked|expired",
  
  "scope": {
    "endpoints": ["/users/{id}", "/users/{id}/orders"],
    "operations": ["GET", "POST"],
    "fields": ["id", "name", "email"]
  },
  
  "purpose": "Customer support dashboard needs read-only access to user profile and order history for troubleshooting",
  
  "data_classification": {
    "max_sensitivity": "confidential",
    "pii_access": true,
    "phi_access": false,
    "pci_access": false
  },
  
  "sla_tier": "gold",
  "rate_limits": {
    "requests_per_second": 100,
    "daily_quota": 1000000,
    "burst_allowance": 150
  },
  
  "throughput_estimate": {
    "peak_rps": 75,
    "avg_rps": 20,
    "justification": "500K active support users, 5% concurrency"
  },
  
  "lifecycle": {
    "approved_by": "api.owner@company.com",
    "approved_at": "2025-11-01T10:00:00Z",
    "expires_at": "2026-11-01T10:00:00Z",
    "auto_renew": true
  },
  
  "compliance": {
    "attestation_required": true,
    "last_review": "2025-10-15",
    "reviewer": "security.team@company.com"
  }
}
```

**Subscription Uniqueness:**

The combination of `(consumer_app_id, api_id, api_version, environment)` forms a **natural composite key** and must be unique:

- **Database constraint**: `UNIQUE (consumer_app_id, api_id, api_version, environment)`
- **Primary lookup pattern**: Gateway authorization checks use these four fields to retrieve the subscription
- **Prevents duplicate subscriptions**: An app cannot have multiple active subscriptions to the same API version in the same environment
- **Version-specific approval**: Access to v2.0 doesn't automatically grant access to v3.0 - each requires separate subscription/approval
- **Environment isolation**: Production approval is separate from dev/staging (different risk, SLA, and compliance requirements)

**Gateway Lookup Example:**
```
GET /subscriptions?app_id={app_id}&api_id={api_id}&version={version}&env={env}
→ Returns single subscription record with all policy metadata
```

The `subscription_id` (UUID) remains useful as:
- Primary key for database performance
- Foreign key in audit logs (`golden_log_envelope.subscription_id`)
- Simpler reference in Backstage UI and notifications

**Why Subscriptions Drive Policy:**

1. **Scoped Access Control**: Endpoint/operation scope defines precisely what the consumer can call - Gateway enforces this without hardcoded rules
2. **Purpose-Based Auditing**: Purpose string enables audit review ("Is this subscription still being used for its stated intent?")
3. **Data Classification Enforcement**: Gateway/OPA checks subscription's `max_sensitivity` against API endpoint data classification
4. **Dynamic Rate Limiting**: SLA tier + throughput estimate inform Gateway rate limiting without manual Kong configuration per consumer
5. **Compliance Traceability**: Every API call logs `subscription_id`, linking traffic to approved purpose and data classification
6. **Time-Bounded Authorization**: `expires_at` enables automatic subscription expiration without manual revocation
7. **Chargeback Accuracy**: Throughput estimate + SLA tier + actual usage = chargeback calculation

**Policy Derivation Examples:**

```rego
# OPA Policy: Check if endpoint is in subscription scope
allow {
  input.subscription.scope.endpoints[_] == input.request.path
  input.subscription.scope.operations[_] == input.request.method
  input.subscription.status == "approved"
  time.now_ns() < time.parse_rfc3339_ns(input.subscription.lifecycle.expires_at)
}

# OPA Policy: Data classification check
allow {
  api_classification := data.apis[input.api_id].classification
  subscription_clearance := input.subscription.data_classification.max_sensitivity
  classification_level(api_classification) <= classification_level(subscription_clearance)
}
```

### Policy Engine Boundary

**Externalize Policy Decisions (OPA/Cedar):**

To avoid hard-coding business rules in Kong plugins, all policy decisions are delegated to an external policy engine:

- **Subscription Authorization**: `allow/deny` decisions based on subscription scope, status, expiration, data classification clearance
- **Deprecation Routing**: Route traffic to preferred versions, inject deprecation warnings, enforce sunset dates
- **PII/Data Classification Access**: Field-level access control derived from subscription's `data_classification` vs API endpoint sensitivity

**Policy Engine Integration:**

**Architecture: Open Source Kong + Custom Policy Service**

We're using Open Source Kong with a custom policy engine service to maximize flexibility and cost-effectiveness:

**Policy Engine Service:**
- **OPA Runtime**: Run Open Policy Agent as sidecar container alongside Registry service
- **Policy Storage**: Rego policy files stored in Git repository (`policies/` directory)
- **Bundle Server**: Registry serves OPA bundles via `/policies/bundle` endpoint
- **Hot Reload**: OPA polls bundle endpoint every 60s for policy updates (configurable)
- **Decision API**: Registry exposes `POST /subscriptions/check` endpoint that internally calls OPA

**Authorization Check Flow:**
1. **Request arrives at Kong Gateway** with API key or JWT
2. **Custom Kong plugin** extracts: `consumer_app_id`, `api_id`, `version`, `environment`, `request.method`, `request.path`
3. **Plugin calls Registry API**: `POST /subscriptions/check` with extracted metadata
4. **Registry queries database** for subscription using composite key `(consumer_app_id, api_id, version, environment)`
5. **Registry calls OPA** with subscription metadata + request context:
   ```json
   {
     "input": {
       "subscription": {...subscription_record...},
       "request": {"method": "GET", "path": "/users/123", "headers": {...}},
       "api": {...api_metadata...}
     }
   }
   ```
6. **OPA evaluates policies**, returns decision with reason:
   ```json
   {
     "allow": true,
     "reason": "subscription_active_and_scoped",
     "headers": {"X-RateLimit-Remaining": "950"},
     "ttl": 30
   }
   ```
7. **Registry returns decision** to Kong plugin with cache TTL
8. **Kong allows/denies** request, adds custom headers if specified

**Custom Kong Plugin Requirements:**

Build a Lua plugin (`kong/plugins/registry-auth`) that:
- **Extracts consumer identity** from API key or JWT claims
- **Looks up consumer_app_id** mapping (cached in Kong's shared dict)
- **Calls Registry `/subscriptions/check` API** with timeout (100ms)
- **Caches responses** in Kong's shared memory using provided TTL
- **Handles failure modes**: fail-closed by default, fail-open if configured per API
- **Logs policy decision** to stdout for Auditor ingestion
- **Injects response headers** for rate limit info, deprecation warnings

**Policy Management Workflow:**
- **Policies as Code**: Rego files in `policies/` Git repo with PR review process
- **Policy Testing**: Unit tests using OPA's test framework (`.rego` → `_test.rego`)
- **Deployment**: CI/CD pipeline validates policies, then Registry rebuilds OPA bundle
- **Versioning**: Bundle includes version hash; OPA only reloads on hash change
- **Audit Trail**: All policy changes tracked via Git history + changelog

**Example Policy (Rego):**

```rego
package authz

default allow = false

# Allow if subscription is active and not expired
allow {
  input.subscription.status == "approved"
  time.now_ns() < time.parse_rfc3339_ns(input.subscription.lifecycle.expires_at)
  endpoint_in_scope
  method_in_scope
}

endpoint_in_scope {
  input.subscription.scope.endpoints[_] == input.request.path
}

method_in_scope {
  input.subscription.scope.operations[_] == input.request.method
}

# Generate reason for deny
reason = msg {
  not input.subscription.status == "approved"
  msg := "subscription_not_approved"
}

reason = msg {
  not time.now_ns() < time.parse_rfc3339_ns(input.subscription.lifecycle.expires_at)
  msg := "subscription_expired"
}
```

**Caching Strategy:**

To minimize latency impact while maintaining security:

- **Short TTL Caching at Gateway**: Authorization decisions cached for 5-60 seconds (configurable per API sensitivity)
- **Push Invalidation from Registry**: Real-time cache invalidation via event hooks
  - `subscription.revoked` → immediate cache purge for that subscription
  - `api.deprecated` → invalidate all cached decisions for that API version
  - `policy.updated` → flush entire authorization cache
- **Cache Keys**: `{subscription_id, api_id, version, env, scopes}` for precise invalidation

**Failure Mode Policy:**

Gateway behavior when policy engine is unreachable:

- **Default: Fail-Closed (Deny)**: If Registry/policy engine is down, deny all requests
  - Appropriate for: PCI/HIPAA-compliant APIs, production environments, PII access
  - Prevents unauthorized access during outages
- **Opt-In: Fail-Open (Allow with Logging)**: For specific non-sensitive APIs, allow traffic with audit log warning
  - Appropriate for: Internal dev/test environments, public read-only APIs
  - Must be explicitly configured per API with justification
- **Circuit Breaker**: After sustained policy engine failures, Gateway can fail-open temporarily with alerting
  - Prevents cascading outages while security team investigates
  - All fail-open requests logged with `policy_decision: "FAILOPEN"` for audit review

---

## 3. Component Specifications

### 3.1 API Registry & Governance Core (Custom Build)

**Responsibilities:**
- Maintain API catalog with version management
- Manage subscription lifecycle (request → approve → revoke → expire)
- Enforce lifecycle state transitions
- Host policy engine (OPA) with policy-as-code
- Provide authorization check API for Gateway
- Emit events for cache invalidation

**Key APIs:**

| Endpoint | Purpose |
|----------|---------|
| `POST /subscriptions/check` | Authorization check called by Gateway on every request |
| `GET /apis/{id}/versions` | Retrieve all versions of an API |
| `POST /subscriptions` | Request new subscription |
| `PATCH /subscriptions/{id}` | Approve/revoke subscription |
| `PATCH /apis/{id}/versions/{version}/lifecycle` | Update lifecycle state (deprecate, retire) |
| `GET /subscriptions?consumer_app_id={id}` | List subscriptions for a consumer |

**Technology Stack:**
- PostgreSQL for persistent data
- OPA (Open Policy Agent) as sidecar for policy decisions
- REST API (Node.js, Go, or Python)
- Event bus for publishing lifecycle events

### 3.2 Kong Gateway (Configure)

**Responsibilities:**
- Route traffic to backend APIs
- Authenticate requests (API keys, JWT)
- Call Registry for authorization checks
- Enforce rate limits from subscription metadata
- Emit structured logs for Auditor
- Cache authorization decisions with TTL

#### URL Structure

All API requests use path-based versioning with this structure:

```
https://{environment}-gateway.company.com/{api-slug}/{version}/{resource-path}

Examples:
GET  https://prod-gateway.company.com/users-api/v2/users/{user_id}
POST https://prod-gateway.company.com/orders-api/v3/orders
GET  https://staging-gateway.company.com/products-api/v1/products?category=electronics
```

**Four-Field Composite Key Extraction:**

1. **Environment**: Derived from gateway cluster hostname (`prod-gateway` → `production`, `staging-gateway` → `staging`)
2. **API ID**: Mapped from URL slug (`/users-api/`) to Registry UUID via Kong route configuration
3. **Version**: Explicit in URL path (`/v2/`)
4. **Consumer App ID**: Extracted from JWT bearer token claims (`app_id` field)

#### Kong Route Configuration

Each API version gets its own Kong service and route with registry metadata:

```yaml
services:
  - name: users-api-v2-prod
    url: http://users-service-v2.internal:8080
    routes:
      - name: users-api-v2-route
        paths:
          - /users-api/v2
        strip_path: true
    plugins:
      - name: registry-auth
        config:
          api_id: "550e8400-e29b-41d4-a716-446655440000"  # Registry UUID for users-api
          version: "2.0"
          environment: "production"
          registry_url: "http://registry-service:8080"
```

#### Custom Plugin Implementation

Build a Lua plugin (`kong/plugins/registry-auth`) with the following behavior:

1. **Validate JWT**: Verify bearer token signature and extract `app_id` claim
2. **Build composite key**: Combine `consumer_app_id` (from JWT), `api_id` (from route config), `version` (from route config), `environment` (from route config)
3. **Check cache**: Look up authorization decision in Kong shared memory using composite key
4. **Call Registry**: If cache miss, call `POST /subscriptions/check` with request metadata
5. **Cache response**: Store decision for TTL seconds (returned by Registry)
6. **Enforce policy**: Allow/deny request based on decision, inject headers for rate limits and subscription ID
7. **Log decision**: Emit structured log with subscription_id, policy_decision, and request metadata for Auditor

**Failure handling**: Default to fail-closed (deny) if Registry is unreachable, with per-API override for fail-open in non-production environments.

### 3.3 Backstage Developer Portal (Customize/Extend)

**Responsibilities:**
- Provide searchable API catalog UI
- Display API documentation and specs
- Enable subscription request workflows
- Show consumer's active subscriptions
- Visualize deprecation timelines
- Surface compliance/attestation status

#### Backstage's Native Entity Model

**Core Entities:**
- **`Component`**: Represents services/applications (your "Consumer Apps")
  - Has `spec.type`: `service`, `website`, `library`, etc.
  - Tracks ownership, lifecycle, and dependencies
  - Does NOT natively track API subscriptions or approvals
  
- **`API`**: Represents API specifications
  - Has `spec.type`: `openapi`, `asyncapi`, `graphql`, `grpc`
  - Contains `spec.definition` pointing to OpenAPI/AsyncAPI file
  - Does NOT natively support versioning as first-class concept
  - Does NOT track subscriptions, approvals, or consumer relationships

**What Backstage Lacks (Registry Must Provide):**

1. **API Versioning Model**: Backstage treats `users-api-v2` and `users-api-v3` as separate entities
2. **Subscription Management**: No concept of Component → API subscriptions with approval workflows
3. **Environment Isolation**: No distinction between dev/staging/production
4. **Scope/Authorization Metadata**: No fields for endpoint scopes, rate limits, data classification
5. **Deprecation Workflows**: Lifecycle is just a label; no enforcement or transition timelines
6. **Compliance/Purpose Tracking**: No purpose strings, attestations, or compliance metadata

#### Custom Backstage Plugins Required

Build these React components to surface Registry data:

1. **`<ApiVersionsCard>`**: Displays version table with maturity badges
2. **`<SubscriptionRequestButton>`**: One-click subscription workflow
3. **`<MySubscriptionsTab>`**: Component-level view of API dependencies
4. **`<DeprecationTimeline>`**: Visual countdown for deprecated APIs
5. **`<ComplianceWidget>`**: Dashboard for security/compliance teams

#### Backstage Catalog YAML Example

```yaml
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: users-api
  description: User management and authentication
  annotations:
    registry.io/api-id: "550e8400-e29b-41d4-a716-446655440000"
    registry.io/classification: "confidential"
spec:
  type: openapi
  lifecycle: production
  owner: platform-team
  definition:
    $text: https://github.com/company/users-api/blob/main/openapi.yaml
```

### 3.4 Auditor (Custom Build)

**Responsibilities:**
- Ingest golden log envelope from Gateway
- Store time-series metrics in Prometheus
- Provide Grafana dashboards for analytics
- Calculate cost attribution/chargeback
- Generate compliance reports
- Track SLA/SLO compliance

**Data Sources:**
- Gateway emits structured logs for every request
- Prometheus scrapes Kong metrics
- Registry provides subscription metadata for enrichment

**Key Metrics:**
- Requests per second by API/version/consumer
- Error rates and latency percentiles
- Quota usage vs. subscription limits
- Deprecated API usage (migration tracking)
- PII access audit trail

---

## 4. Integration Contracts

### 4.1 Registry ↔ Gateway

**Authorization Check API:**

```
POST /subscriptions/check
Request:
{
  "consumer_app_id": "uuid",
  "api_id": "uuid",
  "version": "2.1.0",
  "environment": "production",
  "request": {
    "method": "GET",
    "path": "/users/123",
    "headers": {...}
  }
}

Response:
{
  "allow": true,
  "reason": "subscription_active_and_scoped",
  "subscription_id": "uuid",
  "scopes": ["users:read"],
  "rate_limits": {
    "requests_per_second": 100,
    "burst_allowance": 150
  },
  "headers": {
    "X-RateLimit-Remaining": "950"
  },
  "ttl": 30
}
```

- Gateway calls this on every request using the four-field composite key
- Response includes cache TTL to reduce latency impact
- Returns full subscription metadata needed for policy enforcement

**Event Hooks (Registry → Gateway):**

| Event | Trigger | Gateway Action |
|-------|---------|----------------|
| `subscription.revoked` | Subscription revoked | Immediately invalidate cache for that subscription |
| `version.published` | New API version available | Update routing configuration |
| `api.deprecated` | API marked deprecated | Inject deprecation warnings in response headers |

### 4.2 Gateway ↔ Auditor

**Golden Log Envelope:**

Gateway emits structured log for every API call:

```json
{
  "trace_id": "uuid",
  "subscription_id": "uuid",
  "api_id": "uuid",
  "version": "2.1.0",
  "environment": "production",
  "route": "/users/{id}",
  "verb": "GET",
  "status": 200,
  "latency_ms": 45,
  "size_bytes": 1024,
  "policy_decision": "allow",
  "error_class": null
}
```

Auditor ingests these logs for:
- Analytics dashboards
- Compliance reporting
- Chargeback calculation
- SLA tracking

### 4.3 Backstage ↔ Registry

#### Read Operations

| Backstage View | Registry API Call | Purpose |
|----------------|-------------------|---------|
| API Catalog Page | `GET /apis/{id}/versions` | Display available versions, maturity, lifecycle state |
| API Version Detail | `GET /apis/{id}/versions/{version}` | Show version metadata, deprecation status |
| My Subscriptions | `GET /subscriptions?consumer_app_id={id}` | List all APIs this app consumes |
| Subscription Detail | `GET /subscriptions/{id}` | Show approval status, scope, rate limits |
| Deprecation Dashboard | `GET /apis?lifecycle=deprecated` | List deprecated APIs with sunset timelines |
| Compliance Report | `GET /subscriptions?data_classification=pii` | Audit PII access |

#### Write Operations

| User Action in Backstage | Registry API Call | Outcome |
|---------------------------|-------------------|---------|
| "Request Subscription" | `POST /subscriptions` | Creates pending subscription, notifies API owner |
| "Approve Subscription" | `PATCH /subscriptions/{id}` | Updates status to `approved`, triggers Gateway sync |
| "Mark API Deprecated" | `PATCH /apis/{id}/versions/{version}/lifecycle` | Sets state to `deprecated`, starts sunset countdown |
| "Publish New Version" | `POST /apis/{id}/versions` | Creates new version, triggers CI/CD |
| "Revoke Subscription" | `DELETE /subscriptions/{id}` | Soft deletes subscription, invalidates Gateway cache |

#### Caching Strategy

To minimize Registry API calls:

- **Backstage Backend Cache**: 5-minute TTL for API metadata
- **Event-Driven Invalidation**: Registry publishes webhook events to Backstage
  - `api.version.published` → invalidate API version cache
  - `subscription.approved` → invalidate subscription list cache
  - `api.deprecated` → invalidate API detail cache
- **Long-Lived Cache**: 1-hour TTL for API ownership, team info

#### Security/Authorization

- **Backstage Permissions Plugin**: Restrict who can see/request subscriptions
- **Token Passthrough**: Backstage forwards user OAuth token to Registry APIs
- Registry validates token against identity provider (Okta, Auth0, etc.)

---

## 5. Implementation Guide

### Implementation Phases

**Phase 1 (Months 1-3): Foundation**
- Deploy Kong Gateway + basic config
- Build minimal Registry API (CRUD for APIs, versions, apps)
- Set up Prometheus/Grafana

**Phase 2 (Months 4-6): Governance**
- Implement subscription workflows
- Add lifecycle state management
- Build deprecation/retirement automation
- Deploy Backstage with custom plugins

**Phase 3 (Months 7-9): Analytics & Polish**
- Custom Auditor dashboards
- Compliance reporting
- Maturity model tracking
- CI/CD integrations

---

## Appendix: Entity Mapping

### Backstage ↔ Registry Entity Mapping

| Backstage Concept | Registry Equivalent | Notes |
|-------------------|---------------------|-------|
| `Component` (e.g., `checkout-service`) | `Consumer App` | 1:1 mapping via `metadata.annotations['registry.io/app-id']` |
| `API` entity | `API` + all its versions | Backstage API = umbrella; Registry API = container for versions |
| N/A (separate API entities) | `API Version` | **Key difference**: Registry has first-class versioning |
| `spec.lifecycle` (label only) | `lifecycle_state` enum | Registry enforces transitions; Backstage just displays |
| N/A | `Subscription` | **Backstage has no native concept** - purely Registry domain |
| `metadata.annotations` | Custom governance metadata | Store `registry.io/api-id`, `registry.io/classification`, etc. |
