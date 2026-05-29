---
date: 2026-05-29
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

## Industry Landscape (May 2026)

### Protocols

| Protocol | Purpose | Registration | Discovery |
|----------|---------|-------------|-----------|
| **A2A** (Google → AAIF) | Agent-to-agent communication | Push: Agent Card JSON at `/.well-known/agent-card.json` | Well-known URI, curated registries |
| **MCP** (Anthropic → AAIF) | Agent-to-tool access | Push: publish to MCP Registry API | Registry API at `registry.modelcontextprotocol.io` |
| **ANS** (IETF draft) | DNS-inspired agent directory | Push: DID + PKI certificates | Two-layer resolve (lightweight + semantic) |

**Industry consensus**: A2A for agent-to-agent, MCP for agent-to-tool. Both governed by AAIF (Linux Foundation, 170+ members). All three cloud providers support both.

### Cloud Provider Registries

| Provider | Product | Registration | Discovery | Governance |
|----------|---------|-------------|-----------|------------|
| **AWS** | Bedrock AgentCore Agent Registry | Push + approval workflow | Hybrid search + MCP endpoint | IAM, CloudTrail, EventBridge |
| **Google** | Gemini Enterprise Agent Platform | Platform-indexed | Governed single pane | Crypto Agent Identity, Agent Gateway, Model Armor |
| **Microsoft** | Copilot Studio + Azure AI Foundry | Published to store | Store browsing + A2A | Azure AD/Entra |

### Key Industry Patterns

1. **Agent Card** (A2A) is the metadata standard — name, description, version, capabilities, auth schemes, skills (with I/O modes and examples)
2. **Push-based self-registration** — agents publish their own metadata
3. **Hybrid search** (semantic + keyword) for discovery
4. **Approval workflows** for enterprise governance
5. **Kubernetes Agent CRD** is in CNCF draft but not production-ready — Ybor's PlatformAgent CRD is ahead of this

## Proposed Options

### Option A: A2A Agent Card on PlatformAgent CRD (Recommended)

Extend the existing `PlatformAgent` CRD to embed A2A Agent Card metadata. The operator reconciles CRDs into a discoverable registry backed by a lightweight in-cluster service.

**How it works**:
1. Agent developer defines a `PlatformAgent` CR with an embedded `agentCard` spec (A2A-compatible)
2. The operator reconciles the CR → generates an Agent Card JSON → serves it at the agent's well-known URI
3. A registry controller watches all PlatformAgent CRDs, indexes their Agent Cards, and exposes a discovery API
4. The Orchestrator queries the registry to find agents by capability/skill match
5. Agents discover each other via the same registry or direct well-known URI

**Agent Card metadata** (A2A-aligned):
```yaml
apiVersion: agents.p6m.dev/v1alpha1
kind: PlatformAgent
metadata:
  name: triage-agent
  namespace: p6m-agents
spec:
  agentCard:
    name: "Triage Agent"
    description: "Investigates Grafana alerts, reads logs/metrics, identifies root cause, creates Jira tickets"
    version: "1.2.0"
    provider:
      organization: "p6m"
      contactEmail: "platform@p6m.dev"
    capabilities:
      streaming: true
      pushNotifications: false
    authentication:
      schemes:
        - type: bearer
          bearerFormat: "k8s-sa-token"
    skills:
      - id: "investigate-alert"
        name: "Investigate Alert"
        description: "Analyze Grafana alert, read app logs, check metrics, git blame"
        inputModes: ["application/json"]
        outputModes: ["application/json"]
        inputSchema:
          type: object
          properties:
            alertName: { type: string }
            grafanaUrl: { type: string }
            namespace: { type: string }
        outputSchema:
          type: object
          properties:
            rootCause: { type: string }
            evidence: { type: array }
            jiraTicketKey: { type: string }
      - id: "create-jira-findings"
        name: "Create Jira Findings"
        description: "Create structured Jira ticket with investigation results"
        inputModes: ["application/json"]
        outputModes: ["application/json"]
  # Existing PlatformAgent fields
  channels:
    - name: triage
      trigger:
        type: webhook
        source: grafana
      mcpServers:
        - grafana-alerts
        - jira
  runtime:
    image: "p6m.jfrog.io/platform-agents/triage-agent:1.2.0"
    serviceAccount: triage-agent-sa
    resources:
      requests: { cpu: "500m", memory: "1Gi" }
```

**Discovery API** (served by a registry controller pod):
```
GET /v1/agents                          → list all registered agents
GET /v1/agents?skill=investigate-alert  → find agents with matching skill
GET /v1/agents?tag=observability        → filter by tag
GET /v1/agents/{name}                   → get specific agent + full card
GET /v1/agents/{name}/.well-known/agent-card.json  → A2A-compatible endpoint
```

**Pros**:
- Builds on existing PlatformAgent CRD — incremental, not greenfield
- A2A-aligned — interoperable with AWS/Google/Microsoft agent registries
- Kubernetes-native — etcd as source of truth, RBAC for access control
- Agent Card schema is standardized and adopted by all major vendors
- Discovery is both K8s-native (kubectl) and API-accessible (HTTP)

**Cons**:
- CRD storage limits rich queries (no full-text search in etcd)
- No built-in approval workflow (would need admission webhook or custom status field)
- A2A spec is at v0.3.0 — may evolve

**Implementation scope**:
- Extend PlatformAgent CRD schema (agentCard field)
- Build registry controller (Go, watches PlatformAgent CRDs, serves HTTP API)
- Agent Card well-known endpoint per agent (via Ingress or Gateway API)
- Orchestrator integration (query registry before dispatching)

---

### Option B: Standalone Registry Service (Database-Backed)

Build a dedicated registry service (like a mini Hugging Face) with a PostgreSQL backend, REST API, and optional UI.

**How it works**:
1. Agents self-register by calling a REST API with their metadata
2. Registry stores agent records in PostgreSQL with versioning, usage stats, health status
3. Discovery via hybrid search (semantic + keyword), filtering, and browsing
4. Approval workflow: agents submit → admin approves → agent becomes discoverable
5. Health monitoring: registry periodically probes agent endpoints

**Pros**:
- Rich queries (full-text search, semantic search, faceted filtering)
- Usage tracking (invocation counts, success rates, latency percentiles)
- Version history and changelogs
- Approval workflows built-in
- Could serve a UI for human browsing (like Hugging Face Hub)

**Cons**:
- New service to build and operate (DB, API, migrations)
- Duplicates some CRD metadata (two sources of truth unless synced)
- More infrastructure overhead
- Not Kubernetes-native — separate from the operator pattern

---

### Option C: Hybrid (CRD + DB Sync)

CRD as source of truth, synced to a DB-backed service for rich queries.

**How it works**:
1. Agents defined as PlatformAgent CRDs (Option A pattern)
2. A sync controller watches CRDs and mirrors to PostgreSQL
3. Registry API reads from DB (rich queries, search, stats)
4. CRD remains the single source of truth — DB is a read-optimized projection

**Pros**: Best of both worlds — K8s-native + rich queries
**Cons**: Most complex; sync lag; two stores to maintain

---

## Recommendation

**Start with Option A** (A2A Agent Card on PlatformAgent CRD). Reasons:

1. **Incremental** — extends existing PlatformAgent CRD, doesn't require new infrastructure
2. **Standards-aligned** — A2A Agent Card is the industry-adopted schema (AWS, Google, Microsoft all support it)
3. **Sufficient for current scale** — with <20 agents, K8s-native discovery is adequate
4. **Upgrade path** — can evolve to Option C (add DB sync) when rich queries/stats are needed
5. **Ybor's PlatformAgent CRD is already ahead of the CNCF Agent CRD draft** — adding A2A alignment positions it as a reference implementation

Option B or C becomes relevant when: agent count exceeds ~50, external teams need to browse/register agents, or usage analytics are needed for cost governance.

## Next Steps

1. Review the existing PlatformAgent CRD schema in `p6m-dev/platform-agent-operator` to understand what fields already exist
2. Design the agentCard schema extension (A2A-aligned)
3. Define the registry controller architecture
4. Create a Jira epic under YP6M-1549 for the implementation
5. Share this plan with John Thompson (primary PlatformAgent implementer) for feedback

## Open Questions

- Does the existing PlatformAgent CRD already carry enough metadata, or is the Agent Card a net-new addition?
- Should the registry controller live in `platform-agent-operator` or be a separate component?
- What's the timeline — is this for the current sprint or future planning?
- Should agents be discoverable cross-cluster (multi-cluster registry)?
