# Intent Agent Native Authorization for Agentic Profiles

**Document:** `draft-embesozzi-intent-agent-native-authorization-00`  
**Status:** Individual Draft — Work in Progress  
**Author:** Martin Besozzi  
**Date:** 2026-07-01

## Abstract

This specification defines a framework that enables a first-party AI agent to obtain explicit user approval for the specific operations it plans to execute, and enables enforcement points to verify at runtime that every invocation remains within that approval. The agent discovers the authorization requirements that services publish in their agentic profiles, computes from the user's prompt a structured authorization intent expressed as Rich Authorization Request entries (RFC 9396), and obtains user approval through the native, browser-less challenge flow defined by OAuth 2.0 for First-Party Applications and OAuth 2.0 Agent Native Authorization. The resulting access token carries the approved intent; enforcement points verify each tool invocation against it using a Policy Decision Point, such as the OpenID AuthZEN Authorization API or a local policy engine. This framework is referred to as Intent Agent Native Authorization for Agentic Profiles (IANA-AP).

## Design Intent

This document presents a unified view of the Intent Agent Native Authorization framework. It is intended as a reference integration point that shows how discovery, intent computation, user authorization, and enforcement compose into a coherent whole. Based on community feedback, individual components (e.g., the `x-authz-mapping` discovery mechanism, the `agent_intent` RAR type, the enforcement profile) may be specified in separate IETF documents. Readers should engage with this draft as a framework specification, not necessarily the final form of each component. The known open points and candidate future specifications are enumerated in the draft's "Open Items and Future Specifications" appendix.

## The Problem

Traditional authorization answers a single question: *Is this request allowed?* Agentic systems require an additional runtime guarantee: *Does this request still belong to the execution plan the user explicitly and cryptographically approved?*

Unlike deterministic software, an AI agent may determine which tools and resources it needs only after it understands the user's request — so the required authorization is discovered at the moment of execution, not upfront at login. Before an AI agent can execute on behalf of a user, it must answer four questions, each answered by one phase of the framework:

| Question | Phase |
|---|---|
| What authorization does each service require? | 1. Discover Required Authorization |
| Which operations is the agent planning to execute? | 2. Compute Authorization Intent |
| Has the user explicitly approved those operations? | 3. Review and Approve |
| Does every runtime invocation remain within that approval? | 4. Runtime Enforcement |

## Key Concepts

**Four-phase framework:**

1. **Discover Required Authorization**: The agent reads the authorization requirements declared in the agentic profile via the `x-authz-mapping` field, collecting the SARC-structured (Subject, Action, Resource, Context) mappings for each operation it plans to invoke. Services advertise requirements dynamically, so no service-specific authorization logic needs to be built into the agent.
2. **Compute Authorization Intent**: The agent — optionally via a dedicated Intent Agent that only plans and never executes — maps the user's prompt against the discovered mappings to compute a precise set of `authorization_details` entries of type `agent_intent`, one per planned tool invocation.
3. **Review and Approve**: The user reviews the exact planned operations and approves with a phishing-resistant Passkey. No browser redirect — the challenge is delivered to the agent as a structured, machine-readable elicitation and rendered natively within the agent's UX. Approval produces a cryptographically bound token.
4. **Runtime Enforcement**: The PEP validates the token against each invocation using a PDP (Remote via AuthZEN, or Local via Cedar/OPA), and verifies the actual call arguments match the approved `resource.properties.arguments` — closing the time-of-check to time-of-use gap.

## Composable by Design

IANA-AP follows a Lego-style architecture — each layer is independently replaceable and builds on existing standards:

- **Identity & Delegation Foundation** — OAuth 2.0 and OpenID Connect. No new identity infrastructure required.
- **Zero Trust Enforcement** — Policy Enforcement Points (PEPs) deployed across the AI stack (MCP servers, gateways, APIs) for consistent fine-grained policy decisions. The PDP can be remote (AuthZEN API) or local (Cedar, OPA) — the framework is PDP-agnostic.
- **Runtime Authorization** — IANA-AP adds intent-based authorization on top: the agent discovers authorization requirements, computes a structured intent from the user's prompt, and obtains cryptographic human approval before acting.

The Authorization Server, the Policy Decision Point, and the Agentic Profile are all independently swappable. Swap Keycloak for Auth0, replace a remote AuthZEN PDP with a local Cedar engine, or extend from MCP to A2A — the framework interfaces remain unchanged.

Each layer builds on existing, independently governed specifications:

| Specification | Role in IANA-AP |
|----------|------|
| OAuth 2.0 Rich Authorization Requests (RFC 9396) | Structured intent representation via the `agent_intent` RAR type |
| OAuth 2.0 First-Party Applications (FiPA) | Native authorization challenge/response flow, no browser redirect |
| OAuth 2.0 Agent Native Authorization (ANA) | Structured elicitation format for in-agent user approval |
| WebAuthn / FIDO2 Passkeys | Phishing-resistant, device-bound approval gesture |
| OpenID AuthZEN | Standardized PEP-to-PDP interface |
| SPIFFE (JWT-SVIDs) | Workload identity for agent credentials |

IANA-AP does not replace existing standards. It composes them.

## Open Items

This draft is a framework under active development. The known open points — each a
candidate for a future companion specification — are detailed in the draft's
"Open Items and Future Specifications" appendix:

| Open item | Question |
|---|---|
| A2A / OpenAPI profile bindings | `x-authz-mapping` bindings for A2A Agent Cards and OpenAPI operations |
| `agent_intent` type registration | Final RAR type name and a formal JSON schema |
| Local PDP mappings | Normative Cedar/OPA bindings (AuthZEN is the normative interface today) |
| Intent-aware policy evaluation | How the approved intent is conveyed to the PDP |
| Token attenuation and downscoping | Narrowing the token per server (Token Exchange / Transaction Tokens) and delegation across agent chains |

Feedback on these items is especially welcome — see [Contributing](#contributing).

## Documents

| Document | Link |
|----------|------|
| **Editor's Copy ** | [draft-embesozzi-intent-agent-native-authorization](https://embesozzi.github.io/draft-embesozzi-intent-agent-native-authorization/) |
| **Draft (Markdown)** | [draft-embesozzi-intent-agent-native-authorization.md](./draft-embesozzi-intent-agent-native-authorization.md) |
| **Introductory article** | [Applying Zero Trust to AI Agents with Intent Agent Native Authorization](https://embesozzi.github.io/posts/martin-besozzi/intent-agent-native-authorization-agentic-profiles/) |

### Intent Agent Native Authorization — In Action

[![Demo Video](https://img.youtube.com/vi/2IoPhwswXyE/0.jpg)](https://youtu.be/2IoPhwswXyE)

## Contributing

Feedback is welcome via GitHub Issues or Pull Requests.

## Author

Martin Besozzi — embesozzi@gmail.com
