---
date: 2026-05-29
ticket: ADAIP-351
repo: ybor-cortex
status: approved
---

# ADAIP-351 Phase 2: Pre-Render Source Code Verification

## Context

Phase 1 (completed) added `SemanticValidator` as a pre-commit gate in `commit_github_files`. It detects drift between generated manifests and source code (health endpoint, port, env vars) and blocks the commit with a drift report.

**Problem discovered in live testing**: When SemanticValidator blocks the commit, Claude re-renders with the *same wrong inputs* instead of correcting them. The drift is caught too late — after rendering — so there's no structured way for Claude to fix the values and try again. The live cortex-mcp deployment showed this failure loop: render → commit blocked → re-render (same inputs) → blocked again → Claude gives up.

**Root cause**: Verification runs at commit time (step 5), but variable correction needs to happen at extraction time (step 1-2). By step 5, Claude has forgotten the extraction context.

**Solution**: Add a `verify_deployment_variables` tool that runs BETWEEN extract/detect and user confirmation. It cross-references extracted values against source code and returns a structured drift table with corrected values. The render then gets correct inputs from the start.

## New Pipeline

```
1. extract_deployment_variables  → template defaults (port=8080, health=/health/ready)
2. detect_secrets_usage          → env var classification
3. verify_deployment_variables   → drift table: template vs source code (NEW)
4. Claude corrects values        → uses drift table evidence
5. User confirms corrected values
6. render_deployment_manifests   → renders with corrected values
7. SemanticValidator sign-off    → confirms rendered output matches source (existing)
8. commit_github_files           → commits
```

**Two validators, distinct roles:**
- **verify** (step 3) = pre-render discovery — "here's what's wrong with the defaults, with evidence"
- **semantic validator** (step 7) = post-render sign-off — "the rendered manifests match source code, approved to commit"

## Files to Create/Modify

| File | Change |
|------|--------|
| `cortex-api/cortex_mcp/tools/source_scanning.py` | **NEW** — shared scanning functions (routes, ports, env vars) |
| `cortex-api/cortex_mcp/tools/verify_variables.py` | **NEW** — `VerifyDeploymentVariables` MCP tool |
| `cortex-api/cortex_mcp/tools/commit_validator.py` | Refactor `SemanticValidator` to import from `source_scanning.py` |
| `cortex-api/cortex_core/ai/constitution.py` | Add verify step to GUIDED + ADAPTIVE workflows |
| `cortex-api/cortex_server/main.py` | Register `VerifyDeploymentVariables` |
| `cortex-api/tests/test_verify_variables.py` | **NEW** — tests for verify tool |
| `cortex-api/tests/test_source_scanning.py` | **NEW** — tests for shared scanning functions |

## Implementation

### Step 1: `source_scanning.py` — Shared scanning module

Extract from `commit_validator.py` (lines 1156-1568) into a neutral shared module:

**Constants**: `_ROUTE_PATTERNS`, `_ROUTER_PREFIX_RE`, `_STANDARD_ENV_VARS`, `_MAX_SOURCE_FILES`

**Functions**:
- `extract_routes_with_lines(content: str) -> list[tuple[str, int]]` — regex per-line route scanning
- `extract_routes(content: str) -> set[str]` — routes without line numbers
- `parse_env_refs_with_lines(content: str) -> list[tuple[str, int]]` — AST-based env var extraction
- `parse_dockerfile_port(content: str) -> tuple[int | None, str | None, int | None]` — returns `(port, source_label, lineno)`
- `filter_python_source_files(tree: list[dict]) -> list[str]` — excludes tests/, docs/, .github/

### Step 2: `verify_variables.py` — New MCP tool

**Input schema**:
```python
{
    "owner": str,                    # required
    "repo": str,                     # required
    "ref": str,                      # default: "main"
    "health_endpoint": str,          # required — from extract
    "port": int,                     # required — from extract
    "detected_config_vars": [str],   # default: [] — from detect
    "detected_secrets": [str],       # default: [] — from detect
}
```

**Output (ToolResult.content)** — drift table with evidence:
```
DEPLOYMENT VARIABLE VERIFICATION — 3 drift(s) found

| # | Field           | Template Value | Source Value   | Source File | Line |
|---|-----------------|----------------|----------------|-------------|------|
| 1 | health_endpoint | /health/ready  | /health        | main.py     | 42   |
| 2 | port            | 8080           | 3000           | Dockerfile  | 5    |
| 3 | DATABASE_URL    | (not declared) | os.environ ref | config.py   | 12   |

CORRECTED VALUES (use these for render_deployment_manifests):
  health_endpoint: /health
  port: 3000
  additional_env_vars: DATABASE_URL (classify with detect_secrets_usage if needed)
```

Column definitions:
- **Template Value** — what extract_deployment_variables returned (the default)
- **Source Value** — what the actual application code uses (evidence)
- **Source File / Line** — exact location in the repo (evidence for the correction)

**Output (ToolResult.metadata)** — structured for tool chaining:
```python
{
    "deployment_event": "variables_verified",
    "issues_found": 3,
    "verified_health_endpoint": "/health",
    "verified_port": 3000,
    "additional_env_vars": ["DATABASE_URL"],
    "detected_routes": ["/health", "/api/items"],
    "dockerfile_port": 3000,
}
```

**Three verification checks** (same logic as SemanticValidator, using shared functions):

1. **Health endpoint**: Scan Python files for route decorators, compare against `health_endpoint` input. If mismatch, pick best health-like route as correction.
2. **Port**: Read Dockerfile from repo tree, parse EXPOSE/CMD --port, compare against `port` input.
3. **Env vars**: AST-parse Python files for `os.environ.get()`/`os.getenv()`, compare against `detected_config_vars + detected_secrets`. Report undeclared vars as `additional_env_vars`.

**Graceful degradation**: If no token, tree fetch fails, or empty tree → return "verification skipped" with `issues_found: 0`.

### Step 3: Refactor `commit_validator.py`

Replace local constant/function definitions with imports from `source_scanning.py`:

```python
from cortex_mcp.tools.source_scanning import (
    _MAX_SOURCE_FILES, _ROUTE_PATTERNS, _ROUTER_PREFIX_RE, _STANDARD_ENV_VARS,
    extract_routes_with_lines, filter_python_source_files,
    parse_env_refs_with_lines, parse_dockerfile_port,
)
```

Delegate `SemanticValidator` methods to shared functions:
- `_extract_routes_with_lines()` → `extract_routes_with_lines()`
- `_parse_env_refs_with_lines()` → `parse_env_refs_with_lines()`
- Port parsing in `_check_port_consistency()` → `parse_dockerfile_port()`
- Python file filtering → `filter_python_source_files()`

SemanticValidator stays as the post-render sign-off gate. Its role changes from "catch drift and block" to "confirm rendered output matches source and approve commit." If verify + Claude corrections worked, the semantic validator passes clean. If something slipped through (Claude skipped verify, user overrode to wrong values), it still blocks.

### Step 4: Constitution update (`constitution.py`)

Add to GUIDED mode (after detect_secrets_usage paragraph, before "Do NOT stop"):

```
**Source code verification (additional step):**
After extract and detect, call `verify_deployment_variables` with the same
owner/repo, the extracted health_endpoint and port, and the detected
config_vars and secrets. It returns a verification table and corrected values.
Use the corrected values from its output when presenting the confirmation
table in step 2 and when calling render_deployment_manifests in step 3a.
Show the verification table to the user so they see what was auto-corrected.
```

Add equivalent to ADAPTIVE mode. Add one-liner to SHORT constitutions.

### Step 5: Register tool (`main.py`)

```python
from cortex_mcp.tools.verify_variables import VerifyDeploymentVariables
registry.register(VerifyDeploymentVariables())  # after DetectSecretsUsage
```

No platform token required — uses `_resolve_token` from context (same as extract/detect).

### Step 6: Tests

**`test_verify_variables.py`** (~15 tests):
- Health: exact match (no issue), drift detection, no Python files, health route selection
- Port: match, mismatch, CMD preferred over EXPOSE, no Dockerfile
- Env vars: all detected (no issue), additional vars found, standard vars filtered
- Graceful degradation: no token, tree fetch fails, empty tree
- Output format: table structure, metadata correctness

**`test_source_scanning.py`** (~8 tests):
- Route extraction: FastAPI, Flask, Starlette patterns + line numbers
- Env var extraction: os.environ.get, os.getenv, os.environ["X"] + line numbers
- Dockerfile parsing: EXPOSE, CMD --port, both present
- Python file filtering: excludes tests/, docs/

## Key Design Decisions

- **Two validators, distinct roles** — `verify` discovers drift pre-render with evidence; `SemanticValidator` confirms post-render output and gives sign-off before commit
- **Separate `source_scanning.py`** — neutral shared module; both validators use the same scanning logic (routes, ports, env vars) but at different pipeline stages
- **Drift table with evidence** — every row has Template Value, Source Value, Source File, Line — Claude and the user can see exactly what needs correction and why
- **Health correction heuristic** — prefers routes containing "health" in the name; user can override in confirmation
- **Env var check is additive** — reports vars found in source but not in detect output; doesn't duplicate secret classification
- **Rate limit guard** — `_MAX_SOURCE_FILES = 15` cap on per-file reads, single `get_tree_recursive()` call shared across checks

## Verification

```bash
cd cortex-api
uv run pytest tests/test_source_scanning.py -v        # Shared functions
uv run pytest tests/test_verify_variables.py -v        # New tool
uv run pytest tests/test_commit_validator.py -v        # Existing tests still pass
uv run pytest tests/ -q                                # Full suite
```

Manual test: Deploy cortex-mcp via Cortex and verify:
1. verify_deployment_variables reports health=/health (not /health/ready)
2. Claude presents corrected values in confirmation table
3. Render uses /health → commit passes without SemanticValidator drift
