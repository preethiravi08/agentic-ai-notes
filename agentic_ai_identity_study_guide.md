# Agentic AI: Identity & Security — Study Guide
*Based on Uber Engineering: "Solving the Identity Crisis for AI Agents" (May 2026)*

---

## The Core Problem

Traditional identity systems know two types of actors: **humans** and **workloads** (services/APIs). AI agents don't fit neatly into either bucket — they're workloads that act *on behalf of* humans, often calling other agents along the way.

This creates two specific gaps:

**Problem 1 — Identity Model Gap**
Current identity models (human identity + non-human identity/NHI like service accounts) don't have a concept for "a workload acting as a delegate for a human." An AI agent running as a Kubernetes pod looks identical to any other service.

**Problem 2 — Provenance Drops Across Hops**
When an agent calls another agent, the originating user's identity gets dropped. The final downstream system (e.g. a source control repo receiving a PR) only sees "some service did this" — not *which agent*, acting for *which human*, triggered by *what intent*.

---

## Why This Matters (The Motivation Example)

An on-call engineer asks an **Oncall Agent** to fix a misfiring alert. The Oncall Agent delegates to an **Investigation Agent**, which confirms the alert threshold is wrong, then delegates to a **Monitoring Agent**, which opens a PR to fix it.

The PR is logged as "Monitoring Agent opened this" — with no trail back to the engineer or the reason. This breaks:
- Auditing ("who authorized this change?")
- Compliance ("can we prove the right person approved this?")
- Incident response ("why did this happen?")

---

## Key Properties of Agentic Workflows (vs. Traditional Automation)

| Traditional Automation | Agentic AI |
|---|---|
| Fixed, predetermined steps | Dynamic plans that evolve mid-session |
| Single actor (human or service) | Chains of agents delegating to each other |
| One-hop identity | Multi-hop delegation |
| Static permissions | Context-dependent access |

---

## Uber's Architecture

### Core Components

**Agent Registry** — Source of truth mapping agent IDs to their authorized workloads (Kubernetes pods). Prevents impersonation.

**Security Token Service (STS)** — The only entity allowed to mint JWT tokens for agents. Issues short-lived, single-hop tokens. Acts as a dynamic trust broker.

**AI Agent Mesh** — The data plane where agents talk to each other using JWT-authenticated calls.

**MCP Gateway** — Central policy enforcement point for all tool calls (MCP = Model Context Protocol). Authorizes tool invocations and handles PII redaction via AI Guard.

**AI Gateway** — Mediates all outbound LLM API calls (to OpenAI, Anthropic, etc.). Handles prompt injection detection, jailbreak prevention, content safety.

---

## How Agent Identity Works

Uber extended their existing **Zero Trust Architecture** to cover agents. The foundation is **SPIRE** (SPIFFE-based identity for workloads).

### Token Minting Flow (per agent, at runtime)

1. The workload (Kubernetes pod) fetches a **SVID** (SPIFFE Verifiable Identity Document) from SPIRE — proves the compute environment is legitimate.
2. The agent SDK uses its SVID + inbound JWT context to request a new **JWT from STS**, scoped to the *next hop's audience* only.
3. STS checks the **Agent Registry** to confirm this agent is authorized to run on that workload (prevents impersonation).
4. STS mints and returns a short-lived JWT. This JWT is what the agent uses for the next call.

### Key JWT Design Decisions

- **Single-hop tokens** — Each JWT is valid only for one specific destination (e.g., Oncall Agent → Investigation Agent). Can't be replayed elsewhere.
- **Short TTL** — Minutes, not hours. Limits blast radius if intercepted.
- **Actor chain (`act_chain` claim)** — The JWT carries the full verified lineage of every entity in the call chain: `[user → oncall-agent → investigation-agent]`. Every hop in the chain is cryptographically attested.
- **Extensible** — Session IDs, intent claims, and other context can be added as new JWT claims.

---

## Multi-Hop Token Flow (Traced)

```
Engineer (user1)
    ↓ initiates session
Oncall Agent (Workload-1)
    ↓ requests JWT from STS (audience: Investigation Agent)
    JWT: sub=oncall-agent, aud=workload-2, act_chain=[user1, oncall-agent]
    ↓ calls Investigation Agent
Investigation Agent (Workload-2)
    ↓ verifies JWT, requests new JWT from STS (audience: MCP Gateway)
    JWT: sub=investigation-agent, aud=mcp-gateway, act_chain=[user1, oncall-agent, investigation-agent]
    ↓ calls MCP Gateway with full lineage
MCP Gateway
    ↓ verifies JWT, enforces tool-level policies, proxies to downstream system
```

The token exchange is conceptually based on **OAuth 2.0 Token Exchange (RFC 8693)**, customized for Uber's internal performance and auditing requirements.

---

## Developer Experience: The Paved Path

The risk with security systems: developers bypass them. Uber's fix — make the secure path the *default* path.

They built a **Standardized A2A (Agent-to-Agent) Client** on top of the A2A protocol. Developers use this client for agent-to-agent calls; the JWT exchange and actor chain propagation happen automatically under the hood. Developers don't have to think about it.

Existing agents were migrated in a phased approach to adopt the standard client.

---

## Observability

The system logs every hop in the actor chain, including:
- Which user initiated the session
- Which agents were involved
- Which MCP tools were called
- Authorization decision (ALLOWED / DENIED) at each step

This enables end-to-end audit trails without stitching together partial logs.

**Performance**: P99 latency for STS token exchange is consistently **< 40ms**, making the overhead negligible even at Uber scale.

---

## Strategic Direction (3-Layer IAM Model for Agents)

```
┌─────────────────────────────────────────────┐
│         Unified Enforcement Plane            │
│  Observability · Audit · Governance ·        │
│  Central policy across all tools/protocols  │
├─────────────────────────────────────────────┤
│         Dynamic Access Control               │
│  Adaptive access · Human-in-the-loop ·      │
│  Workflow authorization (full chain context) │
├─────────────────────────────────────────────┤
│         Identity & Trust Foundation          │
│  Agent identity · Context propagation ·     │
│  Trust definition (SPIRE + STS + Registry)  │
└─────────────────────────────────────────────┘
```

The article primarily covers the **bottom layer**. The long-term vision: static human-managed permissions don't scale for agent-driven systems — you need dynamic, context-aware access control that considers the full delegation chain.

---

## Key Standards & References to Know

| Standard / Project | Role |
|---|---|
| **SPIFFE / SPIRE** | Workload identity (cryptographic, infra-level) |
| **JWT (JSON Web Token)** | Token format carrying identity + claims |
| **OAuth 2.0 Token Exchange (RFC 8693)** | Conceptual basis for per-hop token exchange |
| **MCP (Model Context Protocol)** | Standard protocol for AI tool calls |
| **A2A Protocol** | Agent-to-Agent communication protocol |
| **WIMSE (IETF Working Group)** | Emerging standards for workload identity in multi-service environments |
| **IETF draft-klrc-aiagent-auth-01** | Draft spec for AI agent authentication & authorization |

---

## Core Takeaways

1. **Agents need their own identity primitive** — NHI (service accounts) isn't enough; you need an "agent" type that carries delegation context.
2. **Provenance must be carried forward** — Each token exchange must append to (not replace) the actor chain.
3. **Short-lived, single-hop tokens** reduce the attack surface dramatically vs. long-lived credentials.
4. **Secure-by-default > opt-in security** — Build the secure path into the SDK so developers can't easily skip it.
5. **Static permissions won't scale** — Agent IAM needs to be dynamic, evaluating the full delegation chain and session context at enforcement time.
