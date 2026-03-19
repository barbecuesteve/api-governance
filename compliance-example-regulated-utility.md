<script type="module">
	import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
	mermaid.initialize({
		startOnLoad: true,
		theme: 'dark'
	});
</script>
[Back to API Governance Framework](README.md)

# Compliance Framework Example: Regulated Utility 


**Context:** A power utility operates a mix of corporate IT systems, regulated operational environments, and legacy mainframe platforms. Some systems must remain isolated from the public Internet and tightly segmented from other internal environments. The API governance platform provides the **control plane for access, lifecycle management, observability, and evidence collection** across these boundaries.

---

## Platform Scope & Boundaries

### What This Platform Provides

The API governance platform is a **governance, enforcement, and audit layer** that sits between utility applications, modernization services, integration platforms, and restricted environments. It provides:

**Enforcement Capabilities:**
- **Network-Zone-Aware Access Control** - Enforces which applications may call which APIs in which environments and trust zones
- **Gateway-Mediated Connectivity** - Ensures approved traffic flows through controlled ingress/egress points rather than ad hoc direct connections
- **Subscription-Based Authorization** - Requires explicit, auditable approval before one system consumes another
- **Segmentation Enforcement** - Applies different controls for corporate IT, regulated utility networks, and isolated operational environments
- **Policy Enforcement** - Applies environment, protocol, and data-handling rules consistently across modern and legacy integrations

**Lifecycle & Coordination:**
- **System of Record for Interfaces** - Tracks APIs, events, owners, consumers, lifecycle state, and modernization status
- **Dependency Visibility** - Shows which downstream systems depend on which mainframe transactions, APIs, and integration services
- **Deprecation Control** - Prevents teams from retiring legacy interfaces without understanding consumer impact
- **Cross-Team Workflow** - Gives modernization, platform, security, and business teams a common operating model

**Audit & Compliance Evidence:**
- **Immutable Audit Trails** - Captures who accessed what, from where, when, and under which approval
- **Change Traceability** - Links interface publication, approval, version changes, and production access to named owners
- **Continuous Monitoring** - Tracks policy violations, anomalous access patterns, and usage of restricted interfaces
- **Audit-Ready Reporting** - Produces evidence for regulatory reviews, cybersecurity audits, and internal control assessments

**Observability:**
- **Dependency Mapping** - Shows which systems, teams, and workflows depend on each interface
- **Operational Telemetry** - Tracks latency, errors, throughput, and saturation indicators for modern and legacy-backed interfaces
- **Blast-Radius Analysis** - Helps incident responders understand which consumers and business functions are affected by an outage or degraded dependency
- **Migration Visibility** - Shows whether consumers are moving off legacy contracts and onto modernized interfaces

### What This Platform Cannot Provide

The following utility-specific capabilities are **outside the platform's scope** and must be implemented by backend systems or specialized operational technology:

**Operational Control Systems:**
- **SCADA / EMS / DMS Control Logic** - Does not operate substations, distribution systems, energy management systems, or protective relays
- **Safety Interlocks** - Does not replace field safety controls, switching procedures, or engineering protections
- **Real-Time Grid Operations** - Does not perform dispatch, balancing, or control-room functions

**Legacy System Operability:**
- **Data Conversion Programs** - Does not itself transform legacy record layouts into modern domain models, though it can govern services that do
- **Batch Scheduling** - Does not replace enterprise schedulers or mainframe job control

**Network & Regulatory Determinations:**
- **Network Classification Decisions** - Does not decide which systems may connect across trust boundaries; it enforces approved policy, which is where that's done.
- **Regulatory Interpretation** - Provides technical controls and evidence, but does not replace legal, compliance, or regulatory interpretation
- **Unidirectional Transfer Appliances** - Does not itself implement data diodes or one-way gateways, but can govern APIs on either side of those boundaries

### Integration Model

The platform works **in conjunction with** utility systems and segmented environments:

<pre class="mermaid">
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#e8f4f8','primaryTextColor':'#000','primaryBorderColor':'#000','lineColor':'#333'}}}%%
flowchart LR
    Portal[Developer Portal / Registry] --> GW[API Gateway]
    Portal --> AUD[Auditor]

    Corp[Corporate IT Apps] -->|Approved Calls| GW
    GW -->|Governed Access| Int[Integration Services / Facades]
    Int --> MF[Mainframe Systems]
    Int --> Bus[Message Bus / Event Backbone]
    Bus --> Ops[Restricted Utility / OT-Adjacent Systems]
    GW -->|Logs + Metrics| AUD

    style Portal fill:#D0EED0
    style GW fill:#D0EED0
    style AUD fill:#FFE6CC
    style Int fill:#E0D0E0
    style MF fill:#FFD0D0
    style Bus fill:#E0D0E0
    style Ops fill:#FFD0D0
</pre>

The Registry tracks the interfaces, the Gateway enforces the approved connectivity model, and the Auditor provides the evidence trail and operational telemetry. Mainframe modernization teams expose selected capabilities through governed facades, events, or integration services rather than by allowing uncontrolled direct dependencies.

---

## Utility-Specific Governance Concerns

### Segmented Networks and Internet Restrictions

In regulated utility environments, some systems must remain off the public Internet and reachable only through tightly controlled private connectivity. The platform should therefore be deployed to support:

- **On-premises or private-cloud operation** with no dependency on public Internet services for runtime enforcement
- **Separate deployment zones** for corporate, regulated, and restricted environments
- **Controlled synchronization** of metadata, policies, and audit summaries between zones where policy allows it
- **Gateway-only connectivity** for approved system-to-system access across boundaries
- **Environment-specific subscriptions** so approval in development or enterprise IT does not imply approval in production or restricted zones

### Mainframe Modernization as a Governance Problem

For utilities, modernization is not only a rewrite program. It is an interface management problem:

- Legacy capabilities often live in **CICS transactions, IMS services, MQ flows, batch outputs, and DB2-backed applications**
- Multiple teams want those capabilities, but each new point-to-point integration increases fragility
- The modernization team needs a way to expose legacy functionality incrementally without creating another generation of opaque dependencies

API governance addresses this by making each exposed capability a managed product:

- **Registry** records the business capability, system of record, owner, consumers, and lifecycle state
- **Gateway** ensures consumption happens only through approved interfaces
- **Auditor** shows which consumers still depend on legacy contracts, how those interfaces are performing, and where modernization risk remains, which supports sequenced modernization rather than risky cutovers

### Observability in a Utility Context

In this environment, observability is not only a performance concern. It is part of operational control.

- Teams need to know **which applications depend on a mainframe-backed interface before making a change**
- Incident responders need to know **whether a failure is isolated, cascading, or crossing trust boundaries**
- Security and compliance teams need to know **which restricted interfaces are being accessed and whether the traffic matches approved patterns**
- Modernization teams need to know **which consumers are still on legacy contracts, which ones are error-prone, and which cutovers are actually safe**

Without this visibility, modernization becomes guesswork and segmented environments become harder to operate safely.

---

## Compliance and Control Alignment

This example is written for a North American power utility, where cybersecurity and audit expectations often span **NERC CIP**, internal cyber policy, SOX-relevant financial controls, privacy obligations, and state utility commission scrutiny.

### Cybersecurity & Network Segmentation

*API Governance Implementation:*
- All interfaces are assigned a **trust-zone classification** such as corporate, regulated operations, restricted, or external partner
- Gateway policies enforce which zones may communicate and under what protocols, identities, and approval workflows
- Direct system-to-system access outside approved patterns is treated as a policy violation and surfaced in dashboards
- APIs exposing sensitive operational or customer data require stronger authentication, narrower subscription scopes, and enhanced monitoring
- Deployment architecture supports private PKI, internal DNS, and fully private connectivity

*Evidence for Auditors:*
- Registry exports showing interface classifications and ownership
- Network diagrams showing gateway-mediated trust boundaries
- Access approval records proving least-privilege subscriptions
- Policy violation reports showing attempted or detected non-approved access paths

### Asset Ownership and Change Control

*API Governance Implementation:*
- Every API, event stream, and integration facade has a named owner, support team, and escalation path
- Version changes, publication, deprecation, and retirement require auditable workflow steps
- Production subscriptions require business justification, volume estimates, and named approvers
- Mainframe modernization interfaces carry lineage metadata linking modern contracts to legacy systems of record

*Evidence for Auditors:*
- Registry records showing owners, lifecycle state, and consumer lists
- Approval logs for version publication and production subscription requests
- Change history linking interface updates to tickets, code changes, and approvers

### Operational Resilience and Incident Response

*API Governance Implementation:*
- Dependency mapping shows which downstream systems rely on each legacy or modernized interface
- Auditor tracks latency, error rates, and call volumes by consumer so incident teams can assess blast radius quickly
- Restricted interfaces can have tighter rate limits and circuit-breaker behavior to protect fragile legacy systems
- Deprecation dashboards reveal which consumers still block retirement of brittle interfaces

*Evidence for Auditors:*
- Dependency graphs and consumer impact reports
- Incident dashboards showing affected subscriptions and systems
- Historical usage reports for legacy interfaces under modernization

### Customer Data, Billing, and Financial Controls

*API Governance Implementation:*
- Billing, metering, outage, and customer-service APIs carry data classification metadata
- Subscription approval requires a documented purpose for access to customer or billing data
- Gateway logs all access to sensitive endpoints with consumer identity, purpose, and outcome
- Retention and audit policies support reconciliation, dispute investigation, and internal controls

*Evidence for Auditors:*
- Data classification records for APIs handling customer, billing, or metering data
- Access logs tied to subscription IDs and usage justifications
- Reports showing who accessed financially relevant interfaces and when

---

## Mainframe Modernization Patterns

### Pattern 1: Governed Facade

A modernization team exposes a COBOL/CICS capability through a service facade. The platform governs:

- Which consumer applications may call it
- Which version they are allowed to use
- What rate limits protect the backend
- Which downstream consumers still rely on the legacy contract

This lets the utility stabilize consumption before rewriting the underlying system.

### Pattern 2: Event Publication from Legacy Systems

Rather than letting consumers poll legacy stores directly, the modernization team publishes events derived from mainframe updates:

- Registry documents the event contract and owner
- Subscriptions approve which consumers receive it
- Auditor tracks which systems depend on the event stream
- Migration can then shift consumers from batch extracts to governed event contracts

### Pattern 3: Strangler Migration with Parallel Versions

As a mainframe capability is reimplemented:

- `v1` continues to route to the legacy backend
- `v2` routes to a modern service
- Subscriptions determine which consumers are approved for each version
- Auditor shows remaining `v1` traffic, making retirement evidence-based rather than political

---

## Example Utility Scenario

Consider a utility modernizing its customer and outage ecosystem:

- The legacy billing platform remains the system of record
- A modernization team builds a API facade over billing and account functions
- Apps want access: an outage communications application, a customer web portal, and a field-service platform
- Some operational environments are in restricted network zones and cannot accept direct Internet-connected dependencies

With API governance in place:

- The facade is registered with ownership, lifecycle state, sensitivity, and dependency metadata
- Each consuming system requests a subscription with explicit purpose and environment scope
- The Gateway enforces approved access paths and blocks unauthorized reuse
- The Auditor shows which consumers depend on billing-backed interfaces before the modernization team changes data models or retires batch feeds

This gives the utility a controlled way to connect legacy, modern, and regulated systems without waiting for a full mainframe rewrite.

---

## Why This Matters for Utilities

For a regulated utility, API governance is not only a developer experience investment. It is a way to:

- modernize incrementally instead of through a rewrite-first program
- keep segmented and non-Internet-connected systems under explicit control
- give the mainframe modernization team a disciplined publication and migration model
- create audit-ready evidence for security, compliance, and operational review
- reduce fragile point-to-point integration as more teams need the same core capabilities

In this environment, governance is what allows modernization to proceed without losing control of risk.

---

[Back to Technical Design](technical-design.md)
