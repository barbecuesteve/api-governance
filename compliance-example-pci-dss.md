# Compliance Framework Example: Payment Processing (PCI-DSS)

**Context:** An internet payment processor (similar to Stripe) must comply with PCI-DSS, SOC 2, GDPR, and various regional financial regulations. The API governance platform becomes a critical compliance control point.

## PCI-DSS Specific Requirements

### Requirement 1 & 2: Network Security & Secure Configurations

*API Governance Implementation:*
- All APIs handling cardholder data (CHD) designated as "Highly Restricted" classification
- Network segmentation enforced: payment APIs isolated in PCI-compliant zones, only accessible via Gateway
- Gateway deployed in DMZ with strict firewall rules: inbound only from authenticated services, outbound only to PCI backend
- Service configurations managed as code, immutable infrastructure prevents configuration drift
- Quarterly vulnerability scans of Gateway and payment APIs, automated patching for critical CVEs

*Evidence for Auditors:*
- Network diagrams generated from infrastructure-as-code showing segmentation
- Firewall rules exported from cloud providers showing Gateway-only access
- Scan reports from approved scanning vendors (ASV)

### Requirement 3 & 4: Protect Cardholder Data & Encrypt Transmission

*API Governance Implementation:*
- Cardholder data never stored in logs; Gateway automatically redacts card numbers, CVVs from all logs
- Tokenization services integrated with Gateway: raw CHD converted to tokens before reaching most backend services
- Only PCI-compliant vault services handle raw CHD; all others work with tokens
- TLS 1.3 enforced for all cardholder data transmission
- mTLS required for services directly accessing vault or payment processor integrations
- Key rotation managed centrally by platform team; services don't handle encryption keys

*Evidence for Auditors:*
- Data flow diagrams showing tokenization at Gateway ingress
- Encryption configuration exports showing TLS versions, cipher suites
- Key rotation logs from secrets management system
- Sample logs demonstrating redaction of CHD

### Requirement 6: Secure Development & Vulnerability Management

*API Governance Implementation:*
- API Review Panel includes security specialist who reviews all payment-related APIs
- Automated SAST/DAST scanning in CI/CD pipeline for all API changes
- Breaking changes to payment APIs require cryptographic signature of approval from security team
- API specifications scanned for PCI-DSS anti-patterns: CHD in query params, GET requests, unencrypted storage
- Immutable deployment pipeline: no manual changes in production; all changes via reviewed, tested deployments

*Evidence for Auditors:*
- Review records from API Review Panel with security sign-offs
- SAST/DAST scan results for payment APIs
- CI/CD pipeline configurations showing security gates
- Change logs showing all deployments with approval audit trails

### Requirement 7 & 8: Access Control & Authentication

*API Governance Implementation:*
- Subscription-based access enforces need-to-know: only payment processing services can access CHD APIs
- Role-based access control (RBAC) layered on subscriptions: different endpoints require different roles
- Multi-factor authentication required for any human access to production payment APIs (even read-only)
- Service accounts use certificate-based authentication, not passwords
- Credential rotation enforced: OAuth tokens expire within 1 hour, refresh tokens within 24 hours
- Principle of least privilege: subscriptions scoped to minimum necessary endpoints and operations

*Evidence for Auditors:*
- Registry exports showing subscription scoping and role assignments
- Authentication policy configurations from identity provider
- Certificate lifecycle management logs
- Access reviews showing periodic validation of who has access to what

### Requirement 9: Physical Security

*API Governance Implementation:*
- Gateway and platform services deployed in PCI-compliant cloud regions or colocations
- API metadata and subscription data encrypted at rest with FIPS 140-2 validated encryption modules
- Physical access to platform infrastructure controlled by cloud provider SOC reports

*Evidence for Auditors:*
- Cloud provider attestations (SOC 2 Type II, PCI-DSS AOC)
- Encryption configuration showing FIPS compliance

### Requirement 10: Logging & Monitoring

*API Governance Implementation:*
- **Centralized, tamper-proof audit logs** capturing all CHD access:
  - Every request to payment APIs logged with full metadata (who, what, when, where, why)
  - User identity tracked: which merchant, which end-customer, which employee for admin actions
  - Field-level access logging: which specific payment methods or transactions were accessed
  - Logs written to immutable storage with cryptographic integrity protection
- **Real-time monitoring & alerting:**
  - Security Information and Event Management (SIEM) integration consumes Gateway logs
  - Alerts on suspicious patterns: unusual access times, volume anomalies, failed auth attempts
  - Daily automated reports summarize CHD access by user, service, purpose
- **Log retention:** 1 year online, 7 years archived (exceeds PCI-DSS minimum of 1 year)
- **Time synchronization:** All Gateway and Auditor components sync with NTP; timestamps in logs auditable

*Evidence for Auditors:*
- Audit log samples showing required fields (user ID, timestamp, action, resource, outcome)
- SIEM configuration showing alerting rules for security events
- Log retention policy documentation and storage verification
- NTP configuration and drift monitoring

### Requirement 11: Security Testing

*API Governance Implementation:*
- Quarterly penetration testing of Gateway and payment APIs by external firm
- Automated vulnerability scanning weekly; critical/high findings must be remediated within 30 days
- Bug bounty program encourages responsible disclosure of API security issues
- Gateway bypass testing: quarterly internal tests attempt direct API calls to verify network segmentation

*Evidence for Auditors:*
- Penetration test reports with findings and remediation proof
- Vulnerability scan reports showing scanning frequency and remediation timelines
- Bug bounty program documentation and resolved issues

### Requirement 12: Information Security Policy

*API Governance Implementation:*
- API security standards documented and versioned in platform wiki/handbook
- Annual security awareness training for all engineers covers API security best practices
- Incident response playbook includes API-specific scenarios (credential leak, data breach via API)
- Quarterly risk assessments of API ecosystem identify high-risk APIs for additional controls

*Evidence for Auditors:*
- API security standards documentation
- Training completion records for engineering staff
- Incident response procedures and test exercise results
- Risk assessment reports and risk register

---

## GDPR Compliance Through API Governance

**Right to Access (Article 15):**
- Special `/privacy/user/{id}/data` API consolidated view of personal data across all internal APIs
- Audit logs track which APIs were queried to fulfill access request
- Response includes data sources, processing purposes, retention periods

**Right to Erasure / Right to be Forgotten (Article 17):**
- Subscription metadata tracks which APIs process personal data for which purposes
- Data deletion request triggers automated calls to all subscribed APIs handling user's data
- Audit trail proves deletion completed across all systems within mandated timeframe (typically 30 days)
- Tombstone records prevent re-creation of deleted data

**Data Minimization (Article 5):**
- API specifications reviewed for necessity: does this endpoint need to return full data or can it be scoped?
- Gateway enforces field filtering: consumers only receive fields relevant to their purpose
- Review panel challenges data models that expose excessive personal data

**Purpose Limitation (Article 5):**
- Subscription requests require usage_description explaining why consumer needs data
- Gateway logs include purpose of access from subscription metadata
- Auditor can generate reports: "Which services accessed customer data for marketing purposes?"

**Data Portability (Article 20):**
- Standardized export formats (JSON, CSV) for personal data enforced through API standards
- Export APIs required for all services handling significant personal data
- Machine-readable format makes portability to competitors feasible

**Breach Notification (Article 33-34):**
- Security monitoring detects potential breaches (unusual data access, exfiltration patterns)
- Audit logs provide evidence for breach scope: exactly which records were accessed, by whom, when
- Automated breach report generation from Auditor data: affected individuals, data types exposed, timeline
- 72-hour notification deadline trackable through incident timestamps in audit logs

---

## SOC 2 Type II Compliance

**Security (Trust Service Criteria):**
- Access controls (CC6.1): Subscription-based authorization, least privilege enforced by Gateway
- Logical access (CC6.2): Authentication, MFA, credential lifecycle management via platform
- Security monitoring (CC7.2): Real-time SIEM integration, anomaly detection, incident response

**Availability (Trust Service Criteria):**
- SLO monitoring (A1.2): Auditor tracks API availability per SLO commitments
- Capacity management (A1.3): Consumer-level metrics enable capacity planning and scaling

**Processing Integrity (Trust Service Criteria):**
- Data validation (PI1.4): API specifications define data contracts; Gateway validates inbound requests
- Error handling (PI1.5): Standardized error responses, consumer-specific error tracking for diagnosis

**Confidentiality (Trust Service Criteria):**
- Data classification (C1.1): APIs declare sensitivity level, controls enforced accordingly
- Encryption (C1.2): TLS for all, mTLS for confidential data, field-level encryption for highly sensitive data

**Privacy (Trust Service Criteria):**
- Consent management (P3.2): Subscription approval can require end-user consent validation
- Data disposal (P5.2): Retention policies enforced, deletion requests tracked through audit logs

---

## Regional Financial Regulations

**Open Banking (PSD2 in Europe, similar in UK, Australia):**
- Third-party provider (TPP) access to customer financial data via APIs
- Strong Customer Authentication (SCA) enforced at Gateway for payment initiation
- TPP registration and certification tracked in Registry; only certified TPPs get subscriptions
- Audit logs prove consent flow: customer authorized TPP access, which data was shared, when consent expires

**Anti-Money Laundering (AML) & Know Your Customer (KYC):**
- Transaction monitoring APIs integrate with AML engines
- Audit logs provide transaction reconstruction for regulatory investigations
- API access controls ensure only authorized fraud/AML teams can query sensitive transaction data
- Immutable logs prevent tampering with evidence needed for compliance or law enforcement

**California Consumer Privacy Act (CCPA):**
- Similar to GDPR: access, deletion, opt-out rights
- API governance platform provides same mechanisms as GDPR compliance
- Audit logs track California residents' data access separately for CCPA-specific reporting

---

[Back to Technical Design](technical-design.md)
