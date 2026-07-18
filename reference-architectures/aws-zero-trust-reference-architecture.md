title: AWS Zero Trust Reference Architecture
version: 1.0
status: Stable
owner: Leandro Santos
last_updated: 2026-07-18
cloud: AWS
category: Security

# Title

AWS Zero Trust Reference Architecture

## Problem

Agentic AI applications operating in regulated environments must establish strong identity boundaries between users, AI agents, workloads, and cloud resources. Traditional perimeter-based security models are insufficient because AI agents dynamically access multiple systems, invoke tools, and exchange sensitive information.

This reference architecture demonstrates how AWS security services can be combined to implement Zero Trust principles, enforce least-privilege access, provide end-to-end auditability, and reduce the attack surface from the beginning of the project.

## When to use it

Use this reference architecture when designing AI or agentic applications that require:

- Strong identity and authentication
- Least-privilege authorization
- End-to-end auditability
- Separation of duties
- Compliance with regulated environments
- Secure communication between workloads and cloud services

## When NOT to use it

This reference architecture is not intended for:

- Small prototypes or proof-of-concept projects where security requirements are minimal.
- Deployments running primarily outside the AWS ecosystem.
- Applications that do not require strict identity, authorization, or compliance controls.

The architectural principles remain applicable, but the AWS-specific implementation should be adapted for other cloud providers.

## Core concepts

- Verify explicitly
- Least privilege
- Assume breach
- Workload identity
- Short-lived credentials
- Identity federation
- Network segmentation
- Continuous authorization
- Comprehensive audit logging

## Advantages

- Reduces the attack surface.
- Eliminates long-lived credentials.
- Improves regulatory compliance.
- Enables complete audit trails.
- Simplifies security reviews.
- Supports secure multi-agent architectures.

## Trade-offs

- Higher operational complexity.
- Additional IAM design effort.
- Increased monitoring requirements.
- More components to manage.
- Longer implementation time during initial development.

## Security considerations

- Enforce MFA for human identities.
- Use IAM Roles instead of static credentials.
- Prefer workload identities over secrets.
- Encrypt data at rest and in transit.
- Enable CloudTrail organization-wide.
- Rotate KMS keys according to policy.
- Restrict network access using private endpoints where possible.

## Related patterns

- Identity Federation
- Workload Identity
- Least Privilege
- Defense in Depth
- Agent Interoperability
- Secure Secret Management

## References

AWS Well-Architected Security Pillar
AWS Zero Trust Whitepaper
NIST SP 800-207 (Zero Trust Architecture)
NIST AI RMF
OWASP Top 10 for LLM Applications (where relevant)
ISO 42001

## Assumptions

- AWS Organizations is available.
- IAM Identity Center is used for workforce identities.
- Agent workloads run on EKS, ECS or Lambda.
- CloudTrail and centralized logging are enabled.

# Identity-First Zero Trust Pattern for Agentic Workloads on AWS


## Status

Draft architectural pattern.

This document defines an initial security architecture for agentic systems operating in highly regulated environments such as banks, insurers, payment processors, and financial institutions.

The pattern is intentionally incomplete in some implementation areas. Its purpose is to establish the core security principles, trust boundaries, and architectural direction before selecting all supporting technologies.

---

## Context

Agentic systems can autonomously select tools, invoke APIs, retrieve sensitive information, and initiate business operations.

Unlike traditional applications, an agent may determine which operation to execute based on natural-language instructions, retrieved content, model reasoning, and tool availability. This creates additional risks, including:

* Prompt injection influencing tool selection.
* Agents invoking APIs outside the intended business context.
* Excessive permissions assigned to shared MCP servers.
* Loss of the initiating user identity across service boundaries.
* Inability to distinguish legitimate automation from compromised agent behaviour.
* Sensitive data being returned to an agent unnecessarily.
* Authorized workloads performing unauthorized business actions.

Traditional perimeter-based security is insufficient for this model.

The architecture must assume that no agent, network location, tool, MCP server, or downstream service is implicitly trusted.

---

## Architectural Decision

All calls from MCP servers to protected business APIs must use temporary AWS workload credentials and AWS Signature Version 4 authentication.

Each request must be independently authenticated and authorized.

SigV4 and IAM will provide the workload identity and service-level authorization foundation. Additional controls will enforce business authorization, agent permissions, user identity, data access, approval requirements, and runtime safeguards.

The core principle is:

> Every tool invocation must have a verifiable workload identity, an authorized business purpose, minimum necessary privileges, and a complete audit trail.

---

## Security Objectives

The architecture should provide:

1. Strong workload identity for every MCP server and agent runtime.
2. No long-lived access keys or shared API credentials.
3. Least-privilege authorization at the API, method, and resource level.
4. Separation between workload identity and initiating business identity.
5. Explicit authorization for each tool invocation.
6. Strong tenant, customer, and account isolation.
7. Protection against prompt injection and confused-deputy scenarios.
8. Human approval for high-impact operations.
9. Complete traceability from user request to downstream business action.
10. Minimum necessary data disclosure to agents and language models.
11. Rapid revocation and containment of compromised workloads.
12. Centralized monitoring, detection, and forensic evidence.

---

## High-Level Architecture

```text
User or Business System
          │
          │ authenticated request
          ▼
Agent Orchestration Layer
          │
          │ establishes user, tenant,
          │ agent and conversation context
          ▼
Agent Runtime
          │
          │ permitted tool selection
          ▼
MCP Server
          │
          │ temporary IAM credentials
          │ SigV4-signed request
          ▼
API Gateway / VPC Lattice / Private Service Endpoint
          │
          ├── workload identity validation
          ├── IAM authorization
          ├── resource policy enforcement
          ├── trusted context validation
          ├── business authorization
          ├── approval validation
          └── request logging
          ▼
Business Service
          │
          ├── domain validation
          ├── tenant isolation
          ├── data filtering
          └── transaction controls
          ▼
Protected Data or Transaction System
```

---

## Identity Model

The system must distinguish between several identities.

### Workload Identity

The AWS identity used by the calling service.

Examples:

* ECS task role.
* EKS Pod Identity role.
* Lambda execution role.
* EC2 instance profile.
* Assumed IAM role using AWS STS.

Example:

```text
arn:aws:iam::123456789012:role/CustomerLookupMcpRole
```

The workload identity answers:

> Which technical workload made this API request?

---

### Agent Identity

The logical agent that selected and initiated the tool.

Examples:

```text
customer-support-agent
fraud-investigation-agent
treasury-assistant
document-analysis-agent
```

The agent identity answers:

> Which agent was responsible for choosing this operation?

Agent identity must not be accepted directly from model-generated content. It must be established by the trusted orchestration layer.

---

### Business Identity

The authenticated human or business process on whose behalf the agent is acting.

Examples:

* Bank employee.
* Customer.
* Service account.
* Scheduled business workflow.
* Operations team member.

The business identity answers:

> Who requested or authorized the action?

---

### Tenant and Domain Identity

The business boundary within which the action is permitted.

Examples:

* Customer ID.
* Legal entity.
* Business unit.
* Account portfolio.
* Geographic jurisdiction.
* Banking tenant.

This identity answers:

> Within which business boundary may the action operate?

---

## Identity Propagation

Every protected tool invocation should include trusted execution context.

Example:

```json
{
  "userId": "employee-18427",
  "tenantId": "commercial-banking-ca",
  "agentId": "treasury-assistant-v3",
  "agentVersion": "3.4.1",
  "conversationId": "conv-8a71f",
  "toolCallId": "tool-43d20",
  "workflowId": "wire-review-workflow",
  "approvalId": "approval-9821",
  "purpose": "review-payment-status"
}
```

This context must be:

* Created by a trusted orchestration component.
* Cryptographically protected or resolved server-side.
* Validated by the receiving service.
* Immutable from the model’s perspective.
* Included in audit records.
* Limited to non-sensitive identifiers where possible.

Ordinary HTTP headers supplied directly by the agent or language model must not be trusted as authoritative identity information.

---

## Authentication Pattern

Each MCP server must receive temporary AWS credentials through its runtime environment.

The MCP server signs every downstream API request using SigV4.

The receiving service validates:

* AWS access key identity.
* Session token.
* Request signature.
* Signed request components.
* Request timestamp.
* Target AWS service and region.
* IAM permissions.
* Resource policy conditions.

Long-lived access keys must not be stored in:

* Agent prompts.
* Environment files.
* Container images.
* Source repositories.
* MCP configuration files.
* Secrets passed to language models.

---

## Authorization Layers

Authorization must be enforced at multiple layers.

### Layer 1: Workload Authorization

IAM determines whether the MCP server may invoke a specific API operation.

Example:

```json
{
  "Effect": "Allow",
  "Action": "execute-api:Invoke",
  "Resource": [
    "arn:aws:execute-api:ca-central-1:123456789012:abc123/prod/GET/customers/*"
  ]
}
```

This answers:

> May this workload call this route?

---

### Layer 2: Agent Authorization

The orchestration layer or policy engine determines whether the specific agent may use the tool.

Example:

```text
customer-support-agent:
  allowed:
    - get-customer-profile
    - get-case-status
  denied:
    - initiate-wire-transfer
    - change-credit-limit
```

This answers:

> May this agent use this capability?

---

### Layer 3: Business Authorization

The API determines whether the initiating user may perform the requested action.

Examples:

* The employee is authorized for the customer account.
* The customer owns the requested account.
* The user’s role permits the transaction type.
* The action is permitted within the current jurisdiction.
* The requested amount is below the employee’s approval limit.

This answers:

> May this user perform this action on this business resource?

---

### Layer 4: Transaction Authorization

The domain service validates the operation against current business state.

Examples:

* Account is active.
* Transaction has not already been processed.
* Wire transfer destination is approved.
* Refund is within the allowed time window.
* Requested amount is within limits.
* Required dual approval has been completed.

This answers:

> Is this specific operation valid now?

---

## MCP Server Isolation

MCP servers should be separated by business capability and risk level.

Preferred:

```text
CustomerReadMcpRole
CaseManagementMcpRole
FraudInvestigationMcpRole
PaymentReadOnlyMcpRole
PaymentExecutionMcpRole
```

Avoid:

```text
UniversalAgentToolsRole
```

A universal MCP server with broad permissions creates a large blast radius. SigV4 would authenticate it correctly, but a compromised agent or prompt-injected workflow could inherit all of its privileges.

High-risk capabilities should be isolated into dedicated MCP servers with:

* Separate IAM roles.
* Separate network policies.
* Separate deployment pipelines.
* Separate logging.
* Separate approval rules.
* Separate scaling and runtime controls.

---

## Tool Classification

Tools should be classified by impact.

### Read-Only Tools

Examples:

* Retrieve account balance.
* Search internal policies.
* Read transaction status.
* Retrieve customer profile.

Typical controls:

* IAM authorization.
* Business identity validation.
* Data minimization.
* Tenant isolation.
* Sensitive-field masking.
* Rate limits.

---

### Reversible Write Tools

Examples:

* Update a support case.
* Add a note.
* Schedule a callback.
* Create a draft request.

Typical controls:

* All read-only controls.
* Idempotency keys.
* Input schema validation.
* Change logging.
* User confirmation where appropriate.

---

### High-Impact Tools

Examples:

* Transfer funds.
* Change credit limits.
* Freeze an account.
* Approve a loan.
* Modify beneficiary information.
* Export regulated customer data.

Typical controls:

* Dedicated MCP server.
* Narrow IAM role.
* Explicit user confirmation.
* Human approval or dual control.
* Transaction limits.
* Step-up authentication.
* Out-of-band verification.
* Immutable audit records.
* Restricted execution windows.
* Automated fraud or anomaly checks.

The model must not be the final authorization authority for high-impact actions.

---

## Prompt Injection Controls

SigV4 does not prevent an agent from making a correctly authenticated but maliciously influenced request.

The architecture must assume that retrieved documents, emails, web content, user messages, and tool responses may contain hostile instructions.

Required controls include:

* Strict separation of instructions and retrieved data.
* Tool allowlists per agent.
* Structured tool schemas.
* Server-side parameter validation.
* Denial of arbitrary URLs, SQL, file paths, and commands.
* Retrieval source classification.
* Content provenance tracking.
* Maximum data retrieval limits.
* Output filtering.
* High-risk action confirmation.
* Independent policy evaluation outside the model.
* Detection of unusual tool sequences.
* Prevention of tool definitions being modified by model output.

The MCP server must treat all model-generated tool arguments as untrusted input.

---

## Network Architecture

Protected APIs should not be publicly reachable unless there is a documented requirement.

Preferred options include:

* Private API Gateway.
* VPC Lattice with IAM authorization.
* AWS PrivateLink.
* Internal Application Load Balancers.
* VPC endpoints.
* Security groups restricted to known workloads.
* Separate subnets for agent runtime, MCP servers, and business services.

Network authorization should complement identity authorization.

A request should ideally require:

```text
Valid workload identity
AND permitted IAM action
AND approved network path
AND valid business context
AND valid transaction state
```

Network location alone must never be treated as proof of trust.

---

## Confused-Deputy Protection

A shared agent platform may invoke services on behalf of multiple teams, users, or tenants.

Trust policies should use applicable conditions such as:

* `aws:SourceArn`
* `aws:SourceAccount`
* `sts:ExternalId`
* `sts:SourceIdentity`
* Session tags.
* Principal tags.
* Organization ID.
* VPC endpoint conditions.

Role assumption must be constrained to known orchestration services and approved workloads.

A privileged MCP server must not accept arbitrary requests merely because they originate from another internal service.

---

## Data Protection

Agents should receive only the minimum information necessary to complete the task.

Controls should include:

* Encryption in transit.
* Encryption at rest with KMS.
* Separate KMS keys by data classification where appropriate.
* Field-level encryption or tokenization.
* PII masking.
* Account-number truncation.
* Data loss prevention checks.
* Response filtering before returning data to the model.
* Prevention of sensitive values entering prompts or logs.
* Restricted model retention and training settings.
* Regional data residency controls.
* Data-classification tags.
* Purpose-based access controls.

Where possible, business services should return decisions or summaries rather than complete raw records.

Example:

Preferred:

```json
{
  "eligible": false,
  "reasonCode": "ACCOUNT_RESTRICTED"
}
```

Avoid when unnecessary:

```json
{
  "fullCustomerRecord": "...",
  "completeTransactionHistory": "...",
  "identityDocuments": "..."
}
```

---

## Secrets Management

Secrets that cannot be replaced by IAM roles should be stored in AWS Secrets Manager or another approved enterprise secrets platform.

The architecture should support:

* Automatic rotation.
* Scoped access.
* Audit logging.
* Versioning.
* Revocation.
* Runtime retrieval.
* No secret exposure to model context.
* No secret persistence in agent memory.
* No secret inclusion in tool results.

Whenever possible, workload identity should replace shared secrets.

---

## Audit and Traceability

Every agent action should be traceable across the entire execution chain.

Required correlation fields may include:

```text
userId
tenantId
agentId
agentVersion
conversationId
workflowId
toolCallId
approvalId
awsPrincipalArn
awsAccountId
sourceService
destinationService
apiOperation
resourceId
decision
policyVersion
timestamp
requestHash
responseClassification
```

A complete audit trail should answer:

* Who initiated the request?
* Which agent selected the tool?
* Which MCP server executed it?
* Which AWS role signed the request?
* Which API operation was invoked?
* What business resource was accessed?
* Which policy permitted or denied the action?
* Was human approval required?
* What data was returned?
* Did the action alter business state?
* Which model and prompt version were involved?

Logs should be sent to centralized, tamper-resistant storage with retention appropriate to regulatory and legal requirements.

---

## Monitoring and Detection

Security monitoring should detect:

* Unexpected role assumptions.
* Calls to unusual API routes.
* Sudden increases in tool invocation frequency.
* Repeated authorization failures.
* Agent access outside expected business hours.
* Cross-tenant access attempts.
* Large data retrievals.
* New tool sequences.
* Unusual transaction amounts.
* High-risk actions without approval identifiers.
* Requests from unexpected regions, accounts, or VPC endpoints.
* Changes to IAM policies or MCP tool definitions.
* MCP servers returning sensitive data unexpectedly.

Potential AWS services include:

* CloudTrail.
* CloudWatch.
* GuardDuty.
* Security Hub.
* AWS Config.
* IAM Access Analyzer.
* Detective.
* Macie.
* EventBridge.
* Central enterprise SIEM platforms.

---

## Runtime Security

Agent and MCP workloads should be hardened using controls such as:

* Minimal container images.
* Read-only filesystems.
* Non-root execution.
* Signed container images.
* Software composition analysis.
* Vulnerability scanning.
* Runtime threat detection.
* Egress restrictions.
* Admission policies.
* Restricted Linux capabilities.
* Dedicated service accounts.
* Resource limits.
* Patch management.
* Immutable deployments.
* Separation of development, testing, and production accounts.

Production agents should not be able to modify their own tool definitions, IAM permissions, deployment configuration, or policy rules.

---

## Failure and Revocation

The system must support rapid containment.

Required capabilities include:

* Disable an individual agent.
* Remove a tool from an agent allowlist.
* Revoke an MCP server role.
* Deny an API route through a resource policy.
* Disable a high-risk workflow.
* Block a tenant or user session.
* Rotate or revoke downstream credentials.
* Quarantine a compromised container or pod.
* Apply emergency organization-level SCPs.
* Preserve evidence for investigation.

Authorization should fail closed when:

* Identity context is missing.
* Approval data cannot be verified.
* Policy services are unavailable.
* Tenant information is inconsistent.
* Tool arguments fail validation.
* Agent identity is unknown.
* Audit logging cannot be completed for high-risk operations.

---

## Example Authorization Decision

A wire-transfer status agent requests:

```text
GET /payments/transfer/84291
```

The request is permitted only when:

```text
The MCP server presents valid temporary AWS credentials
AND the request has a valid SigV4 signature
AND IAM permits GET on the transfer-status route
AND the agent is authorized to use the payment-status tool
AND the initiating employee may access the customer account
AND the transfer belongs to the employee's permitted business unit
AND the request is associated with an active user session
AND the response is filtered to exclude unnecessary sensitive information
AND the complete invocation is recorded in the audit trail
```

A request to execute a transfer requires additional controls:

```text
All status-query controls
AND explicit customer or employee confirmation
AND transaction amount validation
AND beneficiary validation
AND fraud screening
AND required approval workflow
AND idempotency protection
AND transaction-level audit evidence
```

---

## Proposed AWS Building Blocks

Potential services include:

```text
Agent runtime:
- Amazon Bedrock Agents or custom orchestration
- Amazon ECS
- Amazon EKS
- AWS Lambda

Workload identity:
- IAM roles
- AWS STS
- EKS Pod Identity
- ECS task roles

API protection:
- Amazon API Gateway
- Amazon VPC Lattice
- AWS PrivateLink
- AWS WAF where publicly exposed

Fine-grained authorization:
- IAM
- Amazon Verified Permissions
- Application policy engine
- Domain-specific authorization services

Secrets and encryption:
- AWS Secrets Manager
- AWS KMS
- AWS Certificate Manager

Audit and security:
- AWS CloudTrail
- Amazon CloudWatch
- Amazon GuardDuty
- AWS Security Hub
- AWS Config
- IAM Access Analyzer
- Amazon Macie

Governance:
- AWS Organizations
- Service Control Policies
- Permission boundaries
- Separate AWS accounts
- Infrastructure as code
```

The final service selection will depend on latency, scale, regulatory requirements, deployment model, and the institution's existing control environment.

---

## Consequences

### Positive Consequences

* Strong AWS-native workload authentication.
* No need to invent a custom service authentication scheme.
* Temporary credentials reduce long-lived-secret risk.
* Fine-grained service authorization.
* Better accountability across agent actions.
* Reduced blast radius through dedicated MCP identities.
* Improved regulatory and audit evidence.
* Easier central revocation and policy enforcement.
* Compatible with private AWS networking patterns.

### Negative Consequences

* Increased implementation complexity.
* Additional identity context must be propagated securely.
* More IAM roles and policies must be governed.
* Fine-grained authorization may add latency.
* Approval workflows can reduce agent autonomy.
* Audit storage volumes may be significant.
* Policy design and testing require dedicated effort.
* Troubleshooting multi-layer authorization failures may be difficult.
* Shared MCP server designs may require restructuring.

---

## Risks and Open Questions

The following areas require further design:

1. How will trusted user and tenant identity be propagated?
2. Will the orchestration layer issue signed context tokens or use role sessions?
3. Will permissions be assigned per MCP server, per agent, or per individual workflow?
4. Which service will perform fine-grained business authorization?
5. How will human approvals be represented and verified?
6. How will approval replay be prevented?
7. How will prompt injection attempts be detected and investigated?
8. Which tool actions require step-up authentication?
9. How will sensitive tool responses be filtered before entering model context?
10. What are the permitted model providers and deployment regions?
11. What information may be retained in conversation history or agent memory?
12. How will emergency agent shutdown be implemented?
13. How will policy changes be tested before production deployment?
14. What audit evidence is required by internal risk, compliance, and regulators?
15. How will agent and tool versions be associated with each transaction?
16. Should high-risk MCP servers run in separate AWS accounts?
17. Is VPC Lattice, API Gateway, or a service mesh the preferred enforcement point?
18. What are the acceptable latency and availability impacts of authorization checks?
19. How will downstream non-AWS systems validate workload identity?
20. Which actions must always remain human-controlled?

---

## Recommended Initial Implementation

A practical first iteration could use:

```text
Agent runtime:
Amazon ECS or Amazon EKS

MCP workload identity:
Dedicated IAM role per MCP server

API exposure:
Private API Gateway or VPC Lattice

Authentication:
SigV4 on every request

Service authorization:
Least-privilege IAM policies

Business authorization:
Application-level policy service

Context:
Trusted signed execution context created by the orchestrator

High-risk actions:
Human approval and idempotency enforcement

Secrets:
AWS Secrets Manager

Encryption:
AWS KMS

Audit:
CloudTrail, structured application logs and centralized SIEM

Governance:
Separate production AWS account, SCPs and permission boundaries
```

This creates a strong foundation without attempting to solve every advanced policy concern in the first release.

---

## Summary

SigV4 authentication with IAM roles is the foundational workload identity mechanism for this pattern.

It provides strong cryptographic authentication and AWS-native service authorization, but it must be combined with:

* Agent-specific tool permissions.
* Trusted business identity propagation.
* Fine-grained domain authorization.
* Prompt injection controls.
* Data minimization.
* Private networking.
* High-risk approval workflows.
* Comprehensive audit logging.
* Runtime and organizational governance.

The intended result is not simply an authenticated agent system.

The intended result is an agent system in which every action is attributable, explicitly authorized, narrowly scoped, continuously evaluated, and independently auditable.
