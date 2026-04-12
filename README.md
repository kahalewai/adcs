<div align="center">

<img width="364" height="468" alt="adcs-final1" src="https://github.com/user-attachments/assets/925ce417-56ca-4afe-b271-8e2dd710bca1" />

<br>

![Status: Draft for Public Comment](https://img.shields.io/badge/Status-Draft%20for%20Public%20Comment-yellow)
![Version: 1.0.0](https://img.shields.io/badge/Version-1.0.0-blue)
![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-green)

<br>

</div>

<br>

## Intro

AI agents delegate tasks to other agents, which delegate to other agents, which invoke tools. At every hop, the scope of what an agent is authorized to do must travel with the action, and at no hop may that scope expand beyond what was originally granted. Today, no standard mechanism enforces this. Each framework manages delegation ad hoc, through implicit context passing or application-layer trust assumptions. A sub-agent can end up with broader permissions than its delegator possesses. Constraints imposed by a human originator vanish by the time a tool is invoked. Long delegation chains become opaque. Cross-protocol delegations carry no verifiable authorization record.

<br>

> ADCS is THE authorization chain for multi-agent AI workflows

<br>

ADCS (Agent Delegation Chain Standard) is an open, vendor-neutral standard that defines how permissions and constraints flow from an originating principal through a chain of agent delegations, and how each hop in that chain is verified to be monotonically restrictive, meaning scope can only narrow, never expand. Every delegation is cryptographically signed using Decentralized Identifiers (DIDs). Every chain is verified by a four-phase algorithm that any conformant implementation can execute independently. No agent in the chain can grant more than it received.

<br>

ADCS addresses the authorization gap between session-level identity (OAuth 2.1) and session-level circuit breaking (ALS). It answers the question: given that this agent session is permitted to run, is this specific agent authorized to perform this specific action within the scope that was delegated to it?

<br>

ADCS was designed to support the full complexity of real-world agentic workflows:

* Linear and DAG chains - parallel agent branches that converge with defined merge semantics
* Conditional permissions - permissions gated on human approval, policy evaluation, or time windows
* Cross-protocol delegation - chains spanning MCP, A2A, ACP, and HTTP with verified authorization at every boundary
* Multi-party originator quorum - dual-control and multi-party authorization for regulated industries
* Continuous revocation - real-time authorization change propagation via CAEP
* Permission escalation - agents can request additional scope mid-task without abandoning the chain

<br>
<br>

## Why ADCS?

Existing mechanisms for managing agent authorization are insufficient:

* OAuth scopes are session-level, they don't subdivide across a multi-agent task chain
* Application-layer trust assumptions let sub-agents inherit ambient permissions they were never explicitly granted
* No existing mechanism verifies that constraints imposed by a human originator survive through every delegation hop
* Cross-protocol delegations (A2A task → MCP tool) carry no verifiable authorization record
* Key rotation or agent decommissioning leaves orphaned credentials with residual access
* No existing mechanism provides a cryptographically verifiable, fail-closed, monotonically restrictive delegation chain

ADCS closes those gaps with a normative data model, a four-phase verification algorithm, and cryptographic binding at every hop, independent of the agents' own compliance and independent of any single protocol.

<br>
<br>

## How Does ADCS Work?

**Step 1 - Human (or system) creates an originator declaration**

```
Human → Host Application: "Book me a round trip to Tokyo, budget under $3000"
Host Application: Creates originator declaration with permissions, constraints, and intent
Originator: Signs declaration with DID-bound key
```

**Step 2 - Orchestrating agent delegates to specialist agents**

```
Orchestrator: Receives originator's scope
Orchestrator → Hotel Agent: Delegation with narrowed permissions (search only) and tightened budget ($1200)
Each delegation: Cryptographically signed, references the originator declaration
```

**Step 3 - Specialist agent invokes a tool**

```
Hotel Agent → MCP Tool Server: Capability token + delegation chain
Tool Server: Executes four-phase verification algorithm
Verification: Structural → Cryptographic → Temporal → Monotonic Restriction → Revocation
```

**Step 4 - Verification succeeds or fails closed**

```
All phases pass: Tool executes within verified scope
Any phase fails: Action denied. No partial validity. No fallback.
Error code returned: PERMISSION_EXPANSION, CONSTRAINT_LOOSENED, DELEGATION_EXPIRED, etc.
```

**Step 5 - Full audit trail is preserved**

```
Every chain, every verification event, every originator intent → audit log
Chains retrievable by chain_id for the full retention period
Originator trust tier recorded: hardware_bound, host_mediated, or policy_derived
```

<br>
<br>

## View the ADCS Standard

For the complete formal specification, including definitions, requirements, and conformance criteria, see the full reference document:

[ADCS-Standard-v1.0.0.md](ADCS-Standard-v1.0.0.md)

**Status:** Public Draft for Community Review

**Version:** v1.0.0

**License:** Apache License 2.0

**Date:** 2026-04-11

<br>
<br>

## How Do I Use ADCS?

ADCS defines three conformance levels. Levels are cumulative and designed for incremental adoption — each level delivers independent value.

<br>

**Level 1 - Core Conformance**

Implement the delegation data model and the four-phase verification algorithm. No token infrastructure required.

* Delegation object structure with DID-based principals
* Originator declaration with trust tier classification
* All four phases of the verification algorithm (Structural → Cryptographic → Temporal → Monotonic Restriction)
* All 7 core constraint types (budget, data_scope, time_range, network_scope, resource_path, max_results, action_count)
* Fail-closed behavior: any verification failure = denial
* Constraint evaluation capability declaration

> *Sufficient for: internal agent deployments where the governance layer verifies chains before tool invocation.*

<br>

**Level 2 - Token Conformance**

Level 1 plus JWT capability tokens with proof-of-possession binding.

* JWT capability tokens carrying effective scope, originator trust tier, and chain reference
* Proof-of-possession via mTLS or DPoP, tokens are bound to the holder's transport credential
* Token replay prevention via nonce tracking
* Token-only presentation mode, full chain stays with governance layer, only token reaches the tool server
* Condition satisfaction record verification for conditional permissions

> *Sufficient for: deployments where tool servers independently verify authorization without receiving the full chain.*

<br>

**Level 3 - Full Conformance**

Level 2 plus protocol bindings, revocation infrastructure, and advanced features.

* At least one protocol binding: MCP, A2A, ACP, or HTTP
* Chain compaction for deep hierarchies (>5 hops)
* Revocation service with metadata endpoint
* CAEP continuous revocation for real-time enforcement
* Multi-party originator quorum
* Permission escalation protocol
* ADCS Configuration Discovery endpoint (`/.well-known/adcs-configuration`)
* *Optional:* DAG workflow chain support, Macaroon token support

> *Sufficient for: production multi-organization deployments with regulatory compliance requirements.*

<br>
<br>

## What the ADCS Standard Defines

The ADCS specification defines:

* Delegation object: normative structure for a single delegation hop, who granted what to whom, with what constraints, signed by which key
* Delegation chain: ordered sequence or DAG of delegations from originator to terminal action
* Originator declaration: root of every chain, human or system intent, maximum scope, trust tier, optional multi-party quorum
* Capability token: short-lived, transport-bound JWT carrying chain reference and computed effective scope
* Four-phase verification algorithm: structural, cryptographic, temporal, and monotonic restriction verification, normative and deterministic
* Monotonic restriction invariant: scope can only narrow at each hop; permissions removed, constraints tightened, never expanded
* Constraint type registry: 7 core constraint types with defined restriction rules, plus vendor extension support
* Cross-protocol bindings: MCP, A2A, ACP, and HTTP envelope formats with error response mappings
* Chain compaction: summarization of deep chains while preserving auditability
* DID-based principal identity: keys resolved via DID document lookup, decoupled from chain structure
* Revocation infrastructure: revocation lists, push revocation, CAEP continuous revocation, and revocation service metadata
* Permission escalation: standardized protocol for mid-task scope requests
* Conformance test suite: 48+ test vectors covering every normative requirement
* Error taxonomy: 37 error codes across 6 categories with deterministic failure semantics

<br>
<br>

## What the ADCS Standard Does NOT Do

Understanding the boundary is as important as understanding the capability:

* ADCS does not halt agent sessions. Session-level circuit breaking is the responsibility of ALS. ADCS governs action-level scope within an active session.
* ADCS does not verify tool integrity. Tool artifact signing and integrity verification is the responsibility of the MCP Integrity Standard or equivalent.
* ADCS does not enforce semantic constraints. ADCS verifies that constraints are structurally present and structurally non-loosened at each hop. It does not verify that a terminal action's parameters satisfy those constraints unless a semantic mapping layer is deployed (Section 16.5).
* ADCS does not evaluate policy. Policy engines (OPA, Cedar, Rego) may be used to evaluate chains, but are not specified by ADCS.
* ADCS does not provision identities. ADCS relies on externally established identities anchored in DID documents. Identity provisioning is the responsibility of DID method specifications, SPIFFE/SPIRE, or OAuth 2.1.
* ADCS does not detect prompt injection. A structurally valid chain produced by a manipulated agent will pass verification. ADCS is a layer in a defense-in-depth architecture, not a substitute for adversarial input detection.

<br>
<br>

## The Monotonic Restriction Invariant

The central security property of ADCS:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   ORIGINATOR DECLARATION                                    │
│   permissions: [search, book, delegate]                     │
│   budget: $3000 · data_scope: internal                      │
│                                                             │
│         │  Scope can only NARROW ↓                          │
│         ▼                                                   │
│   DELEGATION 1 (Orchestrator → Hotel Agent)                 │
│   permissions: [search, delegate]        ← book withheld    │
│   budget: $1200                          ← narrowed         │
│                                                             │
│         │  Scope can only NARROW ↓                          │
│         ▼                                                   │
│   DELEGATION 2 (Hotel Agent → Search Tool)                  │
│   permissions: [search]                  ← delegate removed │
│   budget: $1200 · max_results: 10        ← constraint added │
│                                                             │
│         │                                                   │
│         ▼                                                   │
│   TERMINAL ACTION: tools/call search_hotels                 │
│   effective_permissions: [search]                           │
│   effective_constraints: {budget: $1200, max_results: 10}   │
│                                                             │
│   ✗ Cannot add permissions at any hop                       │
│   ✗ Cannot loosen constraints at any hop                    │
│   ✗ Cannot sub-delegate without "delegate" permission       │
│   ✓ Verified by algorithm, not by agent compliance          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

The invariant is enforced by the verification algorithm running outside the agent runtime. A manipulated agent cannot grant its delegate broader scope than the agent received.

<br>
<br>

## Relationship to Other Standards

ADCS is one layer in a defense-in-depth architecture. It is designed to interoperate with, not replace, adjacent standards.

```
┌─────────────────────────────────────────────────────────────┐
│                   AGENT SECURITY STACK                      │
│                                                             │
│   ┌───────────────────────────────────────────────────┐     │
│   │  APPLICATION LAYER                                │     │
│   │  Agent frameworks, prompt injection defense,      │     │
│   │  content filters, policy engines (OPA, Cedar)     │     │
│   └───────────────────────┬───────────────────────────┘     │
│                           │                                 │
│   ┌───────────────────────▼───────────────────────────┐     │
│   │  ADCS — Action-Level Scope Authorization          │     │
│   │  "Is this agent authorized for this action        │     │
│   │   within the scope delegated to it?"              │     │
│   └───────────────────────┬───────────────────────────┘     │
│                           │                                 │
│   ┌───────────────────────▼───────────────────────────┐     │
│   │  OAuth 2.1 — Session-Level Identity               │     │
│   │  "Is this session authenticated and authorized    │     │
│   │   to operate?"                                    │     │
│   └───────────────────────┬───────────────────────────┘     │
│                           │                                 │
│   ┌───────────────────────▼───────────────────────────┐     │
│   │  ALS — Transport-Level Circuit Breaking           │     │
│   │  "Should this agent's connections continue        │     │
│   │   to be permitted?"                               │     │
│   └───────────────────────┬───────────────────────────┘     │
│                           │                                 │
│   ┌───────────────────────▼───────────────────────────┐     │
│   │  TLS 1.3 — Transport Security                     │     │
│   └───────────────────────────────────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Standard | Relationship |
|---|---|
| **ALS** | ALS halts sessions; ADCS authorizes actions within sessions. A valid ADCS chain does not override an ALS halt. An ALS NORMAL state does not substitute for ADCS chain verification. |
| **OAuth 2.1** | OAuth establishes identity and initial authorization. ADCS tracks how that authorization is subdivided and delegated across agent hops. |
| **MCP Integrity** | MCP Integrity verifies tool artifacts are genuine. ADCS verifies the invoking agent has scope to use them. Publisher-declared constraint mappings provide the bridge between the two standards. |
| **CAEP** | CAEP provides real-time revocation event streams. ADCS integrates with CAEP for continuous revocation propagation during active chain execution. |
| **UCAN** | An independently developed capability chain standard for decentralized contexts. ADCS targets centralized enterprise AI workflows with typed constraint registries; UCAN targets decentralized peer-to-peer contexts. ADCS adopts DID-based principal identification informed by UCAN's production experience. |
| **Macaroons** | Contextual caveats as an attenuation mechanism. ADCS supports Macaroon tokens as an optional alternative to JWT, but uses JWT as the baseline due to enterprise tooling ubiquity. |
| **SPIFFE/SPIRE** | Workload identity for cloud-native microservices. Enterprise deployments should evaluate SPIFFE/SVID as the identity substrate for DID principals in containerized agent environments. |
| **IETF Transaction Tokens** | Immutable, propagated call-chain context for distributed systems. Transaction Tokens establish who is calling what; ADCS establishes what they are authorized to do. They are complementary. |
| **IETF Identity Chaining** | Cross-domain OAuth identity preservation. Addresses identity propagation across organizational boundaries; ADCS addresses permission restriction within a task chain. |
| **ACP** | Agent Communication Protocol (IBM/Linux Foundation). ADCS defines a protocol binding for ACP alongside MCP, A2A, and HTTP. |

<br>
<br>

## Security

ADCS was designed with a comprehensive security model covering privilege escalation, chain forgery, chain replay, confused deputy attacks, prompt injection, chain injection, chain truncation, and delegation chain privacy.

Key security properties:

| Property | Mechanism |
|---|---|
| Scope can only narrow | Monotonic restriction invariant enforced by verification algorithm |
| Verification outside agent runtime | Agent cannot manipulate its own scope verification |
| Fail-closed by default | Any verification failure = denial; no partial validity |
| Chain forgery requires DID compromise | Signatures verified against externally resolved DID documents |
| Chain replay prevented | `terminal_action.action_id` binding + token nonce tracking |
| Confused deputy defeated | Effective scope recomputed from first principles at verification; token claims compared against recomputation |
| Chain injection prevented | Every delegation must reference originator; originator must be present |
| Delegation chain privacy | Token-only presentation mode keeps full chain within organizational trust boundary |
| Key compromise response | Normative time windows: revocation within 1 hour, downstream notification within 2 hours |

> **⚠ Important:** ADCS verification cannot detect whether a structurally valid chain was constructed by a prompt-injected agent. ADCS is a layer in a defense-in-depth architecture. Governance layers SHOULD implement human approval gates, anomaly detection, rate limits on chain construction, and intent-action consistency checks as complementary controls.

<br>
<br>

## Deployment Topologies

ADCS supports three reference deployment topologies:

| Topology | Description | Conformance Target |
|---|---|---|
| **Centralized Enterprise** | Single organization, all agents within one trust domain. Shared Agent Registry, Governance Layer, and Revocation Service. | Level 3; CAEP recommended |
| **Federated Multi-Organization** | Cross-org chains with independently operated infrastructure. Each org runs its own Agent Registry and Revocation Service. Mutual DID method agreement required. | Level 3 |
| **Embedded / Offline** | Edge, air-gapped, or mobile agents without persistent network access. Inline keys, short TTLs, aggressive compaction. | Level 1 or 2 |

### Incremental Adoption Path

Organizations do not need to deploy ADCS everywhere simultaneously:

* **Phase 1 - Boundary Enforcement:** Verify chains only at terminal tool servers and external-facing endpoints. Internal agents use ambient trust.
* **Phase 2 - Orchestrator Adoption:** Orchestrating agents construct chains. Terminal tools verify.
* **Phase 3 - Full Chain Depth:** All agents construct delegations. Chains verified end-to-end.

Phase 1 alone provides meaningful security: the terminal action is verified, the full chain is checked, and scope restriction is enforced at the point of consequence, with no changes to internal agent implementations.

<br>
<br>

## Licensing

The ADCS specification and all reference materials are released under the Apache License 2.0. The standard is open, royalty-free, and vendor-neutral. Organizations are encouraged to adopt, implement, reference, and build upon it.

<br>
<br>

## Future Work

Future ADCS extensions planned:

* **Semantic Constraint Mapping Standard** - normative specification for mapping constraint values to tool parameters
* **Constraint Accounting Service (CAS) Reference Implementation** - cumulative budget tracking across chains
* **Additional DID Method Bindings** - `did:tdw`, `did:ion`, and other emerging DID methods
* **ADCS-ALS Integration Profile** - coordinated chain revocation on ALS session halt
* **Cross-Organization Trust Framework** - mutual recognition agreements and federated DID resolution profiles

<br>
<br>

## Community and Collaboration

ADCS is an open initiative. The standard is published as a Draft for Public Comment and the community's input will shape its evolution.

The project welcomes:

* Feedback on the specification - technical review, edge case analysis, implementation experience
* Protocol binding implementations - MCP, A2A, ACP, and HTTP binding libraries
* Reference SDK implementations - chain builder, verifier, token issuer, revocation client (see Appendix E for interface contracts)
* Conformance test contributions - help complete and extend the 48+ test vector suite
* Constraint type extensions - vendor-specific constraint types with restriction rule definitions
* DID method integration - production experience reports with `did:web`, `did:jwk`, `did:key`, and SPIFFE/SVID
* Governance layer integrations - policy engine adapters, semantic mapping implementations, intent-action consistency checkers
* Ecosystem tooling - chain visualization, audit log dashboards, CAS implementations

The goal is to establish ADCS as the standard authorization chain for multi-agent AI workflows, so that every agent framework, tool server, and governance platform has a single, reliable, interoperable mechanism to verify that an agent is authorized for the action it is attempting, within the scope that was delegated to it.

<br>
<br>
