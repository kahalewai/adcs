# Agent Delegation Chain Standard (ADCS)

**Status:** Proposed Standard  
**Version:** 1.0.0  
**Date:** April 11, 2026  
**License:** Apache 2.0  

---

*Begin Agent Delegation Chain Standard*

## Abstract

The Agent Delegation Chain Standard (ADCS) defines a normative data model, verification algorithm, and cryptographic binding specification for the authorization of scope across multi-agent, multi-protocol AI workflows. ADCS establishes how permissions and constraints flow from an originating principal through a chain of agent delegations, and how each hop in that chain is verified to be monotonically restrictive — meaning scope can only narrow, never expand.

ADCS addresses the authorization gap between session-level identity (OAuth 2.1) and session-level circuit breaking (ALS). It answers the question: **given that this agent session is permitted to run, is this specific agent authorized to perform this specific action within the scope that was delegated to it?**

Principals are identified using Decentralized Identifiers (DIDs), anchoring public keys in externally resolvable identity documents and decoupling key rotation from chain structure. Chains support both linear delegation sequences and directed acyclic graph (DAG) workflows with defined merge semantics. Originator declarations may express multi-party quorum authorization for regulated-industry use cases. Conditional permissions, continuous revocation signals via CAEP, and just-in-time permission escalation are supported. ADCS additionally defines a normative audit event taxonomy aligned with the AI Logging Standard (AILS) so that ADCS audit events are natively interoperable with AILS-aware dashboards, SIEMs, and log pipelines; AILS conformance is declarable but not required.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Normative Language](#2-normative-language)
3. [Scope](#3-scope)
4. [Core Concepts](#4-core-concepts)
5. [Delegation Object](#5-delegation-object)
6. [Delegation Chain Structure](#6-delegation-chain-structure)
7. [Capability Token Binding](#7-capability-token-binding)
8. [Originator Declaration](#8-originator-declaration)
9. [Constraint Type Registry](#9-constraint-type-registry)
10. [Verification Algorithm](#10-verification-algorithm)
11. [Chain Compaction](#11-chain-compaction)
12. [Cross-Protocol Chains](#12-cross-protocol-chains)
13. [Cryptographic Requirements](#13-cryptographic-requirements)
14. [Revocation](#14-revocation)
15. [Error Codes](#15-error-codes)
16. [Security Considerations](#16-security-considerations)
17. [Conformance](#17-conformance)
18. [Relationship to Other Standards](#18-relationship-to-other-standards)
19. [Versioning](#19-versioning)
20. [Glossary](#20-glossary)
21. [References](#21-references)
22. [Deployment Topologies](#22-deployment-topologies)
23. [Audit Event Emission](#23-audit-event-emission)
- Appendix A — Core Permission Vocabulary
- Appendix B — Example: Complete Delegation Chain
- Appendix C — Agent Identity Lifecycle
- Appendix D — Conformance Test Vectors
- Appendix E — Reference SDK Architecture
- Appendix F — Implementer Quick Reference

---

## 1. Introduction

### 1.1 The Problem

AI agent systems increasingly operate as networks of autonomous agents that delegate tasks to one another. A human authorizes an action. An orchestrating agent interprets that authorization and delegates subtasks to specialist agents. Those agents invoke tools. At each step, authorization scope must travel with the action — and at no step may scope expand beyond what was originally granted.

Current agent frameworks do not provide a standard mechanism for this. Each framework manages delegation ad hoc, typically through implicit context passing or application-layer trust assumptions. This creates a class of vulnerabilities:

- A sub-agent is granted broader permissions than its delegator possesses
- Constraints imposed by an originating human are not enforced at the tool invocation layer
- A long delegation chain becomes opaque — there is no way to verify that each hop honored the scope it received
- Cross-protocol delegations (from an A2A task to an MCP tool) carry no verifiable authorization record
- Key rotation or agent decommissioning leaves orphaned credentials with residual access

### 1.2 What ADCS Provides

ADCS provides:

- A **normative data structure** for expressing a delegation, including what was granted, what was withheld, and what constraints were added
- A **chain structure** that links delegations from originator to terminal action, supporting both linear sequences and directed acyclic graph (DAG) workflows
- A **verification algorithm** that any conformant implementation can execute to determine whether a chain is valid
- A **monotonic restriction invariant** enforced cryptographically at each hop
- A **capability token binding** that makes the chain the bearer artifact for tool authorization
- A **constraint type registry** that establishes the shared vocabulary for what constraints mean
- **DID-based principal identification** anchoring public keys in externally resolvable identity documents
- **Multi-party originator quorum** for regulated-industry dual-control requirements
- **Continuous revocation signals** via CAEP for real-time authorization change propagation
- **Permission escalation protocol** for agents that discover mid-task they require additional scope

### 1.3 What ADCS Does Not Provide

ADCS does not specify:

- Session-level circuit breaking — that is the responsibility of ALS
- Tool integrity verification — that is the responsibility of the MCP Integrity Standard or equivalent
- Policy evaluation engines (OPA, Cedar, etc.) — these MAY be used to evaluate chains but are not specified here
- The semantic mapping between abstract constraints and concrete action parameters — this is an open problem documented in Section 16.5
- Identity provisioning or authentication — ADCS relies on externally established identities anchored in DID documents

> **⚠ WARNING — Constraint Enforcement Boundary:** ADCS carries constraints through a delegation chain and verifies that constraints are structurally present and structurally non-loosened at each hop. **ADCS does not enforce that a terminal action's parameters satisfy those constraints.** A chain declaring `budget.max = 1500` provides no monetary enforcement guarantee unless the governance layer implements semantic mapping for the target tool's parameters. Implementations MUST declare their constraint evaluation capability per Section 9.4. Deployers MUST NOT rely on ADCS alone as a semantic enforcement mechanism.

---

## 2. Normative Language

The key words **MUST**, **MUST NOT**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119.

---

## 3. Scope

### 3.1 In Scope

| Capability | Description |
|---|---|
| Delegation Object | Normative structure for a single delegation hop |
| Delegation Chain | Normative structure for an ordered sequence or DAG of delegations from originator to action |
| Originator Declaration | Normative structure for the root of a chain, including optional multi-party quorum |
| Capability Token | Normative token format binding a chain to a specific tool invocation |
| Verification Algorithm | Normative procedure for validating a chain |
| Constraint Type Registry | Standard vocabulary for constraint categories |
| Chain Compaction | Normative procedure for compacting chains exceeding depth thresholds |
| Cross-Protocol Binding | Requirements for chains that span multiple agent protocols |
| Cryptographic Requirements | Signing algorithms, DID-based key requirements, and proof-of-possession binding |
| Error Codes | Normative error taxonomy for chain verification failures |
| Conformance | Levels and test requirements for implementations |
| Revocation Service Metadata | Normative discovery endpoint for revocation infrastructure |
| Permission Escalation Protocol | Normative message format for mid-task permission requests |
| Continuous Revocation Signals | CAEP integration for real-time revocation propagation |

### 3.2 Explicitly Out of Scope

| Capability | Where It Belongs |
|---|---|
| Session halt and circuit breaking | ALS Standard |
| Tool artifact integrity and signing | MCP Integrity Standard or equivalent |
| Policy language (OPA, Cedar, Rego, etc.) | Respective policy engine specifications |
| Semantic constraint-to-parameter mapping | Open problem; see Section 16.5 |
| Identity provisioning | DID method specifications, SPIFFE/SPIRE, OAuth 2.1 |
| Agent runtime implementation | Implementer discretion within conformance requirements |
| Cumulative constraint accounting across chains | Optional Constraint Accounting Service; see Section 9.5 |

---

## 4. Core Concepts

### 4.1 Principal

A **principal** is any entity that can initiate or participate in a delegation. Principals include:

- **Human originators** — the natural person whose intent initiates the chain
- **System originators** — automated systems acting as the root of a chain on behalf of an organization
- **Agents** — AI agent runtimes that receive delegated scope and may further delegate

Every principal in a chain MUST have a Decentralized Identifier (DID) that is resolvable to a DID document containing the principal's current public key(s). ADCS requires that principal identifiers be resolvable DIDs and that signing keys be discoverable via DID document resolution at verification time.

### 4.2 Decentralized Identifiers (DIDs)

A **Decentralized Identifier** (DID) is a globally unique identifier conforming to the W3C DID Core specification. A DID resolves to a **DID document** that contains, among other fields, the public key material associated with that identity.

ADCS requires DID-based principals for the following reasons:

- Key rotation happens at the DID document layer without touching or invalidating existing chain structures
- Verifiers can independently confirm that a signing key legitimately belongs to the declared principal by resolving the DID
- Identity lifecycle (provisioning, rotation, decommissioning) is decoupled from chain construction

**Supported DID methods** (implementations MUST support at least one):

| DID Method | Description | Recommended Use |
|---|---|---|
| `did:web` | DID anchored in a well-known URL at a controlled domain | Enterprise agents and services |
| `did:jwk` | DID derived directly from a JWK; self-contained | Ephemeral/short-lived agents |
| `did:key` | DID derived from a single public key | Minimal deployments, test environments |

Additional DID methods MAY be supported as the DID method ecosystem evolves. Implementations SHOULD be designed to accommodate new DID methods without breaking existing chain verification. Verifiers MUST declare which DID methods they support in their conformance metadata.

**Fast-path optimization:** In closed deployments where external DID resolution is unavailable or undesirable, an OPTIONAL inline `public_key` (JWK) field MAY be included in delegation objects alongside `did`. When `public_key` is present, verifiers MAY use it directly without DID resolution.

When inline keys are used, a `key_provenance` field MUST accompany `public_key`:

| Value | Meaning |
|---|---|
| `did_resolved` | Key was resolved from the DID document at chain construction time and cached inline for verifier convenience. Verifiers MAY re-resolve the DID to confirm |
| `registry_backed` | Key is registered in an organizational Agent Registry (Appendix C.5) but not published via DID document. Verifiers can confirm via registry lookup |
| `self_declared` | Key is self-asserted by the principal. No external confirmation path exists. Appropriate for ephemeral agents in closed deployments |

Verifiers MUST record the `key_provenance` value in audit logs. Governance layers MAY apply different trust policies based on `key_provenance` — for example, requiring `did_resolved` or `registry_backed` for `purchase` or `delete` permissions. There is no normative trust ordering between provenance values; trust policy is a deployment decision.

Open interoperability deployments SHOULD use DID resolution and SHOULD prefer `did_resolved` provenance. Inline keys with any provenance value are a conformant operating mode, not a reduced-trust fallback.

### 4.3 Delegation

A **delegation** is a cryptographically signed record expressing that a delegator has granted a delegate a specific set of permissions, subject to a specific set of constraints, for a bounded time period. A delegation is the atomic unit of scope transfer.

A delegation does not grant capabilities to the delegate. It records that the delegator — who already possesses certain capabilities — has chosen to share a subset of them with the delegate. The delegate cannot receive more than the delegator has.

### 4.4 Delegation Chain

A **delegation chain** is an ordered sequence of delegations from an originator declaration to the terminal action. In the standard linear case, every delegation references exactly one predecessor. In DAG workflows (Section 6.6), a delegation may reference multiple predecessors, subject to defined merge semantics. The chain is valid only if every hop satisfies the monotonic restriction invariant and every signature is verifiable.

### 4.5 Monotonic Restriction Invariant

The **monotonic restriction invariant** is the central security property of ADCS:

> At every hop in a delegation chain, the effective scope available to the delegate MUST be equal to or strictly narrower than the effective scope available to the delegator.

Scope may be narrowed in two ways:

1. **Permission removal** — a permission present at hop N is absent at hop N+1
2. **Constraint addition** — a constraint is added or tightened at hop N+1

No hop may add permissions or loosen constraints. This invariant is enforced by the verification algorithm (Section 10) and is not dependent on the agents' own compliance.

### 4.6 Effective Scope

The **effective scope** at any point in a chain is the intersection of all permissions and the union of all constraints accumulated from the originator to that point. At a DAG merge point, effective permissions are the intersection of all merging branches' effective permissions, and effective constraints are the most restrictive value for each constraint type across all merging branches. Effective scope is computed by the verification algorithm, not declared by the delegate.

### 4.7 Terminal Action

The **terminal action** is the concrete operation that a chain authorizes — a `tools/call` in MCP, a `tasks/send` in A2A, or equivalent in any bound protocol. A chain exists to authorize exactly one terminal action context. It MUST NOT be reused across different action invocations.

### 4.8 Originator Trust Tier

The **originator trust tier** expresses the degree to which the originator declaration's signature cryptographically proves the specific human's authorization, as distinct from the host system that holds the signing key on their behalf. Three tiers are defined:

| Tier | Value | Description |
|---|---|---|
| Hardware-bound | `hardware_bound` | Key held in TPM, HSM, or secure enclave; user gesture (biometric or PIN) required to sign |
| Host-mediated | `host_mediated` | Host application holds key on behalf of the authenticated user; user authenticated but signature is not a direct cryptographic user act |
| Policy-derived | `policy_derived` | System-generated originator declaration; no interactive human consent; governed by organizational policy |

Governance systems SHOULD apply different trust weights based on originator trust tier. A `hardware_bound` declaration provides stronger assurance of human authorization than a `host_mediated` declaration; both are stronger than `policy_derived`. See Section 8.3.

---

## 5. Delegation Object

### 5.1 Structure

Each delegation in a chain SHALL conform to the following structure:

```json
{
  "adcs_version": "1.0.0",
  "delegation_id": "<uuid-v4>",
  "parent_delegation_id": "<uuid-v4 | null>",
  "parent_delegation_ids": ["<uuid-v4>"],
  "delegator": {
    "id": "<principal identifier>",
    "did": "<DID URI>",
    "type": "human | system | agent",
    "public_key": "<JWK — OPTIONAL; see Section 4.2>",
    "key_provenance": "did_resolved | registry_backed | self_declared"
  },
  "delegate": {
    "id": "<principal identifier>",
    "did": "<DID URI>",
    "type": "human | system | agent"
  },
  "protocol": "<protocol identifier>",
  "issued_at": "<ISO 8601 timestamp>",
  "ttl": "<ISO 8601 duration, e.g. PT1H>",
  "expires_at": "<ISO 8601 timestamp>",
  "permissions_granted": ["<permission>"],
  "permissions_withheld": ["<permission>"],
  "permissions_conditional": [
    {
      "permission": "<permission string>",
      "condition_type": "human_approval | policy_gate | time_window | other",
      "condition_ref": "<opaque string, deployment-specific>",
      "delegation_scope": {
        "allowed_delegate_types": ["<agent type>"]
      }
    }
  ],
  "constraints_added": {
    "<constraint_type>": "<constraint_value>"
  },
  "intent_reference": "<delegation_id of originator declaration | null>",
  "signature": "<base64url-encoded signature over canonical delegation body>"
}
```

### 5.2 Field Definitions

| Field | Required | Description |
|---|---|---|
| `adcs_version` | REQUIRED | Version of this standard. MUST be `"1.0.0"` for conformant v1 implementations |
| `delegation_id` | REQUIRED | UUID v4. Globally unique identifier for this delegation |
| `parent_delegation_id` | CONDITIONAL | UUID v4 of the single parent delegation. MUST be `null` only for the first delegation whose parent is the originator declaration. MUST be absent if `parent_delegation_ids` is present |
| `parent_delegation_ids` | CONDITIONAL | Array of UUID v4 values for DAG merge delegations with multiple parents. MUST be absent if `parent_delegation_id` is present. See Section 6.6 |
| `delegator` | REQUIRED | The principal granting scope |
| `delegator.id` | REQUIRED | Verifiable identifier for the delegator. SHOULD be the DID string from `delegator.did` or a stable application-layer identifier traceable to it |
| `delegator.did` | REQUIRED | The DID URI for the delegator. Used by verifiers to resolve the current public key via DID document lookup |
| `delegator.type` | REQUIRED | One of `human`, `system`, `agent` |
| `delegator.public_key` | OPTIONAL | JWK-formatted public key. See Section 4.2. MUST be accompanied by `key_provenance` when present |
| `delegator.key_provenance` | CONDITIONAL | REQUIRED when `public_key` is present. One of `did_resolved`, `registry_backed`, `self_declared`. See Section 4.2 |
| `delegate` | REQUIRED | The principal receiving scope |
| `delegate.id` | REQUIRED | Verifiable identifier for the delegate |
| `delegate.did` | REQUIRED | The DID URI for the delegate |
| `delegate.type` | REQUIRED | One of `human`, `system`, `agent` |
| `protocol` | REQUIRED | The protocol through which this delegation is conveyed. See Section 12 for registered protocol identifiers |
| `issued_at` | REQUIRED | ISO 8601 timestamp at which this delegation was created |
| `ttl` | REQUIRED | ISO 8601 duration. MUST NOT exceed the remaining TTL of the parent delegation |
| `expires_at` | REQUIRED | ISO 8601 timestamp. MUST equal `issued_at + ttl` |
| `permissions_granted` | REQUIRED | Array of permission strings granted. MUST be a subset of the parent's effective permissions. MAY be empty |
| `permissions_withheld` | REQUIRED | Array of permission strings explicitly withheld. Informational. SHOULD list all parent permissions not in `permissions_granted` |
| `permissions_conditional` | OPTIONAL | Array of conditional permission objects. See Section 5.5 |
| `constraints_added` | REQUIRED | Constraint type/value pairs added or tightened at this hop. MAY be empty. Values MUST be equal to or more restrictive than the parent's constraint for the same type |
| `intent_reference` | REQUIRED | The `delegation_id` of the originator declaration at the root of this chain |
| `signature` | REQUIRED | Base64url-encoded signature over the canonical delegation body, signed by the delegator's key as resolved from `delegator.did` |

### 5.3 Permission Strings

Permission strings SHALL use reverse-domain notation to avoid collisions:

```
read
write
delete
search
compute
send_message
book
purchase
admin
key_management
delegate
<vendor-specific>: "com.example.custom_permission"
```

A core permission vocabulary is defined in Appendix A. Vendors MAY define additional permissions using reverse-domain notation. Implementations MUST treat unrecognized permission strings as opaque identifiers and MUST still enforce the subset invariant on them.

### 5.4 Canonical Delegation Body

The canonical body for signature computation is the JSON object containing all fields except `signature`, serialized using JSON Canonicalization Scheme (JCS) as defined in RFC 8785. Specifically:

- Keys sorted lexicographically (Unicode code point order) per RFC 8785 §3.2.3
- UTF-8 encoding
- No trailing whitespace
- No pretty-printing (compact representation)
- Number serialization per RFC 8785 §3.2.2.3 (no trailing zeros, no positive sign on exponent)

Implementations MUST produce and verify signatures over this canonical form. Implementations MUST NOT use ad-hoc canonicalization; a conformant JCS implementation is REQUIRED.

### 5.5 Conditional Permissions

A **conditional permission** is a permission that is granted to a delegate but may only be exercised after an external condition is satisfied.

```json
{
  "permission": "<permission string>",
  "condition_type": "human_approval | policy_gate | time_window | other",
  "condition_ref": "<opaque string, deployment-specific>",
  "delegation_scope": {
    "allowed_delegate_types": ["<agent type string>"]
  }
}
```

| Field | Required | Description |
|---|---|---|
| `permission` | REQUIRED | The permission string being conditionally granted |
| `condition_type` | REQUIRED | Category of condition: `human_approval`, `policy_gate`, `time_window`, or `other` |
| `condition_ref` | OPTIONAL | Opaque deployment-specific reference to the condition configuration or approval channel |
| `delegation_scope` | OPTIONAL | When applied to the `delegate` permission, restricts which agent types the receiving agent may further delegate to. See Appendix A |

**Monotonic restriction rules for conditional permissions:**

1. A permission in the parent's `permissions_conditional` MAY appear in the child's `permissions_conditional` with the same or more restrictive condition.
2. A permission in the parent's `permissions_conditional` MAY be moved to the child's `permissions_withheld`.
3. A permission in the parent's `permissions_conditional` MUST NOT appear in the child's `permissions_granted`.
4. A permission not present in the parent's `permissions_granted` or `permissions_conditional` MUST NOT appear in the child's `permissions_conditional`.

Conditional permissions MUST be tracked separately in `effective_conditional_permissions` throughout verification. Governance systems receiving a capability token with `effective_conditional_permissions` are responsible for enforcing the conditions before permitting the action.

### 5.6 Condition Satisfaction Record

When a conditional permission's condition is satisfied (e.g., a human approves the action), the governance layer MUST produce a **condition satisfaction record** providing cryptographic evidence of the approval.

```json
{
  "permission": "<permission string>",
  "condition_type": "human_approval | policy_gate | time_window | other",
  "satisfied_at": "<ISO 8601 timestamp>",
  "satisfied_by": "<DID of the approver or satisfying principal>",
  "evidence_ref": "<opaque reference to the approval artifact, e.g. approval ticket ID>",
  "chain_id": "<chain_id of the chain this satisfaction applies to>",
  "signature": "<base64url signature by the approver over canonical record body>"
}
```

| Field | Required | Description |
|---|---|---|
| `permission` | REQUIRED | The permission whose condition was satisfied |
| `condition_type` | REQUIRED | MUST match the `condition_type` from the conditional permission |
| `satisfied_at` | REQUIRED | Timestamp of condition satisfaction |
| `satisfied_by` | REQUIRED | DID of the principal who satisfied the condition. For `human_approval`, this is the approving human's DID. For `policy_gate`, this is the policy engine's DID |
| `evidence_ref` | OPTIONAL | Opaque reference to external approval evidence |
| `chain_id` | REQUIRED | The chain to which this satisfaction applies |
| `signature` | REQUIRED | Signature over the canonical record body (all fields except `signature`), signed by the `satisfied_by` principal |

**Delivery mechanism:** The condition satisfaction record is attached to the capability token as an additional JWT claim `condition_satisfactions` (an array of records). The verifier MUST verify each record's signature against the `satisfied_by` DID and MUST confirm that the `permission` and `condition_type` match an entry in `effective_conditional_permissions`. A satisfied conditional permission MAY then be treated as an effective permission for the terminal action.

**Audit requirement:** Condition satisfaction records MUST be logged alongside the chain verification event and retained for the audit retention period (Section 22.6).

---

## 6. Delegation Chain Structure

### 6.1 Chain Object

```json
{
  "adcs_version": "1.0.0",
  "chain_id": "<uuid-v4>",
  "originator_declaration": { },
  "delegations": [ ],
  "terminal_action": {
    "protocol": "<protocol identifier>",
    "operation": "<protocol-specific operation name>",
    "target": "<tool name | agent id | resource identifier>",
    "action_id": "<protocol-specific action identifier>"
  },
  "chain_digest": "<sha256 of canonical chain body, hex-encoded>",
  "revocation_service": "<URI of revocation service metadata endpoint>",
  "context_snapshot": {
    "task_state_hash": "<sha256 of current task or conversation state, hex-encoded>",
    "preceding_action_count": "<integer>",
    "preceding_action_summary": "<optional human-readable description>",
    "additional_context": { "<key>": "<value>" }
  }
}
```

### 6.2 Field Definitions

| Field | Required | Description |
|---|---|---|
| `adcs_version` | REQUIRED | MUST be `"1.0.0"` |
| `chain_id` | REQUIRED | UUID v4. Globally unique identifier for this chain instance |
| `originator_declaration` | REQUIRED | The originator declaration object. See Section 8 |
| `delegations` | REQUIRED | Ordered array of delegation objects. For linear chains: ordered from first delegation (delegator = originator) to last (delegate = terminal actor). For DAG chains: topologically sorted. MUST NOT be empty |
| `terminal_action` | REQUIRED | The action this chain authorizes |
| `terminal_action.protocol` | REQUIRED | Protocol of the terminal action |
| `terminal_action.operation` | REQUIRED | Operation being authorized |
| `terminal_action.target` | REQUIRED | The specific tool, agent, or resource being accessed |
| `terminal_action.action_id` | REQUIRED | Protocol-specific identifier for the action instance. Used to prevent chain reuse |
| `chain_digest` | REQUIRED | SHA-256 hash of the canonical chain body (all fields except `chain_digest`), hex-encoded |
| `revocation_service` | REQUIRED | URI of the revocation service metadata endpoint for the issuing organization. Gives verifiers a deterministic discovery path for revocation lists and CAEP endpoints. See Section 14.6 |
| `context_snapshot` | OPTIONAL | Observational snapshot of the runtime context at chain construction time. Not subject to verification. See Section 6.5 |

### 6.3 Chain Depth

A chain MUST contain at least one delegation. Chains exceeding five delegations SHOULD use chain compaction (Section 11). Chains MUST NOT exceed 20 delegations without compaction.

### 6.4 Chain Uniqueness

Each chain instance is bound to exactly one terminal action via `terminal_action.action_id`. A chain MUST NOT be presented for a different action than the one declared in `terminal_action`. Implementations MUST verify this binding before accepting a chain.

### 6.5 Context Snapshot

The `context_snapshot` field is an OPTIONAL observational record of the runtime state at chain construction time. It is not subject to any phase of the verification algorithm. Verifiers MUST NOT reject a chain because `context_snapshot` fields are absent, unrecognized, or differ from expectations.

| Field | Required | Description |
|---|---|---|
| `task_state_hash` | OPTIONAL | SHA-256 hash of the current task or conversation state at chain construction time |
| `preceding_action_count` | OPTIONAL | Number of actions already executed within the current session |
| `preceding_action_summary` | OPTIONAL | Human-readable or structured summary of preceding actions |
| `additional_context` | OPTIONAL | Free-form key-value object for deployment-specific data |

### 6.6 DAG Workflow Chains

ADCS supports **directed acyclic graph (DAG) workflow chains** where an orchestrator delegates to multiple sub-agents in parallel, and a downstream agent needs authorization derived from multiple converging branches.

**Structure:** A delegation with multiple parents declares `parent_delegation_ids` (an array) instead of `parent_delegation_id` (a single value). The `delegations` array MUST be topologically sorted such that every delegation appears after all of its parents.

**Merge semantics at a DAG convergence point:**

- **Effective permissions** = intersection of all parent branches' effective permissions
- **Effective constraints** = for each constraint type, the most restrictive value across all parent branches

The monotonic restriction invariant holds at merge points: the merged effective scope is always equal to or narrower than any individual parent branch's scope.

**DAG conformance** is an OPTIONAL capability at Level 3 (see Section 17). Implementations that do not support DAG chains MUST reject chains containing `parent_delegation_ids` with `DAG_NOT_SUPPORTED`.

---

## 7. Capability Token Binding

### 7.1 Purpose

A capability token is the bearer artifact that carries the delegation chain to the point of action invocation. The token is short-lived, transport-bound, and references the chain that authorizes the action.

### 7.2 Token Structure

Capability tokens SHALL be JSON Web Tokens (JWT) conforming to RFC 7519:

```json
{
  "iss": "<token issuer identifier>",
  "sub": "<delegate principal DID>",
  "aud": "<target server or agent DID or identifier>",
  "exp": "<Unix timestamp>",
  "nbf": "<Unix timestamp>",
  "iat": "<Unix timestamp>",
  "jti": "<uuid-v4>",
  "adcs_version": "1.0.0",
  "chain_id": "<chain_id from delegation chain>",
  "chain_digest": "<chain_digest from delegation chain>",
  "effective_permissions": ["<permission>"],
  "effective_conditional_permissions": [
    {
      "permission": "<permission string>",
      "condition_type": "human_approval | policy_gate | time_window | other",
      "condition_ref": "<opaque string>"
    }
  ],
  "effective_constraints": {
    "<constraint_type>": "<constraint_value>"
  },
  "originator_trust_tier": "hardware_bound | host_mediated | policy_derived",
  "condition_satisfactions": [
    {
      "permission": "<permission string>",
      "condition_type": "human_approval | policy_gate | time_window | other",
      "satisfied_at": "<ISO 8601 timestamp>",
      "satisfied_by": "<DID>",
      "evidence_ref": "<opaque string>",
      "chain_id": "<chain_id>",
      "signature": "<base64url signature>"
    }
  ],
  "cnf": {
    "x5t#S256": "<SHA-256 thumbprint of client certificate>",
    "jkt": "<JWK thumbprint for DPoP>"
  }
}
```

### 7.3 Field Requirements

| Claim | Required | Description |
|---|---|---|
| `iss` | REQUIRED | Identifier of the token issuer |
| `sub` | REQUIRED | DID of the delegate invoking the action |
| `aud` | REQUIRED | Identifier of the target server or agent |
| `exp` | REQUIRED | Expiration. MUST NOT exceed the expiration of the last delegation in the chain |
| `nbf` | REQUIRED | Not-before time |
| `iat` | REQUIRED | Issued-at time |
| `jti` | REQUIRED | Unique token identifier. Used for replay prevention |
| `adcs_version` | REQUIRED | MUST be `"1.0.0"` |
| `chain_id` | REQUIRED | References the full chain object |
| `chain_digest` | REQUIRED | SHA-256 digest of the referenced chain |
| `effective_permissions` | REQUIRED | Computed effective permissions at the terminal hop. Verifiers MUST recompute and reject mismatches |
| `effective_conditional_permissions` | REQUIRED | Computed effective conditional permissions. MUST be empty array if none. Subject to recomputation |
| `effective_constraints` | REQUIRED | Computed effective constraints. Subject to recomputation |
| `originator_trust_tier` | REQUIRED | Trust tier from the originator declaration. Carried to the token for governance system use |
| `condition_satisfactions` | OPTIONAL | Array of condition satisfaction records (Section 5.6). Empty array if no conditions have been satisfied. Verifiers MUST verify each record's signature |
| `cnf` | REQUIRED | Proof-of-possession claim. MUST include `x5t#S256` or `jkt`. MUST NOT be absent |

### 7.4 Token Lifetime

```
exp ≤ min(chain.delegations[-1].expires_at, issuer_max_ttl)
```

Implementations SHOULD issue tokens with a TTL of 15 minutes or less. Tokens with a TTL exceeding 1 hour MUST NOT be issued.

### 7.5 Proof-of-Possession Requirement

Every capability token MUST include a `cnf` claim binding the token to the holder's transport credential. A token without a `cnf` claim is non-conformant and MUST be rejected. The verifier MUST confirm that the TLS client certificate thumbprint (for mTLS) or the DPoP proof (for DPoP) matches the `cnf` claim.

### 7.6 Macaroon-Based Tokens

Implementations MAY use Macaroons in place of JWTs, provided:

- The Macaroon carries caveats equivalent to the JWT claims in Section 7.2
- Each delegation hop is expressed as an attenuation caveat on the parent Macaroon
- The monotonic restriction invariant is enforced through caveat addition
- Proof-of-possession is included as a contextual caveat bound to the transport session

JWT is the required baseline format. Implementations supporting Macaroon tokens MUST also support JWT.

### 7.7 Token-Only Presentation Mode (Privacy-Preserving)

To protect deployment topology and organizational trust hierarchy from disclosure to third-party tool servers, implementations SHOULD support a **token-only presentation mode**:

- The tool server receives only the capability token (with effective permissions and constraints)
- The full chain is retained by the issuing governance layer, not transmitted to the tool server
- The governance layer vouches for the chain's validity via the token signature

When operating in token-only mode, the tool server MUST trust the token issuer's governance layer for chain verification. The full chain MUST remain available for audit retrieval via the `chain_id`. This mode is OPTIONAL but RECOMMENDED for all third-party tool integrations.

---

## 8. Originator Declaration

### 8.1 Purpose

The originator declaration is the root of every delegation chain. It declares the human or system intent that the chain serves, the permissions the originator is granting, and the constraints that apply from the root.

### 8.2 Structure

```json
{
  "adcs_version": "1.0.0",
  "delegation_id": "<uuid-v4>",
  "originator": {
    "id": "<principal identifier>",
    "did": "<DID URI>",
    "type": "human | system",
    "public_key": "<JWK — OPTIONAL; see Section 4.2>",
    "key_provenance": "did_resolved | registry_backed | self_declared"
  },
  "originator_trust_tier": "hardware_bound | host_mediated | policy_derived",
  "originator_quorum": {
    "required": 2,
    "signers": [
      {
        "id": "<principal identifier>",
        "did": "<DID URI>",
        "signature": "<base64url-encoded signature over canonical originator declaration body>"
      }
    ]
  },
  "issued_at": "<ISO 8601 timestamp>",
  "ttl": "<ISO 8601 duration>",
  "expires_at": "<ISO 8601 timestamp>",
  "intent": "<natural language description of the task being authorized>",
  "intent_structured": {
    "action_category": "booking | procurement | data_access | communication | administration | computation | other",
    "resource_types": ["<deployment-defined resource type strings>"],
    "sensitivity": "low | medium | high | critical"
  },
  "permissions_granted": ["<permission>"],
  "permissions_conditional": [
    {
      "permission": "<permission string>",
      "condition_type": "human_approval | policy_gate | time_window | other",
      "condition_ref": "<opaque string, deployment-specific>"
    }
  ],
  "constraints": {
    "<constraint_type>": "<constraint_value>"
  },
  "context": {
    "bootstrapping_method": "human_explicit | host_derived | policy_derived",
    "<key>": "<value>"
  },
  "signature": "<base64url-encoded signature over canonical originator declaration body>"
}
```

### 8.3 Field Definitions

| Field | Required | Description |
|---|---|---|
| `delegation_id` | REQUIRED | UUID v4. The root delegation_id for this chain |
| `originator` | REQUIRED | The human or system initiating the chain |
| `originator.id` | REQUIRED | Verifiable identifier for the originator |
| `originator.did` | REQUIRED | DID URI for the originator |
| `originator.type` | REQUIRED | MUST be `human` or `system`. MUST NOT be `agent` |
| `originator.public_key` | OPTIONAL | JWK-formatted public key. See Section 4.2. MUST be accompanied by `key_provenance` when present |
| `originator.key_provenance` | CONDITIONAL | REQUIRED when `public_key` is present. One of `did_resolved`, `registry_backed`, `self_declared`. See Section 4.2 |
| `originator_trust_tier` | REQUIRED | One of `hardware_bound`, `host_mediated`, `policy_derived`. See Section 4.8 |
| `originator_quorum` | OPTIONAL | Multi-party quorum authorization. See Section 8.5 |
| `issued_at` | REQUIRED | When this declaration was created |
| `ttl` | REQUIRED | Duration for which this authorization is valid |
| `expires_at` | REQUIRED | Absolute expiration. MUST equal `issued_at + ttl` |
| `intent` | REQUIRED | Natural language description. Minimum 10 characters. MUST NOT be empty. MUST be preserved throughout the chain. Serves audit readability; governance layers SHOULD NOT rely on `intent` alone for programmatic action-consistency checks |
| `intent_structured` | OPTIONAL | Machine-readable intent classification. When present, governance layers SHOULD use it for intent-action consistency checks (Section 16.8). See Section 8.8 |
| `permissions_granted` | REQUIRED | Maximum set of permissions any delegation in this chain may grant |
| `permissions_conditional` | OPTIONAL | Permissions the originator grants conditionally at the chain root |
| `constraints` | REQUIRED | Root constraints applying to the entire chain |
| `context` | OPTIONAL | Additional structured context. MUST include `bootstrapping_method`. MUST NOT contain permissions or constraints |
| `signature` | REQUIRED | Signature over the canonical originator declaration body |

### 8.4 Human vs. System Originators

A **human originator** is a natural person whose identity is established through an external authentication system. The `originator.did` SHOULD resolve to a DID document whose controller is the human's authenticated identity.

A **system originator** is an automated system acting as the root principal for agent workflows triggered by scheduled jobs, event-driven processes, or API calls. System originators MUST have DIDs resolvable via `did:web` or equivalent organizational DID infrastructure.

### 8.5 Multi-Party Originator Quorum

For regulated-industry use cases requiring dual control or broader multi-party authorization, the `originator_quorum` field allows multiple principals to co-sign an originator declaration.

```json
"originator_quorum": {
  "required": 2,
  "signers": [
    {
      "id": "user:alice@example.com",
      "did": "did:web:example.com:users:alice",
      "signature": "<base64url-encoded signature over canonical originator declaration body>"
    },
    {
      "id": "user:bob@example.com",
      "did": "did:web:example.com:users:bob",
      "signature": "<base64url-encoded signature over canonical originator declaration body>"
    }
  ]
}
```

**Quorum rules:**

- `required` MUST be a positive integer ≤ the number of entries in `signers`
- `required` MAY be 1 (effectively a single alternate signer); it MUST NOT be 0
- Each signer's `signature` is computed over the same canonical originator declaration body as the primary `signature`, excluding all `signature` fields
- Quorum verification (Phase 2, Step 2.1b) requires that at least `required` valid signatures from distinct signers are present
- The primary `signature` field (signed by the declared `originator`) counts toward the quorum if the originator's DID also appears in `signers`; otherwise the primary signature is verified independently
- Organizations with more than two required approvers MAY specify `required` values greater than 2

When `originator_quorum` is present, the `originator_trust_tier` MUST be `hardware_bound` or `host_mediated` for all signing principals. A quorum originator declaration with `policy_derived` tier for any signer MUST be treated as `policy_derived` for the entire declaration.

### 8.6 Intent Preservation

The `intent` field MUST be preserved and accessible throughout the chain. The intent field MUST NOT be modified after signing.

### 8.7 Originator Declaration Bootstrapping (Normative)

This section defines normative requirements for how originator declarations are created. Three approaches are recognized, each with associated requirements.

**Human Explicit:** The host application creates the originator declaration from explicit user input. The user confirms the intent statement and the permission set before signing.

- MUST present the inferred scope to the user before signing when interaction is possible
- MUST NOT proceed if the user does not confirm
- `context.bootstrapping_method` MUST be `"human_explicit"`
- `originator_trust_tier` SHOULD be `hardware_bound` or `host_mediated`

**Host-Derived:** The host application infers scope from conversation context, task metadata, or LLM interpretation.

- SHOULD surface inferred intent and permissions to the user for confirmation before signing in interactive sessions
- MUST NOT use Host-Derived bootstrapping for chains granting `purchase` or `delete` permissions in interactive sessions without an explicit user confirmation step
- `context.bootstrapping_method` MUST be `"host_derived"`
- `originator_trust_tier` MUST be `host_mediated`

**Policy-Derived:** The deployment's policy configuration generates the declaration automatically.

- The originating system, not a human, signs the declaration
- `context.bootstrapping_method` MUST be `"policy_derived"`
- `context` MUST include a `policy_rule_id` field referencing the policy that generated the declaration
- `originator_trust_tier` MUST be `policy_derived`

Regardless of method:

- The originator MUST sign with the private key corresponding to their DID's current verification key
- The `intent` field MUST accurately reflect the scope being authorized
- The bootstrapping method MUST be recorded in `context.bootstrapping_method`

### 8.8 Structured Intent

The `intent_structured` field provides machine-readable classification of the originator's intent, enabling programmatic intent-action consistency checks by governance layers.

```json
"intent_structured": {
  "action_category": "booking",
  "resource_types": ["flights", "hotels"],
  "sensitivity": "medium"
}
```

| Field | Required | Description |
|---|---|---|
| `action_category` | REQUIRED | One of: `booking`, `procurement`, `data_access`, `communication`, `administration`, `computation`, `other` |
| `resource_types` | REQUIRED | Array of deployment-defined resource type strings describing what the chain acts upon. MUST NOT be empty |
| `sensitivity` | REQUIRED | One of: `low`, `medium`, `high`, `critical`. Governance layers MAY use this to determine approval thresholds |

**Intent-action consistency:** Governance layers SHOULD compare `intent_structured.action_category` against `terminal_action.operation` and the permissions in the chain. A chain with `action_category: data_access` whose terminal action invokes a `purchase` tool is a signal warranting human review. The consistency check is a governance-layer responsibility; ADCS does not define a normative consistency algorithm.

`intent_structured` MUST be preserved and accessible throughout the chain alongside `intent`. When present in the originator declaration, it MUST NOT be modified after signing.

---

## 9. Constraint Type Registry

### 9.1 Purpose

Constraints are only useful if all parties share a common vocabulary for what a constraint means and how to determine whether a proposed action violates it.

### 9.2 Core Constraint Types

All conformant implementations MUST support evaluation of these types.

#### 9.2.1 `budget`

```json
"budget": { "max": 1500, "currency": "USD" }
```

| Field | Required | Description |
|---|---|---|
| `max` | REQUIRED | Maximum monetary value. MUST be non-negative |
| `currency` | REQUIRED | ISO 4217 currency code |

**Restriction rule:** A hop MAY reduce `max`. A hop MUST NOT increase `max`. Currency changes SHOULD be treated with caution; implementations MAY reject them.

> **⚠ WARNING — Per-Chain Limit Only:** The `budget` constraint limits spending authorization within a single chain. An originator issuing multiple chains can authorize cumulative spending exceeding any individual chain's `budget.max`. Deployments with financial accountability requirements MUST implement a Constraint Accounting Service (Section 9.5) or equivalent external budget tracking. ADCS alone does not enforce cumulative budget limits.

#### 9.2.2 `data_scope`

```json
"data_scope": "public_only | internal | confidential | restricted"
```

**Ordering (least to most restrictive):** `public_only` < `internal` < `confidential` < `restricted`

**Restriction rule:** A hop MAY change to a more restrictive value. MUST NOT change to a less restrictive value.

#### 9.2.3 `time_range`

```json
"time_range": { "start": "<ISO 8601>", "end": "<ISO 8601>" }
```

**Restriction rule:** A hop MAY narrow the range. MUST NOT widen it.

#### 9.2.4 `network_scope`

```json
"network_scope": {
  "allowed_hosts": ["api.example.com", "*.internal.example.com"],
  "denied_hosts": ["*.external.example.com"]
}
```

**Restriction rule:** A hop MAY remove from `allowed_hosts` or add to `denied_hosts`. MUST NOT add to `allowed_hosts` or remove from `denied_hosts`.

#### 9.2.5 `resource_path`

```json
"resource_path": {
  "allowed": ["/home/alice/docs/"],
  "denied": ["/home/alice/docs/private/"]
}
```

**Restriction rule:** A hop MAY narrow `allowed` paths or expand `denied` paths. MUST NOT add new allowed paths or remove denied paths.

#### 9.2.6 `max_results`

```json
"max_results": 20
```

**Restriction rule:** A hop MAY decrease. MUST NOT increase.

#### 9.2.7 `action_count`

```json
"action_count": { "max": 10, "window": "PT1H" }
```

**Restriction rule:** A hop MAY decrease `max` or decrease `window`. MUST NOT increase either.

#### 9.2.8 `max_delegation_depth`

Constrains how many additional delegation hops may be created below the hop at which this constraint is introduced.

```json
"max_delegation_depth": 3
```

**Restriction rule:** A hop MAY decrease `max_delegation_depth`. MUST NOT increase it. When `max_delegation_depth` reaches 0 at a given hop, that agent MUST NOT create further sub-delegations regardless of whether `delegate` permission is present.

The verification algorithm MUST track the current delegation depth relative to where `max_delegation_depth` was set and fail with `DELEGATION_DEPTH_EXCEEDED` if a delegation is attempted beyond the allowed depth.

### 9.3 Vendor Constraint Types

Vendors MAY define additional constraint types using reverse-domain notation. Implementations encountering an unrecognized constraint type MUST treat it as opaque and MUST deny if a hop modifies it with `UNKNOWN_CONSTRAINT_MODIFIED`, unless a trusted policy engine that understands the type has verified the modification.

### 9.4 Constraint Evaluation Capability Declaration

> **⚠ IMPORTANT:** Structural verification of constraint presence is not the same as semantic enforcement. See Section 1.3 Warning.

Implementations MUST declare their constraint evaluation capability in a machine-readable capability document:

```json
{
  "adcs_constraint_evaluation": "structural_only | semantic",
  "semantic_mapping_method": "publisher_declared | pattern_matching | llm_assisted | none"
}
```

| Value | Meaning |
|---|---|
| `structural_only` | Implementation verifies constraints are present and structurally non-loosened. Does not evaluate whether action parameters satisfy constraint values |
| `semantic` | Implementation evaluates whether action parameters satisfy constraint values using the declared mapping method |

Implementations that declare `structural_only` MUST NOT represent themselves as enforcing semantic constraints. This capability declaration MUST be included in any conformance documentation and SHOULD be accessible via the deployment's governance API.

### 9.5 Cumulative Constraint Accounting

ADCS constraint values are per-chain structural limits. ADCS does not maintain running totals against constraint budgets across multiple chains from the same originator. An originator granting `budget.max = $1500` across five separate chains authorizing $400 each would result in $2000 in total authorized spend — exceeding the constraint — without ADCS detecting it.

Deployments requiring cumulative constraint enforcement SHOULD implement a **Constraint Accounting Service (CAS)** that is consulted at chain construction time. The CAS maintains running totals per originator declaration and per constraint type, and approves or denies chain construction based on remaining budget.

**CAS Reservation API (Normative Interface):**

```
POST /adcs/cas/reserve
Content-Type: application/json

{
  "originator_did": "<DID of the originator>",
  "constraint_type": "budget",
  "requested_value": { "max": 400, "currency": "USD" },
  "chain_id": "<chain_id being constructed>",
  "ttl": "<ISO 8601 duration — reservation expires if chain is not finalized>"
}
```

**Success response (200):**

```json
{
  "reservation_id": "<uuid-v4>",
  "constraint_type": "budget",
  "reserved_value": { "max": 400, "currency": "USD" },
  "remaining": { "max": 1100, "currency": "USD" },
  "expires_at": "<ISO 8601 timestamp>"
}
```

**Failure response (409):**

```json
{
  "error": "BUDGET_EXHAUSTED",
  "constraint_type": "budget",
  "requested_value": { "max": 400, "currency": "USD" },
  "remaining": { "max": 200, "currency": "USD" }
}
```

**Release endpoint:**

```
DELETE /adcs/cas/reserve/{reservation_id}
```

Releases a reservation when a chain is abandoned before finalization.

**CAS integration with chain objects:** When a CAS reservation is obtained, the `cas_reservation_id` field SHOULD be included in the chain object's `context_snapshot.additional_context`:

```json
"additional_context": {
  "cas_reservation_id": "<reservation_id from CAS>"
}
```

Verifiers that enforce cumulative constraints SHOULD confirm the reservation is valid and not expired before accepting the chain. The CAS is an OPTIONAL deployment component; implementations that do not deploy a CAS MUST still support the `budget` constraint type for per-chain structural verification.

---

## 10. Verification Algorithm

### 10.1 Overview

The verification algorithm is the normative procedure by which any conformant implementation determines whether a delegation chain is valid for a given terminal action. Every implementation that accepts delegation chains MUST implement this algorithm completely and MUST NOT accept a chain that fails any step.

Verification has four phases:

1. **Structural verification** — chain is well-formed and internally consistent
2. **Cryptographic verification** — all signatures are valid
3. **Temporal verification** — all TTLs are valid at the time of verification
4. **Monotonic restriction verification** — each hop narrows or preserves scope

`context_snapshot`, if present, is not subject to any verification phase.

### 10.2 Inputs

```
chain:                  A complete delegation chain object (Section 6)
token:                  A capability token (Section 7)
current_time:           The verifier's current time (ISO 8601)
terminal_action:        The action being authorized (protocol, operation, target, action_id)
clock_skew_tolerance:   ISO 8601 duration tolerance for temporal checks (default: PT60S)
```

### 10.3 Phase 1 — Structural Verification

```
1.1  chain.adcs_version MUST equal "1.0.0"
     → FAIL: VERSION_MISMATCH

1.2  chain.originator_declaration MUST be present and non-null
     → FAIL: MISSING_ORIGINATOR

1.3  chain.delegations MUST be a non-empty ordered array
     → FAIL: EMPTY_CHAIN

1.4  chain.terminal_action MUST be present
     → FAIL: MISSING_TERMINAL_ACTION

1.5  chain.terminal_action.action_id MUST equal terminal_action.action_id provided as input
     → FAIL: ACTION_ID_MISMATCH

1.6  chain.terminal_action.protocol MUST equal terminal_action.protocol provided as input
     → FAIL: PROTOCOL_MISMATCH

1.7  chain.terminal_action.operation MUST equal terminal_action.operation provided as input
     → FAIL: OPERATION_MISMATCH

1.8  chain.terminal_action.target MUST equal terminal_action.target provided as input
     → FAIL: TARGET_MISMATCH

1.9  For linear chains (chain.delegations[0].parent_delegation_id is not null):
     chain.delegations[0].parent_delegation_id MUST equal
     chain.originator_declaration.delegation_id
     → FAIL: BROKEN_CHAIN

1.9b For DAG chains (chain.delegations[0].parent_delegation_ids is present):
     If the implementation does not support DAG chains:
       → FAIL: DAG_NOT_SUPPORTED
     Verify that the delegations array is topologically sorted:
       For each delegation d with parent_delegation_ids:
         Every id in parent_delegation_ids MUST correspond to a delegation
         that appears earlier in the delegations array, or to the originator
         declaration delegation_id
       → FAIL: BROKEN_CHAIN (if any reference not satisfied)

1.10 For i in 1..len(chain.delegations)-1:
     If chain.delegations[i].parent_delegation_id is present:
       MUST equal chain.delegations[i-1].delegation_id
     If chain.delegations[i].parent_delegation_ids is present:
       Each referenced id MUST correspond to a preceding delegation
       or the originator declaration
     → FAIL: BROKEN_CHAIN

1.11 For every delegation d in chain.delegations:
     d.intent_reference MUST equal chain.originator_declaration.delegation_id
     → FAIL: INTENT_REFERENCE_MISMATCH

1.12 Recompute chain.chain_digest over canonical chain body (all fields except chain_digest)
     chain.chain_digest MUST match the computed value
     → FAIL: CHAIN_DIGEST_MISMATCH

1.13 token.chain_id MUST equal chain.chain_id
     → FAIL: TOKEN_CHAIN_MISMATCH

1.14 token.chain_digest MUST equal chain.chain_digest
     → FAIL: TOKEN_DIGEST_MISMATCH

1.15 chain.revocation_service MUST be present and non-empty
     → FAIL: MISSING_REVOCATION_SERVICE
```

### 10.4 Phase 2 — Cryptographic Verification

```
2.1  Resolve chain.originator_declaration.originator.did to obtain current public key.
     If inline public_key is present:
       MAY use inline public_key without DID resolution.
       key_provenance MUST be recorded in the verification audit log.
     Verify chain.originator_declaration.signature using resolved or inline key
     → FAIL: INVALID_ORIGINATOR_SIGNATURE
     → FAIL: DID_RESOLUTION_FAILURE (if DID cannot be resolved)

2.1b If chain.originator_declaration.originator_quorum is present:
     Verify that at least originator_quorum.required valid signatures are present
     in originator_quorum.signers, each verified against the signer's resolved DID key.
     Each signer's DID MUST be resolvable.
     → FAIL: QUORUM_INSUFFICIENT (if fewer than required valid signatures)
     → FAIL: DID_RESOLUTION_FAILURE (if any signer DID cannot be resolved)

2.2  For every delegation d in chain.delegations:
     Resolve d.delegator.did to obtain current public key.
     Verify d.signature using resolved key.
     → FAIL: INVALID_DELEGATION_SIGNATURE (include delegation_id in error)
     → FAIL: DID_RESOLUTION_FAILURE (include delegation_id if DID cannot be resolved)

2.3  Verify token signature using the token issuer's public key
     → FAIL: INVALID_TOKEN_SIGNATURE

2.4  Verify token proof-of-possession:
     If token.cnf contains x5t#S256:
       The SHA-256 thumbprint of the client's TLS certificate on the
       current connection MUST equal token.cnf.x5t#S256
       → FAIL: POP_MISMATCH
     If token.cnf contains jkt:
       The DPoP proof MUST be valid and the JWK thumbprint MUST
       equal token.cnf.jkt
       → FAIL: POP_MISMATCH
     If token.cnf is absent:
       → FAIL: MISSING_POP

2.5  Verify that the token has not been presented before (replay prevention)
     token.jti MUST NOT appear in the verifier's seen-tokens record for this session
     → FAIL: TOKEN_REPLAY
     Record token.jti as seen
```

### 10.5 Phase 3 — Temporal Verification

Clock skew tolerance of `clock_skew_tolerance` (default PT60S) is applied to all expiry comparisons. The tolerance window is additive: a delegation with `expires_at` in the past by less than `clock_skew_tolerance` MAY be accepted. Implementations SHOULD log when a chain passes only within the tolerance window.

```
3.1  chain.originator_declaration.expires_at MUST be > (current_time - clock_skew_tolerance)
     → FAIL: ORIGINATOR_EXPIRED

3.2  chain.originator_declaration.expires_at MUST equal
     chain.originator_declaration.issued_at + chain.originator_declaration.ttl
     → FAIL: ORIGINATOR_TTL_INCONSISTENT

3.3  For every delegation d in chain.delegations:
     d.expires_at MUST be > (current_time - clock_skew_tolerance)
     → FAIL: DELEGATION_EXPIRED (include delegation_id)

3.4  For every delegation d in chain.delegations:
     d.expires_at MUST equal d.issued_at + d.ttl
     → FAIL: DELEGATION_TTL_INCONSISTENT (include delegation_id)

3.5  For every delegation d with a single parent p:
     d.expires_at MUST be ≤ p.expires_at
     → FAIL: TTL_EXCEEDS_PARENT (include delegation_id)

3.5b For every DAG delegation d with parent_delegation_ids:
     d.expires_at MUST be ≤ min(expires_at of all parent delegations)
     → FAIL: TTL_EXCEEDS_PARENT (include delegation_id)

3.6  token.exp MUST be > (current_time - clock_skew_tolerance)
     → FAIL: TOKEN_EXPIRED

3.7  token.nbf MUST be ≤ (current_time + clock_skew_tolerance)
     → FAIL: TOKEN_NOT_YET_VALID

3.8  token.exp MUST be ≤ chain.delegations[-1].expires_at
     → FAIL: TOKEN_TTL_EXCEEDS_CHAIN
```

### 10.6 Phase 4 — Monotonic Restriction Verification

```
Initialize:
  effective_permissions = chain.originator_declaration.permissions_granted
  effective_conditional_permissions = chain.originator_declaration.permissions_conditional (if present, else [])
  effective_constraints = chain.originator_declaration.constraints
  current_depth = 0

For each delegation d in chain.delegations (in topological order):

  4.0  If current_depth > 0 (i.e., this is not the first delegation):
       The delegator of d received its scope from the preceding hop(s).
       That delegator MUST hold "delegate" in its effective_permissions to create
       further delegations.
       
       For linear delegations:
         "delegate" MUST be present in effective_permissions at the preceding hop.
       
       For DAG delegations (parent_delegation_ids present):
         "delegate" MUST be present in the effective_permissions of EVERY parent
         branch that feeds into this delegation's delegator.
       
       → FAIL: UNAUTHORIZED_SUBDELEGATION (include delegation_id)
       
       If "max_delegation_depth" is present in effective_constraints:
         current_depth MUST be ≤ effective_constraints.max_delegation_depth
         → FAIL: DELEGATION_DEPTH_EXCEEDED (include delegation_id)

       Note: At current_depth == 0, the delegator is the originator or receives
       scope directly from the originator declaration. The originator does not
       need "delegate" permission — the originator is the root authority.
       The first delegate's ability to sub-delegate is governed by whether
       "delegate" appears in the first delegation's permissions_granted.

  4.1  d.permissions_granted MUST be a subset of effective_permissions
       For each p in d.permissions_granted:
         p MUST be in effective_permissions
         → FAIL: PERMISSION_EXPANSION (include delegation_id and permission)

  4.1b For each conditional permission cp in d.permissions_conditional (if present):
       cp.permission MUST be in effective_permissions OR in
       {p.permission for p in effective_conditional_permissions}
       → FAIL: PERMISSION_EXPANSION (include delegation_id and permission)
       cp.permission MUST NOT be in d.permissions_granted if it appeared
       only in effective_conditional_permissions
       → FAIL: PERMISSION_CONDITION_LOOSENED (include delegation_id and permission)

  4.2  For each constraint type ct in d.constraints_added:
       If ct is a known type (Section 9.2):
         Apply the monotonic restriction rule for that type.
         New value MUST be equal to or more restrictive than effective_constraints[ct]
         → FAIL: CONSTRAINT_LOOSENED (include delegation_id and constraint_type)
       If ct is an unknown type:
         If effective_constraints[ct] exists and d.constraints_added[ct] differs:
           → FAIL: UNKNOWN_CONSTRAINT_MODIFIED (include delegation_id and constraint_type)

  4.2b For DAG delegations (parent_delegation_ids present):
       Compute merged_effective_permissions = intersection of all parent branches' effective_permissions
       Compute merged_effective_constraints = most restrictive value per type across all parent branches
       d.permissions_granted MUST be a subset of merged_effective_permissions
       → FAIL: DAG_MERGE_VIOLATION (include delegation_id)

  4.3  Update effective state:
       If linear delegation:
         effective_permissions = d.permissions_granted
         effective_conditional_permissions = d.permissions_conditional (if present, else [])
         For each ct in d.constraints_added:
           effective_constraints[ct] = d.constraints_added[ct]
       If DAG delegation:
         effective_permissions = merged_effective_permissions ∩ d.permissions_granted
         effective_conditional_permissions = d.permissions_conditional (if present, else [])
         For each ct: effective_constraints[ct] = most restrictive(merged_effective_constraints[ct], d.constraints_added[ct])
       current_depth += 1

After final delegation:

  4.4  Verify token.effective_permissions equals effective_permissions (set equality)
       → FAIL: TOKEN_PERMISSIONS_MISMATCH

  4.4b Verify token.effective_conditional_permissions equals effective_conditional_permissions
       (set equality; condition_type and condition_ref MUST also match)
       → FAIL: TOKEN_CONDITIONAL_PERMISSIONS_MISMATCH

  4.5  Verify token.effective_constraints equals effective_constraints (deep equality)
       → FAIL: TOKEN_CONSTRAINTS_MISMATCH
```

### 10.7 Revocation Check

After Phase 4, perform revocation checks using the revocation service discovered via `chain.revocation_service`:

```
R.1  Fetch or consult cached revocation data from chain.revocation_service metadata endpoint.

R.2  chain.originator_declaration.delegation_id MUST NOT appear in the revocation list.
     → FAIL: ORIGINATOR_REVOKED

R.3  For every delegation d in chain.delegations:
     d.delegation_id MUST NOT appear in the revocation list.
     → FAIL: DELEGATION_REVOKED (include delegation_id)

R.4  token.jti MUST NOT appear in the token revocation list.
     → FAIL: TOKEN_REVOKED
```

### 10.8 Verification Result

A chain that passes all phases and revocation checks SHALL produce:

```json
{
  "result": "VALID",
  "chain_id": "<chain_id>",
  "effective_permissions": ["<permission>"],
  "effective_constraints": { },
  "originator_trust_tier": "<tier>",
  "verified_at": "<ISO 8601 timestamp>",
  "originator_intent": "<intent string from originator declaration>"
}
```

A chain that fails any step SHALL produce:

```json
{
  "result": "INVALID",
  "chain_id": "<chain_id>",
  "error_code": "<error code from Section 15>",
  "error_detail": "<human-readable explanation>",
  "failed_at": "<delegation_id or 'originator' or 'token'>",
  "verified_at": "<ISO 8601 timestamp>"
}
```

### 10.9 Fail-Closed Requirement

Implementations MUST treat any verification failure as a denial of the terminal action. There is no partial validity. Implementations MUST NOT fall back to broader authorization on chain failure.

### 10.10 Permission Escalation Protocol

When an agent discovers mid-task that it requires a permission not present in its effective scope, it MAY issue a **permission escalation request** to the governance layer rather than abandoning the chain.

**Escalation request format:**

```json
{
  "adcs_version": "1.0.0",
  "escalation_id": "<uuid-v4>",
  "chain_id": "<chain_id of the active chain>",
  "requesting_agent_did": "<DID of the requesting agent>",
  "requested_permission": "<permission string>",
  "requested_condition_type": "human_approval | policy_gate",
  "justification": "<natural language explanation of why the permission is needed>",
  "issued_at": "<ISO 8601 timestamp>",
  "signature": "<base64url signature over canonical escalation body, signed by requesting agent>"
}
```

**Governance layer response:**

If the escalation is approved, the governance layer constructs a new delegation hop extending the active chain, expressing the approved permission as a conditional or unconditional grant, signed by an authorized approver. This new hop is appended to the chain; the chain_digest is recomputed, and a new capability token is issued.

If denied, the governance layer returns a denial with a reason code. The agent MUST NOT proceed with the action.

**Constraints:** An escalation MUST NOT request a permission not present in the originator declaration's `permissions_granted` or `permissions_conditional`. Governance layers MUST reject escalations that would expand scope beyond the originator's root grant.

---

## 11. Chain Compaction

### 11.1 Purpose

Delegation chains in deep multi-agent hierarchies may accumulate many hops. Chain compaction allows intermediate hops to be summarized while preserving auditability.

### 11.2 Compaction Threshold

Chains SHOULD be compacted when depth exceeds five delegations. Chains exceeding 20 delegations MUST use compaction or MUST NOT be created.

### 11.3 Compacted Chain Object

```json
{
  "adcs_version": "1.0.0",
  "chain_id": "<uuid-v4 — same as original chain>",
  "compacted": true,
  "originator_declaration": { },
  "first_delegation": { },
  "last_delegation": { },
  "effective_permissions": ["<permission>"],
  "effective_constraints": { },
  "chain_length": 8,
  "intermediate_digest": "<sha256 of canonical intermediate delegations array, hex-encoded>",
  "full_chain_locator": "<URI where full chain is available>",
  "terminal_action": { },
  "chain_digest": "<sha256 of canonical compacted chain body, hex-encoded>",
  "revocation_service": "<URI of revocation service metadata endpoint>",
  "compaction_signature": "<base64url-encoded signature over canonical compacted body>"
}
```

### 11.4 Field Definitions

| Field | Required | Description |
|---|---|---|
| `compacted` | REQUIRED | MUST be `true` |
| `originator_declaration` | REQUIRED | Full originator declaration; MUST NOT be compacted |
| `first_delegation` | REQUIRED | First delegation in the chain; MUST NOT be compacted |
| `last_delegation` | REQUIRED | Last delegation (delegate = terminal actor); MUST NOT be compacted |
| `effective_permissions` | REQUIRED | Computed effective permissions at terminal hop |
| `effective_constraints` | REQUIRED | Computed effective constraints at terminal hop |
| `chain_length` | REQUIRED | Total number of delegations including first and last |
| `intermediate_digest` | REQUIRED | SHA-256 hash of the canonical JSON array of all intermediate delegations |
| `full_chain_locator` | REQUIRED | URI for the full uncompacted chain. MUST be accessible for audit. Full chain MUST be retained per applicable policy |
| `revocation_service` | REQUIRED | Same as in full chain object |
| `compaction_signature` | REQUIRED | Signature over canonical compacted chain body, signed by the compacting party |

### 11.5 Compaction Verification

Verification of a compacted chain executes Phases 1–3 adapted for the compacted structure, plus a compaction signature verification. Phase 4 is verified for first and last delegation against the originator declaration; `effective_permissions` and `effective_constraints` are trusted subject to the compaction signature.

Implementations performing high-assurance verification SHOULD retrieve the full chain from `full_chain_locator` and execute the complete algorithm.

---

## 12. Cross-Protocol Chains

### 12.1 Protocol-Agnosticism

A delegation chain is protocol-agnostic. Each delegation records which protocol was used for that hop, but chain structure and verification are identical regardless of protocol.

### 12.2 Registered Protocol Identifiers

| Identifier | Protocol |
|---|---|
| `mcp` | Model Context Protocol |
| `a2a` | Agent-to-Agent Protocol |
| `acp` | Agent Communication Protocol (IBM/Linux Foundation) |
| `direct` | Direct invocation (no protocol intermediary) |
| `http` | Generic HTTP |

### 12.3 Protocol Binding Requirements

A protocol binding specification MUST specify:

1. How the chain is attached to or referenced from protocol messages
2. Which protocol operations are governed
3. How `terminal_action.action_id` is derived from the protocol message
4. How verification failures are communicated in the protocol's error format

### 12.4 MCP Binding

For MCP governed operations (`tools/call`, `resources/read`, `resources/subscribe`, `prompts/get`):

- The chain or token MUST be attached in the `_meta` field of the MCP `params` object
- `terminal_action.action_id` SHALL be the JSON-RPC request `id`
- `terminal_action.operation` SHALL be the MCP method name
- `terminal_action.target` SHALL be the tool name, resource URI, or prompt name
- Verification failures SHALL be returned as JSON-RPC error responses with the ADCS error code in the `data` field

**Envelope example (`tools/call` with token-only mode):**

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "search_hotels",
    "arguments": { "destination": "Tokyo", "checkin": "2026-04-01" },
    "_meta": {
      "adcs_token": "<JWT capability token>"
    }
  },
  "id": "jsonrpc-id-7f3b"
}
```

**Envelope example (`tools/call` with full chain):**

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "search_hotels",
    "arguments": { "destination": "Tokyo", "checkin": "2026-04-01" },
    "_meta": {
      "adcs_chain": { "...full chain object..." }
    }
  },
  "id": "jsonrpc-id-7f3b"
}
```

**Error response:**

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "ADCS verification failed",
    "data": {
      "adcs_error_code": "PERMISSION_EXPANSION",
      "adcs_error_detail": "Delegation del-001 grants 'write' not present in parent scope",
      "chain_id": "chain-abc123"
    }
  },
  "id": "jsonrpc-id-7f3b"
}
```

### 12.5 A2A Binding

For A2A governed operations (`tasks/send`, `messages/send`):

- The chain or token MUST be attached in the A2A task or message `metadata` object under the key `adcs_chain` or `adcs_token`
- `terminal_action.action_id` SHALL be the task ID or message ID
- `terminal_action.operation` SHALL be `tasks/send` or `messages/send`
- `terminal_action.target` SHALL be the target agent identifier
- Verification failures SHALL be returned in the A2A error response with the ADCS error code

**Envelope example (`tasks/send` with token-only mode):**

```json
{
  "jsonrpc": "2.0",
  "method": "tasks/send",
  "params": {
    "id": "task-9f2a",
    "message": {
      "role": "user",
      "parts": [{ "type": "text", "text": "Find hotels in Tokyo for April 1-7" }]
    },
    "metadata": {
      "adcs_token": "<JWT capability token>"
    }
  }
}
```

**Error response:**

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "ADCS verification failed",
    "data": {
      "adcs_error_code": "CONSTRAINT_LOOSENED",
      "adcs_error_detail": "Delegation del-002 increased budget.max from 1200 to 1500",
      "chain_id": "chain-abc123"
    }
  },
  "id": "task-9f2a"
}
```

### 12.6 ACP Binding

For ACP (Agent Communication Protocol, IBM/Linux Foundation) governed operations:

- The chain or token MUST be attached in the ACP message envelope's `metadata` object under the key `adcs_chain` or `adcs_token`
- `terminal_action.action_id` SHALL be set to the ACP message `message_id` field
- `terminal_action.operation` SHALL be set to the ACP operation name (e.g., `invoke`, `query`, `notify`)
- `terminal_action.target` SHALL be set to the target agent's ACP endpoint identifier
- Verification failures SHALL be returned as ACP error responses with the ADCS error code in the error payload's `adcs_error_code` field

**Envelope example (`invoke` with token-only mode):**

```json
{
  "message_id": "msg-4e1c",
  "operation": "invoke",
  "target": "agent://tools.example.com/search-hotels",
  "payload": { "destination": "Tokyo" },
  "metadata": {
    "adcs_token": "<JWT capability token>"
  }
}
```

### 12.7 HTTP Generic Binding

For any HTTP-based tool invocation not covered by a more specific binding:

- The chain SHALL be carried in the request header `ADCS-Chain` (full chain, base64url-encoded) or `ADCS-Token` (capability token, JWT)
- The `action_id` SHALL be derived from the `X-Request-ID` request header. If absent, the server MUST generate a UUID and include it in the response as `X-Request-ID`
- `terminal_action.operation` SHALL be the HTTP method (e.g., `GET`, `POST`)
- `terminal_action.target` SHALL be the request URI path
- Verification failures SHALL be returned as HTTP 403 responses with a JSON body:

```json
{
  "error": "adcs_verification_failed",
  "adcs_error_code": "<error code from Section 15>",
  "adcs_error_detail": "<human-readable explanation>",
  "chain_id": "<chain_id if parseable>"
}
```

---

## 13. Cryptographic Requirements

### 13.1 Signing Algorithms

| Algorithm | Requirement | Notes |
|---|---|---|
| `ES256` (ECDSA P-256, SHA-256) | REQUIRED | Minimum for all conformant implementations |
| `ES384` (ECDSA P-384, SHA-384) | RECOMMENDED | Recommended for originator declarations |
| `EdDSA` (Ed25519) | OPTIONAL | MAY be supported |
| `RS256` (RSA-PKCS1, SHA-256) | MUST NOT | Not permitted |
| `RS512` (RSA-PSS, SHA-512) | OPTIONAL for manifest signing only | |

All signing algorithms MUST be declared in the JWK `alg` field. Implementations MUST reject signatures produced by algorithms not in this table.

### 13.2 Key Requirements

- All keys MUST be at least 256 bits in effective security strength
- Keys used for **originator declarations** MUST be stored in hardware-backed secure storage (HSM, TPM, or platform secure enclave) in production deployments
- Agent signing keys MUST NOT be hardcoded in source code, configuration files, or container images
- Keys MUST be stable for the lifetime of the issuing principal's deployment; ephemeral keys MUST NOT be used for delegation signing
- Keys MUST be rotated at intervals not exceeding 90 days
- Compromised keys MUST be revoked and revocation MUST be published to the trust anchor

### 13.3 Key Distribution via DID

Public keys for chain verification are resolved via DID document lookup at verification time. The `delegator.did` and `originator.did` fields are the authoritative key references. Implementations MUST resolve the current DID document at verification time and use the key designated for signing/verification in that document.

Implementations MAY cache DID document resolutions with a TTL not exceeding 1 hour for performance. Cached resolutions MUST be invalidated upon revocation notification.

### 13.4 Hash Requirements

- Chain digests and intermediate digests MUST use SHA-256
- Token binding thumbprints MUST use SHA-256 (per RFC 8705)
- Implementations MAY support SHA-384 or SHA-512 for chain digests; if used, the algorithm MUST be declared alongside the digest

### 13.5 Canonical Serialization

All signatures and digests are computed over JSON Canonicalization Scheme (JCS) per RFC 8785 (Section 5.4). Implementations MUST use a conformant JCS library and MUST NOT accept signatures computed over non-canonical forms.

### 13.6 Key Compromise Response

When a signing key is suspected or confirmed compromised, the following actions MUST be taken within the specified time windows:

| Action | Time Window |
|---|---|
| Revoke all delegation chains signed by the compromised key | Within 1 hour of confirmed compromise |
| Publish key revocation to the DID document (remove or rotate verification key) | Within 1 hour |
| Publish delegation revocation to the revocation service | Within 1 hour |
| Notify downstream verifiers via push revocation or CAEP event | Within 2 hours |
| Issue replacement key and re-establish agent identity | Within 24 hours |

Implementations MUST provide an emergency revocation endpoint that accepts authenticated key compromise reports and initiates this response automatically where feasible. The emergency endpoint URI SHOULD be declared in the revocation service metadata (Section 14.6).

---

## 14. Revocation

### 14.1 Delegation Revocation

A delegation MAY be revoked before its TTL expires. Revocation is appropriate when:

- The delegator determines the delegate should no longer act under delegated scope
- The delegation was issued in error
- The session is terminated by an external governance signal
- A signing key is compromised (Section 13.6)

### 14.2 Revocation Methods

Implementations MUST support at least one of the following:

**Method 1 — Revocation List:** A signed, timestamped list of revoked `delegation_id` values at a well-known endpoint.

```json
{
  "revocation_list_version": "1.0.0",
  "issued_at": "<ISO 8601 timestamp>",
  "valid_until": "<ISO 8601 timestamp>",
  "issuer": "<revocation authority DID>",
  "revoked": [
    {
      "delegation_id": "<uuid-v4>",
      "revoked_at": "<ISO 8601 timestamp>",
      "reason": "<human-readable reason>"
    }
  ],
  "signature": "<base64url-encoded signature>"
}
```

**Method 2 — Push Revocation:** The revocation authority pushes a signed revocation notice to registered verifiers. Format is a single entry from the revocation list, signed individually.

### 14.3 Originator Declaration Revocation

Originator declarations are subject to revocation in the same manner as delegation objects. When an originator declaration is revoked:

- All delegation chains whose `originator_declaration.delegation_id` matches the revoked ID are immediately invalid
- Verifiers MUST check the originator declaration's `delegation_id` against the revocation list (Step R.2 of Section 10.7)
- Error code `ORIGINATOR_REVOKED` MUST be returned

### 14.4 Token Revocation

Capability tokens MUST be revoked when:

- The delegation chain they reference is revoked
- The session they are associated with is halted
- The tool artifact they authorize has changed

Token revocation uses `token.jti` as identifier and follows the same methods as delegation revocation.

### 14.5 Revocation Check Timing

Implementations MUST check revocation at verification time. A chain passing all algorithm phases but referencing a revoked delegation MUST be rejected per Section 10.7.

### 14.6 Continuous Revocation Signals (CAEP)

Implementations SHOULD support the **Continuous Access Evaluation Protocol (CAEP)** for real-time revocation propagation during active chain execution.

CAEP provides an event stream by which a security event provider (the revocation authority) pushes session-change signals to subscribers (chain verifiers and governance layers) without requiring polling.

**Minimum required CAEP event types:**

| Event Type | Trigger | ADCS Response |
|---|---|---|
| `session.revoked` | Authorization session terminated | Immediately invalidate all chains for this session |
| `credential.compromise` | Signing key compromised | Revoke all chains signed by that key; trigger key compromise response (Section 13.6) |
| `token.claims_change` | Effective scope of a principal changed | Re-verify active chains for the affected principal |

Implementations that support CAEP MUST:

- Subscribe to the CAEP endpoint declared in the revocation service metadata for all active chain sessions
- Upon receiving a triggering event, immediately invalidate affected capability tokens and reject any in-flight verifications for the affected chain
- Log the CAEP event alongside the revocation action

### 14.7 Revocation Service Metadata

Every chain MUST reference a revocation service via `chain.revocation_service`. Verifiers discover revocation infrastructure by fetching the metadata document at that URI.

**Revocation service metadata document:**

```json
{
  "issuer": "<revocation authority DID>",
  "revocation_list_uri": "<URI of current revocation list>",
  "push_revocation_endpoint": "<URI for push revocation webhook registration>",
  "emergency_revocation_endpoint": "<URI for emergency key compromise reports>",
  "supported_methods": ["list", "push"],
  "max_list_staleness": "PT5M",
  "caep_endpoint": "<URI of CAEP event stream endpoint, if supported>",
  "signature": "<base64url signature over canonical metadata body>"
}
```

| Field | Required | Description |
|---|---|---|
| `issuer` | REQUIRED | DID of the revocation authority |
| `revocation_list_uri` | REQUIRED | URI of the current signed revocation list |
| `push_revocation_endpoint` | OPTIONAL | Webhook registration endpoint for push revocation |
| `emergency_revocation_endpoint` | OPTIONAL | Endpoint for key compromise emergency reports |
| `supported_methods` | REQUIRED | Array of supported revocation methods |
| `max_list_staleness` | REQUIRED | Maximum age of a cached revocation list (ISO 8601 duration) |
| `caep_endpoint` | OPTIONAL | CAEP event stream endpoint; REQUIRED if implementation declares CAEP support |
| `signature` | REQUIRED | Signature over canonical metadata body |

---

## 15. Error Codes

### 15.1 Structural Errors

| Code | Description |
|---|---|
| `VERSION_MISMATCH` | `adcs_version` does not match verifier's supported version |
| `MISSING_ORIGINATOR` | Originator declaration absent or null |
| `EMPTY_CHAIN` | Delegations array empty or absent |
| `MISSING_TERMINAL_ACTION` | Terminal action absent or null |
| `ACTION_ID_MISMATCH` | Terminal action's `action_id` does not match presented action |
| `PROTOCOL_MISMATCH` | Terminal action's protocol does not match |
| `OPERATION_MISMATCH` | Terminal action's operation does not match |
| `TARGET_MISMATCH` | Terminal action's target does not match |
| `BROKEN_CHAIN` | A delegation's parent reference does not resolve correctly |
| `INTENT_REFERENCE_MISMATCH` | A delegation's `intent_reference` does not match the originator declaration |
| `CHAIN_DIGEST_MISMATCH` | Computed chain digest does not match declared value |
| `TOKEN_CHAIN_MISMATCH` | Token's `chain_id` does not match the presented chain |
| `TOKEN_DIGEST_MISMATCH` | Token's `chain_digest` does not match the chain |
| `MISSING_REVOCATION_SERVICE` | `chain.revocation_service` is absent or empty |
| `DAG_NOT_SUPPORTED` | Chain contains DAG delegation structure but implementation does not support DAG |

### 15.2 Cryptographic Errors

| Code | Description |
|---|---|
| `INVALID_ORIGINATOR_SIGNATURE` | Originator declaration signature verification failed |
| `INVALID_DELEGATION_SIGNATURE` | Delegation signature verification failed |
| `INVALID_TOKEN_SIGNATURE` | Token signature verification failed |
| `POP_MISMATCH` | Token proof-of-possession does not match current transport credential |
| `MISSING_POP` | Token does not contain a `cnf` claim |
| `TOKEN_REPLAY` | Token `jti` has been seen before |
| `DID_RESOLUTION_FAILURE` | A principal's DID could not be resolved to a valid DID document |
| `QUORUM_INSUFFICIENT` | Originator quorum requires more valid signatures than are present |

### 15.3 Temporal Errors

| Code | Description |
|---|---|
| `ORIGINATOR_EXPIRED` | Originator declaration has expired (beyond clock skew tolerance) |
| `ORIGINATOR_TTL_INCONSISTENT` | `expires_at` ≠ `issued_at + ttl` in originator declaration |
| `DELEGATION_EXPIRED` | A delegation has expired |
| `DELEGATION_TTL_INCONSISTENT` | `expires_at` ≠ `issued_at + ttl` in a delegation |
| `TTL_EXCEEDS_PARENT` | A delegation's `expires_at` exceeds its parent's `expires_at` |
| `TOKEN_EXPIRED` | Token has expired |
| `TOKEN_NOT_YET_VALID` | Token `nbf` is in the future |
| `TOKEN_TTL_EXCEEDS_CHAIN` | Token `exp` exceeds chain's terminal delegation expiration |

### 15.4 Monotonic Restriction Errors

| Code | Description |
|---|---|
| `PERMISSION_EXPANSION` | A delegation grants a permission not present in the parent |
| `CONSTRAINT_LOOSENED` | A delegation loosens a constraint relative to the parent |
| `UNKNOWN_CONSTRAINT_MODIFIED` | A delegation modifies an unrecognized constraint type |
| `PERMISSION_CONDITION_LOOSENED` | A delegation promotes a permission from conditional to unconditional |
| `TOKEN_PERMISSIONS_MISMATCH` | Token's `effective_permissions` do not match computed value |
| `TOKEN_CONDITIONAL_PERMISSIONS_MISMATCH` | Token's `effective_conditional_permissions` do not match computed value |
| `TOKEN_CONSTRAINTS_MISMATCH` | Token's `effective_constraints` do not match computed value |
| `UNAUTHORIZED_SUBDELEGATION` | An agent creates a sub-delegation without holding `delegate` permission |
| `DELEGATION_DEPTH_EXCEEDED` | `max_delegation_depth` constraint has been exceeded |
| `DAG_MERGE_VIOLATION` | A DAG merge delegation grants permissions not present in the intersection of all parent branches |

### 15.5 Revocation Errors

| Code | Description |
|---|---|
| `ORIGINATOR_REVOKED` | The originator declaration has been revoked |
| `DELEGATION_REVOKED` | A delegation in the chain has been revoked |
| `TOKEN_REVOKED` | The capability token has been revoked |

### 15.6 Compaction Errors

| Code | Description |
|---|---|
| `INVALID_COMPACTION_SIGNATURE` | Compacted chain's compaction signature verification failed |
| `INTERMEDIATE_DIGEST_UNAVAILABLE` | Full chain could not be retrieved for high-assurance verification |

---

## 16. Security Considerations

### 16.1 Privilege Escalation Through Delegation

The monotonic restriction invariant (Section 4.5) is the primary defense against privilege escalation. The invariant is enforced by the verification algorithm, not by the agents themselves — a manipulated agent cannot grant its delegate broader scope than the agent received.

Implementers MUST ensure that the verification algorithm runs outside the agent runtime, in a component the agent cannot write to.

### 16.2 Chain Forgery

Chain forgery requires either a principal's private signing key or the ability to bypass both DID resolution and signature verification. Key security (Section 13.2) is the primary mitigation; DID-based key resolution prevents forged keys from being accepted as legitimate.

### 16.3 Chain Replay

The `terminal_action.action_id` binding prevents chain replay across different invocations. Token replay is prevented by nonce tracking (Step 2.5). Implementations MUST maintain a seen-tokens record for the session lifetime.

### 16.4 Confused Deputy via Scope Misrepresentation

The verification algorithm recomputes effective scope from first principles and compares against the token's declared values (Steps 4.4–4.5). Implementations MUST NOT skip this recomputation.

### 16.5 Semantic Constraint Mapping

> See Section 1.3 Warning and Section 9.4 Constraint Evaluation Capability Declaration.

ADCS verifies structural constraint presence and non-loosening. It does not evaluate whether a terminal action's parameters satisfy constraint values. Three partial approaches exist:

**Publisher-Declared Mappings:** Tool manifests declare which parameters map to which constraint types. Implementations SHOULD support this approach for MCP tools via an `adcs_constraint_mapping` field in the tool manifest:

```json
"adcs_constraint_mapping": {
  "price": "budget",
  "max_records": "max_results",
  "date_range": "time_range"
}
```

**Pattern Matching:** Heuristic rules map parameter names to constraint types. Effective for common cases; fragile for unusual naming conventions.

**LLM-Assisted Evaluation:** A model evaluates semantic alignment between parameters and constraints. Introduces latency and creates a recursive trust problem — the evaluating model is subject to the same manipulation risks as the governed agent.

Implementations MUST declare their approach per Section 9.4. Implementations using `structural_only` MUST NOT represent themselves as enforcing semantic constraints.

### 16.6 Long Chain Opacity

Deep delegation hierarchies can obscure terminal scope. Implementations SHOULD log the full chain at every verification event to preserve forensic auditability.

### 16.7 Cross-Protocol Trust Boundaries

When a chain crosses protocol boundaries, each delegation's signature is verified against the signing key resolved from that delegation's `delegator.did`. The DID document is the per-hop trust anchor. Implementations MUST NOT use a global trust store as a shortcut for cross-protocol chain verification.

### 16.8 Prompt Injection and Adversarial Chain Construction

**ADCS verification cannot detect whether a structurally valid chain was constructed by a manipulated agent.** A chain produced by an agent whose reasoning was hijacked via prompt injection will pass all four verification phases if the signatures are valid and the scope is monotonically restricted. The chain is cryptographically correct even though it authorizes an action the human originator never intended.

Prompt injection targeting agents with tool access was the most widely observed real-world attack against agentic systems in 2025. This threat is not theoretical.

Mitigations that governance layers SHOULD implement:

- **Human approval gates** on `purchase`, `delete`, `send_message`, and other high-consequence permissions, implemented via `permissions_conditional` with `condition_type: human_approval`
- **Anomaly detection** on `context_snapshot` fields: compare `task_state_hash` at chain construction time against state at execution time; flag deviations
- **Rate limits on chain construction**: an abnormal burst of chain construction requests from a single agent is a signal of compromise
- **Intent-action consistency checks**: governance layers SHOULD compare the originator's `intent_structured` fields (Section 8.8) against the terminal action's operation and permissions; a chain with `action_category: data_access` invoking a `purchase` tool is a signal worth human review. When `intent_structured` is absent, governance layers MAY fall back to the natural language `intent` field but SHOULD NOT rely on it for programmatic checks
- **Scope restriction at originator**: the `delegate` permission requirement limits which agents can extend a chain, reducing the surface area for injection-driven escalation

ADCS is not a substitute for a threat model that includes adversarial LLM inputs. It is a layer in a defense-in-depth architecture.

### 16.9 Chain Injection and Truncation

**Chain injection** — replacing the originator declaration and initial delegations with a forged prefix — requires forging valid signatures under the claimed delegators' DIDs. DID-based key resolution defeats this: a forged chain cannot produce a signature that verifies against the real delegator's DID document unless the attacker controls that DID.

**Chain truncation** — presenting a subchain starting from an intermediate hop — is defeated by the `intent_reference` requirement: every delegation must reference the originator declaration's `delegation_id`, and the originator declaration must be present. Step 1.2 (MISSING_ORIGINATOR) and Step 1.11 (INTENT_REFERENCE_MISMATCH) provide the verification.

Implementers SHOULD be aware that both attacks are structurally prevented by the algorithm and should test for them explicitly in their conformance test suites.

### 16.10 Delegation Chain Privacy

A full delegation chain reveals:

- Identities of all principals in the pipeline (via DID)
- The full permission and constraint structure
- Originator intent in natural language
- Organizational trust hierarchy

Third-party tool servers receiving full chains gain detailed intelligence about the calling system's architecture. Implementations SHOULD use token-only presentation mode (Section 7.7) for all third-party tool invocations. The full chain MUST remain available for audit retrieval but SHOULD NOT be transmitted beyond organizational trust boundaries as a default.

---

## 17. Conformance

### 17.1 Conformance Levels

**Level 1 — Core Conformance**

- Delegation object structure with DID-based principals (Section 5)
- Originator declaration structure including trust tier (Section 8)
- Delegation chain structure with revocation_service (Section 6)
- Complete verification algorithm, all four phases plus revocation check (Section 10)
- All core constraint types (Section 9.2)
- Fail-closed behavior (Section 10.9)
- All error codes in Section 15
- Constraint evaluation capability declaration (Section 9.4)

**Level 2 — Token Conformance**

Level 1 plus:

- Capability token structure with all required claims including `originator_trust_tier` (Section 7.2)
- Proof-of-possession via mTLS or DPoP (Section 7.5)
- Token replay prevention (Section 10.4, Step 2.5)
- Token revocation checking (Section 14.4)
- Token-only presentation mode support (Section 7.7)

**Level 3 — Full Conformance**

Level 2 plus:

- Chain compaction for chains exceeding five delegations (Section 11)
- At least one protocol binding (Sections 12.4–12.7)
- At least one revocation method (Section 14.2)
- Revocation service metadata endpoint (Section 14.7)
- CAEP continuous revocation support (Section 14.6)
- DAG workflow chain support (Section 6.6) — OPTIONAL at Level 3
- Macaroon token support (Section 7.6) — OPTIONAL at Level 3
- Multi-party originator quorum support (Section 8.5)
- Permission escalation protocol (Section 10.10)

### 17.2 Conformance Test Suite

A conformance test suite is maintained alongside this standard. Implementations MUST pass the test suite for their declared conformance level. The test suite is normative. Test vectors are provided in Appendix D.

Test categories:

- Structural verification: all valid structures accepted; all malformed structures rejected
- DID resolution: valid DIDs resolve correctly; unresolvable DIDs produce `DID_RESOLUTION_FAILURE`
- Cryptographic verification: valid signatures accepted; invalid, tampered, and missing signatures rejected
- Temporal verification: expired chains rejected; TTL exceeds parent rejected; inconsistent TTL rejected; clock skew tolerance applied correctly
- Monotonic restriction: all expansion cases rejected; all constraint loosening cases rejected; all recomputation mismatches rejected; `UNAUTHORIZED_SUBDELEGATION` for missing `delegate` permission
- Conditional permissions: promotion from conditional to unconditional rejected; correct multi-hop tracking
- Originator quorum: sufficient quorum accepted; insufficient quorum rejected
- DAG chains: valid DAG topology accepted; invalid topology rejected; merge semantics correct
- Context snapshot: chains with and without `context_snapshot` accepted without verification impact
- Proof-of-possession: mismatched thumbprints rejected; absent `cnf` rejected
- Token replay: second presentation of seen `jti` rejected
- Revocation: revoked originator, delegation, and token correctly rejected
- Fail-closed: any verification failure results in denial
- Error codes: all failures produce correct error codes
- Cross-protocol: chains spanning registered protocols verified correctly

### 17.3 AILS Conformance (OPTIONAL capability)

Implementations at any ADCS Conformance Level (1, 2, or 3) MAY additionally declare AILS (AI Logging Standard) conformance per §23.7. Declaring AILS conformance is OPTIONAL. It is not required for any ADCS Conformance Level and does not affect ADCS Level 1/2/3 satisfaction. Implementations that do not declare AILS conformance remain fully conformant ADCS implementations at their declared level.

Implementations that declare AILS conformance MUST emit the audit events defined in §23.2 in the AILS Event Envelope form specified in AILS §5 and MUST meet the integrity and retention requirements of the declared AILS Level as defined in §23.6 and §23.7.

Cross-standard conformance claims follow the format prescribed by AILS §28.5.2. Example claims:

- `"ADCS v1.0.0 Level 2 Conformant; AILS v1.0.0 Level 2 audit events emitted"`
- `"ADCS v1.0.0 Level 3 Conformant; AILS v1.0.0 Level 3 audit events emitted"`
- `"ADCS v1.0.0 Level 1 Conformant; AILS v1.0.0 Level 1 audit events emitted"`

An implementation that emits a subset of the §23.2 events in AILS envelope form but not all of them MUST NOT claim full AILS conformance; partial conformance MAY be claimed per AILS §28.7.1 with explicit enumeration of the event types satisfied.

---

## 18. Relationship to Other Standards

### 18.1 ALS (Agent-Layer Security Standard)

ALS provides session-level circuit breaking — whether an agent session may continue to execute. ADCS provides action-level scope authorization — whether a specific action is within the delegated scope. Both are required and neither replaces the other.

A valid ADCS chain does not override an ALS halt. An ALS NORMAL state does not substitute for ADCS chain verification.

### 18.2 OAuth 2.1

OAuth 2.1 establishes identity and initial authorization at session start. ADCS operates after OAuth authorization — tracking how that authorization was subdivided and delegated across agent hops. OAuth tokens MAY be referenced in the originator declaration `context` field. ADCS may be used with other identity systems; it does not depend on OAuth.

### 18.3 MCP Integrity Standard

The MCP Integrity Standard establishes that a tool artifact is what it claims to be. ADCS establishes that the agent invoking the tool has the scope to do so. Publisher-declared constraint mappings (Section 16.5) provide the bridge between these two standards.

### 18.4 CAEP (Continuous Access Evaluation Protocol)

CAEP (OpenID Foundation) defines an event stream for pushing session-change signals to relying parties in real time. ADCS integrates with CAEP for continuous revocation propagation during active chain execution (Section 14.6). CAEP is the mechanism by which changes in authorization state — compromise events, session revocation, claims changes — reach active ADCS verifiers without polling latency.

### 18.5 Related Work and Engineering Precedents

**UCAN (User Controlled Authorization Networks):** An independently developed capability chain standard for decentralized storage and compute. Key architectural difference: ADCS targets centralized enterprise AI workflows with typed constraint registries; UCAN targets decentralized peer-to-peer contexts with URI-scoped capabilities. ADCS adopts DID-based principal identification from lessons learned in UCAN's production deployment experience.

**Macaroons (Birgisson et al., 2014):** Contextual caveats as an attenuation mechanism. ADCS notes Macaroon support as optional (Section 7.6); JWT is the baseline because JWT tooling is ubiquitous in enterprise environments and JWT claims are more structurally typed than Macaroon caveats.

**SPIFFE/SPIRE:** Workload identity for cloud-native microservices. The agent identity provisioning problem ADCS faces — how agent runtimes receive stable, verifiable identities bound to their task context — is closely related to what SPIFFE solves for ephemeral workloads. Enterprise deployments SHOULD evaluate SPIFFE/SVID as the identity substrate for `did:web` or `did:jwk` principals in containerized agent environments. See Appendix C.

**IETF Transaction Tokens (draft-ietf-oauth-transaction-tokens):** Immutable, propagated call-chain context for distributed systems. Transaction Tokens address call-chain context preservation; ADCS addresses scope restriction within that chain. They are complementary: Transaction Tokens establish *who* is calling *what* in a distributed request; ADCS establishes *what they are authorized to do*.

**IETF Identity Chaining (draft-ietf-oauth-identity-chaining):** Cross-domain OAuth identity preservation. Addresses identity propagation across organizational boundaries; ADCS addresses permission restriction within a task chain. In cross-organization deployments, Identity Chaining and ADCS serve different layers of the same trust problem.

### 18.6 AILS (AI Logging Standard)

AILS (AI Logging Standard) defines a canonical event envelope, event taxonomy, graduated integrity model, provenance classification, redaction framework, and retention tiers for AI system telemetry and audit records. It is the intended reference audit event format for ADCS.

The ADCS-to-AILS event mapping is defined normatively in §23 (Audit Event Emission). ADCS chain verification outcomes, capability token lifecycle events, delegation hop creation, permission escalation requests, condition satisfaction records, and revocation events map directly to AILS event types in the `authority.*`, `agent.*`, `policy.*`, `human.*`, `session.*`, and `credential.*` categories. The AILS Event Envelope (AILS §5, Appendix A) is the reference serialization for ADCS audit events; alternative serializations are permitted per §23.5 provided event type names, payload field names, and semantic values are preserved.

ADCS does not require AILS adoption. An ADCS implementation that emits audit records in OpenTelemetry log form, syslog RFC 5424, CEF, or a proprietary format is fully conformant to ADCS §22.6 and §23 provided the event type names and payload field names defined in §23 are preserved. Declaring AILS conformance per §23.7 is an OPTIONAL capability that makes ADCS audit events natively ingestible by AILS-aware consumers — dashboards, SIEMs, log pipelines, and cross-standard correlation tools — without custom transformation.

AILS v1.0.0 does not currently list ADCS in its own cross-standard event matrix (AILS §29, Appendix E). A complementary AILS update proposing an §29.12 subsection and an Appendix E.10 entry mirroring the mapping defined in §23.2 of this standard is planned for a future AILS revision.

---

## 19. Versioning

This standard uses semantic versioning: `MAJOR.MINOR.PATCH`.

- **MAJOR** increments indicate breaking changes to the delegation object, chain structure, or verification algorithm
- **MINOR** increments indicate backwards-compatible additions
- **PATCH** increments indicate clarifications, errata, and non-normative changes

Verifiers MUST reject objects with a MAJOR version they do not support. Verifiers SHOULD accept objects with a higher MINOR version by treating unrecognized optional fields as absent.

---

## 20. Glossary

| Term | Definition |
|---|---|
| **Action** | A concrete operation executed by an agent across a protocol boundary |
| **ADCS** | Agent Delegation Chain Standard |
| **CAEP** | Continuous Access Evaluation Protocol (OpenID Foundation). Defines real-time security event streams for session-change propagation |
| **Canonical JSON** | JSON serialized per RFC 8785 (JSON Canonicalization Scheme): keys sorted lexicographically, UTF-8 encoded, no trailing whitespace, no pretty-printing, numbers per RFC 8785 §3.2.2.3 |
| **Capability Token** | A short-lived, transport-bound JWT carrying a reference to a delegation chain and the computed effective scope |
| **CAS** | Constraint Accounting Service. An optional deployment component that tracks cumulative constraint usage across multiple chains from the same originator |
| **Chain Compaction** | A procedure for summarizing intermediate delegation hops in a long chain while preserving auditability |
| **Clock Skew Tolerance** | A configurable duration applied to temporal verification checks to accommodate clock drift across distributed systems (default PT60S) |
| **Conditional Permission** | A permission granted to a delegate but exercisable only after an external condition is satisfied |
| **Condition Satisfaction Record** | A cryptographically signed record providing evidence that a conditional permission's condition has been met (Section 5.6) |
| **Constraint** | A typed restriction on the scope of a delegation |
| **Context Snapshot** | An optional observational record of runtime state at chain construction time. Not subject to verification |
| **DAG Chain** | A delegation chain whose structure forms a directed acyclic graph, supporting parallel agent branches that converge at a merge delegation |
| **Delegation** | A signed record expressing that a delegator has granted a delegate a subset of the delegator's scope |
| **Delegation Chain** | An ordered sequence or DAG of delegations from an originator declaration to a terminal action |
| **DID** | Decentralized Identifier. A globally unique, resolvable identifier conforming to the W3C DID Core specification, resolving to a DID document containing public key material |
| **Effective Scope** | The intersection of all permissions and union of all constraints computed from the originator to a given delegation hop |
| **Fail-Closed** | Any verification failure results in denial; no partial validity |
| **Intent** | The natural language description of the task being authorized, declared by the originator and preserved throughout the chain |
| **JIT Identity Provisioning** | Just-in-time identity provisioning: agent receives a scoped, ephemeral identity bound to its task at startup and retired upon task completion |
| **Key Provenance** | Classification of how a principal's public key was obtained: `did_resolved` (from DID document), `registry_backed` (from organizational registry), or `self_declared` (self-asserted). Recorded in audit logs for trust policy evaluation |
| **Monotonic Restriction Invariant** | Scope can only narrow at each hop; permissions can be removed and constraints tightened, but neither can be expanded |
| **Originator Declaration** | The root of a delegation chain, signed by the originating human or system, declaring initial intent and maximum scope |
| **Originator Trust Tier** | The classification of how strongly the originator signature cryptographically proves human authorization: `hardware_bound`, `host_mediated`, or `policy_derived` |
| **Permission** | A named capability that may be granted or withheld through delegation |
| **Permission Escalation** | A standardized protocol by which an agent may request an additional permission mid-task without abandoning the chain |
| **Principal** | Any entity that can participate in a delegation: human, system, or agent |
| **Proof-of-Possession (PoP)** | A cryptographic mechanism binding a capability token to the holder's transport credential |
| **Quorum** | A multi-party originator authorization requiring signatures from at least N of M designated principals |
| **Terminal Action** | The specific protocol operation that a delegation chain authorizes |
| **Token-Only Mode** | A capability token presentation mode in which only the token, not the full chain, is transmitted to the terminal tool server |

---

## 21. References

- **RFC 2119** — Key words for use in RFCs. Bradner, 1997
- **RFC 7519** — JSON Web Token (JWT). Jones, Bradley, Sakimura, 2015
- **RFC 7517** — JSON Web Key (JWK). Jones, 2015
- **RFC 7518** — JSON Web Algorithms (JWA). Jones, 2015
- **RFC 8785** — JSON Canonicalization Scheme (JCS). Rundgren, Jordan, Erdtman, 2020
- **RFC 7800** — Proof-of-Possession Key Semantics for JSON Web Tokens. Jones, Bradley, Sakimura, 2016
- **RFC 8392** — CBOR Web Token (CWT). 2018
- **RFC 8414** — OAuth 2.0 Authorization Server Metadata. Jones, Sakimura, Bradley, 2018
- **RFC 8705** — OAuth 2.0 Mutual-TLS Client Authentication. Campbell et al., 2020
- **RFC 9449** — OAuth 2.0 Demonstrating Proof of Possession (DPoP). Fett et al., 2023
- **RFC 8446** — TLS 1.3. Rescorla, 2018
- **OAuth 2.1** — The OAuth 2.1 Authorization Framework. Hardt, Parecki, Lodderstedt (draft)
- **W3C DID Core** — Decentralized Identifiers (DIDs) v1.0. W3C Recommendation, 2022
- **Macaroons** — Macaroons: Cookies with Contextual Caveats. Birgisson et al., Google Research, 2014
- **ALS** — Agent-Layer Security Standard v2.1.0, 2026
- **AILS** — AI Logging Standard v1.0.0, 2026
- **CAEP** — Continuous Access Evaluation Protocol. OpenID Foundation, 2023
- **UCAN** — User Controlled Authorization Networks. Fission/DAG House, 2022
- **SPIFFE** — Secure Production Identity Framework for Everyone. CNCF, 2018
- **draft-ietf-oauth-transaction-tokens** — Transaction Tokens. IETF OAuth WG (active draft)
- **draft-ietf-oauth-identity-chaining** — Identity Chaining Across Trust Domains. IETF OAuth WG (active draft)
- **ACP** — Agent Communication Protocol. IBM / Linux Foundation AI & Data, 2024

---

## 22. Deployment Topologies

*This section is non-normative. It provides reference architectures to assist implementers in understanding how ADCS components are deployed and operated.*

### 22.1 Centralized Enterprise

**Scenario:** Single organization, all agents within one trust domain.

**Components:**
- **Agent Registry:** Maintains DID documents and agent metadata for all organizational agents. Accessible by all chain verifiers.
- **Originator Service:** Host application that creates and signs originator declarations on behalf of authenticated users.
- **Governance Layer:** Verifies chains, evaluates constraints (where semantic mapping is available), and records verification events.
- **Chain Ledger:** Stores full chains for audit retrieval; indexed by `chain_id`.
- **Revocation Service:** Publishes revocation lists and push notifications. Exposes CAEP event stream for internal subscribers.

**Trust model:** All agents share a common CA or organizational DID infrastructure. The governance layer is a trusted internal service.

**Conformance target:** Level 3. CAEP support RECOMMENDED.

### 22.2 Federated Multi-Organization

**Scenario:** Two or more organizations each operating their own agent infrastructure, with chains crossing organizational boundaries (e.g., a customer's orchestrating agent delegating to a vendor's specialist agent).

**Key differences from centralized:**
- Each organization operates its own Agent Registry, Revocation Service, and Governance Layer
- Cross-org chains include delegations signed by principals from different DID domains
- DID resolution must traverse multiple DID method implementations
- Revocation propagation must cross organizational boundaries via CAEP or push revocation

**Requirements:**
- Each organization MUST expose a publicly accessible revocation service metadata endpoint
- Cross-org verifiers MUST resolve DIDs from external DID registries
- Organizations MUST establish mutual agreement on supported DID methods before chain exchange
- `originator_trust_tier` carries trust tier information across boundaries; receiving organizations MAY downgrade effective trust tier per their own policy

### 22.3 Embedded / Offline

**Scenario:** Agents operating without persistent network access (edge deployments, air-gapped environments, mobile contexts).

**Key differences:**
- DID resolution may not be available at verification time
- Revocation list polling is not viable; push revocation may not be reachable
- Full chains are transmitted with each invocation rather than referenced by locator

**Adaptations permitted:**
- Inline `public_key` with `key_provenance: self_declared` (Section 4.2) MAY be used when DID resolution is unavailable
- Short TTLs (≤15 minutes) MUST be used in lieu of real-time revocation checks
- The `revocation_service` field MUST still be present but MAY reference a local or cached endpoint
- Chain compaction SHOULD be used aggressively to limit serialization size

**Conformance target:** Level 1 or Level 2. Level 3 revocation requirements may be relaxed to short-TTL-only mode with explicit capability declaration.

### 22.4 Incremental Adoption Guidance

Organizations will not deploy ADCS everywhere simultaneously. The following adoption path minimizes disruption:

**Phase 1 — Boundary Enforcement Only:** Implement ADCS verification at terminal tool servers and external-facing agent endpoints only. Internal agent-to-agent delegations continue to use ambient trust. Chains are created at the boundary between the internally-trusted zone and the verified zone.

**Phase 2 — Orchestrator Adoption:** Implement ADCS chain construction in orchestrating agents. Terminal tools continue to verify chains from Phase 1.

**Phase 3 — Full Chain Depth:** Roll out chain construction to all agent types. Each agent constructs delegations and passes chains to downstream agents.

**Passthrough mode:** A non-ADCS-aware component in the middle of a chain MUST forward the `adcs_chain` or `adcs_token` metadata field unchanged. This is a passthrough contract, not a verification contract. Non-aware components do not verify; they do not strip or modify chain data.

**Minimum viable deployment:** Phase 1 provides meaningful security guarantees: the terminal action is verified, the full chain is checked, and scope restriction is enforced at the point of consequence. This is achievable without modifying internal agent implementations.

### 22.5 Performance Considerations

**Reference chain sizes:**

| Chain Depth | Approx. Size (no compaction) | Approx. Size (compacted) |
|---|---|---|
| 1 hop | ~2 KB | N/A |
| 3 hops | ~5 KB | N/A |
| 5 hops | ~8 KB | ~4 KB |
| 10 hops | ~15 KB | ~4.5 KB |

Sizes assume DID strings (~60 chars), ES256 signatures (~128 bytes base64), and a moderate constraint set. Inline JWK fast-path keys (~300 bytes each) significantly increase these estimates.

**Verification latency targets by conformance level:**

| Level | Target (excluding DID resolution) | Target (with cached DID resolution) |
|---|---|---|
| Level 1 | < 10 ms for 5-hop chain | < 5 ms |
| Level 2 | < 15 ms for 5-hop chain | < 8 ms |
| Level 3 (with compaction) | < 20 ms | < 10 ms |

DID resolution introduces variable network latency. Implementations MUST cache DID document resolutions (TTL ≤ 1 hour) and SHOULD warm the cache before high-frequency verification workloads.

**Verification result caching:** Implementations MAY cache successful verification results for a chain using the key `chain_digest + current_time_bucket` (where `time_bucket` is a rounded interval shorter than the minimum TTL in the chain). Cached results MUST be invalidated immediately upon receiving a revocation event for any component of the chain.

### 22.6 Audit Log Requirements

Implementations MUST maintain an audit log of all chain verification events. The fields below are the minimum per-event audit record. Implementations SHOULD emit audit events per §23 (Audit Event Emission), which defines normative event types aligned with AILS and supersedes this minimum-field list when adopted. The following fields MUST be recorded per event:

| Field | Description |
|---|---|
| `event_time` | ISO 8601 timestamp of the verification event |
| `chain_id` | The chain being verified |
| `result` | `VALID` or `INVALID` |
| `error_code` | Error code if INVALID; absent if VALID |
| `terminal_action` | Protocol, operation, and target of the authorized action |
| `originator_did` | DID of the chain's originator |
| `terminal_actor_did` | DID of the terminal delegate |
| `originator_trust_tier` | Trust tier of the originator |
| `effective_permissions` | Effective permissions at the terminal hop (if VALID) |
| `verifier_id` | Identifier of the verifying component |

Audit logs MUST be retained for a minimum of 90 days. Deployments subject to regulatory requirements (SOX, HIPAA, GDPR, etc.) MUST apply the longer of 90 days and their applicable regulatory retention period. Full chains MUST be retrievable by `chain_id` for the duration of the audit retention period.

### 22.7 ADCS Configuration Discovery

Implementations SHOULD expose a well-known discovery endpoint that allows clients to determine supported capabilities before constructing or presenting chains. This prevents opaque `VERSION_MISMATCH` and `DAG_NOT_SUPPORTED` failures in cross-organization deployments.

**Endpoint:** `GET /.well-known/adcs-configuration`

**Response:**

```json
{
  "adcs_versions_supported": ["1.0.0"],
  "conformance_level": 2,
  "did_methods_supported": ["did:web", "did:jwk", "did:key"],
  "key_provenance_accepted": ["did_resolved", "registry_backed", "self_declared"],
  "constraint_evaluation": "structural_only",
  "semantic_mapping_method": "none",
  "dag_support": false,
  "macaroon_support": false,
  "caep_support": true,
  "ails_conformance": "none",
  "revocation_service": "https://trust.example.com/.well-known/adcs-revocation",
  "cas_endpoint": "https://trust.example.com/adcs/cas/reserve",
  "max_chain_depth": 20
}
```

| Field | Required | Description |
|---|---|---|
| `adcs_versions_supported` | REQUIRED | Array of supported `adcs_version` strings |
| `conformance_level` | REQUIRED | Highest conformance level supported (1, 2, or 3) |
| `did_methods_supported` | REQUIRED | Array of accepted DID method prefixes |
| `key_provenance_accepted` | REQUIRED | Array of accepted `key_provenance` values |
| `constraint_evaluation` | REQUIRED | `structural_only` or `semantic` (Section 9.4) |
| `semantic_mapping_method` | REQUIRED | `none`, `publisher_declared`, `pattern_matching`, or `llm_assisted` |
| `dag_support` | REQUIRED | Whether DAG chains are accepted |
| `macaroon_support` | REQUIRED | Whether Macaroon tokens are accepted |
| `caep_support` | REQUIRED | Whether CAEP continuous revocation is supported |
| `ails_conformance` | REQUIRED | Declared AILS (AI Logging Standard) conformance level for audit event emission. One of `"none"`, `"level_1"`, `"level_2"`, `"level_3"`. See §23.7 |
| `revocation_service` | REQUIRED | URI of revocation service metadata |
| `cas_endpoint` | OPTIONAL | URI of Constraint Accounting Service, if deployed |
| `max_chain_depth` | REQUIRED | Maximum accepted chain depth |

The discovery document SHOULD be cacheable with a `Cache-Control` header of `max-age=3600`.

---

## 23. Audit Event Emission

### 23.1 Purpose

ADCS defines a normative audit event taxonomy aligned with the AI Logging Standard (AILS) so that ADCS audit events are portable across observability and security tooling. Implementations emit events describing chain verification outcomes, capability token lifecycle, delegation hop creation, permission escalation, condition satisfaction, and revocation in a form that any AILS-aware consumer — dashboards, SIEMs, log pipelines, cross-standard correlation tools — can ingest without custom transformation.

Implementations MAY declare AILS conformance (§23.7) to obtain native interoperability with AILS consumers. Declaring AILS conformance is OPTIONAL. Implementations that do not declare AILS conformance remain fully conformant ADCS implementations and MAY emit §23.2 events in any serialization format provided the event type names, payload field names, and semantic values defined in this section are preserved.

This section supersedes the minimum per-event audit record in §22.6 when adopted. §22.6 remains the minimum field set for v1.0.0-only implementations and for implementations that choose not to adopt the event-based model.

### 23.2 Event Type Mapping (Normative)

The following table defines the normative mapping from ADCS triggers to AILS event types. Implementations emitting §23 audit events MUST emit an event of the specified AILS event type for each listed ADCS trigger, with at minimum the listed required payload fields populated. AILS event type names and payload field names are drawn from AILS Appendix B (Event Type Registry) and from the payload requirements in AILS §11, §12, §13, §14, and §15.

| ADCS Trigger | AILS Event Type | Required Payload Fields |
|---|---|---|
| Chain verification succeeds (§10.8 VALID) | `authority.gate_passed` | `gate_id`, `token_id` (= token `jti`), `action_id` (= `terminal_action.action_id`), `validations_performed` (array containing at minimum `"signature_verified"`, `"temporal_valid"`, `"monotonic_valid"`, `"pop_verified"`, `"not_revoked"`), `validated_at` |
| Chain verification fails (§10.8 INVALID) | `authority.gate_denied` | `gate_id`, `action_id`, `denial_reason` (mapped from ADCS error code per §23.3), `token_id` (if a token was presented), `denied_at`, `escalation_triggered` |
| Capability token issued (§7) | `authority.token_issued` | `token_id` (= `jti`), `action_id` (= `terminal_action.action_id`), `scope` (object containing `effective_permissions`, `effective_constraints`, `originator_trust_tier`), `issued_at`, `expires_at` (= token `exp`), `issuer` (= token `iss`) |
| Capability token consumed at terminal action | `authority.token_consumed` | `token_id`, `action_id`, `consumed_at`, `consumed_by`, `execution_result` (`"success"`, `"error"`, or `"partial"`) |
| Capability token reaches `exp` without consumption | `authority.token_expired` | `token_id`, `action_id`, `issued_at`, `expired_at`, `reason` (`"timeout"`, `"plan_cancelled"`, `"superseded"`, or implementation-specific value) |
| Delegation or token revoked (§14) | `authority.token_revoked` | `token_id` (= `jti` for token revocation; set to `delegation_id` for delegation-object revocation — see convention note below), `action_id`, `revoked_at`, `revoked_by`, `revocation_reason` (mapped per revocation-reason convention below) |
| Each delegation hop creation | `agent.delegation` | `delegating_agent_id` (= `delegator.did`), `receiving_agent_id` (= `delegate.did`), `delegation_type` (`"sub_goal"` for in-protocol delegation; `"custom"` for cross-protocol delegation per §12), `delegation_depth`, `constraints` (= `constraints_added` from the delegation object), `status` (`"initiated"` at hop creation) |
| Permission escalation request issued (§10.10) | `policy.escalation` | `escalation_id`, `escalation_reason` (`"scope_boundary"`), `severity`, `escalated_at`, `execution_state` (`"halted"`), `action_pending` (= `requested_permission`), `authority_frozen` (= `true`), `session_id` (= `chain_id`) |
| Escalation approved or denied | `human.escalation_response` | `escalation_event_id`, `response_action` (`"resume"`, `"terminate"`, or `"modify_and_resume"`), `reviewer_id`, `reviewed_at`, `authority_granted` |
| Condition satisfaction record attached (§5.6) | `human.confirmation` | `confirmation_type` (`"action_approval"`), `target_event_id` (= the `event_id` of the corresponding `authority.token_issued` event), `decision` (`"approved"`), `reviewer_id` (= `satisfied_by`), `reviewed_at` (= `satisfied_at`), `scope` (object with `action` = the permission being confirmed and `resource` = `terminal_action.target`) |
| Originator declaration revoked (§14.3) | `policy.violation` | `violation_type` (`"unauthorized_action"`), `severity` (`"critical"`), `policy_name` (`"adcs_originator_revocation"`), `description`, `detected_at`, `violating_entity` (= `originator.did`), `action_taken` (`"blocked"`) |
| CAEP `session.revoked` received (§14.6) | `session.state_changed` | `session_id` (= `chain_id`), `previous_state`, `new_state` (`"terminated"`), `transition_trigger` (`"policy"`), `triggered_by`, `transitioned_at`, `reason` |
| CAEP `credential.compromise` received (§14.6) | `authority.token_revoked` | Emitted once per affected token with `revocation_reason` = `"security_incident"` (see revocation-reason convention below). All other required fields as in the "Delegation or token revoked" row. |
| DID verification key rotated (§13.3, Appendix C.4) | `credential.renewed` | `credential_type` = `"custom"` (with a payload annotation documenting that the custom credential is an ADCS DID verification key), `previous_fingerprint` (thumbprint of the retired verification key), `new_fingerprint` (thumbprint of the new verification key), `renewed_at`, `valid_from`, `valid_until` |

**Revocation-reason convention for `authority.token_revoked`:** ADCS revocation reasons MUST be mapped to AILS `revocation_reason` enum values as follows. The AILS enum values for this field are fixed by AILS §12.5.1: `policy_change`, `security_incident`, `scope_violation`, `manual_override`, `session_terminated`, `custom`.

| ADCS revocation cause | AILS `revocation_reason` value |
|---|---|
| Organizational policy change revokes scope | `policy_change` |
| Signing key compromise (§13.6) | `security_incident` |
| CAEP `credential.compromise` received | `security_incident` |
| Constraint or permission scope breach detected | `scope_violation` |
| Administrative or operator revocation | `manual_override` |
| Session termination cascades to tokens | `session_terminated` |
| Any other deployment-specific reason (including key compromise classified as an operational rather than security event, and chain-abandonment cascades) | `custom` |

**`token_id` vs. `delegation_id` convention for `authority.token_revoked`:** ADCS revocation operates over two distinct identifier spaces — capability token `jti` (§14.4) and delegation/originator `delegation_id` (§14.1, §14.3). When a `authority.token_revoked` event records revocation of a capability token, the AILS `token_id` field MUST be populated with the token's `jti`. When a `authority.token_revoked` event records revocation of a delegation object or an originator declaration, the AILS `token_id` field MUST be populated with the revoked `delegation_id` and the event payload SHOULD include an implementation-specific annotation (e.g., under `context.labels` or an `x-adcs.revoked_identifier_kind` payload field) distinguishing `"token_jti"` from `"delegation_id"` to permit unambiguous downstream correlation with ADCS chain objects. Originator declaration revocation additionally triggers a `policy.violation` event per the table above; the two events share the same `correlation.session_id` (= `chain_id`).

### 23.3 Error Code to Denial Reason Mapping (Normative)

When a chain verification failure produces an `authority.gate_denied` event, the event's `denial_reason` payload field MUST be set per the following mapping. The AILS `denial_reason` enum values are fixed by AILS §12.7.1: `no_token`, `token_expired`, `token_revoked`, `binding_mismatch`, `scope_exceeded`, `intent_mismatch`, `plan_mismatch`, `tenant_mismatch`, `policy_denied`. Implementations MUST include the original ADCS error code in the event payload (e.g., under an `x-adcs.error_code` field or within `context.labels`) to preserve full diagnostic detail.

**Structural errors (§15.1) → `policy_denied`** (except where noted):

| ADCS Error Code | AILS `denial_reason` |
|---|---|
| `VERSION_MISMATCH` | `policy_denied` |
| `MISSING_ORIGINATOR` | `policy_denied` |
| `EMPTY_CHAIN` | `policy_denied` |
| `MISSING_TERMINAL_ACTION` | `policy_denied` |
| `ACTION_ID_MISMATCH` | `intent_mismatch` |
| `PROTOCOL_MISMATCH` | `intent_mismatch` |
| `OPERATION_MISMATCH` | `intent_mismatch` |
| `TARGET_MISMATCH` | `intent_mismatch` |
| `BROKEN_CHAIN` | `policy_denied` |
| `INTENT_REFERENCE_MISMATCH` | `intent_mismatch` |
| `CHAIN_DIGEST_MISMATCH` | `policy_denied` |
| `TOKEN_CHAIN_MISMATCH` | `binding_mismatch` |
| `TOKEN_DIGEST_MISMATCH` | `binding_mismatch` |
| `MISSING_REVOCATION_SERVICE` | `policy_denied` |
| `DAG_NOT_SUPPORTED` | `policy_denied` |

**Cryptographic errors (§15.2):**

| ADCS Error Code | AILS `denial_reason` |
|---|---|
| `INVALID_ORIGINATOR_SIGNATURE` | `policy_denied` |
| `INVALID_DELEGATION_SIGNATURE` | `policy_denied` |
| `INVALID_TOKEN_SIGNATURE` | `policy_denied` |
| `POP_MISMATCH` | `binding_mismatch` |
| `MISSING_POP` | `binding_mismatch` |
| `TOKEN_REPLAY` | `binding_mismatch` |
| `DID_RESOLUTION_FAILURE` | `policy_denied` |
| `QUORUM_INSUFFICIENT` | `policy_denied` |

**Temporal errors (§15.3):**

| ADCS Error Code | AILS `denial_reason` |
|---|---|
| `ORIGINATOR_EXPIRED` | `token_expired` |
| `ORIGINATOR_TTL_INCONSISTENT` | `policy_denied` |
| `DELEGATION_EXPIRED` | `token_expired` |
| `DELEGATION_TTL_INCONSISTENT` | `policy_denied` |
| `TTL_EXCEEDS_PARENT` | `scope_exceeded` |
| `TOKEN_EXPIRED` | `token_expired` |
| `TOKEN_NOT_YET_VALID` | `token_expired` |
| `TOKEN_TTL_EXCEEDS_CHAIN` | `scope_exceeded` |

**Monotonic restriction errors (§15.4):**

| ADCS Error Code | AILS `denial_reason` |
|---|---|
| `PERMISSION_EXPANSION` | `scope_exceeded` |
| `CONSTRAINT_LOOSENED` | `scope_exceeded` |
| `UNKNOWN_CONSTRAINT_MODIFIED` | `scope_exceeded` |
| `PERMISSION_CONDITION_LOOSENED` | `scope_exceeded` |
| `TOKEN_PERMISSIONS_MISMATCH` | `scope_exceeded` |
| `TOKEN_CONDITIONAL_PERMISSIONS_MISMATCH` | `scope_exceeded` |
| `TOKEN_CONSTRAINTS_MISMATCH` | `scope_exceeded` |
| `UNAUTHORIZED_SUBDELEGATION` | `scope_exceeded` |
| `DELEGATION_DEPTH_EXCEEDED` | `scope_exceeded` |
| `DAG_MERGE_VIOLATION` | `scope_exceeded` |

**Revocation errors (§15.5):**

| ADCS Error Code | AILS `denial_reason` |
|---|---|
| `ORIGINATOR_REVOKED` | `token_revoked` |
| `DELEGATION_REVOKED` | `token_revoked` |
| `TOKEN_REVOKED` | `token_revoked` |

**Compaction errors (§15.6):**

| ADCS Error Code | AILS `denial_reason` |
|---|---|
| `INVALID_COMPACTION_SIGNATURE` | `policy_denied` |
| `INTERMEDIATE_DIGEST_UNAVAILABLE` | `policy_denied` |

Implementations encountering an error code added in a future ADCS revision not listed here MUST default the `denial_reason` to `policy_denied` and MUST include the new error code verbatim in the payload annotation described above.

### 23.4 Envelope Mapping Rules (Normative)

When ADCS audit events are emitted in AILS Event Envelope form (AILS §5, Appendix A), the following mapping rules apply. These rules ensure that AILS envelope fields carry the correct ADCS-layer identity and correlation information without redefining the envelope's semantics.

**Correlation fields (AILS §5.6):**

- `correlation.session_id` MUST equal the chain's `chain_id`. This binds every ADCS event emitted during verification, consumption, or revocation of a chain to the chain's lifetime as an AILS session.
- `correlation.request_id` SHOULD equal `terminal_action.action_id` when the event is scoped to a specific terminal action invocation.
- `correlation.trace_id` and `correlation.span_id` MUST conform to W3C Trace Context per AILS REQ-5.6.2. When an ADCS verification runs within an existing OpenTelemetry trace context, emitters MUST propagate the active `trace_id` and `span_id` per AILS REQ-5.6.3; when emitted outside an active trace, a new `trace_id` and `span_id` conforming to the W3C Trace Context format MUST be generated.
- `correlation.parent_span_id`, when used, SHOULD reference the span of the enclosing protocol operation (the MCP `tools/call`, A2A `tasks/send`, or equivalent) whose authorization the chain is verifying.

**Actor field (AILS §5.5):**

- `actor.id` MUST be set to the DID of the principal whose action caused the event. For delegation-hop creation events (`agent.delegation`), this is the delegator. For permission escalation events (`policy.escalation`), this is the requesting agent. For condition satisfaction events (`human.confirmation`), this is the satisfying principal (= `satisfied_by`).
- For verifier-emitted events (`authority.gate_passed`, `authority.gate_denied`), `actor.id` MUST be the verifier's own identity, and `actor.type` MUST be `system`.
- `actor.type` maps from the ADCS principal `type` field as follows: ADCS `human` → AILS `human`; ADCS `system` → AILS `system`; ADCS `agent` → AILS `agent`.
- When `actor.type` is `agent`, the optional `actor.agent_identity` object SHOULD be populated with the agent's framework (if known) and SPIFFE ID (if the agent's DID is SPIFFE-derived per Appendix C.3).

**Emitter field (AILS §5.4):**

The `emitter` object represents the software component that produced the event and is distinct from `actor`. The following emitter-type mappings apply:

| Component emitting the event | `emitter.type` |
|---|---|
| Chain verifier | `security_service` |
| Governance layer issuing capability tokens | `security_service` |
| Policy engine handling escalation or condition satisfaction | `security_service` |
| Agent runtime emitting `agent.delegation` at hop creation | `runtime` |
| CAEP subscriber processing inbound security events | `security_service` |
| Revocation service publishing revocation records | `security_service` |

**Data classification (AILS §5.7, §23.3.3):**

- Verifier-emitted events (`authority.gate_passed`, `authority.gate_denied`) default to `SYSTEM_TRUSTED`.
- Token-issuance events (`authority.token_issued`) where the token authorizes an originator-initiated action default to `USER_TRUSTED` when the originator trust tier is `hardware_bound` or `host_mediated`, and to `SYSTEM_TRUSTED` when the tier is `policy_derived`.
- Events where the terminal action involves external input (e.g., MCP tool responses, A2A messages received from external agents) MUST be classified `EXTERNAL_UNTRUSTED` per AILS REQ-23.3.3.
- Condition satisfaction events (`human.confirmation`) emitted in response to a human-approval condition default to `USER_TRUSTED`.

**Originator trust tier carry-through:**

The ADCS originator trust tier from §4.8 — `hardware_bound`, `host_mediated`, or `policy_derived` — MUST be carried in the event payload on `authority.token_issued` events as a field named `originator_trust_tier`, placed within the `scope` object (AILS REQ-12.2.2). This enables AILS consumers to filter, weight, or alert on tokens by the cryptographic strength of the originating human authorization without inspecting the underlying chain.

### 23.5 Serialization and Transport (Normative)

The AILS Event Envelope (AILS §5, Appendix A) is the **reference serialization** for ADCS audit events. An implementation that emits §23.2 events as AILS-conformant envelopes is automatically interoperable with any AILS-conformant consumer without transformation.

Implementations MAY use alternative serializations — OpenTelemetry log records, syslog RFC 5424 messages, CEF (Common Event Format), or proprietary formats — provided that:

1. The ADCS event type names defined in §23.2 are preserved and recoverable from the serialized event.
2. The payload field names and semantic values defined in §23.2, §23.3, and §23.4 are preserved and recoverable.
3. The originator trust tier carry-through from §23.4 is preserved on `authority.token_issued` events.

Implementations MUST NOT silently drop fields required by §23.2 when serializing to an alternative format. If a target serialization lacks a native representation for a required field, the implementation MUST encode the field as a well-known extension attribute in a deterministic way documented in the implementation's conformance claim.

Implementations that declare AILS conformance per §23.7 MUST use the AILS Event Envelope and MUST NOT omit required AILS envelope fields (per AILS REQ-5.2.2, REQ-5.15.1, REQ-5.15.2). Implementations declaring `"none"` for `ails_conformance` MAY still emit §23.2 events in any serialization.

Transport of AILS-enveloped events is out of scope for ADCS and is governed by AILS DP-1 (Semantics Over Transport). Events MAY be transported via OpenTelemetry, message queues, HTTP, or any other mechanism that preserves envelope integrity.

### 23.6 Integrity, Retention, and Redaction

**Integrity.** Implementations SHOULD emit audit events with AILS Level 2 integrity fields — `sequence_number`, `stream_id`, and `chain_hash` per AILS §22.4 — to provide tamper-evidence. When the stream scope is per-session, the AILS `stream_id` MUST be set equal to the ADCS `chain_id` per AILS REQ-22.4.7 (applied to ADCS's session analog). Implementations serving deployments with regulatory or high-assurance requirements — including EU AI Act high-risk systems, financial-industry dual-control workflows, and forensic-evidence contexts — SHOULD emit Level 3 signed events per AILS §22.5, using the Ed25519 or ES256 signing algorithms permitted by AILS REQ-22.5.3. Event signatures are computed over the canonical event after any redaction has been applied, per AILS REQ-24.7.1, so redacted events remain integrity-verifiable.

**Retention.** ADCS audit events — as `authority.*`, `session.*`, `credential.*`, and `policy.*` category events — fall within the AILS Governance retention tier per AILS REQ-25.2.1 and SHOULD be retained for a minimum of 3 years from chain termination (measured from the `session.state_changed` event recording the chain's transition to a terminal state, or from the final `authority.token_consumed`/`authority.token_expired` event when no explicit termination event is emitted). This 3-year minimum supersedes the 90-day minimum in §22.6 when audit-grade logging is required. The AILS Regulatory tier (10 years, per AILS REQ-25.2.1) applies when the deployment is subject to EU AI Act Article 12 high-risk-system requirements or equivalent regulatory retention obligations, per AILS §25.6. Deployers MUST retain events for the longer of the AILS minimum and any applicable regulatory requirement per AILS REQ-25.2.2.

**Redaction.** The ADCS `intent` field (§8.3) and human-principal DIDs are sensitive per AILS §24 and SHOULD be redacted under the AILS `default` or `strict` redaction profiles in environments where intent text may contain PII or where regulatory constraints apply. When the ADCS `intent` field is redacted, the redacted value MUST follow AILS REQ-24.4.1 format conventions (`"[REDACTED]"` for omitted, `"sha256:<hex>"` for hashed, etc.) and the redaction MUST be recorded in the envelope `redaction.redacted_fields` array. Cryptographic signatures, `chain_digest` values, DIDs of non-human principals (agents, systems, tools), delegation identifiers, token identifiers, and structural metadata (`permissions_granted`, `constraints_added`, `action_id`, etc.) are **not** sensitive under AILS §24.2 and MUST NOT be redacted — redacting these fields would destroy the audit value of the event and break chain reconstruction. The absolute AILS prohibition on credential material (AILS REQ-24.3.2) applies: raw signing keys, bearer token contents, and private key material MUST NOT appear in ADCS audit events under any configuration.

### 23.7 AILS Conformance Declaration (Normative, OPTIONAL Capability)

The ADCS Configuration Discovery endpoint (§22.7) carries a field `ails_conformance` taking one of four values: `"none"`, `"level_1"`, `"level_2"`, or `"level_3"`. This field declares the deployment's AILS conformance posture and is REQUIRED in the discovery response.

| Value | Meaning |
|---|---|
| `"none"` | The implementation does not claim AILS conformance. It MAY still emit §23.2 events in any serialization provided event type and field names are preserved. |
| `"level_1"` | The implementation emits all §23.2 mapped events in AILS Event Envelope form conforming to AILS Level 1 (AILS §28.2). The integrity envelope is not required. Retention follows the AILS Standard tier (90 days) or Governance tier as appropriate per AILS §25. |
| `"level_2"` | Level 1 plus: events include the AILS integrity envelope with `sequence_number`, `stream_id`, and `chain_hash` per AILS §22.4. The Governance retention tier (3 years) applies to audit-category events. Mandatory AILS alert conditions (AILS §27) are emitted, including those bound in §23.8. |
| `"level_3"` | Level 2 plus: events are cryptographically signed per AILS §22.5 with Ed25519 or ES256. The Regulatory retention tier (10 years) is available when applicable per AILS §25. |

Implementations declaring `"level_1"` or higher MUST emit all §23.2 mapped events in AILS Event Envelope form. Implementations declaring `"level_2"` or `"level_3"` MUST additionally meet the corresponding AILS integrity requirements. Implementations declaring `"none"` MAY emit §23.2 events in any serialization, including no serialization at all, and MUST NOT make AILS-conformance claims.

**Cross-standard conformance claim format** (per AILS REQ-28.5.2):

```
"ADCS v1.0.0 Level N Conformant; AILS v1.0.0 Level K audit events emitted"
```

where N is the declared ADCS Conformance Level (1, 2, or 3) and K is the declared AILS Conformance Level (1, 2, or 3). ADCS Conformance Level and AILS Conformance Level are independent — an ADCS Level 1 implementation MAY declare AILS Level 3, and an ADCS Level 3 implementation MAY declare AILS Level 1.

Declaring AILS conformance is OPTIONAL and does not affect satisfaction of ADCS Conformance Levels 1, 2, or 3 per §17.1. See §17.3.

### 23.8 Alert Binding (Normative)

Implementations emitting §23.2 events in AILS envelope form MUST emit AILS `system.alert` events under the following conditions, in accordance with AILS §27.2. These alert bindings are in addition to any other alert conditions defined in AILS §27 that apply generically to the categories of events ADCS emits.

**Repeated gate denials.** Three or more `authority.gate_denied` events carrying the same `correlation.session_id` (= ADCS `chain_id`) within a 60-second window, measured per AILS REQ-27.3.2 from the `timestamp` fields of the triggering events, MUST trigger a `system.alert` event with `alert_name` = `authority_gate_denied_repeated` and severity `high`. This binding is the ADCS instantiation of the mandatory AILS alert condition of the same name defined in AILS §27.2.1.

**Critical verification failures.** A chain verification failure producing any of the following ADCS error codes MUST trigger a `system.alert` event with `alert_name` = `policy_violation_critical` and severity `critical`:

- `QUORUM_INSUFFICIENT` (§15.2) — a multi-party originator declaration failed to meet its required quorum threshold
- `INVALID_ORIGINATOR_SIGNATURE` (§15.2) — a signature forgery or key-compromise scenario at the chain root
- `CHAIN_DIGEST_MISMATCH` (§15.1) — a tamper or integrity-break condition on the chain object
- `TOKEN_REPLAY` (§15.2) — a token-replay attack indicator
- Any `_REVOKED` error code (`ORIGINATOR_REVOKED`, `DELEGATION_REVOKED`, `TOKEN_REVOKED` per §15.5) — action attempted with a revoked authorization artifact

These conditions indicate adversarial activity, chain integrity compromise, or governance-rule violation of a severity warranting immediate operator attention. The alert MUST include `triggering_event_ids` referencing the `event_id` of the originating `authority.gate_denied` event per AILS REQ-27.3.3.

Implementations MAY additionally emit `system.alert` events for other error-code patterns based on deployment risk tolerance. Custom alert conditions MUST use AILS `x-` prefix naming per AILS REQ-27.5.1.

---

## Appendix A — Core Permission Vocabulary

The following permission strings are registered in ADCS v1.0.0. All conformant implementations MUST recognize these identifiers.

| Permission | Description |
|---|---|
| `read` | Read access to data, files, or resources |
| `write` | Write or modify access |
| `delete` | Delete or remove data, files, or resources |
| `search` | Search or query operations |
| `compute` | Execute computation or processing |
| `send_message` | Send a message to an external party |
| `book` | Reserve or schedule a resource, appointment, or service |
| `purchase` | Commit to a financial transaction |
| `admin` | Administrative or privileged system operations |
| `key_management` | Operations involving cryptographic keys or credentials |
| `delegate` | Create further delegations from the received scope |

**The `delegate` permission** is special: an agent that does not have `delegate` in its `permissions_granted` MUST NOT create further delegations. This is enforced by the verification algorithm at Step 4.0. A delegation created by an agent without `delegate` permission produces `UNAUTHORIZED_SUBDELEGATION`.

The `delegate` permission MAY be granted conditionally via `permissions_conditional`. When granted conditionally, further sub-delegation requires the condition to be satisfied first. The optional `delegation_scope` field on a conditional `delegate` grant restricts which agent types the receiving agent may delegate to:

```json
{
  "permission": "delegate",
  "condition_type": "policy_gate",
  "condition_ref": "policy:allow-search-delegation",
  "delegation_scope": {
    "allowed_delegate_types": ["search_agent", "summarizer_agent"]
  }
}
```

The `allowed_delegate_types` values are deployment-defined agent type strings. Governance layers enforcing `delegation_scope` MUST verify that the target delegate's registered agent type matches a value in `allowed_delegate_types` before permitting sub-delegation.

---

## Appendix B — Example: Complete Delegation Chain

A travel booking workflow with two hops, conditional permissions, and updated structure including DID-based principals.

```json
{
  "adcs_version": "1.0.0",
  "chain_id": "chain-abc123",
  "revocation_service": "https://trust.example.com/.well-known/adcs-revocation",
  "originator_declaration": {
    "adcs_version": "1.0.0",
    "delegation_id": "orig-001",
    "originator": {
      "id": "user:alice@example.com",
      "did": "did:web:example.com:users:alice",
      "type": "human"
    },
    "originator_trust_tier": "host_mediated",
    "issued_at": "2026-03-12T10:00:00Z",
    "ttl": "PT4H",
    "expires_at": "2026-03-12T14:00:00Z",
    "intent": "Book a round trip to Tokyo departing April 1, returning April 7, total budget under $3000 USD",
    "intent_structured": {
      "action_category": "booking",
      "resource_types": ["flights", "hotels"],
      "sensitivity": "medium"
    },
    "permissions_granted": ["search", "delegate"],
    "permissions_conditional": [
      {
        "permission": "book",
        "condition_type": "human_approval",
        "condition_ref": "approval-channel:ops-console"
      }
    ],
    "constraints": {
      "budget": { "max": 3000, "currency": "USD" },
      "time_range": { "start": "2026-04-01", "end": "2026-04-07" },
      "max_delegation_depth": 3
    },
    "context": {
      "bootstrapping_method": "human_explicit"
    },
    "signature": "base64url..."
  },
  "delegations": [
    {
      "adcs_version": "1.0.0",
      "delegation_id": "del-001",
      "parent_delegation_id": "orig-001",
      "delegator": {
        "id": "agent:travel-coordinator",
        "did": "did:web:agents.example.com:travel-coordinator",
        "type": "agent"
      },
      "delegate": {
        "id": "agent:hotel-specialist",
        "did": "did:web:agents.example.com:hotel-specialist",
        "type": "agent"
      },
      "protocol": "a2a",
      "issued_at": "2026-03-12T10:01:00Z",
      "ttl": "PT2H",
      "expires_at": "2026-03-12T12:01:00Z",
      "permissions_granted": ["search"],
      "permissions_withheld": ["delegate"],
      "permissions_conditional": [
        {
          "permission": "book",
          "condition_type": "human_approval",
          "condition_ref": "approval-channel:ops-console"
        }
      ],
      "constraints_added": {
        "budget": { "max": 1200, "currency": "USD" }
      },
      "intent_reference": "orig-001",
      "signature": "base64url..."
    },
    {
      "adcs_version": "1.0.0",
      "delegation_id": "del-002",
      "parent_delegation_id": "del-001",
      "delegator": {
        "id": "agent:hotel-specialist",
        "did": "did:web:agents.example.com:hotel-specialist",
        "type": "agent"
      },
      "delegate": {
        "id": "tool:search_hotels",
        "did": "did:web:tools.example.com:search-hotels",
        "type": "system"
      },
      "protocol": "mcp",
      "issued_at": "2026-03-12T10:02:00Z",
      "ttl": "PT30M",
      "expires_at": "2026-03-12T10:32:00Z",
      "permissions_granted": ["search"],
      "permissions_withheld": [],
      "constraints_added": {
        "max_results": 10
      },
      "intent_reference": "orig-001",
      "signature": "base64url..."
    }
  ],
  "terminal_action": {
    "protocol": "mcp",
    "operation": "tools/call",
    "target": "search_hotels",
    "action_id": "jsonrpc-id-7f3b"
  },
  "chain_digest": "sha256hex...",
  "context_snapshot": {
    "task_state_hash": "sha256hex...",
    "preceding_action_count": 2,
    "preceding_action_summary": "Searched flights for Tokyo April 1-7; selected Star Alliance options within budget",
    "additional_context": {}
  }
}
```

**Effective scope at terminal action:**
- `effective_permissions`: `["search"]`
- `effective_conditional_permissions`: `[{"permission": "book", "condition_type": "human_approval", "condition_ref": "approval-channel:ops-console"}]`
- `effective_constraints`: `{"budget": {"max": 1200, "currency": "USD"}, "time_range": {"start": "2026-04-01", "end": "2026-04-07"}, "max_results": 10, "max_delegation_depth": 3}`

The booking permission propagates as conditional — not available for unconditional exercise by the hotel search tool. Budget narrowed from $3000 to $1200. `max_delegation_depth` inherited at 3; `hotel-specialist` withheld `delegate`, so no further sub-delegation is possible from this branch regardless.

---

## Appendix C — Agent Identity Lifecycle

*This appendix is non-normative. It describes how agent identities are provisioned, maintained, and decommissioned in practice.*

### C.1 The Identity Provisioning Problem

ADCS requires every agent to have a DID that resolves to a current public key. In static, long-lived service deployments this is straightforward. In cloud-native environments where hundreds or thousands of agents are created dynamically, pre-provisioning DIDs for every possible agent instance is impractical and leads to credential sprawl and over-permissioning.

### C.2 Just-in-Time (JIT) Identity Provisioning

The recommended model for ephemeral and dynamically instantiated agents is **just-in-time identity provisioning**:

1. When an agent is instantiated for a task, the platform's **Agent Identity Service** generates a new key pair and creates a `did:jwk` or `did:web` DID bound to that instance
2. The DID is registered in the organization's Agent Registry and associated with the agent's task context, the originating chain's `chain_id`, and the agent's type classification
3. The agent receives its private key via a secure credential injection mechanism (e.g., Kubernetes Secrets, SPIFFE SVID, HSM-backed injection)
4. Upon task completion or agent decommissioning, the DID is revoked in the Agent Registry, the private key is destroyed, and the revocation is published

This model ensures no orphaned credentials — an agent's identity has the same lifetime as its task.

### C.3 SPIFFE/SPIRE Integration

Organizations using SPIFFE/SPIRE for workload identity MAY derive ADCS agent DIDs from SPIFFE SVIDs (X.509 or JWT format). The SPIFFE ID (`spiffe://trust-domain/workload`) maps naturally to a `did:web` identifier. The SVID's X.509 certificate provides the verification key material for the DID document.

This approach provides:
- Automatic rotation aligned with SVID issuance intervals (typically 1 hour)
- Integration with existing PKI infrastructure
- Attestation of the workload's runtime environment (node attestation, pod attestation)

### C.4 Long-Lived Agent Identities

Some agents (persistent orchestrators, registered service agents) have long-lived identities appropriate for `did:web` anchored at a stable domain. Requirements:

- The DID document MUST be accessible at `https://<domain>/.well-known/did.json` or equivalent
- The verification key MUST be rotated at intervals not exceeding 90 days (Section 13.2)
- Key rotation MUST update the DID document and MUST NOT break active chains; new keys are used for new delegations, old keys remain valid for verifying existing signed delegations until those chains expire
- The DID document MUST be served over HTTPS with a valid certificate

### C.5 Agent Registry Requirements

Organizations MUST maintain an Agent Registry that:

- Maps each active agent DID to its agent type, owner, task context, and registration timestamp
- Supports DID document lookup by verifiers
- Supports revocation of agent DIDs
- Records the `chain_id` values for which each agent has received delegations (for cascade revocation)
- Is accessible with availability requirements appropriate to the real-time nature of ADCS verification (target: 99.9% uptime; ≤ 100ms p99 response time for DID lookup)

---

## Appendix D — Conformance Test Vectors

*The complete machine-readable test vector corpus is maintained in the standard's repository. This appendix defines the required test vector categories and provides annotated summaries.*

### D.1 Required Test Vector Categories

Each conformant implementation MUST pass test vectors in the following categories. Test vectors are identified by category code and number.

| Category | Code | Count | Description |
|---|---|---|---|
| Minimal valid chain | TV-VALID-01 | 1 | One hop, no constraints, all phases pass |
| Valid with all constraint types | TV-VALID-02 | 1 | All Section 9.2 constraint types exercised |
| Valid compacted chain | TV-VALID-03 | 1 | 8-hop chain in compacted form |
| Valid cross-protocol chain | TV-VALID-04 | 1 | A2A hop → MCP terminal action |
| Valid DAG chain | TV-VALID-05 | 1 | Two parallel branches converging at merge delegation |
| Valid with originator quorum | TV-VALID-06 | 1 | Two-of-two quorum originator |
| Invalid: every Phase 1 error | TV-STRUCT | 15 | One vector per structural error code |
| Invalid: every Phase 2 error | TV-CRYPTO | 8 | One vector per cryptographic error code |
| Invalid: every Phase 3 error | TV-TEMP | 8 | One vector per temporal error code |
| Invalid: every Phase 4 error | TV-MONO | 10 | One vector per monotonic restriction error code |
| Invalid: revocation | TV-REVOKE | 3 | Revoked originator, delegation, and token |
| Edge cases | TV-EDGE | 10 | Clock skew boundary, empty permissions, min TTL, max depth |

### D.2 Minimal Valid Chain (TV-VALID-01) — Summary

```
Chain:
  originator: did:jwk:... (alice), trust_tier: host_mediated
  delegation 0: alice → agent-A (protocol: direct)
    permissions_granted: ["search"]
    constraints_added: {}
  terminal_action: direct, "invoke", "search-tool", action-id-001

Expected result: VALID
  effective_permissions: ["search"]
  effective_constraints: {}
```

Full JSON in repository at `test-vectors/TV-VALID-01.json`.

### D.3 Permission Expansion (TV-MONO-01) — Summary

```
Chain:
  originator grants: ["search"]
  delegation 0: grants ["search", "write"]   ← expansion

Expected result: INVALID
  error_code: PERMISSION_EXPANSION
  failed_at: <delegation_id of delegation 0>
```

Full JSON in repository at `test-vectors/TV-MONO-01.json`.

### D.4 Unauthorized Subdelegation (TV-MONO-08) — Summary

```
Chain:
  originator grants: ["search"]  (no "delegate")
  delegation 0: agent-A → agent-B, grants ["search"]
  delegation 1: agent-B → agent-C, grants ["search"]   ← B has no delegate permission

Expected result: INVALID
  error_code: UNAUTHORIZED_SUBDELEGATION
  failed_at: <delegation_id of delegation 1>
```

Full JSON in repository at `test-vectors/TV-MONO-08.json`.

---

## Appendix E — Reference SDK Architecture

*This appendix is non-normative. It defines an interface contract for conformant SDK implementations to ensure API-level interoperability.*

### E.1 Module Overview

A conformant ADCS SDK SHOULD expose the following modules:

```
adcs/
  chain_builder       — constructs and signs delegation objects and chains
  originator_service  — creates and signs originator declarations
  capability_token_issuer — issues JWTs from verified chains
  chain_verifier      — executes the four-phase verification algorithm
  revocation_client   — fetches/caches revocation lists, subscribes to CAEP
  constraint_registry — pluggable registry for constraint type evaluators
  did_resolver        — resolves DID documents; caches per Section 13.3
```

### E.2 chain_verifier Interface

```typescript
interface ChainVerificationInput {
  chain: DelegationChain;
  token: CapabilityToken;
  terminalAction: TerminalAction;
  currentTime?: ISO8601Timestamp;         // defaults to system time
  clockSkewTolerance?: ISO8601Duration;   // defaults to PT60S
}

interface ChainVerificationResult {
  result: "VALID" | "INVALID";
  chainId: string;
  effectivePermissions?: string[];
  effectiveConstraints?: Record<string, unknown>;
  originatorTrustTier?: OriginatorTrustTier;
  originatorIntent?: string;
  errorCode?: AdcsErrorCode;
  errorDetail?: string;
  failedAt?: string;
  verifiedAt: ISO8601Timestamp;
}

interface ChainVerifier {
  verify(input: ChainVerificationInput): Promise<ChainVerificationResult>;
  setRevocationClient(client: RevocationClient): void;
  setDidResolver(resolver: DidResolver): void;
  setConstraintRegistry(registry: ConstraintRegistry): void;
}
```

### E.3 chain_builder Interface

```typescript
interface DelegationInput {
  delegatorDid: string;
  delegateDid: string;
  delegatorType: PrincipalType;
  delegateType: PrincipalType;
  protocol: string;
  permissionsGranted: string[];
  permissionsWithheld: string[];
  permissionsConditional?: ConditionalPermission[];
  constraintsAdded: Record<string, unknown>;
  parentDelegationId: string;
  intentReference: string;
  ttl: ISO8601Duration;
  signingKey: PrivateKey;
}

interface ChainBuilder {
  addDelegation(input: DelegationInput): ChainBuilder;
  addDagDelegation(input: DagDelegationInput): ChainBuilder;
  setTerminalAction(action: TerminalAction): ChainBuilder;
  setRevocationService(uri: string): ChainBuilder;
  build(): DelegationChain;
}
```

### E.4 constraint_registry Interface

```typescript
interface ConstraintEvaluator {
  constraintType: string;
  isMoreRestrictive(current: unknown, proposed: unknown): boolean;
  // For semantic enforcement (optional):
  evaluate?(constraintValue: unknown, actionParameters: Record<string, unknown>): boolean;
}

interface ConstraintRegistry {
  register(evaluator: ConstraintEvaluator): void;
  get(constraintType: string): ConstraintEvaluator | null;
  evaluationCapability(): "structural_only" | "semantic";
}
```

---

## Appendix F — Implementer Quick Reference

### F.1 Level 1 Conformance Checklist

- [ ] Implement DID resolution for at least one DID method (`did:web`, `did:jwk`, or `did:key`)
- [ ] Support inline `public_key` with `key_provenance` field; record provenance in audit logs (Section 4.2)
- [ ] Implement delegation object structure with `delegator.did`, `delegate.did` (Section 5)
- [ ] Implement originator declaration structure with `originator_trust_tier` (Section 8)
- [ ] Include `revocation_service` in all chain objects (Section 6.2)
- [ ] Use RFC 8785 (JCS) for all canonical JSON serialization (Section 5.4)
- [ ] Implement all four phases of the verification algorithm (Section 10.3–10.6), including Step 4.0 (`delegate` permission check, with DAG semantics if DAG is supported)
- [ ] Implement revocation check (Section 10.7), including originator declaration revocation
- [ ] Support all 7 core constraint types (Section 9.2)
- [ ] Apply clock skew tolerance (default PT60S) in Phase 3 (Section 10.5)
- [ ] Implement fail-closed behavior: no partial validity (Section 10.9)
- [ ] Publish constraint evaluation capability declaration (Section 9.4)
- [ ] Implement all error codes in Section 15 and include them in verification results
- [ ] Emit audit events per §23 using AILS-aligned event types and payload fields (AILS Event Envelope is the reference serialization; alternatives permitted provided event type names and payload field names are preserved per §23.5)

### F.2 Additional Requirements for Level 2

- [ ] Issue JWT capability tokens with all required claims including `originator_trust_tier` and `condition_satisfactions` (Section 7.2)
- [ ] Implement condition satisfaction record verification when `condition_satisfactions` is present (Section 5.6)
- [ ] Implement proof-of-possession verification via mTLS (`x5t#S256`) or DPoP (`jkt`) (Section 7.5)
- [ ] Implement token replay prevention: maintain per-session seen-`jti` record (Section 10.4, Step 2.5)
- [ ] Check capability token revocation at verification time (Section 14.4)
- [ ] Support token-only presentation mode (Section 7.7)
- [ ] *Optional:* Declare AILS Level 2 conformance (§23.7) by emitting §23.2 audit events in AILS Event Envelope form with the AILS integrity envelope (sequence_number, stream_id, chain_hash per AILS §22.4) and retaining audit-category events for the Governance tier (3 years per AILS §25)

### F.3 Additional Requirements for Level 3

- [ ] Implement chain compaction for chains > 5 delegations; enforce 20-delegation maximum (Section 11)
- [ ] Implement at least one protocol binding: MCP (12.4), A2A (12.5), ACP (12.6), or HTTP (12.7)
- [ ] Implement at least one revocation method: list (14.2) or push (14.2)
- [ ] Expose revocation service metadata endpoint (Section 14.7)
- [ ] Implement CAEP continuous revocation subscription and event handling (Section 14.6)
- [ ] Implement multi-party originator quorum verification (Section 8.5, Step 2.1b)
- [ ] Implement permission escalation request handling (Section 10.10)
- [ ] Expose ADCS Configuration Discovery endpoint (`/.well-known/adcs-configuration`) (Section 22.7)
- [ ] *Optional:* DAG workflow chain support (Section 6.6)
- [ ] *Optional:* Macaroon token support (Section 7.6)
- [ ] *Optional:* Declare AILS Level 3 conformance (§23.7) by signing §23.2 audit events with Ed25519 or ES256 per AILS §22.5 and retaining audit-category events for the Regulatory tier (10 years per AILS §25) when subject to EU AI Act high-risk-system requirements or equivalent regulation

### F.4 Verification Algorithm — Four-Phase Summary

```
Input: chain, token, terminal_action, current_time, clock_skew_tolerance

Phase 1 — Structural (Steps 1.1–1.15)
  Verify version, originator present, delegations non-empty,
  terminal action matches, chain is linked, digests match,
  token references this chain, revocation_service present.

Phase 2 — Cryptographic (Steps 2.1–2.5)
  Resolve delegator DIDs → verify all signatures.
  Verify originator quorum if present.
  Verify token signature + proof-of-possession.
  Check token JTI for replay.

Phase 3 — Temporal (Steps 3.1–3.8)
  All TTLs valid (with clock_skew_tolerance).
  All expires_at consistent with issued_at + ttl.
  No delegation exceeds its parent's TTL.
  Token not expired, not before NBF, within chain TTL.

Phase 4 — Monotonic Restriction (Steps 4.0–4.5)
  4.0: Delegator at each hop (except first) must hold "delegate" permission.
       For DAG: "delegate" must be in ALL parent branches' effective permissions.
  4.1: Permissions only narrow (subset invariant).
  4.1b: Conditional permissions correctly propagated.
  4.2: Constraints only tighten.
  4.3: Update effective state per hop.
  4.4-4.5: Token's declared effective scope matches recomputed scope.

Revocation (Steps R.1–R.4)
  Check originator, all delegations, and token against revocation service.

→ VALID with effective scope, OR INVALID with error code.
```

### F.5 Complete Error Code Reference

| Code | Phase | Section |
|---|---|---|
| `VERSION_MISMATCH` | Phase 1 | 15.1 |
| `MISSING_ORIGINATOR` | Phase 1 | 15.1 |
| `EMPTY_CHAIN` | Phase 1 | 15.1 |
| `MISSING_TERMINAL_ACTION` | Phase 1 | 15.1 |
| `ACTION_ID_MISMATCH` | Phase 1 | 15.1 |
| `PROTOCOL_MISMATCH` | Phase 1 | 15.1 |
| `OPERATION_MISMATCH` | Phase 1 | 15.1 |
| `TARGET_MISMATCH` | Phase 1 | 15.1 |
| `BROKEN_CHAIN` | Phase 1 | 15.1 |
| `INTENT_REFERENCE_MISMATCH` | Phase 1 | 15.1 |
| `CHAIN_DIGEST_MISMATCH` | Phase 1 | 15.1 |
| `TOKEN_CHAIN_MISMATCH` | Phase 1 | 15.1 |
| `TOKEN_DIGEST_MISMATCH` | Phase 1 | 15.1 |
| `MISSING_REVOCATION_SERVICE` | Phase 1 | 15.1 |
| `DAG_NOT_SUPPORTED` | Phase 1 | 15.1 |
| `INVALID_ORIGINATOR_SIGNATURE` | Phase 2 | 15.2 |
| `INVALID_DELEGATION_SIGNATURE` | Phase 2 | 15.2 |
| `INVALID_TOKEN_SIGNATURE` | Phase 2 | 15.2 |
| `POP_MISMATCH` | Phase 2 | 15.2 |
| `MISSING_POP` | Phase 2 | 15.2 |
| `TOKEN_REPLAY` | Phase 2 | 15.2 |
| `DID_RESOLUTION_FAILURE` | Phase 2 | 15.2 |
| `QUORUM_INSUFFICIENT` | Phase 2 | 15.2 |
| `ORIGINATOR_EXPIRED` | Phase 3 | 15.3 |
| `ORIGINATOR_TTL_INCONSISTENT` | Phase 3 | 15.3 |
| `DELEGATION_EXPIRED` | Phase 3 | 15.3 |
| `DELEGATION_TTL_INCONSISTENT` | Phase 3 | 15.3 |
| `TTL_EXCEEDS_PARENT` | Phase 3 | 15.3 |
| `TOKEN_EXPIRED` | Phase 3 | 15.3 |
| `TOKEN_NOT_YET_VALID` | Phase 3 | 15.3 |
| `TOKEN_TTL_EXCEEDS_CHAIN` | Phase 3 | 15.3 |
| `PERMISSION_EXPANSION` | Phase 4 | 15.4 |
| `CONSTRAINT_LOOSENED` | Phase 4 | 15.4 |
| `UNKNOWN_CONSTRAINT_MODIFIED` | Phase 4 | 15.4 |
| `PERMISSION_CONDITION_LOOSENED` | Phase 4 | 15.4 |
| `TOKEN_PERMISSIONS_MISMATCH` | Phase 4 | 15.4 |
| `TOKEN_CONDITIONAL_PERMISSIONS_MISMATCH` | Phase 4 | 15.4 |
| `TOKEN_CONSTRAINTS_MISMATCH` | Phase 4 | 15.4 |
| `UNAUTHORIZED_SUBDELEGATION` | Phase 4 | 15.4 |
| `DELEGATION_DEPTH_EXCEEDED` | Phase 4 | 15.4 |
| `DAG_MERGE_VIOLATION` | Phase 4 | 15.4 |
| `ORIGINATOR_REVOKED` | Revocation | 15.5 |
| `DELEGATION_REVOKED` | Revocation | 15.5 |
| `TOKEN_REVOKED` | Revocation | 15.5 |
| `INVALID_COMPACTION_SIGNATURE` | Compaction | 15.6 |
| `INTERMEDIATE_DIGEST_UNAVAILABLE` | Compaction | 15.6 |

---

*End of Agent Delegation Chain Standard*
