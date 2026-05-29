---
date: 2026-05-28
ticket: YP6M-1549
repo: ybor-cortex
status: approved
---

# Plan: Agent Registry for the Ybor Platform

## Context

The PlatformAgent architecture (YP6M-1549) enables event-driven autonomous agents — a Jira webhook triggers Argo Events, the Orchestrator dispatches an Agent Job (Claude Code), which interacts with MCP servers via a Code-Mode Gateway. The triage→PR agent chain (YP6M-1871/1872) already works as a demo.

**Problem**: There is no registry where agents can register themselves and be discovered by the Orchestrator or by other agents. Today, agent routing is hardcoded. As the number of agents grows (triage, PR fix, deploy, test, report), the Orchestrator needs a dynamic way to discover what agents exist, what they can do, and how to invoke them — analogous to how a model registry (Hugging Face) catalogues models with metadata, benchmarks, and versioning.

**Existing Ybor assets**:
- `PlatformAgent` CRD (implemented, YP6M-1529) — declares agent identity, permissions, channels
- `PlatformSkill`, `PlatformMCP`, `PlatformStream` companion CRDs
- Two-SA security model (orchestrator SA vs agent SA)
- Cross-namespace Argo Events (EventSource/Sensor in `p6m-events`)
- Repos: `p6m-dev/platform-agent-operator`, `p6m-dev/platform-nanoclaw`, `p6m-dev/platform-agents`

## Recommendation

**Start with Option A** (A2A Agent Card on PlatformAgent CRD). Reasons:

1. **Incremental** — extends existing PlatformAgent CRD, doesn't require new infrastructure
2. **Standards-aligned** — A2A Agent Card is the industry-adopted schema (AWS, Google, Microsoft all support it)
3. **Sufficient for current scale** — with <20 agents, K8s-native discovery is adequate
4. **Upgrade path** — can evolve to Option C (add DB sync) when rich queries/stats are needed
5. **Ybor's PlatformAgent CRD is already ahead of the CNCF Agent CRD draft** — adding A2A alignment positions it as a reference implementation
