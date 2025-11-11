<script type="module">
	import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
	mermaid.initialize({
		startOnLoad: true,
		theme: 'dark'
	});
</script>
## Composition-Based Architecture

<pre class="mermaid">
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

### Integration Contracts

**Registry ↔ Gateway:**
- **Authorization Check API**: `subscription.check(consumer_app_id, api_id, version, env) → {allow|deny, reason, ttl, subscription_id, scopes, rate_limits}`
  - Gateway calls this on every request using the four-field composite key to retrieve subscription
  - Response includes cache TTL to reduce latency impact
  - Returns full subscription metadata needed for policy enforcement (scopes, rate limits, data classification)
- **Event Hooks** (Registry → Gateway):
  - `subscription.revoked`: Immediately invalidate Gateway cache for revoked subscriptions
  - `version.published`: Notify Gateway of new API version availability for routing
  - `api.deprecated`: Trigger Gateway warnings/headers for deprecated API calls

**Gateway ↔ Auditor:**
- **Golden Log Envelope**: Gateway emits structured log for every API call
  - Fields: `trace_id, subscription_id, api_id, version, env, route, verb, status, latency_ms, size_bytes, policy_decision, error_class`
  - Auditor ingests these logs for analytics, compliance reporting, and chargeback calculation
  - Prometheus metrics scraped from Gateway for real-time dashboards

**Backstage ↔ Registry:**
- **Read-Mostly Integration**: Backstage plugins display API catalog, subscription status, lifecycle state
- **Write Operations via Registry APIs**: All mutations (create API, request subscription, trigger deprecation) call Registry APIs
- **No Direct Database Access**: Backstage never writes directly to Registry database; always through Registry API layer

#### Backstage's Native Entity Model

Backstage has built-in support for APIs and applications through its **Software Catalog**:

**Core Entities:**
- **`Component`**: Represents services/applications (your "Consumer Apps")
  - Has `spec.type`: `service`, `website`, `library`, etc.
  - Tracks ownership (`spec.owner`), lifecycle (`spec.lifecycle`), and dependencies
  - Does NOT natively track API subscriptions or approvals
  
- **`API`**: Represents API specifications
  - Has `spec.type`: `openapi`, `asyncapi`, `graphql`, `grpc`
  - Contains `spec.definition` pointing to OpenAPI/AsyncAPI file
  - Has `spec.lifecycle`: `experimental`, `production`, `deprecated`
  - Does NOT natively support versioning as first-class concept (treats each version as separate API entity)
  - Does NOT track subscriptions, approvals, or consumer relationships

- **`Resource`**: Infrastructure elements (databases, queues, cloud resources)

**What Backstage Lacks (Your Registry Must Provide):**

1. **API Versioning Model**: Backstage treats `users-api-v2` and `users-api-v3` as separate API entities, not versions of the same API
2. **Subscription Management**: No concept of Component → API subscriptions with approval workflows
3. **Environment Isolation**: No distinction between dev/staging/production for the same API
4. **Scope/Authorization Metadata**: No fields for endpoint scopes, rate limits, data classification
5. **Deprecation Workflows**: Lifecycle is just a label; no enforcement, transition timelines, or migration tracking
6. **Compliance/Purpose Tracking**: No purpose strings, attestations, or compliance metadata

#### Integration Architecture: Registry as Source of Truth

**Registry owns all governance data; Backstage reads from Registry APIs.**

<pre class="mermaid">
graph TB
    subgraph Backstage["Backstage Software Catalog"]
        subgraph APICard["API Entity (Catalog Card)"]
            Native["- Name, Description, Owner (native)<br/>- OpenAPI spec link (native)"]
            
            subgraph Plugin["API Governance Plugin"]
                Versions["Versions: v2.1 (stable), v3.0 (beta)"]
                Lifecycle["Lifecycle: Production"]
                Subs["Subscriptions: 47 active, 3 pending"]
                Deprecation["Deprecation: v2.0 sunset in 90 days"]
            end
        end
    end
    
    Registry["API Registry Service (PostgreSQL)<br/>- APIs, Versions, Subscriptions, Lifecycle"]
    
    Plugin -->|"API Calls"| Registry
    
    style Backstage fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style APICard fill:#fff,stroke:#90caf9,stroke-width:1px
    style Plugin fill:#fff3e0,stroke:#fb8c00,stroke-width:2px
    style Registry fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style Native fill:#f5f5f5,stroke:#bdbdbd,stroke-width:1px
</pre>

**Why This Approach:**
- **Single source of truth** for governance data avoids synchronization complexity
- **Registry can evolve schema** without Backstage entity model changes
- **Complex queries** (subscription analytics, compliance reports) stay in Registry where they belong
- **Backstage remains lightweight** as pure UI layer, not a data store

**Trade-offs:**
- Backstage native search won't index Registry-specific fields (acceptable - use Registry search APIs)
- Extra latency for page loads (mitigated with 5-minute caching + event-driven invalidation)
- Backstage graph visualizations won't automatically show subscription relationships (build custom visualization plugin if needed)

#### Backstage ↔ Registry Integration Patterns

**Read Operations (Backstage → Registry):**

| Backstage View | Registry API Call | Purpose |
|----------------|-------------------|---------|
| API Catalog Page | `GET /apis/{id}/versions` | Display available versions, maturity, lifecycle state |
| API Version Detail | `GET /apis/{id}/versions/{version}` | Show version metadata, deprecation status, breaking changes |
| My Subscriptions (Component) | `GET /subscriptions?consumer_app_id={id}` | List all APIs this app consumes |
| Subscription Detail | `GET /subscriptions/{id}` | Show approval status, scope, rate limits, expiry |
| Deprecation Dashboard | `GET /apis?lifecycle=deprecated` | List deprecated APIs, sunset timelines, migration paths |
| Compliance Report | `GET /subscriptions?data_classification=pii` | Audit PII access, attestation status |

**Write Operations (Backstage UI → Registry APIs):**

| User Action in Backstage | Registry API Call | Outcome |
|---------------------------|-------------------|---------|
| "Request Subscription" button | `POST /subscriptions` | Creates pending subscription, notifies API owner for approval |
| "Approve Subscription" (API Owner) | `PATCH /subscriptions/{id}` | Updates status to `approved`, triggers Gateway sync |
| "Mark API Deprecated" | `PATCH /apis/{id}/versions/{version}/lifecycle` | Sets state to `deprecated`, starts sunset countdown |
| "Publish New Version" | `POST /apis/{id}/versions` | Creates new version, triggers CI/CD pipeline |
| "Revoke Subscription" | `DELETE /subscriptions/{id}` | Soft deletes subscription, invalidates Gateway cache |

**Entity Mapping (Backstage ↔ Registry):**

| Backstage Concept | Registry Equivalent | Notes |
|-------------------|---------------------|-------|
| `Component` (e.g., `checkout-service`) | `Consumer App` | 1:1 mapping via `metadata.annotations['registry.io/app-id']` |
| `API` entity | `API` + all its versions | Backstage API = umbrella; Registry API = container for versions |
| N/A (separate API entities) | `API Version` | **Key difference**: Registry has first-class versioning, Backstage does not |
| `spec.lifecycle` (label only) | `lifecycle_state` enum | Registry enforces transitions; Backstage just displays |
| N/A | `Subscription` | **Backstage has no native concept** - purely Registry domain |
| `metadata.annotations` | Custom governance metadata | Store `registry.io/api-id`, `registry.io/classification`, etc. |

**Backstage Catalog YAML Example (Enhanced with Registry Metadata):**

```yaml
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: users-api
  description: User management and authentication
  annotations:
    registry.io/api-id: "550e8400-e29b-41d4-a716-446655440000"
    registry.io/classification: "confidential"
    backstage.io/techdocs-ref: dir:.
spec:
  type: openapi
  lifecycle: production  # Basic label; Registry tracks detailed lifecycle
  owner: platform-team
  definition:
    $text: https://github.com/company/users-api/blob/main/openapi.yaml
  system: identity-platform
```

**Custom Backstage Plugin Components:**

Build these React components to surface Registry data:

1. **`<ApiVersionsCard>`**: Displays version table with maturity badges
   - Fetches `GET /apis/{id}/versions`
   - Shows current, preview, deprecated versions
   - Links to migration guides for deprecated versions

2. **`<SubscriptionRequestButton>`**: One-click subscription workflow
   - Opens modal to select version, environment, request purpose
   - Calls `POST /subscriptions` to create pending subscription
   - Shows approval status and estimated SLA

3. **`<MySubscriptionsTab>`**: Component-level view of API dependencies
   - Queries `GET /subscriptions?consumer_app_id={component_id}`
   - Shows rate limits, quota usage, expiry warnings
   - Quick actions: renew, upgrade version, revoke

4. **`<DeprecationTimeline>`**: Visual countdown for deprecated APIs
   - Fetches `GET /apis/{id}/versions/{version}/deprecation`
   - Shows sunset date, migration path, blocking consumers
   - API owner view: list of consumers to migrate

5. **`<ComplianceWidget>`**: Dashboard for security/compliance teams
   - Queries `GET /subscriptions?compliance.attestation_required=true`
   - Shows overdue attestations, PII access reviews
   - Links to subscription detail for remediation

**Caching Strategy:**

To minimize Registry API calls:

- **Backstage Backend Cache**: 5-minute TTL for API metadata (versions, lifecycle)
- **Event-Driven Invalidation**: Registry publishes webhook events to Backstage
  - `api.version.published` → invalidate API version cache
  - `subscription.approved` → invalidate subscription list cache
  - `api.deprecated` → invalidate API detail cache
- **Long-Lived Cache for Stable Data**: 1-hour TTL for API ownership, team info

**Security/Authorization:**

- **Backstage Permissions Plugin**: Restrict who can see/request subscriptions
  - Integration with Registry's `permission.check(user, resource, action)` API
  - Example: Only API owners can approve subscriptions to their APIs
- **Token Passthrough**: Backstage UI forwards user OAuth token to Registry APIs
  - Registry validates token against identity provider (Okta, Auth0, etc.)
  - No need for service-to-service credentials

#### Data Ownership Boundaries

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

**Backstage is the UX layer; Registry is the policy engine.**

Developers discover APIs in Backstage, but all governance actions (request subscription, approve access, deprecate version) flow through Registry APIs. Backstage plugins make Registry data visible and actionable, but never bypass Registry business logic.

### Subscriptions as First-Class Citizens

**Subscriptions are the source of truth for policy enforcement.** Rather than encoding rules at the API or Gateway level, all authorization and governance decisions derive from Subscription metadata.

**Subscription Schema (Minimum Viable):**

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

## Implementation Phases

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

## Bottom Line

**Compose where you can, build where you must.** The infrastructure (gateway, observability, portal UI) is commoditized. Your competitive advantage is in the **governance workflows and business logic** - that's where custom development pays off. Budget: ~60% integration/configuration, ~40% custom development. This gets you to production faster with lower risk than building everything from scratch.