# Intent Agent Native Authorization for Agentic Profiles

**Document:** `draft-embesozzi-intent-agent-native-authorization-00`  
**Status:** Individual Draft — Informational  
**Author:** Martin Besozzi  
**Date:** 2026-07-01

## Abstract

This document defines Intent Agent Native Authorization (IANA-AP), a framework that enables first-party AI agents to discover the authorization requirements of agentic profiles, compute a structured authorization intent from user prompts, obtain human approval via a native authorization flow, and present a token that enforcement points verify using fine-grained policy decisions.

The framework composes existing specifications: an `x-authz-mapping` field in agentic profile tool definitions for SARC-structured authorization requirement discovery; OAuth 2.0 Rich Authorization Requests (RFC 9396) for structured intent representation; the OAuth 2.0 First-Party Applications draft and the OAuth 2.0 Agent Native Authorization draft for the human approval flow; and a SARC-structured authorization request that enforcement points evaluate using a Policy Decision Point (PDP) — AuthZEN by default, or any compatible policy engine including local evaluation.

This version defines the framework for the Model Context Protocol (MCP). Extension to Agent2Agent (A2A) and OpenAPI-described REST APIs is noted as future work.

## Key Concepts

**Four-phase framework:**

1. **Discovery** — The agent reads `x-authz-mapping` fields from the agentic profile's tool definitions to understand what authorization each tool requires.
2. **Intent Computation** — The agent (optionally via a dedicated Intent Agent) maps the user's prompt against the discovered mappings to compute a precise set of `authorization_details` entries.
3. **User Authorization** — The user reviews and approves the intent with strong authentication (e.g., passkey), producing a cryptographically bound token.
4. **Enforcement** — The PEP validates the token against each tool call using a PDP (Remote via AuthZEN, or Local via Cedar/OPA).

**Shifts authorization from login time to runtime** — the token is not a broad credential, it is a cryptographically bound record of what the user chose to authorize for a specific task.

## Documents

| Document | Link |
|----------|------|
| **Draft (Markdown)** | [draft-embesozzi-intent-agent-native-authorization-00.md](./draft-embesozzi-intent-agent-native-authorization-00.md) |

## Related Specifications

| Specification | Role |
|---|---|
| [draft-embesozzi-oauth-agent-native-authorization](https://datatracker.ietf.org/doc/draft-embesozzi-oauth-agent-native-authorization/) | User authorization phase (ANA) |
| [draft-ietf-oauth-first-party-apps](https://datatracker.ietf.org/doc/draft-ietf-oauth-first-party-apps/) | First-party OAuth flow (FiPA) |
| [RFC 9396 — Rich Authorization Requests](https://datatracker.ietf.org/doc/html/rfc9396) | `authorization_details` / `agent_intent` type |
| [AuthZEN Authorization API 1.0](https://openid.net/specs/authorization-api-1_0.html) | Remote PDP interface |


### Intent Agent Native Authorization — In Action

[![Demo Video](https://img.youtube.com/vi/2IoPhwswXyE/0.jpg)](https://youtu.be/2IoPhwswXyE)

## Contributing

Feedback is welcome via GitHub Issues or Pull Requests.

## Author

Martin Besozzi — embesozzi@gmail.com
