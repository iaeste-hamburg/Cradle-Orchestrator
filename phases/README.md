# Antigravity Integration: Four-Phase Laufgitter Workflow

This directory contains sequential Laufgitter manifests implementing end-to-end integration of Google's Antigravity CLI (`agy`) as a worker engine. The spec treats all information about antigravity as unverified until Phase 1 confirms it via live web search.

## Quick Start

1. **Ensure prerequisites**:
   - Python 3.12+
   - `laufgitter.py` in the parent directory
   - OpenCode CLI installed and authenticated (for Phase 0–2)
   - ~/.config/laufgitter/config.toml with `[engines.opencode]` configured
   - Export litellm proxy env vars before Phase 0: `export OPENAI_BASE_URL=<litellm_proxy>` etc.

2. **Run phases sequentially** (cannot run in parallel due to cross-phase dependencies):

```bash
cd ..
./laufgitter.py run phases/phase0-litellm-probe.json
# Check Phase 0 results manually (see verification checklist below)

./laufgitter.py run phases/phase1-research-antigravity.json
# **MANUAL GATE**: Read /tmp/laufgitter-phase1/research-antigravity-cli/antigravity-cli-facts.json
# Verify sources match your expectations before proceeding

./laufgitter.py run phases/phase2-author-antigravity-engine.json
# **MANUAL GATE**: Read /tmp/laufgitter-phase2/write-antigravity-engine-config/antigravity-engine-block.toml
# Merge into ~/.config/laufgitter/config.toml by hand (human-in-the-loop by design)

./laufgitter.py run phases/phase3-antigravity-screenshot.json
# Verify final artifact: /tmp/laufgitter-phase3/artifacts-out/screenshot.png
```

## Phase Descriptions

### Phase 0: Prove gpt-5.4 via litellm

**Purpose**: Confirm that OpenCode's provider routing through litellm works end-to-end before committing to downstream phases that depend on gpt-5.4.

**What it does**:
- Single probe task requesting gpt-5.4 to echo its model identity
- Weak check (only grep, cannot assert backend without manual inspection)
- **Manual verification**: read `/tmp/laufgitter-phase0/.../worker.log` and confirm the reported model identity is your Azure-routed gpt-5.4

**Environment**:
```bash
export OPENAI_BASE_URL=https://<your-litellm-proxy>:8000/v1
export OPENAI_API_KEY=<litellm-key>
```

### Phase 1: Live Web Research on Antigravity CLI

**Purpose**: Treat all claims about antigravity's flags, env vars, and MCP discovery as unverified hearsay until a worker with live web access confirms them.

**What it does**:
- gpt-5.4 searches official Antigravity docs (web search within OpenCode)
- Produces `antigravity-cli-facts.json` with structured findings and source URLs
- Check validates schema presence and citation, not truth

**Output**: `/tmp/laufgitter-phase1/research-antigravity-cli/antigravity-cli-facts.json`

**Expected keys** (set to null if not found in official sources):
- `headless_flag`: flag for non-interactive execution
- `exit_code_convention`: how the CLI signals success/failure
- `base_url_mechanism`: object with per-provider (Gemini, Vertex) flag/env-var names
- `mcp_discovery_paths`: array of file paths or CLI subcommands for MCP registration
- `model_flag`: flag to select model
- `model_slug_examples`: array of valid model identifiers
- `sandbox_flags`: flags for sandboxed/permission-bypass execution
- `sources`: array of `{claim, url}` pairs — one per fact

**Manual gate**: Read the output JSON and cross-check against your own research or docs. If sources contradict your knowledge or cite irrelevant URLs, **stop and re-scope Phase 2** — do not encode hallucinated flags into real config.

### Phase 2: Author [engines.antigravity] Block

**Purpose**: Use gpt-5.4 to write a valid TOML engine config block, **using only facts from Phase 1's output**.

**What it does**:
- Reads Phase 1's `antigravity-cli-facts.json`
- Generates `antigravity-engine-block.toml` with commented lines justifying each value
- Validates schema (all required keys, placeholders present)

**Output**: `/tmp/laufgitter-phase2/write-antigravity-engine-config/antigravity-engine-block.toml`

**Manual gate**: Read the generated TOML and verify:
1. Every line comment references a fact from Phase 1
2. No CLI flags or env vars are invented outside Phase 1's facts
3. All placeholders (`{spec}`, `{model}`, `{taskdir}`, `{access_args}`, `{sandbox_args}`, `{engine_args}`) are present

**Next step**: Manually merge this block into `~/.config/laufgitter/config.toml`:
```bash
cat /tmp/laufgitter-phase2/.../antigravity-engine-block.toml >> ~/.config/laufgitter/config.toml
```

Laufgitter does not self-modify its config (by design) — hand-merge like any engine block in `config.sample.toml`.

### Phase 3: Contract-Driven Execution

**Purpose**: Demonstrate gpt-5.4 writing a contract, antigravity executing it, and a deterministic check verifying the result — no model calls in the check itself.

**What it does**:
1. **write-contract**: gpt-5.4 writes `contract.json` declaring artifact requirements (file name, min dimensions for an image)
2. **execute-screenshot**: antigravity uses available MCP tools to produce `screenshot.png` matching the contract
3. **Check**: Python script reads contract.json, opens screenshot.png, verifies dimensions without calling a model

**Output**: `/tmp/laufgitter-phase3/artifacts-out/screenshot.png`

**Verification**:
- `contract.json` validates to schema
- `screenshot.png` is a valid image file
- Image dimensions ≥ contract minimums

## Verification Checklist

Before running each phase, lint it:

```bash
./laufgitter.py lint phases/phase0-litellm-probe.json
./laufgitter.py lint phases/phase1-research-antigravity.json
./laufgitter.py lint phases/phase2-author-antigravity-engine.json
./laufgitter.py lint phases/phase3-antigravity-screenshot.json
```

Optionally, run baseline (no workers, just checks against empty state) to prove checks fail correctly:

```bash
./laufgitter.py run phases/phase0-litellm-probe.json --baseline
# Expected: FAIL (no worker output to grep)

./laufgitter.py run phases/phase1-research-antigravity.json --baseline
# Expected: FAIL (no antigravity-cli-facts.json in scratch dir)

# etc.
```

### Full Verification Workflow

1. **Phase 0**:
   - [ ] `./laufgitter.py run phases/phase0-litellm-probe.json` exits PASS
   - [ ] Manually inspect `/tmp/laufgitter-phase0/.../worker.log` and confirm model identity is Azure-routed gpt-5.4

2. **Phase 1**:
   - [ ] `./laufgitter.py run phases/phase1-research-antigravity.json` exits PASS
   - [ ] Read `/tmp/laufgitter-phase1/research-antigravity-cli/antigravity-cli-facts.json`
   - [ ] Cross-check each source URL against official Antigravity docs
   - [ ] Decision: proceed (Phase 2) or re-scope (Phase 1 again with refined search)

3. **Phase 2**:
   - [ ] `./laufgitter.py run phases/phase2-author-antigravity-engine.json` exits PASS
   - [ ] Read `/tmp/laufgitter-phase2/write-antigravity-engine-config/antigravity-engine-block.toml`
   - [ ] Verify each line's comment justifies a Phase 1 fact
   - [ ] Manually merge into `~/.config/laufgitter/config.toml`
   - [ ] Set env vars per Phase 1 findings (e.g., `GOOGLE_GEMINI_BASE_URL`)

4. **Phase 3**:
   - [ ] `./laufgitter.py run phases/phase3-antigravity-screenshot.json` exits PASS
   - [ ] Check `/tmp/laufgitter-phase3/artifacts-out/screenshot.png` exists and opens in an image viewer
   - [ ] Verify dimensions match or exceed contract minimums

## Design Notes

### Why Four Separate Manifests?

Laufgitter has no cross-manifest dependency mechanism. Tasks within one manifest run in parallel; ordering across phases requires running separate manifests sequentially. This design:

1. **Forces human gates**: Between each phase, a human reads output and decides whether to proceed. This catches hallucinated facts before they propagate.
2. **Isolates concerns**: Phase 0 tests the model routing layer; Phase 1 tests research capability; Phase 2 tests code generation from sourced data; Phase 3 tests end-to-end execution and deterministic validation.
3. **Enables reruns**: If Phase 1 finds bad sources, re-run it in isolation without repeating Phase 0.

### Why max_parallel: 1 in Phase 3?

Phase 3's second task (`execute-screenshot`) depends on the first task's output (`contract.json`). Setting `max_parallel: 1` ensures write-contract completes before antigravity starts. (Laufgitter does not otherwise express cross-task dependencies.)

### Why worktrees: false in Phase 3's execute-screenshot?

The task spec references an absolute path outside its worktree: `/tmp/laufgitter-phase3/write-contract/contract.json`. With worktrees enabled, that directory would be deleted after write-contract passes. `worktrees: false` keeps it available. The downside: parallel workers in this manifest could theoretically collide, but `max_parallel: 1` makes that moot here.

### Why no full_access in Phase 3?

Antigravity uses existing MCP servers, not spawning sub-workers. No sandbox escape is needed, so `full_access` is omitted (defaults to false/sandboxed).

## Environment Setup

### For Phase 0–2 (OpenCode/gpt-5.4)

```bash
# 1. Install OpenCode
curl -fsSL https://opencode.ai/install | bash

# 2. Auth to OpenCode with litellm proxy
OPENAI_BASE_URL=https://<litellm-host>:8000/v1OPENAI_API_KEY=<key> opencode auth login

# 3. Uncomment and configure [engines.opencode] in ~/.config/laufgitter/config.toml
# Set bin = /absolute/path/to/laufgitter/engines/opencode-sandboxed.sh

# 4. Before running phases, export litellm proxy env vars
export OPENAI_BASE_URL=https://<litellm-host>:8000/v1
export OPENAI_API_KEY=<key>
```

### For Phase 3 (Antigravity/Gemini)

```bash
# After Phase 2, merge config and set Gemini env vars
export GOOGLE_GEMINI_BASE_URL=<if-required-by-phase-1>
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json  # if Vertex
```

## Troubleshooting

### Phase 0 fails with "FAIL: gpt-5.4 via litellm did not respond"

- Check litellm proxy is running and accessible
- Verify `OPENAI_BASE_URL` and `OPENAI_API_KEY` are exported
- Inspect `/tmp/laufgitter-phase0/.../worker.log` for OpenCode errors

### Phase 1 produces empty or null facts

- Web search inside OpenCode may be rate-limited or blocked
- Re-run with `max_attempts: 2` to let the retry pick different search terms
- Manual research: visit Antigravity docs and correct `antigravity-cli-facts.json` by hand before Phase 2

### Phase 2 generates invalid TOML

- Likely Phase 1 facts are malformed or incomplete
- Re-run Phase 1 with refined search or manual correction
- Inspect `/tmp/laufgitter-phase2/.../antigravity-engine-block.toml` for syntax errors

### Phase 3 execute-screenshot fails to read contract

- Confirm Phase 3 is run with `max_parallel: 1` so write-contract completes first
- Check file path `/tmp/laufgitter-phase3/write-contract/contract.json` matches task key

## References

- [Laufgitter README](../README.md) — full documentation
- [config.sample.toml](../config.sample.toml) — engine block examples
- Phase manifests above: each JSON file documents its fields per Laufgitter spec
