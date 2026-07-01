---
title: "Intent Agent Native Authorization for Agentic Profiles"
abbrev: "IANA-AP"
category: info

docname: draft-embesozzi-intent-agent-native-authorization-00
submissiontype: independent
number:
date: 2026-06-22
consensus: false
v: 3
keyword:
 - authorization
 - agent
 - intent
 - MCP
 - AuthZEN
 - SARC
 - OAuth
 - first-party

author:
 -
    fullname: Martin Besozzi
    organization: Independent
    email: embesozzi@gmail.com

normative:
  RFC2119:
  RFC8174:
  RFC9396:
  RFC7519:
  FIPA:
    title: "OAuth 2.0 for First-Party Applications"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-first-party-apps/
    author:
      org: IETF OAuth Working Group
    date: 2025
  ANA:
    title: "OAuth 2.0 Agent Native Authorization"
    target: https://datatracker.ietf.org/doc/draft-embesozzi-oauth-agent-native-authorization/
    author:
      -
        fullname: Martin Besozzi
    date: 2025
  AUTHZEN:
    title: "Authorization API 1.0"
    target: https://openid.net/specs/authorization-api-1_0.html
    author:
      org: OpenID Foundation
    date: 2026
informative:
  RFC8707:
  COAZ:
    title: "AuthZEN Profile for Model Context Protocol Tool Authorization"
    target: https://github.com/openid/authzen/blob/main/profiles/authzen-mcp-profile-1_0.md
    author:
      org: OpenID AuthZEN Working Group
    date: 2026
  RFC6749:
  RFC8693:
  MCP:
    title: "Model Context Protocol Specification"
    target: https://spec.modelcontextprotocol.io/
    author:
      org: Anthropic
    date: 2025
  A2A:
    title: "Agent2Agent Protocol"
    target: https://google.github.io/A2A/
    author:
      org: Google
    date: 2025
  OPENAPI:
    title: "OpenAPI Specification"
    target: https://spec.openapis.org/oas/latest.html
    author:
      org: OpenAPI Initiative
    date: 2024
  TXN-TOKENS:
    title: "Transaction Tokens"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-transaction-tokens/
    author:
      org: IETF OAuth Working Group
    date: 2025
  CEL:
    title: "Common Expression Language"
    target: https://github.com/google/cel-spec
    author:
      org: Google
    date: 2023

--- abstract

This document defines Intent Agent Native Authorization (IANA-AP), a framework that enables first-party AI agents to discover the authorization requirements of agentic profiles, compute a structured authorization intent from user prompts, obtain human approval via a native authorization flow, and present a token that enforcement points verify using fine-grained policy decisions.

The framework composes existing specifications: a machine-readable authorization mapping in agentic profile tool definitions for SARC-structured authorization requirement discovery; OAuth 2.0 Rich Authorization Requests (RFC 9396) for structured intent representation; the OAuth 2.0 First-Party Applications draft and the OAuth 2.0 Agent Native Authorization draft for the human approval flow; and a SARC-structured authorization request that enforcement points evaluate using a Policy Decision Point (PDP) — AuthZEN by default, or any compatible policy engine including local evaluation. The mapping format is inspired by the COAZ profile and is designed to be PDP-agnostic.

This version defines the framework for the Model Context Protocol (MCP). Extension to Agent2Agent (A2A) and OpenAPI-described REST APIs is noted as future work.

--- middle

# Introduction

OAuth 2.0 {{RFC6749}} and its extensions define how credentials are issued and scoped. They do not define how an AI agent discovers what permissions a given agentic profile requires before execution, nor how the agent presents those requirements to a human user in a comprehensible form for approval before any tool is called.

This gap is acute in first-party agentic deployments. A user instructs an agent to carry out a task. The agent must call one or more tools on MCP servers, A2A agents, or REST APIs. Each of those tools requires specific authorized actions on specific resources. Without a discovery mechanism, the agent either requests overly broad permissions upfront or fails at runtime when enforcement points reject its token.

Traditional OAuth deployments grant permissions at login time: the user approves a set of scopes once, and every subsequent action is authorized implicitly under those scopes. This model is unsuitable for agentic deployments, where the specific actions and resources involved are not known until the user issues a prompt. This specification shifts authorization from login time to runtime: the agent computes the exact intent from the user's prompt, the user approves it at the moment of action with a cryptographic gesture via {{ANA}}, and the resulting token carries that precise approved intent. The token is not a broad credential — it is a cryptographically bound record of what the user chose to authorize for this specific task. Unlike browser-based OAuth flows, this approval happens natively within the agent's UX — no redirect, no browser pop-up — via a structured machine-readable challenge that the agent renders directly to the user.

This document defines a framework with four phases:

1. **Discovery**: The agent reads the SARC mappings published in the agentic profile's tool definitions before execution. For MCP, these mappings use the `x-authz-mapping` field defined in Section 4.2.

2. **Intent Computation**: The agent maps the user's prompt against the discovered SARC entries to produce a set of `authorization_details` entries ({{RFC9396}}) that represent the precise set of authorized actions the user will be asked to approve.

3. **User Authorization**: The agent drives an OAuth 2.0 first-party authorization challenge carrying the computed `authorization_details`. The user sees the exact planned tool invocations and approves natively — no browser redirect — via a structured machine-readable challenge ({{ANA}}). This phase follows {{FIPA}} and {{ANA}}.

4. **Enforcement**: The agent presents the resulting token to the Enforcement Point (PEP). The PEP maps each tool invocation to the token's `authorization_details`, extracts the SARC parameters, and calls a Policy Decision Point (PDP) for a fine-grained allow/deny decision. AuthZEN {{AUTHZEN}} is the normative PDP interface; other compatible policy engines MAY be used.

This framework is designed to be composable with existing infrastructure. An organization that has deployed OAuth 2.0, OpenID Connect, and AuthZEN-compatible Policy Enforcement Points can extend to intent-based authorization by adopting this specification without replacing any existing component. The Authorization Server, the Policy Decision Point, and the Agentic Profile are independently interchangeable — this specification defines the interfaces between them, not their implementations. An operator may start with a remote AuthZEN PDP and later replace it with a local Cedar or OPA engine; swap one MCP server for another; or adopt a different OAuth 2.0 AS — without modifying the framework. No redesign; just composition.

## Relationship to Existing Specifications

This specification builds on and composes the following:

- **COAZ {{COAZ}}**: Informatively referenced as the origin of the `x-authz-mapping` field concept and SARC-structured tool mapping. IANA-AP defines `x-authz-mapping` independently (Section 4.2) to be PDP-agnostic and AuthZEN-compatible, and intentionally diverges from COAZ's enforcement-time resource construction behavior (Section 8.3). COAZ is cited as background; IANA-AP implementations are not required to conform to it.

- **RFC 9396 {{RFC9396}}**: The `authorization_details` parameter and RAR token structure are used as-is. This specification defines a new RAR type `agent_intent` to carry SARC-structured intent entries.

- **draft-ietf-oauth-first-party-apps {{FIPA}} and draft-embesozzi-oauth-agent-native-authorization {{ANA}}**: The user authorization phase is fully delegated to these two specifications. This document normatively references them and does not respecify the challenge/response mechanics.

- **AuthZEN {{AUTHZEN}}**: The normative Remote PDP interface. This specification defines how the PEP constructs the AuthZEN request from the token's `authorization_details` and the active tool call. Implementations using a Local PDP (Cedar, OPA) follow the same SARC structure with a different evaluation mechanism (Section 8.3).

## Relationship to Mission-Bound Authorization

Mission-Bound Authorization defines a durable Mission object that persists across token rotations and delegation chains. That work and this specification share the intuition that a token alone is insufficient to capture a human-approved task. The key difference is focus: Mission-Bound Authorization defines the lifecycle governance of a mission artifact; this specification defines how an agent *discovers* what to request in the first place and how the resulting token is enforced at the point of action execution. The two specifications are complementary and future work may define a binding between `agent_intent` entries and a Mission object.

## Requirements Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals.

# Terminology

The following terms are used in this document:

Agent:
: A first-party AI agent acting on behalf of a human user. The agent reads agentic profiles, computes authorization intent, and drives the user authorization flow.

Agentic Profile:
: A machine-readable description of a service that an agent can invoke. In this version, the Agentic MCP Profile is the authoritative source of tool definitions and authorization mappings — exposed via the MCP Server Card or a `tools/list` response, as defined by the MCP specification {{MCP}}. Future versions extend this concept to A2A Agent Cards and OpenAPI descriptions.

SARC Model:
: The Subject-Action-Resource-Context model as defined in {{AUTHZEN}}. In this specification, SARC parameters are expressed in the `agent_intent` RAR type (Section 5.2); the `x-authz-mapping` field (Section 4.2) carries CEL expressions for `action`, `resource`, and `arguments` that are evaluated to populate the SARC fields at intent computation time.

Authorization Mapping:
: The `x-authz-mapping` extension field within the `_meta` object of an agentic profile tool definition, as defined in Section 4.2. Its presence declares the tool as participating in IANA-AP (opt-in signal). It carries CEL expressions for `action`, `resource`, and `arguments` that the agent evaluates to construct the `agent_intent` entry. The field structure is AuthZEN-compatible and PDP-agnostic.

Authorization Intent:
: The set of `authorization_details` entries of type `agent_intent` that the agent computes from the user's prompt and the discovered SARC mappings. Each entry represents one planned tool invocation.

Intent Agent:
: A specialized agent component that performs authorization intent planning. It operates before any tool invocation, consuming the discovery corpus and the user's prompt, and outputs the set of `agent_intent` entries for user approval. It has no tool execution capability. Implementations MAY realize this as a separate LLM call with a constrained system prompt dedicated solely to intent planning.

RAR Token:
: An OAuth 2.0 access token that carries an `authorization_details` claim ({{RFC9396}}) containing one or more `agent_intent` entries.

Enforcement Point (PEP):
: The component that intercepts tool invocations, extracts the token's `authorization_details`, and calls the PDP before executing the tool. A single abstract actor; in the Agentic MCP Profile this is an MCP server or MCP gateway; other profiles define their own equivalent enforcement points.

PDP:
: A Policy Decision Point that evaluates a SARC-structured authorization request and returns an allow or deny decision. This specification supports two deployment modes:
  - **Remote PDP**: The PEP calls an external authorization service over the network using the AuthZEN Authorization API {{AUTHZEN}}. This is the normative interface defined in this specification.
  - **Local PDP**: The PEP evaluates policy locally using an embedded policy engine (e.g., Cedar, OPA). The PEP extracts the same SARC fields from the `agent_intent` entry and maps them to the engine's native input format.

  In both modes, the SARC parameters sourced from the `agent_intent` entry are identical.

Authorization Server (AS):
: The OAuth 2.0 Authorization Server that issues RAR tokens after the user has approved the authorization intent via the first-party flow.

# Protocol Overview

The following diagram shows the full four-phase flow. In this version of the
specification, the Agentic Profile is an MCP Server; future versions extend this
framework to A2A Agent Cards and OpenAPI descriptions (Section 4.1).

~~~
User          Agent          AS      Agentic Profile / PEP    PDP
 |              |             |                    |                |
 |              | [Phase 1: Discovery]              |                |
 |              |             |                    |                |
 |  (1) Prompt  |             |                    |                |
 |------------->|             |                    |                |
 |              |             |                    |                |
 |              | (2) Agentic Profile request       |                |
 |              |---------------------------------->|                |
 |              | (3) Agentic Profile + x-authz-mapping             |
 |              |<----------------------------------|                |
 |              |             |                    |                |
 |              | [Phase 2: Intent Computation]     |                |
 |              |             |                    |                |
 |              | (4) Compute authorization_details (agent_intent)  |
 |              |             |                    |                |
 |              | [Phase 3: User Authorization]     |                |
 |              |             |                    |                |
 |              | (5) Auth Req + authorization_details     |                |
 |              |------------>|                    |                |
 |              |             |                    |                |
 |  (6) Present intent for approval (via ANA)      |                |
 |<-------------|             |                    |                |
 |              |             |                    |                |
 |  (7) Approve (passkey / strong authn)           |                |
 |------------->|             |                    |                |
 |              | (8) Complete challenge            |                |
 |              |------------>|                    |                |
 |              | (9) RAR token (authorization_details: agent_intent)|
 |              |<------------|                    |                |
 |              |             |                    |                |
 |              | [Phase 4: Enforcement]            |                |
 |              |             |                    |                |
 |              | (10) Tool call + Bearer RAR token|                |
 |              |---------------------------------->|                |
 |              |             |                    | (11) PDP       |
 |              |             |                    |  Request       |
 |              |             |                    |--------------->|
 |              |             |                    | (12) Decision  |
 |              |             |                    |<---------------|
 |              |             |                    |                |
 |              | (13) Tool result                 |                |
 |              |<----------------------------------|                |
~~~

Steps:

1. The user submits a natural language prompt to the agent.
2. The agent reads the Agentic MCP Profile (via the MCP Server Card or `tools/list` response, as supported by the server).
3. The Agentic Profile returns tool definitions including `x-authz-mapping` entries as defined in Section 4.2.
4. The agent maps the prompt against the discovered authorization mappings to compute an `authorization_details` array of `agent_intent` entries (Section 5).
5. The agent submits an authorization request to the AS carrying the computed `authorization_details`, following the flow defined in {{FIPA}} and {{ANA}}.
6. The AS returns an authorization challenge; the agent presents the intent to the user following {{ANA}}.
7. The user reviews the presented SARC entries and approves with strong authentication (e.g., passkey).
8. The agent completes the challenge response following {{FIPA}} and {{ANA}}.
9. The AS issues a RAR token containing the approved `authorization_details`.
10. The agent calls the tool on the Agentic Profile, presenting the RAR token as a Bearer token.
11. The PEP maps the tool call to the matching `agent_intent` entry in the token and calls the PDP (Remote via AuthZEN, or Local via an embedded engine).
12. The PDP returns an allow or deny decision.
13. On allow, the PEP executes the tool and returns the result.

# Agentic Profile Discovery

## Overview

Before computing an authorization intent, the agent MUST discover the authorization mappings for the tools it plans to invoke. The discovery mechanism is profile-specific: each Agentic Profile type exposes mappings in a different way. This section defines discovery for the MCP profile and reserves subsections for A2A and OpenAPI profiles.

## Authorization Mapping in Agentic MCP Profile

This specification defines the `x-authz-mapping` extension field for declaring that a tool participates in IANA-AP and for expressing the full SARC model as CEL expressions. The field is inspired by the COAZ profile {{COAZ}} and is designed to be AuthZEN-compatible while remaining PDP-agnostic.

The `x-authz-mapping` field serves two purposes: (1) as an opt-in signal — its presence declares the tool participates in IANA-AP; (2) as a SARC mapping — its CEL expressions define how to derive `action`, `resource`, and call `arguments` from the request context at both intent computation time and enforcement time.

The field contains three sub-objects:

- `action`: CEL expressions for the `action` component of the SARC model.
- `resource`: CEL expressions for the `resource` component. Single-quoted strings (e.g., `'mcp-tool'`) are CEL string literals.
- `arguments`: a map of argument names to CEL expressions that resolve argument values from the user's prompt at intent computation time.

`resource.properties.aud` is NOT carried in the per-tool mapping. It is sourced from the MCP Server Card at intent computation time (the server-level identity URI is a server-scoped, not tool-scoped, attribute).

### CEL Evaluation Context

Expression evaluation uses Common Expression Language (CEL) {{CEL}}. The use of CEL expressions in tool authorization mappings follows the convention established by the COAZ profile {{COAZ}}. The evaluation context is defined identically for both Phase 2 (planned call) and Phase 4 (actual MCP request), enabling the same expressions to be evaluated by both the agent and the PEP:

| CEL variable | Value |
|---|---|
| `params.method` | MCP JSON-RPC method name (e.g., `"tools/call"`) |
| `params.name` | Tool name from the request (e.g., `"disable_user"`) |
| `params.arguments.X` | Argument X from the tool call parameters |
| `token.X` | Claim X from the caller's JWT |

At Phase 2, the agent evaluates expressions against the planned call values (tool name known from discovery, argument values extracted from the user's prompt). At Phase 4, the PEP evaluates the same expressions against the actual incoming MCP request.

An MCP tool that wishes to support pre-computed authorization intent MUST include an `x-authz-mapping` field within its `_meta` object. Tools that omit this field cannot be pre-authorized; when the agent calls such a tool, the PEP will find no matching `agent_intent` entry and MUST apply just-in-time re-authorization (Section 8.4).

A conforming tool definition example:

~~~json
{
  "name": "disable_user",
  "description": "Disable a user account by userId.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "userId": { "type": "string" }
    },
    "required": ["userId"]
  },
  "_meta": {
    "x-authz-mapping": {
      "action":   { "name": "params.method" },
      "resource": { "type": "'mcp-tool'", "id": "params.name" },
      "arguments": { "userId": "params.arguments.userId" }
    }
  }
}
~~~

For tools with no call arguments, `arguments` MAY be omitted:

~~~json
"_meta": {
  "x-authz-mapping": {
    "action":   { "name": "params.method" },
    "resource": { "type": "'mcp-tool'", "id": "params.name" }
  }
}
~~~

## Agent Discovery Procedure

The agent MUST perform discovery before execution begins, not lazily at the point of each tool call. Specifically:

1. The agent reads the Agentic MCP Profile from each MCP server it intends to use (via the MCP Server Card or `tools/list` response, as supported by the server {{MCP}}).
2. For each tool in the profile, the agent checks for the presence of `x-authz-mapping` within the tool's `_meta` object.
3. The agent retains the set of tools with an `x-authz-mapping` entry as the discovery corpus for intent computation.

Tools without an `x-authz-mapping` field in their `_meta` cannot be included in pre-computed authorization intent. If the agent calls such a tool, no matching `agent_intent` entry will exist in the presented token. The PEP MUST apply just-in-time re-authorization as described in Section 8.4.

## Authorization Mapping in Agentic A2A Profile

This section is reserved for future specification. A future version of this document will define an equivalent discovery procedure for A2A Agent Cards {{A2A}}, where SARC mappings are expressed as an extension to the Agent Card JSON document served at `/.well-known/agent.json`. The `x-authz-mapping` field structure (`action`, `resource`, `arguments` as CEL expressions) is profile-agnostic; a future A2A binding will define the appropriate CEL evaluation context and `resource.type` literal for that profile.

## Authorization Mapping in Agentic OpenAPI Profile

This section is reserved for future specification. A future version of this document will define an equivalent discovery procedure for REST APIs described with OpenAPI {{OPENAPI}}, where SARC mappings are expressed as an `x-authz-mapping` extension field on individual operation objects. A future OpenAPI binding will define the CEL evaluation context and `resource.type` literal appropriate for that profile.

# Intent Computation

## Overview

Intent computation is the process by which the agent translates a user's natural language prompt, together with the discovered authorization mappings (`x-authz-mapping`), into a concrete set of `authorization_details` entries of type `agent_intent`. These entries represent the exact tool invocations the agent plans to make and the specific resources and actions involved. This role MAY be realized as a dedicated Intent Agent — a constrained component with no tool execution capability, whose sole responsibility is authorization intent planning (see Terminology).

## agent_intent RAR Type

This specification defines a new Rich Authorization Requests {{RFC9396}} type with the working name `agent_intent`. The type name is subject to IANA registration (Section 10).

An `agent_intent` entry is structured as a SARC-based request template compatible with the AuthZEN Authorization API {{AUTHZEN}}. The PEP reads it directly, replaces `resource.properties.arguments` with the actual call arguments, adds the `subject` from the token, and calls the PDP — no structural translation required.

~~~json
{
  "type": "agent_intent",
  "action": { "name": "tools/call" },
  "resource": {
    "type": "mcp-tool",
    "id": "disable_user",
    "properties": {
      "aud": "urn:example:mcp:identity-server",
      "arguments": { "userId": "alice" }
    }
  }
}
~~~

The fields are:

type:
: REQUIRED. The string `agent_intent`.

action:
: REQUIRED. An object with a `name` field. For MCP, `name` is always the string `"tools/call"` — the protocol-level method name, uniform across all MCP tool invocations.

resource:
: REQUIRED. An object with the following fields:
  - `type`: the string `"mcp-tool"`
  - `id`: the tool name being invoked
  - `properties.aud`: the URI identifying the MCP server (sourced from the Server Card at intent computation time; see Section 5.4)
  - `properties.arguments`: the approved call arguments resolved from the user's prompt

## Subject Inference

The `agent_intent` entry does not carry a Subject field. The Subject is the principal who approved the token, identified by the `sub` claim of the RAR token ({{RFC7519}}). The agent's identity, if relevant to policy evaluation, is carried at the token level — in the `client_id` claim or the `act` claim ({{RFC8693}}) — not inside individual `agent_intent` entries. The PEP reads agent identity from the token directly when constructing the PDP request `context` (Section 8.3).

## Computation Procedure

The following procedure applies to the Agentic MCP Profile. Future versions of this specification define equivalent procedures for A2A and OpenAPI profiles.

The agent MUST compute the authorization intent as follows:

1. For each tool the agent determines it will call (based on the user's prompt and the available tool definitions), locate the corresponding `x-authz-mapping` entry from the discovery corpus (Section 4.2). If no `x-authz-mapping` entry exists for a tool, that tool MUST NOT be included in the pre-computed authorization intent; the agent MAY still attempt to call it, but the PEP will apply just-in-time re-authorization (Section 8.4).
2. Evaluate the `x-authz-mapping` CEL expressions against the planned call context (Section 4.2) to construct the `agent_intent` entry:
   - `action.name`: evaluate `x-authz-mapping.action.name` (e.g., `params.method` → `"tools/call"`)
   - `resource.type`: evaluate `x-authz-mapping.resource.type` (e.g., `'mcp-tool'` → `"mcp-tool"`)
   - `resource.id`: evaluate `x-authz-mapping.resource.id` (e.g., `params.name` → tool name)
   - `resource.properties.aud`: set to the MCP server's identity URI sourced from the Server Card (not from the per-tool mapping)
   - `resource.properties.arguments`: evaluate each expression in `x-authz-mapping.arguments` against the prompt context, extracting argument values from the user's natural language input

3. Produce one `agent_intent` entry per planned tool invocation.
4. Collect all entries into an `authorization_details` array.

When the agent plans tool invocations across multiple Agentic Profile servers, the authorization request SHOULD include all server URIs as `resource` parameter values ({{RFC8707}}) so the issued token carries an `aud` claim covering all intended servers.

The agent MUST present the computed `authorization_details` to the user for review before submitting the authorization request (Step 6 in Section 3).

# User Authorization

Unlike browser-based OAuth flows where the user is redirected to a consent page, this framework delivers the authorization challenge directly to the agent in a structured machine-readable format. The agent renders this natively — as a prompt, dialog, or UI component within its own interface — without any browser interaction. The user sees exactly what the agent plans to do (which tools, on which resources, with which arguments) before any authentication gesture is requested.

The user authorization phase is fully delegated to two existing specifications:

- **{{FIPA}}** defines the OAuth 2.0 first-party authorization challenge/response flow. The agent MUST use this flow to submit the authorization request carrying the `authorization_details` and to complete the challenge.

- **{{ANA}}** defines the structured elicitation format that the AS uses to communicate the challenge to the agent. The AS returns HTTP 400 with `error: "insufficient_authorization"` and an `elicitations` array that carries a human-readable `message` summarizing the agent_intent entries awaiting approval and a `requestedSchema` for authenticator selection. The agent renders this natively to the user. The agent MUST implement the `elicitations` response format defined in {{ANA}}.

The authorization challenge request MUST include the `authorization_details` parameter carrying the `agent_intent` entries computed in Section 5.

## Authorization Challenge Example

The following example shows the AS challenge response for two `agent_intent` entries. The AS returns HTTP 400 with `error: "insufficient_authorization"` and an `elicitations` array as defined in {{ANA}}. The `message` field provides a human-readable summary of the planned tool invocations; the `requestedSchema` requests the user's authenticator selection.

~~~http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "insufficient_authorization",
  "auth_session": "sess_abc123",
  "elicitations": [
    {
      "mode": "form",
      "message": "The agent wants to perform the following actions:\n- Disable user alice (identity-server)\n- Delete user alice (identity-server)\n\nApprove with your passkey to proceed.",
      "requestedSchema": {
        "type": "object",
        "properties": {
          "authenticator": {
            "type": "string",
            "title": "Authentication Method",
            "oneOf": [
              { "const": "passkey", "title": "Passkey" }
            ]
          }
        },
        "required": ["authenticator"]
      }
    }
  ]
}
~~~

The user reviews the two planned invocations (`disable_user` and `delete_user` on the identity server) and selects `passkey`. The passkey WebAuthn challenge delivery follows {{ANA}}; the WebAuthn ceremony is performed in-band by the first-party agent without browser interaction. On successful authentication, the AS issues a RAR token containing the approved `authorization_details`.

# Token Structure

## RAR Token Claims

The Authorization Server issues an OAuth 2.0 access token that carries the approved `authorization_details` as a top-level claim ({{RFC9396}}, Section 9). The token MUST contain:

- `sub`: The subject (user) who approved the intent.
- `aud`: One or more server URIs identifying the Agentic Profile servers at which this token is valid. When the agent plans invocations across multiple servers, `aud` MUST be an array covering all server URIs referenced in the `authorization_details` entries (see Section 5.4 and {{RFC8707}}).
- `authorization_details`: An array of one or more `agent_intent` entries as defined in Section 5.2, reflecting only the entries the AS approved.

The token SHOULD also carry the agent's identity in the `client_id` claim or the `act` claim ({{RFC8693}}). The PEP uses this to populate the PDP `context` at enforcement time (Section 8.3).

The AS MAY reduce the set of requested `agent_intent` entries (e.g., deny individual entries while approving others). The agent MUST inspect the issued token's `authorization_details` and MUST NOT proceed to invoke tools whose corresponding `agent_intent` entry was not approved.

## Example Token Payload

~~~json
{
  "sub": "user@example.com",
  "iss": "https://as.example.com",
  "aud": ["urn:example:mcp:identity-server"],
  "client_id": "spiffe://example.org/agent/identity",
  "exp": 1750000000,
  "authorization_details": [
    {
      "type": "agent_intent",
      "action": { "name": "tools/call" },
      "resource": {
        "type": "mcp-tool",
        "id": "disable_user",
        "properties": {
          "aud": "urn:example:mcp:identity-server",
          "arguments": { "userId": "alice" }
        }
      }
    }
  ]
}
~~~

## Future Token Attenuation

In orchestration scenarios, an agent may obtain a single RAR token covering all planned server audiences and all `agent_intent` entries, then attenuate it before calling each individual server. The intended pattern is:

1. The orchestrator obtains a master token with `aud` covering all servers and `authorization_details` containing all `agent_intent` entries.
2. Before calling server A, the agent exchanges the master token for a narrower token via {{RFC8693}} token exchange (or a Transaction Token {{TXN-TOKENS}}), reducing `aud` to server A's URI and `authorization_details` to only the entries where `resource.properties.aud` matches server A.
3. Server A never receives intent entries intended for other servers.

The per-entry `resource.properties.aud` field (Section 5.2) is what makes this attenuation possible — it identifies which `agent_intent` entries belong to which server.

Full specification of the attenuation exchange, including which claims MUST be preserved and which MAY be reduced, is deferred to a future version of this specification. Implementers MAY prototype this pattern using {{RFC8693}} token exchange or Transaction Tokens {{TXN-TOKENS}}.

# Enforcement

## Overview

In the Agentic MCP Profile, the Enforcement Point (PEP) is an MCP server or MCP gateway that intercepts each tool invocation and evaluates whether the presented token authorizes that specific action on that specific resource before executing the tool. The same enforcement pattern applies to other Agentic Profiles (A2A, OpenAPI) — the profile determines the interception mechanism, but the SARC extraction and PDP call are identical across profiles.

## PEP Procedure

When the agent calls a tool carrying a Bearer token, the PEP MUST:

1. Validate the token (signature, expiry, audience).
2. Extract the `authorization_details` claim and identify the matching `agent_intent` entry for the current invocation. If no matching entry exists, the PEP MUST apply just-in-time re-authorization (Section 8.4).
3. Extract the SARC parameters from the matching `agent_intent` entry and construct a PDP authorization request (Section 8.3).
4. If the PDP returns `allow: true`, execute the invocation and return the result; otherwise reject the request with an appropriate error.

## SARC Matching

The `agent_intent` entry carries the SARC parameters used to evaluate the authorization decision. The PEP extracts these parameters as follows:

- **Subject**: sourced from the token's `sub` claim (not from the `agent_intent` entry).
- **Action**: read from `agent_intent.action`.
- **Resource**: read from `agent_intent.resource`, with `properties.arguments` replaced by the actual invocation arguments. The PEP SHOULD verify the actual arguments are consistent with the approved `resource.properties.arguments` before proceeding.
- **Context**: sourced from the token's `client_id` or `act` claim to identify the agent.

For matching the invocation to the correct `agent_intent` entry, the PEP compares the invoked tool name against `resource.id` and the server's identity URI against `resource.properties.aud`.

Refer to {{AUTHZEN}} for the normative definition of the SARC model and its field semantics. Implementations using other PDP backends (e.g., Cedar, OPA) MUST extract the same SARC fields and map them to the PDP's native input format (Section 8.3).

## PDP Request Construction

### AuthZEN PDP

When the PDP is an AuthZEN endpoint {{AUTHZEN}}, the `agent_intent` entry is already structured as an AuthZEN request template (Section 5.2). The PEP constructs the `access` request with minimal processing: subject from the token's `sub` claim, action and resource copied from the matching `agent_intent` entry (with `properties.arguments` replaced by actual call arguments), and context from the token's `client_id` or `act` claim.

~~~json
{
  "subject": {
    "type": "user",
    "id": "user@example.com"
  },
  "action": {
    "name": "tools/call"
  },
  "resource": {
    "type": "mcp-tool",
    "id": "disable_user",
    "properties": {
      "aud": "urn:example:mcp:identity-server",
      "arguments": {
        "userId": "alice"
      }
    }
  },
  "context": {
    "agent": "spiffe://example.org/agent/identity"
  }
}
~~~

### Local PDP (Cedar, OPA)

When authorization is evaluated locally — using an embedded policy engine such as Cedar or OPA — the PEP does not make a network call to an external service. Instead, it extracts the same SARC fields from the matching `agent_intent` entry and feeds them directly into the local engine:

- **Cedar**: map `subject` to a Cedar `Principal`, `action.name` to a Cedar `Action`, and `resource.type` + `resource.id` to a Cedar `Resource`. Policy is evaluated in-process or via a co-located Cedar engine.
- **OPA**: pass the SARC fields as the OPA `input` document. The policy bundle evaluates the request locally and returns an allow or deny result.

In both cases the SARC fields extracted from `agent_intent` are identical to those used in the Remote PDP (AuthZEN) case. The normative field definitions remain those in {{AUTHZEN}}; only the evaluation mechanism and serialization format differ.

## Just-in-Time Re-Authorization

If the PEP receives an invocation for which no matching `agent_intent` entry exists in the token, this may be because: (a) the tool has no `x-authz-mapping` in the agentic profile and was therefore excluded from the pre-computed authorization intent; (b) the agent is attempting an unplanned invocation; or (c) the approved intent did not include that tool. In all cases, the PEP cannot authorize the call from the existing token. A just-in-time re-authorization mechanism — where the PEP signals to the agent that it must obtain an additional `agent_intent` entry — is a noted extension point and is out of scope for this version.

Implementations encountering this case SHOULD return an HTTP 401 response with a `WWW-Authenticate` header that indicates the missing authorization, to facilitate future standardization of the JIT challenge. This applies regardless of the underlying Agentic Profile, as all profiles defined in this specification operate over HTTP.

# Security Considerations

## Intent Integrity

The `authorization_details` entries in the issued token represent what the user reviewed and approved. Implementations MUST ensure the authorization request is submitted over a secure channel (TLS) and that the AS binds the approved intent to the token without modification beyond what is explicitly permitted (e.g., denial of individual entries).

## Cryptographically Approved Intent

Because the user authorization phase requires strong authentication via {{ANA}} (e.g., a passkey gesture), the `authorization_details` in the issued token constitutes cryptographically approved intent: it is evidence that a specific human approved a specific set of actions on specific resources at a specific time, bound to a specific authentication event. This property is stronger than traditional OAuth scope consent, which may be granted once at login and silently reused across many subsequent operations.

Implementations MUST NOT accept tokens whose `authorization_details` entries were approved via a weaker authentication method than the sensitivity of the authorized actions warrants. The AS SHOULD record the authentication method used during the approval gesture (e.g., as an `amr` or `acr` claim) so that enforcement points can verify the approval strength at tool-call time.

## Resource Binding at Enforcement

The PEP MUST verify that the actual tool call parameters (e.g., the `userId` argument) match the resource ID carried in the approved `agent_intent` entry. An agent that presents a token approved for `alice` and then calls the tool with `userId=bob` MUST be rejected at enforcement time. This check closes the time-of-check-to-time-of-use gap for resource-level authorization.

## Agent Identity

This specification does not define how the agent authenticates to the AS or the PEP. Implementations SHOULD use strong agent identity (e.g., mTLS with SPIFFE/SVID, DPoP) to bind tokens to the agent and prevent token theft or misuse by other agents.

## Scope of User Consent

The user approves a specific set of `agent_intent` entries. The agent MUST NOT invoke tools beyond those entries without first obtaining a new user approval. Enforcement points MUST reject invocations not covered by the token's `authorization_details`.

## First-Party Agent Assumption

This specification is designed for first-party agents — agents operating in the same trust domain as the user and the Authorization Server. Application of this framework to third-party or autonomous agents raises additional trust and consent questions that are out of scope for this version.

# IANA Considerations

TODO

--- back

# Use Cases

## Identity Management — Multi-Tool Authorization

This section illustrates the complete IANA-AP flow for a scenario involving one MCP server and three tools, demonstrating how the Intent Agent computes multiple intents from a single user prompt.

### Scenario

A user instructs an agent:

> "Disable alice's account, delete her data, and list all disabled users."

The agent has access to one MCP server:

- **Identity Server** (`urn:example:mcp:identity-server`) — tools: `disable_user`, `delete_user`, `list_users`

### Phase 1: Discovery

The agent reads the Agentic MCP Profile from the identity server and collects tools with `x-authz-mapping` in their `_meta`. The discovery corpus contains:

| Tool | `action.name` | `resource.type` | `resource.id` | `arguments` |
|---|---|---|---|---|
| `disable_user` | `params.method` | `'mcp-tool'` | `params.name` | `userId: params.arguments.userId` |
| `delete_user` | `params.method` | `'mcp-tool'` | `params.name` | `userId: params.arguments.userId` |
| `list_users` | `params.method` | `'mcp-tool'` | `params.name` | `filter: params.arguments.filter` |

`aud` is not in the per-tool mapping; the agent reads it from the Server Card (`urn:example:mcp:identity-server`).

### Phase 2: Intent Computation (Intent Agent)

The Intent Agent receives the user's prompt and the discovery corpus. It resolves the CEL expressions against the prompt context and produces three `agent_intent` entries:

~~~json
[
  {
    "type": "agent_intent",
    "action": { "name": "tools/call" },
    "resource": {
      "type": "mcp-tool",
      "id": "disable_user",
      "properties": {
        "aud": "urn:example:mcp:identity-server",
        "arguments": { "userId": "alice" }
      }
    }
  },
  {
    "type": "agent_intent",
    "action": { "name": "tools/call" },
    "resource": {
      "type": "mcp-tool",
      "id": "delete_user",
      "properties": {
        "aud": "urn:example:mcp:identity-server",
        "arguments": { "userId": "alice" }
      }
    }
  },
  {
    "type": "agent_intent",
    "action": { "name": "tools/call" },
    "resource": {
      "type": "mcp-tool",
      "id": "list_users",
      "properties": {
        "aud": "urn:example:mcp:identity-server",
        "arguments": { "filter": "disabled" }
      }
    }
  }
]
~~~

Note that `list_users` uses `filter: "disabled"` — the Intent Agent resolved this from the phrase "list all disabled users" in the prompt via the CEL expression `params.arguments.filter`.

### Phase 3: User Authorization

The agent submits an authorization request to the AS carrying the three `agent_intent` entries:

~~~json
{
  "client_id": "spiffe://example.org/agent/identity",
  "authorization_details": [
    { "type": "agent_intent", "action": { "name": "tools/call" },
      "resource": { "type": "mcp-tool", "id": "disable_user",
        "properties": { "aud": "urn:example:mcp:identity-server",
          "arguments": { "userId": "alice" } } } },
    { "type": "agent_intent", "action": { "name": "tools/call" },
      "resource": { "type": "mcp-tool", "id": "delete_user",
        "properties": { "aud": "urn:example:mcp:identity-server",
          "arguments": { "userId": "alice" } } } },
    { "type": "agent_intent", "action": { "name": "tools/call" },
      "resource": { "type": "mcp-tool", "id": "list_users",
        "properties": { "aud": "urn:example:mcp:identity-server",
          "arguments": { "filter": "disabled" } } } }
  ]
}
~~~

The AS returns HTTP 400 with `error: "insufficient_authorization"` and an `elicitations` array ({{ANA}}). The `message` field summarizes the three planned tool invocations in human-readable form. The agent renders this natively — no browser redirect:

~~~http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "insufficient_authorization",
  "auth_session": "sess_abc123",
  "elicitations": [
    {
      "mode": "form",
      "message": "The agent wants to perform the following actions:\n- Disable user alice (identity-server)\n- Delete user alice (identity-server)\n- List disabled users (identity-server)\n\nApprove with your passkey to proceed.",
      "requestedSchema": {
        "type": "object",
        "properties": {
          "authenticator": {
            "type": "string",
            "title": "Authentication Method",
            "oneOf": [
              { "const": "passkey", "title": "Passkey" }
            ]
          }
        },
        "required": ["authenticator"]
      }
    }
  ]
}
~~~

The user reviews the three planned invocations and selects `passkey`. The passkey WebAuthn challenge delivery follows {{ANA}}; the WebAuthn ceremony is performed in-band by the first-party agent without browser interaction. On successful authentication, the AS issues a RAR token:

~~~json
{
  "sub": "user@example.com",
  "aud": ["urn:example:mcp:identity-server"],
  "client_id": "spiffe://example.org/agent/identity",
  "authorization_details": [
    { "type": "agent_intent", "action": { "name": "tools/call" },
      "resource": { "type": "mcp-tool", "id": "disable_user",
        "properties": { "aud": "urn:example:mcp:identity-server",
          "arguments": { "userId": "alice" } } } },
    { "type": "agent_intent", "action": { "name": "tools/call" },
      "resource": { "type": "mcp-tool", "id": "delete_user",
        "properties": { "aud": "urn:example:mcp:identity-server",
          "arguments": { "userId": "alice" } } } },
    { "type": "agent_intent", "action": { "name": "tools/call" },
      "resource": { "type": "mcp-tool", "id": "list_users",
        "properties": { "aud": "urn:example:mcp:identity-server",
          "arguments": { "filter": "disabled" } } } }
  ]
}
~~~

### Phase 4: Enforcement

The agent calls each tool in sequence. The PEP matches each invocation against the token's `authorization_details` and calls the PDP before executing.

### Flow Diagram

~~~
User      Intent Agent      Agent           AS       Identity / PEP    PDP
 |              |              |              |               |           |
 | Prompt: "disable alice, delete, list disabled users"      |           |
 |------------->|              |              |               |           |
 |              |              |              |               |           |
 |              | [Phase 1: Discovery]        |               |           |
 |              |<------------>|              |               |           |
 |              | Agentic Profile req |------------------------->         |
 |              |<-------------|<-Agentic Profile + x-authz-mapping------|
 |              |              |              |               |           |
 |              | [Phase 2: Intent Computation]               |           |
 |              |<-prompt + corpus-|           |               |           |
 |              |--3 agent_intent->|           |               |           |
 |              |              |              |               |           |
 |              | [Phase 3: User Authorization]               |           |
 |              |              |-Auth Req + 3 intents-------->|           |
 |              |              |<-elicitations challenge------|           |
 |<-present 3 intents----------|              |               |           |
 |--approve (passkey)---------->              |               |           |
 |              |              |-complete challenge---------->|           |
 |              |              |<-RAR token (3 agent_intent entries)------|
 |              |              |              |               |           |
 |              | [Phase 4: Enforcement]      |               |           |
 |              |              |-disable_user + token-------->|           |
 |              |              |              |               |--PDP req->|
 |              |              |              |               |<--allow---|
 |              |              |-delete_user + token--------->|           |
 |              |              |              |               |--PDP req->|
 |              |              |              |               |<--allow---|
 |              |              |-list_users + token---------->|           |
 |              |              |              |               |--PDP req->|
 |              |              |              |               |<--allow---|
 |              |              |<-results-----|               |           |
~~~

The Intent Agent appears only during Phase 2: it receives the prompt and the discovery corpus, produces the three `agent_intent` entries, and plays no further role. The RAR token is presented to the single PEP for each tool call; the PEP evaluates each invocation independently against the token's `authorization_details`.

# Acknowledgments

The author thanks the contributors to the OpenID AuthZEN, the authors of draft-ietf-oauth-first-party-apps, the Mission-Bound Authorization work by Karl McGuinness, and the broader IETF OAuth Working Group.

# Document History

## draft-embesozzi-intent-agent-native-authorization-00

- Initial version.
