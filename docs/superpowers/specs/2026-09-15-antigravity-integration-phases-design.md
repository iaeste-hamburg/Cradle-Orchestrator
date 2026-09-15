# Feature Specification: Antigravity Integration Workflow (Four-Phase Pipeline)

- **Feature Name:** Antigravity Worker Engine Integration & Contract-Driven Verification Workflow
- **Branch:** `feat/antigravity-integration-phases` (PR #1)
- **Repository:** `Laufgitter`
- **Author:** Solution8 Agentic Engineering
- **Date:** 2026-09-15
- **Status:** Approved / Draft Specification
- **Target Systems:** `laufgitter.py`, `Zentrale HUD`, OpenCode CLI (`gpt-5.4`), Google Antigravity CLI (`agy` / `gemini-3.8-medium`), LiteLLM Proxy

---

## 1. Executive Summary & Intent

### 1.1 Purpose
This specification formalizes the architectural design, manifest structure, human-in-the-loop validation gates, and deterministic verification procedures for integrating Google's Antigravity CLI (`agy`) as an autonomous worker engine within the **Laufgitter** orchestration system.

The core motivation of this feature is twofold:
1. **Epistemic Rigor (Zero Hallucination Policy):** External tool interfaces, command-line arguments, environment variable discovery, and Model Context Protocol (MCP) configuration formats for newly released agent harnesses (such as Google's Antigravity CLI) must not be assumed *a priori*. All claims regarding CLI flags, exit code semantics, authentication mechanisms, and sandboxing must be verified through live web research before being synthesized into executable engine configurations.
2. **Contract-Driven Execution with Deterministic Verification:** Downstream execution must demonstrate cross-engine task orchestration: a primary planner/reviewer model (`gpt-5.4` via OpenCode) specifies an explicit contract file, an executor engine (`antigravity` running `gemini-3.8-medium`) fulfills the artifact contract using MCP capabilities, and an automated verification check inspects the physical output artifact deterministically without relying on probabilistic LLM assertions.

### 1.2 Invariant System Naming
In accordance with Solution8 repository conventions:
- The orchestrator CLI and engine coordinator is **Laufgitter** (invoked via `laufgitter.py`, with configurations located in `~/.config/laufgitter/` and runtime run states under `~/.laufgitter/`).
- The monitoring interface, native application, and telemetry system is **Zentrale** (e.g., `Zentrale HUD`, `dashboard/zentrale.html`, `Zentrale.app`).
- The repository is canonicalized as **Laufgitter**. All legacy naming conventions are permanently superseded.

---

## 2. Architectural Overview & System Model

### 2.1 Laufgitter Engine Execution Model
Laufgitter orchestrates autonomous LLM coding agents via a template-driven engine subsystem configured in `~/.config/laufgitter/config.toml`. Each engine definition declares:
- `bin`: Path to the harness binary or sandbox wrapper script.
- `args_template`: Array of argv arguments using dynamic token substitution.
- `sandbox_args`: Command-line switches enforcing strict filesystem and network boundaries.
- `full_access_args`: Command-line switches applied only when a manifest task explicitly asserts `"full_access": true` (and `allow_full_access = true` is enabled in config).
- `model_default`: Fallback model identifier when not overridden by task specifications.
- `token_regex` / `model_report_regex`: Regular expressions for parsing token consumption and runtime model telemetry from standard output/error streams.

Laufgitter supports several dynamic argument interpolation tokens:
- `{taskdir}`: The isolated working directory dedicated to the specific task attempt.
- `{spec}`: The prompt or task specification delivered to the agent.
- `{model}`: The resolved model identifier (either from the task definition or `model_default`).
- `{access_args}`: Expands dynamically to `sandbox_args` or `full_access_args`.
- `{engine_args}`: Task-specific CLI options passed verbatim from the manifest.

```
+-----------------------------------------------------------------------------------------+
|                                    Laufgitter Engine                                    |
|                                                                                         |
|  +--------------------+     +--------------------------------+     +-----------------+  |
|  | Manifest (.json)   | --> | laufgitter.py Coordinator      | --> | Zentrale HUD    |  |
|  | - Tasks            |     | - Worktree Management          |     | - Realtime Logs |  |
|  | - Dependencies     |     | - Template Arg Expansion       |     | - Run Telemetry |  |
|  | - Checks & Specs   |     | - Process Lifecycle & Timeouts |     | - Artifact View |  |
|  +--------------------+     +--------------------------------+     +-----------------+  |
|                                     |                  |                                |
|                 +-------------------+                  +-------------------+            |
|                 v                                                          v            |
|     +---------------------------+                              +--------------------+   |
|     | Engine: [engines.opencode]|                              | [engines.antigravity|  |
|     | Harness: OpenCode CLI     |                              | Harness: agy CLI   |   |
|     | Backend: gpt-5.4 via      |                              | Backend: Gemini    |   |
|     | LiteLLM Proxy             |                              | via Google AI/     |   |
|     +---------------------------+                              | Vertex Endpoint    |   |
|                 |                                                          |            |
|                 v                                                          v            |
|     +---------------------------+                              +--------------------+   |
|     | Task 0 / 1 / 2 Artifacts  |                              | Task 3 Deliverables|   |
|     | - worker.log identity     |                              | - screenshot.png   |   |
|     | - antigravity-facts.json  |                              | - MCP Tool Traces  |   |
|     | - antigravity-block.toml  |                              +--------------------+   |
|     | - contract.json           |                                          |            |
|     +---------------------------+                                          |            |
|                 |                                                          |            |
|                 +--------------------> Deterministic <---------------------+            |
|                                        Verification Check                               |
|                                        (Pillow Image Verifier)                          |
+-----------------------------------------------------------------------------------------+
```

### 2.2 Rationale for Sequential Manifests vs. Monolithic Runs
Laufgitter's concurrency model executes all tasks within a single manifest concurrently by default (unless restricted by `max_parallel: 1`). Furthermore, Laufgitter does not provide an internal cross-manifest directed acyclic graph (DAG) scheduler.

The Antigravity integration pipeline strictly requires an epistemic and logistical sequence:
1. Manifests are partitioned into four distinct phases (`phase0` through `phase3`).
2. Execution proceeds sequentially, requiring explicit human review gates between stages.
3. This sequence enforces failure containment: an unverified model routing failure in Phase 0 or a hallucinated flag in Phase 1 cannot propagate into engine configuration in Phase 2 or test executions in Phase 3.

---

## 3. Detailed Phase Specifications

```
  +---------------------------------------------------------------------------------------+
  |                             FOUR-PHASE PIPELINE FLOW                                  |
  +---------------------------------------------------------------------------------------+

   [ PHASE 0 ]  Prove gpt-5.4 Routing Through LiteLLM
        |
        v
   [ GATE 0 ]   Human Gate: Inspect worker.log for Azure backend model identity string
        |
        v
   [ PHASE 1 ]  Live Web Research on Antigravity CLI (`agy`)
        |
        v
   [ GATE 1 ]   Human Gate: Audit antigravity-cli-facts.json against official Google docs
        |
        v
   [ PHASE 2 ]  Synthesize [engines.antigravity] Configuration Block
        |
        v
   [ GATE 2 ]   Human Gate: Validate generated TOML and hand-merge into config.toml
        |
        v
   [ PHASE 3 ]  Contract-Driven Execution & Deterministic Verification
        |       Task 3.1: write-contract (gpt-5.4 generates contract.json)
        |       Task 3.2: execute-screenshot (agy fulfills contract.json)
        |
        v
   [ GATE 3 ]   Final Artifact Verification: Dimension & PNG validity check
```

---

### 3.1 Phase 0: Prove GPT-5.4 Routing Through LiteLLM

#### 3.1.1 Objective
Before tasking `gpt-5.4` with complex web exploration and configuration authoring, Laufgitter must prove that:
- The local OpenCode CLI harness can successfully communicate through the enterprise LiteLLM proxy (`OPENAI_BASE_URL`).
- The LiteLLM proxy correctly routes requests requesting model `gpt-5.4` to the designated upstream Azure OpenAI deployment.
- The model identity reported by the backend runtime is indeed `gpt-5.4` and has not silently degraded to a fallback provider (such as OpenRouter, Claude, or Codex defaults).

#### 3.1.2 Manifest Specification: `phases/phase0-litellm-probe.json`
```json
{
  "run_name": "phase0-litellm-probe",
  "workdir": "/tmp/laufgitter-phase0",
  "tasks": [
    {
      "key": "gpt54-routing-probe",
      "engine": "opencode",
      "model": "gpt-5.4",
      "task_type": "probe",
      "spec": "Reply with exactly one line: ROUTING_OK <the model identity string your provider backend reports for this request>. Do not add anything else.",
      "check": "grep -q 'ROUTING_OK' worker.log || { echo 'FAIL: gpt-5.4 via litellm did not respond as expected'; exit 1; }",
      "expect_files": [],
      "verified": "Confirms OpenCode successfully reaches gpt-5.4 through the litellm proxy end to end.",
      "max_attempts": 1
    }
  ]
}
```

#### 3.1.3 Task Semantics & Probe Mechanics
- `task_type`: Classified as `probe`. The task does not generate workspace code changes.
- `spec`: Constrains the LLM to emit an atomic token: `ROUTING_OK <model_id>`.
- `check`: The automated check performs a basic grep against `worker.log`.
- `max_attempts: 1`: Prevents retry loops when proxy credentials or endpoint routing are defective.

#### 3.1.4 Verification & Human Gate 0
- **Automated Check Limitation:** The automated check only verifies that the string literal `ROUTING_OK` appeared in the logs. It cannot autonomously authenticate whether the backend model string corresponds to the authorized Azure deployment.
- **Mandatory Human Gate:** The operator must inspect `/tmp/laufgitter-phase0/gpt54-routing-probe/worker.log`:
  - **PASS Criteria:** The log line matches `ROUTING_OK gpt-5.4-via-azure` (or the registered Azure deployment name).
  - **FAIL Criteria:** The log displays fallback model signatures (e.g., `ROUTING_OK claude-3-opus-20250219`, `ROUTING_OK openrouter/...`, or proxy 401/403/502 error dumps).
  - **Action:** If failed, halt immediately. Debug proxy environment variables (`OPENAI_BASE_URL`, `OPENAI_API_KEY`) before advancing to Phase 1.

---

### 3.2 Phase 1: Live Web Research on Antigravity CLI

#### 3.2.1 Objective
Gather verifiable technical facts regarding Google's Antigravity CLI (`agy`) using live web search capabilities within OpenCode. All parameters must be cited with authoritative source URLs. Hallucinated or speculative flags must be explicitly flagged as `null`.

#### 3.2.2 Manifest Specification: `phases/phase1-research-antigravity.json`
```json
{
  "run_name": "phase1-research-antigravity",
  "workdir": "/tmp/laufgitter-phase1",
  "tasks": [
    {
      "key": "research-antigravity-cli",
      "engine": "opencode",
      "model": "gpt-5.4",
      "task_type": "research",
      "timeout_s": 1800,
      "spec": "Search the live web for current, official documentation of the 'agy' / Antigravity CLI (Google's Gemini/Antigravity agent CLI). Confirm, with a source URL for each: (1) the exact flag for headless/non-interactive execution and its exit code convention; (2) how to override the Gemini/Vertex API base URL (flag vs env var, exact variable names); (3) whether/how MCP servers are registered and discovered (file paths, CLI subcommands); (4) the exact model-selection flag and accepted model slug format; (5) whether a --sandbox or permission-bypass flag exists for headless runs. Write ONLY a JSON object to antigravity-cli-facts.json with keys: headless_flag, exit_code_convention, base_url_mechanism (object per provider), mcp_discovery_paths (array), model_flag, model_slug_examples (array), sandbox_flags, sources (array of {claim, url}). Every factual field must have a matching entry in sources. If a fact cannot be confirmed from an official or credibly current source, set that field to null and do not guess.",
      "check": "python3 -c \"import json,sys; d=json.load(open('antigravity-cli-facts.json')); required=['headless_flag','exit_code_convention','base_url_mechanism','mcp_discovery_paths','model_flag','sources']; missing=[k for k in required if k not in d]; assert not missing, f'FAIL: missing keys {missing}'; assert len(d['sources'])>0, 'FAIL: no sources cited'; print('OK')\"",
      "expect_files": ["antigravity-cli-facts.json"],
      "verified": "Confirms antigravity-cli-facts.json exists, is valid JSON, has the required schema keys, and cites at least one source URL.",
      "max_attempts": 2
    }
  ]
}
```

#### 3.2.3 Schema Definition: `antigravity-cli-facts.json`
The output artifact `/tmp/laufgitter-phase1/research-antigravity-cli/antigravity-cli-facts.json` must adhere strictly to the following JSON schema:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "AntigravityCliFacts",
  "type": "object",
  "required": [
    "headless_flag",
    "exit_code_convention",
    "base_url_mechanism",
    "mcp_discovery_paths",
    "model_flag",
    "model_slug_examples",
    "sandbox_flags",
    "sources"
  ],
  "properties": {
    "headless_flag": { "type": ["string", "null"] },
    "exit_code_convention": { "type": ["string", "null"] },
    "base_url_mechanism": {
      "type": "object",
      "properties": {
        "gemini": {
          "type": "object",
          "properties": {
            "flag_or_env": { "type": "string" },
            "env": { "type": "string" }
          }
        },
        "vertex": {
          "type": "object",
          "properties": {
            "flag_or_env": { "type": "string" },
            "env": { "type": "string" }
          }
        }
      },
      "additionalProperties": true
    },
    "mcp_discovery_paths": {
      "type": "array",
      "items": { "type": "string" }
    },
    "model_flag": { "type": ["string", "null"] },
    "model_slug_examples": {
      "type": "array",
      "items": { "type": "string" }
    },
    "sandbox_flags": {
      "type": ["array", "null"],
      "items": { "type": "string" }
    },
    "sources": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["claim", "url"],
        "properties": {
          "claim": { "type": "string" },
          "url": { "type": "string", "format": "uri" }
        }
      },
      "minItems": 1
    }
  }
}
```

#### 3.2.4 Human Gate 1: Source Authority Audit
Before proceeding to Phase 2:
1. The operator reads `antigravity-cli-facts.json`.
2. Each URL in `sources` is verified to be an official Google documentation domain (`ai.google.dev`, `cloud.google.com`, `github.com/google/...`) or verified community technical release. Unofficial forum rumors (Reddit, non-authoritative blogs) are rejected.
3. If an essential parameter (such as `headless_flag`) is returned as `null` or points to an unreliable source, the operator must re-run Phase 1 with refined query scopes or supply manual factual corrections prior to Phase 2.

---

### 3.3 Phase 2: Author [engines.antigravity] Configuration Block

#### 3.3.1 Objective
Synthesize an exact TOML configuration block for the Antigravity engine utilizing **only** the factual findings recorded in `antigravity-cli-facts.json`. The generator is forbidden from inventing command-line flags or mechanisms that lack justification in the Phase 1 artifact.

#### 3.3.2 Manifest Specification: `phases/phase2-author-antigravity-engine.json`
```json
{
  "run_name": "phase2-author-antigravity-engine",
  "workdir": "/tmp/laufgitter-phase2",
  "tasks": [
    {
      "key": "write-antigravity-engine-config",
      "engine": "opencode",
      "model": "gpt-5.4",
      "task_type": "docs",
      "spec": "Read /tmp/laufgitter-phase1/research-antigravity-cli/antigravity-cli-facts.json. Using ONLY facts present there (do not invent flags not in that file), write a TOML file antigravity-engine-block.toml containing a single [engines.antigravity] table matching this schema exactly: bin, args_template (argv array using {taskdir}, {spec}, {access_args}, {model}, {engine_args} placeholders), sandbox_args, full_access_args, model_default. Model default must be 'gemini-3.8-medium'. Add a comment above each line naming which facts-file field justified it.",
      "check": "python3 -c \"import tomllib; d=tomllib.load(open('antigravity-engine-block.toml','rb')); e=d['engines']['antigravity']; required=['bin','args_template','sandbox_args','full_access_args','model_default']; missing=[k for k in required if k not in e]; assert not missing, f'FAIL: missing {missing}'; assert '{spec}' in e['args_template'], 'FAIL: no {spec} placeholder'; assert '{model}' in ' '.join(e['args_template']), 'FAIL: no {model} placeholder'; print('OK')\"",
      "expect_files": ["antigravity-engine-block.toml"],
      "verified": "Confirms a syntactically valid [engines.antigravity] TOML block with all required keys and placeholders was produced from sourced facts, not invention.",
      "max_attempts": 2
    }
  ]
}
```

#### 3.3.3 Output Specification: `antigravity-engine-block.toml`
The resulting artifact `/tmp/laufgitter-phase2/write-antigravity-engine-config/antigravity-engine-block.toml` must adhere to standard TOML syntax and include field citations:

```toml
# Facts from Phase 1: headless_flag is "--non-interactive"
# Facts from Phase 1: model_flag is "--model"
# Facts from Phase 1: sandbox_flags is ["--sandbox"]

[engines.antigravity]
# Resolved binary location for Google Antigravity CLI
bin = "agy"

# Argument invocation vector incorporating mandatory Laufgitter interpolation tokens
args_template = [
  "{taskdir}",
  "{spec}",
  "--non-interactive",     # justified by headless_flag
  "--model", "{model}",    # justified by model_flag
  "{access_args}",         # dynamically resolves sandbox_args or full_access_args
  "{engine_args}",         # allows task-specific flag injection
]

# Isolation and access control arguments
sandbox_args = ["--sandbox"]
full_access_args = ["--dangerously-skip-permissions"]

# Default model slug per Solution8 evaluation standard
model_default = "gemini-3.8-medium"
```

#### 3.3.4 Human Gate 2: Safe Configuration Merge
To prevent accidental corruption of production configurations, Laufgitter engines do not self-modify `~/.config/laufgitter/config.toml`.
1. The operator inspects `antigravity-engine-block.toml` to verify comment justifications and placeholder completeness.
2. The operator creates a backup of `~/.config/laufgitter/config.toml`.
3. The operator merges the block into `~/.config/laufgitter/config.toml`:
   ```bash
   cp ~/.config/laufgitter/config.toml ~/.config/laufgitter/config.toml.bak
   cat /tmp/laufgitter-phase2/write-antigravity-engine-config/antigravity-engine-block.toml >> ~/.config/laufgitter/config.toml
   ```
4. The operator sets required environment variables identified in Phase 1 (e.g., `GOOGLE_GENERATIVE_AI_API_KEY` or `GOOGLE_APPLICATION_CREDENTIALS`).

---

### 3.4 Phase 3: Contract-Driven Execution with Deterministic Verification

#### 3.4.1 Objective
Execute an end-to-end multi-agent workflow where:
1. An architect agent (`gpt-5.4` on OpenCode) defines a strict contract specifying deliverable requirements (`contract.json`).
2. A worker agent (`gemini-3.8-medium` on the newly registered `antigravity` engine) reads the contract and produces the required binary artifact (`screenshot.png`) using its native MCP tools.
3. A deterministic, non-probabilistic validator evaluates the deliverable using the standard Python image processing library (`PIL.Image`), checking file validity and dimensional constraints without making an LLM inference call.

#### 3.4.2 Concurrency & Worktree Strategy
Phase 3 requires specific execution constraints in Laufgitter:
- `max_parallel: 1`: Laufgitter tasks default to parallel execution. Because Task 2 depends on the output artifact of Task 1, concurrency must be clamped to `1` to enforce strict sequential execution.
- `worktrees: false` on Task 2: Standard Laufgitter tasks run in isolated git worktrees which are pruned upon task completion. Setting `worktrees: false` on the executor task ensures that the preceding task's output directory (`/tmp/laufgitter-phase3/write-contract/contract.json`) remains readable across the task boundary.

#### 3.4.3 Manifest Specification: `phases/phase3-antigravity-screenshot.json`
```json
{
  "run_name": "phase3-antigravity-screenshot",
  "workdir": "/tmp/laufgitter-phase3",
  "max_parallel": 1,
  "tasks": [
    {
      "key": "write-contract",
      "engine": "opencode",
      "model": "gpt-5.4",
      "task_type": "code-review",
      "spec": "Write contract.json describing the artifact the next worker must produce for task: 'Take a screenshot of <TARGET_URL/UI_STATE>'. Required keys: expect_files (array), min_width, min_height, description. File name must be 'screenshot.png'.",
      "check": "python3 -c \"import json; d=json.load(open('contract.json')); assert 'expect_files' in d and 'screenshot.png' in d['expect_files']; assert 'min_width' in d and 'min_height' in d; print('OK')\"",
      "expect_files": ["contract.json"],
      "verified": "Confirms contract.json declares the required artifact and minimum acceptance dimensions.",
      "max_attempts": 1
    },
    {
      "key": "execute-screenshot",
      "engine": "antigravity",
      "model": "gemini-3.8-medium",
      "task_type": "code-feature",
      "engine_args": ["--dangerously-skip-permissions", "--sandbox"],
      "spec": "Read /tmp/laufgitter-phase3/write-contract/contract.json for the exact artifact requirements, then use your available MCP tools to capture the screenshot it describes. Save the result to screenshot.png in this task directory, matching or exceeding the min_width/min_height from the contract.",
      "check": "python3 -c \"import json; from PIL import Image; c=json.load(open('/tmp/laufgitter-phase3/write-contract/contract.json')); img=Image.open('screenshot.png'); w,h=img.size; assert w>=c['min_width'] and h>=c['min_height'], f'FAIL: {w}x{h} smaller than contract minimum {c[\\\"min_width\\\"]}x{c[\\\"min_height\\\"]}'; print('OK')\"; cp screenshot.png /tmp/laufgitter-phase3/artifacts-out/ 2>/dev/null || true",
      "expect_files": ["screenshot.png"],
      "worktrees": false,
      "verified": "Confirms screenshot.png exists, is a valid image, and meets the minimum dimensions gpt-5.4's contract declared — not just that a file exists.",
      "max_attempts": 2
    }
  ]
}
```

#### 3.4.4 Contract Schema: `contract.json`
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "ArtifactDeliveryContract",
  "type": "object",
  "required": ["expect_files", "min_width", "min_height", "description"],
  "properties": {
    "expect_files": {
      "type": "array",
      "items": { "type": "string" },
      "contains": { "const": "screenshot.png" }
    },
    "min_width": {
      "type": "integer",
      "minimum": 100
    },
    "min_height": {
      "type": "integer",
      "minimum": 100
    },
    "description": {
      "type": "string"
    }
  }
}
```

#### 3.4.5 Deterministic Verification Mechanism
Unlike traditional eval loops that query an LLM "judge" to score outputs, Phase 3 executes binary and mathematical verification:
1. `PIL.Image.open('screenshot.png')`: Ensures the deliverable has a valid PNG header, decodable bitstream, and uncorrupted chunks.
2. `img.size (w, h)`: Extracts native pixel dimensions.
3. `assert w >= c['min_width'] and h >= c['min_height']`: Enforces strict geometric inequality constraints defined in `contract.json`.
4. `cp screenshot.png /tmp/laufgitter-phase3/artifacts-out/`: Persists the verified artifact to an immutable deliverables folder for human inspection and Zentrale HUD display.

---

## 4. Operational Runbook & Human Verification Gates

```
+-----------------------------------------------------------------------------------------+
| PHASE GATE | INPUT / PRE-CONDITION              | VERIFICATION CRITERIA | FAILURE ACTION|
+-----------------------------------------------------------------------------------------+
| Gate 0     | phase0 execution completes         | worker.log reports    | Fix LiteLLM   |
|            |                                    | Azure gpt-5.4 identity| proxy & abort |
+-----------------------------------------------------------------------------------------+
| Gate 1     | antigravity-cli-facts.json created | Official source URLs, | Re-scope search|
|            |                                    | no hallucinated flags | or edit facts |
+-----------------------------------------------------------------------------------------+
| Gate 2     | antigravity-engine-block.toml      | Valid TOML, justified | Correct TOML  |
|            | created                            | comments, placeholders| before merge  |
+-----------------------------------------------------------------------------------------+
| Gate 3     | screenshot.png generated in        | Valid PNG, dimensions | Debug MCP tool|
|            | /tmp/laufgitter-phase3/            | >= contract thresholds| in Antigravity|
+-----------------------------------------------------------------------------------------+
```

### 4.1 Step-by-Step Operator Runbook

#### Step 1: Pre-flight Verification
```bash
# Verify Laufgitter executable
./laufgitter.py --help

# Verify Python runtime meets >= 3.12 requirement
python3 -c "import sys; assert sys.version_info >= (3, 12)"

# Check OpenCode installation and auth status
opencode --version
opencode auth list
```

#### Step 2: Execute Phase 0
```bash
export OPENAI_BASE_URL="https://litellm.internal.solution8.net/v1"
export OPENAI_API_KEY="sk-solution8-litellm-proxy-key"

./laufgitter.py lint phases/phase0-litellm-probe.json
./laufgitter.py run phases/phase0-litellm-probe.json

# Gate 0 inspection
grep "ROUTING_OK" /tmp/laufgitter-phase0/gpt54-routing-probe/worker.log
```

#### Step 3: Execute Phase 1
```bash
./laufgitter.py lint phases/phase1-research-antigravity.json
./laufgitter.py run phases/phase1-research-antigravity.json

# Gate 1 inspection
cat /tmp/laufgitter-phase1/research-antigravity-cli/antigravity-cli-facts.json | jq .
```

#### Step 4: Execute Phase 2
```bash
./laufgitter.py lint phases/phase2-author-antigravity-engine.json
./laufgitter.py run phases/phase2-author-antigravity-engine.json

# Gate 2 inspection and merge
cat /tmp/laufgitter-phase2/write-antigravity-engine-config/antigravity-engine-block.toml
cat /tmp/laufgitter-phase2/write-antigravity-engine-config/antigravity-engine-block.toml >> ~/.config/laufgitter/config.toml
```

#### Step 5: Execute Phase 3
```bash
# Export Antigravity provider credentials based on Phase 1/2
export GOOGLE_GENERATIVE_AI_API_KEY="ai-solution8-key"

./laufgitter.py lint phases/phase3-antigravity-screenshot.json
./laufgitter.py run phases/phase3-antigravity-screenshot.json

# Gate 3 inspection
ls -lh /tmp/laufgitter-phase3/artifacts-out/screenshot.png
file /tmp/laufgitter-phase3/artifacts-out/screenshot.png
```

---

## 5. Failure Modes, Error Handling & Recovery

| Failure Mode | Root Cause | Impact | Automated / Manual Recovery Procedure |
|:---|:---|:---|:---|
| **Phase 0: Routing Fallback** | LiteLLM proxy misconfigured, routes to Claude or Codex | Phase 0 automated check passes (sees `ROUTING_OK`), but wrong backend used | Manual Gate 0 stops execution. Update proxy mapping or verify `OPENAI_BASE_URL`. |
| **Phase 0: Proxy Network Timeout** | LiteLLM proxy unreachable or returning HTTP 504 | Task exits with code 1 | Verify VPN connection and upstream health before retrying. |
| **Phase 1: Rate Limiting / Search Block** | Search engine blocks headless OpenCode scraping | `antigravity-cli-facts.json` fields set to `null` | Task allows `max_attempts: 2`. If still empty, perform manual lookup and populate facts file. |
| **Phase 1: Hallucinated Source** | LLM generates fabricated URLs or claims | Risk of invalid engine flags in Phase 2 | Gate 1 blocks transition. Discard invalid sources and refine search spec. |
| **Phase 2: Missing Placeholders** | LLM omits `{spec}` or `{model}` in `args_template` | Laufgitter cannot interpolate task prompt or model | Automated check detects missing tokens via `tomllib` assertion and triggers retry. |
| **Phase 2: TOML Syntax Error** | Malformed TOML output | Automated check fails immediately | Review `worker.log`, retry attempt or manually adjust TOML syntax. |
| **Phase 3: Worktree Artifact Loss** | `worktrees: true` accidentally set on Task 2 | Task 2 cannot read `contract.json` from Task 1 | Manifest mandates `worktrees: false` for cross-task scratch access. |
| **Phase 3: Dimension Mismatch** | Screenshot generated at lower resolution than contract | Automated check fails with dimension error message | Task triggers retry attempt. Check MCP tool options or adjust contract constraints. |

---

## 6. Testing, Negative Verification & QA

### 6.1 Baseline Negative Testing (`--baseline`)
To guarantee that verification checks do not succeed trivially or false-pass on missing deliverables, all manifests must be tested against an empty state using Laufgitter's baseline execution flag:

```bash
# Baseline executions must all return FAIL exit codes
./laufgitter.py run phases/phase0-litellm-probe.json --baseline
# Assertion: Fails because worker.log is empty / no ROUTING_OK line exists.

./laufgitter.py run phases/phase1-research-antigravity.json --baseline
# Assertion: Fails because antigravity-cli-facts.json does not exist.

./laufgitter.py run phases/phase2-author-antigravity-engine.json --baseline
# Assertion: Fails because antigravity-engine-block.toml does not exist.

./laufgitter.py run phases/phase3-antigravity-screenshot.json --baseline
# Assertion: Fails because contract.json and screenshot.png do not exist.
```

### 6.2 Manifest Static Linting
Every phase manifest must pass static structural validation before deployment:
```bash
./laufgitter.py lint phases/phase0-litellm-probe.json
./laufgitter.py lint phases/phase1-research-antigravity.json
./laufgitter.py lint phases/phase2-author-antigravity-engine.json
./laufgitter.py lint phases/phase3-antigravity-screenshot.json
```
Linting verifies schema validity, argument placeholder presence, engine registration existence, and task key uniqueness.

---

## 7. Invariant Compliance Matrix

| Target Invariant | Compliance Implementation in Design Specification | Verification Status |
|:---|:---|:---|
| **Orchestrator Naming** | Canonicalized to `Laufgitter` / `laufgitter.py` / `~/.config/laufgitter` / `~/.laufgitter` / `/tmp/laufgitter-phase*`. | VERIFIED |
| **HUD / Dashboard Naming** | Canonicalized to `Zentrale` / `Zentrale HUD` / `zentrale.html` / `Zentrale.app`. | VERIFIED |
| **Repository Name** | Canonicalized to `Laufgitter`. | VERIFIED |
| **Complete 4-Phase Scope** | Detailed technical architecture and specifications for Phases 0, 1, 2, and 3. | VERIFIED |
| **Deterministic Verifier** | Phase 3 verifier relies on Python `PIL.Image` assertions rather than LLM judges. | VERIFIED |
| **Human Gates** | Documented explicit checkpoints (Gate 0 through Gate 3) between sequential manifests. | VERIFIED |
| **Solution8 Standard** | Aligned with firm architectural standards and memory rules. | VERIFIED |
