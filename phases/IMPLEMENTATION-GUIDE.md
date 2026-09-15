# Implementation Guide: Antigravity Integration Phases

This guide walks through the actual execution and manual gates between phases.

## Pre-Flight Checklist

- [ ] `laufgitter.py` executable and in parent directory (`../laufgitter.py`)
- [ ] Python 3.12+ installed
- [ ] OpenCode CLI installed: `opencode --version`
- [ ] OpenCode authenticated with litellm proxy: `opencode auth list` shows a provider
- [ ] `~/.config/laufgitter/config.toml` exists with `[engines.opencode]` block
- [ ] `opencode-sandboxed.sh` exists and is referenced in config by absolute path
- [ ] Temporary directories writable: `/tmp/laufgitter-phase0/`, `/tmp/laufgitter-phase1/`, etc.

## Phase 0 Execution

### 1. Export litellm proxy env vars

```bash
export OPENAI_BASE_URL=https://your-litellm-host:8000/v1
export OPENAI_API_KEY=your-litellm-key
```

### 2. Lint the manifest

```bash
cd /path/to/Laufgitter
./laufgitter.py lint phases/phase0-litellm-probe.json
# Expected output: "lint: clean (1 tasks)"
```

### 3. Run Phase 0

```bash
./laufgitter.py run phases/phase0-litellm-probe.json
```

Expected output: task "gpt54-routing-probe" should PASS or RETRY (if the check grep fails initially).

### 4. Manual verification

**This is the critical gate.** The check only grepped for "ROUTING_OK"; you must verify the backend:

```bash
cat /tmp/laufgitter-phase0/gpt54-routing-probe/worker.log
```

Look for a line like:
```
ROUTING_OK gpt-5.4-via-azure
```

or

```
ROUTING_OK claude-3-opus-20250219  # (if it fell back to a default provider)
```

**Decision**:
- If it says `gpt-5.4-*` or `gpt-5.4-via-azure` or similar: ✅ Proceed to Phase 1
- If it says anything else (OpenRouter model, fallback): ❌ Stop. Litellm proxy routing is broken. Debug before proceeding.

---

## Phase 1 Execution

### 1. Lint

```bash
./laufgitter.py lint phases/phase1-research-antigravity.json
```

Expected: "lint: clean (1 tasks)"

### 2. Run Phase 1

```bash
./laufgitter.py run phases/phase1-research-antigravity.json
```

This will spawn one OpenCode worker that searches the web for Antigravity CLI docs. Timeout is 1800s (30 min). Expected PASS or RETRY with max_attempts: 2.

### 3. Read and verify output

```bash
jq . /tmp/laufgitter-phase1/research-antigravity-cli/antigravity-cli-facts.json | less
```

**Verification checklist**:
- [ ] All keys present: `headless_flag`, `exit_code_convention`, `base_url_mechanism`, `mcp_discovery_paths`, `model_flag`, `model_slug_examples`, `sandbox_flags`, `sources`
- [ ] `sources` is non-empty array of `{claim, url}` objects
- [ ] For each factual field (not `sources`), if it's **not** null, there's a matching source entry
- [ ] Sample source URLs are from official Antigravity docs, Google Cloud docs, or credible AI community docs (not random blogs)
- [ ] Facts match your own research or knowledge of Antigravity CLI

**Example of good output**:
```json
{
  "headless_flag": "--non-interactive",
  "exit_code_convention": "0 for success, 1+ for error",
  "base_url_mechanism": {
    "gemini": {"flag_or_env": "GOOGLE_GENERATIVE_AI_API_KEY"},
    "vertex": {"flag_or_env": "--google-vertex-endpoint", "env": "GOOGLE_APPLICATION_CREDENTIALS"}
  },
  "mcp_discovery_paths": ["/home/user/.config/agy/mcp.json", "CLI subcommand: agy mcp list"],
  "model_flag": "--model",
  "model_slug_examples": ["gemini-2.0-flash", "gemini-1.5-pro"],
  "sandbox_flags": ["--sandbox"],
  "sources": [
    {"claim": "headless_flag is --non-interactive", "url": "https://ai.google.dev/docs/agy-cli"},
    {"claim": "model_flag is --model", "url": "https://ai.google.dev/docs/agy-cli#model-selection"}
  ]
}
```

**Example of bad output** (reason to stop):
```json
{
  "headless_flag": "--headless",  # OK
  "exit_code_convention": "always 0, success is in stdout",  # Suspicious — contradicts Unix convention
  "base_url_mechanism": {
    "gemini": {"flag_or_env": "--gemini-base-url", "env": "CUSTOM_GEMINI_ENDPOINT"}
  },  # Invented — no source
  "sources": [
    {"claim": "headless_flag is --headless", "url": "https://reddit.com/r/google_ai/..."}
  ]
}
```

### 4. Decision gate

- **Good facts**: Proceed to Phase 2.
- **Missing sources**: Decide: (a) Re-run Phase 1 with refined search, (b) Manually research and edit `antigravity-cli-facts.json` by hand, then skip Phase 2's config generation and write it yourself.
- **Contradictory sources**: Stop. Reconsider whether Antigravity is the right integration path, or wait for clearer docs.

---

## Phase 2 Execution

### 1. Ensure Phase 1 facts are finalized

If you edited `antigravity-cli-facts.json` by hand in Phase 1, confirm it's valid JSON:

```bash
python3 -c "import json; json.load(open('/tmp/laufgitter-phase1/research-antigravity-cli/antigravity-cli-facts.json'))"; echo "Valid JSON"
```

### 2. Lint Phase 2

```bash
./laufgitter.py lint phases/phase2-author-antigravity-engine.json
```

### 3. Run Phase 2

```bash
./laufgitter.py run phases/phase2-author-antigravity-engine.json
```

Expected: gpt-5.4 reads Phase 1's facts and writes a TOML config block. PASS or RETRY.

### 4. Read and verify output

```bash
cat /tmp/laufgitter-phase2/write-antigravity-engine-config/antigravity-engine-block.toml
```

Expected structure:
```toml
# Facts from Phase 1: headless_flag is "--non-interactive"
# Facts from Phase 1: model_flag is "--model"

[engines.antigravity]
bin = "/usr/local/bin/agy"  # or wherever agy is installed
args_template = [
  "{taskdir}",
  "{spec}",
  "--non-interactive",     # from headless_flag
  "--model", "{model}",    # from model_flag
  "--sandbox",             # from sandbox_flags
  "{engine_args}",
]
sandbox_args = ["--sandbox"]
full_access_args = []  # or based on Phase 1's permission mechanism
model_default = "gemini-3.8-medium"
```

**Verification checklist**:
- [ ] Every line has a comment justifying it from Phase 1 facts
- [ ] No CLI flags are invented (all come from Phase 1's fields)
- [ ] Placeholders present: `{taskdir}`, `{spec}`, `{model}`, `{engine_args}`, `{access_args}`, `{sandbox_args}`, `{full_access_args}`
- [ ] `model_default` is exactly `"gemini-3.8-medium"`
- [ ] TOML syntax is valid (can parse with Python's `tomllib`)

### 5. Manual merge into config

```bash
# Backup your current config
cp ~/.config/laufgitter/config.toml ~/.config/laufgitter/config.toml.backup

# Append the antigravity engine block
cat /tmp/laufgitter-phase2/write-antigravity-engine-config/antigravity-engine-block.toml >> ~/.config/laufgitter/config.toml

# Verify TOML is still valid
python3 -c "import tomllib; tomllib.load(open('/home/$USER/.config/laufgitter/config.toml', 'rb'))"; echo "Config valid"
```

### 6. Set environment variables for Phase 3

Based on Phase 1's `base_url_mechanism`, export any needed env vars:

```bash
# Example if Phase 1 found GOOGLE_GENERATIVE_AI_API_KEY:
export GOOGLE_GENERATIVE_AI_API_KEY=<your-key>

# Example if Phase 1 found GOOGLE_VERTEX_BASE_URL:
export GOOGLE_VERTEX_BASE_URL=<your-vertex-endpoint>

# Example if Phase 1 found GOOGLE_APPLICATION_CREDENTIALS:
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/creds.json
```

---

## Phase 3 Execution

### 1. Ensure Antigravity CLI is installed and authenticated

```bash
agy --version
```

If not installed, per Phase 1's findings, install it (likely via `npm`, `pip`, or `brew`).

### 2. Lint Phase 3

```bash
./laufgitter.py lint phases/phase3-antigravity-screenshot.json
```

### 3. Run Phase 3

```bash
export GOOGLE_GENERATIVE_AI_API_KEY=<key>  # or other env vars from Phase 1/2
./laufgitter.py run phases/phase3-antigravity-screenshot.json
```

Expected:
- Task "write-contract": gpt-5.4 writes `contract.json`. PASS immediately.
- Task "execute-screenshot": antigravity reads the contract, uses MCP tools to capture a screenshot, saves `screenshot.png`. PASS or RETRY.
- After both PASS: check copies `screenshot.png` to `/tmp/laufgitter-phase3/artifacts-out/`.

### 4. Verify final artifact

```bash
ls -lh /tmp/laufgitter-phase3/artifacts-out/screenshot.png
file /tmp/laufgitter-phase3/artifacts-out/screenshot.png  # Should say "image data"

# Open and view
open /tmp/laufgitter-phase3/artifacts-out/screenshot.png  # macOS
# or: xdg-open /tmp/laufgitter-phase3/artifacts-out/screenshot.png  # Linux
```

Expected: A valid PNG image with dimensions meeting or exceeding the contract minimums.

### 5. Check contract dimensions

```bash
python3 << 'EOF'
import json
from PIL import Image

contract = json.load(open('/tmp/laufgitter-phase3/write-contract/contract.json'))
img = Image.open('/tmp/laufgitter-phase3/artifacts-out/screenshot.png')

print(f"Contract min: {contract['min_width']}x{contract['min_height']}")
print(f"Screenshot:   {img.width}x{img.height}")
print(f"Pass:         {img.width >= contract['min_width'] and img.height >= contract['min_height']}")
EOF
```

Expected output: `Pass: True`

---

## Troubleshooting Reference

### General

| Problem | Diagnosis | Fix |
|---------|-----------|-----|
| `laufgitter.py: command not found` | laufgitter.py not executable or not in PATH | `cd /path/to/Laufgitter && chmod +x laufgitter.py` |
| `ModuleNotFoundError: No module named 'tomllib'` | Python < 3.12 | Upgrade Python to 3.12+ |
| `/tmp/laufgitter-phaseN/...` not found after run | Laufgitter cleaned up temp directory | Re-run the phase; check live results in `~/.laufgitter/runs/` |

### Phase 0

| Problem | Diagnosis | Fix |
|---------|-----------|-----|
| FAIL: "gpt-5.4 via litellm did not respond" | OpenCode not reaching litellm, or litellm not routing gpt-5.4 | Check OPENAI_BASE_URL/OPENAI_API_KEY; test litellm with `curl` |
| worker.log says "Codex" or "OpenRouter" | litellm proxy is down; OpenCode fell back to default | Restart litellm; check `OPENAI_BASE_URL` |

### Phase 1

| Problem | Diagnosis | Fix |
|---------|-----------|-----|
| `antigravity-cli-facts.json` is mostly nulls | Web search timed out or returned no results | Re-run with max_attempts: 2; or manually research and populate facts |
| `sources` is empty or very few entries | gpt-5.4 did research but didn't cite | Inspect worker.log for errors; consider manual fact-checking |
| JSON parse error | gpt-5.4 didn't output valid JSON | Inspect worker output; re-run Phase 1 |

### Phase 2

| Problem | Diagnosis | Fix |
|---------|-----------|-----|
| `antigravity-engine-block.toml` references Phase 1 facts that don't exist (null) | Phase 1 didn't find those facts; gpt-5.4 invented values | Check Phase 1 output; edit TOML by hand to remove invented values |
| TOML syntax error | gpt-5.4 made a typo in args_template array | Edit TOML by hand or re-run Phase 2 |
| `{spec}` or `{model}` placeholder missing | gpt-5.4 forgot to include them | Edit TOML by hand to add them |

### Phase 3

| Problem | Diagnosis | Fix |
|---------|-----------|-----|
| execute-screenshot task fails with "No such file" | `/tmp/laufgitter-phase3/write-contract/contract.json` doesn't exist | Ensure Phase 3 runs with `max_parallel: 1` so write-contract completes first |
| `agy: command not found` | Antigravity CLI not installed or not in PATH | Install per Phase 1's findings; ensure it's executable |
| PIL import error in check | Python doesn't have Pillow installed | `pip install Pillow` |
| Screenshot dimensions too small | antigravity produced smaller image than contract requires | Adjust contract minimums in Phase 3 spec, or debug antigravity's MCP tool |

---

## Success Criteria

Your integration is complete when:

1. ✅ Phase 0: `worker.log` confirms gpt-5.4 is routed through litellm
2. ✅ Phase 1: `antigravity-cli-facts.json` is populated with sourced facts and no key contradictions
3. ✅ Phase 2: `antigravity-engine-block.toml` is valid TOML and merged into `~/.config/laufgitter/config.toml`
4. ✅ Phase 3: `screenshot.png` is a valid image meeting contract dimensions
5. ✅ Antigravity engine is registered and can be used in future Laufgitter manifests:
   ```bash
   ./laufgitter.py run my-manifest.json --engine antigravity
   ```
