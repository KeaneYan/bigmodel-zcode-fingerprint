---
name: bigmodel-zcode-fingerprint
description: "Fix BigModel/GLM error 1305 rate-limiting by matching ZCode's exact HTTP fingerprint."
version: 1.1.0
author: KeaneYan
license: MIT
metadata:
  hermes:
    tags: [hermes, bigmodel, glm, zcode, rate-limiting, anthropic-adapter]
---

# BigModel/GLM ZCode Fingerprint Matching

## Problem

BigModel (Zhipu) serves GLM models via an Anthropic-compatible endpoint at
`https://open.bigmodel.cn/api/anthropic` (or `https://api.z.ai/api/anthropic`).
BigModel's own coding agent (ZCode) uses this same endpoint.

BigModel uses **request fingerprinting** to distinguish real ZCode clients
from impostors. When it detects non-ZCode traffic, it applies aggressive
rate-limiting — returning **error 1305** ("该模型当前访问量过大") or HTTP 429.

Hermes Agent uses the Anthropic Python SDK (Stainless-generated), which
auto-injects **7 `X-Stainless-*` telemetry headers** and **`anthropic-beta`**
headers that ZCode never sends. BigModel treats these as a non-ZCode signal
and throttles.

Additionally, Hermes was using adaptive thinking (vs ZCode's manual thinking),
was sending `temperature=1` (ZCode doesn't), was missing `x-query-id` and
`x-session-id` headers, and was sending lowercase model names (`glm-5.2`
instead of ZCode's `GLM-5.2`).

## Solution

Four changes in `agent/anthropic_adapter.py`:

1. **Strip `X-Stainless-*` and `anthropic-beta` headers** via an httpx
   `event_hooks` callback on a custom `httpx.Client` (replaces the old
   `None`-value approach that crashed on httpx ≥0.80).
2. **Switch to manual thinking** (`{"type":"enabled","budget_tokens":N}`) and
   drop `temperature` for BigModel endpoints.
3. **Uppercase the model name** (`glm-5.2` → `GLM-5.2`).
4. **Add `x-query-id` and `x-session-id`** per-request headers.

## Prerequisites

- Hermes Agent source code at `~/.hermes/hermes-agent/`
- The file `agent/anthropic_adapter.py` must exist
- Python venv with httpx installed
- **GLM provider must be configured with `api_mode: anthropic_messages`** —
  the fingerprint matching code only runs on the Anthropic Messages API path

## GLM Provider Configuration

In `~/.hermes/config.yaml`, the GLM provider must be configured as:

```yaml
providers:
  custom:
    glm-anthropic:
      base_url: https://open.bigmodel.cn/api/anthropic
      api_key: ${GLM_API_KEY}            # read from ~/.hermes/.env
      api_mode: anthropic_messages       # ← CRITICAL: must be anthropic_messages
      models:
        glm-5.2:
          context_length: 200000
          max_tokens: 64000
          reasoning:
            enabled: true
            effort: xhigh                # → budget_tokens=32000, output effort=max
```

Then in `~/.hermes/.env`:
```
GLM_API_KEY=your-bigmodel-api-key
```

### Notes

- **`api_mode: anthropic_messages`** is required — without it, requests go
  through the ChatCompletions transport and none of the fingerprint matching
  code runs.
- **`base_url`** must contain `open.bigmodel.cn` or `api.z.ai` to trigger
  `_is_bigmodel_endpoint()` detection.
- **`api.z.ai`** is Zhipu's alternative endpoint — both work with this fix.
- **`max_tokens: 64000`** matches ZCode's default output cap for GLM-5.2.
- **`reasoning.effort: xhigh`** maps to `budget_tokens=32000` and
  `output_config.effort=max` (matching ZCode). See the budget mapping table
  in `references/fingerprint-comparison.md`.

## How to Apply

### Step 1: Locate insertion points

```bash
grep -n '_is_bigmodel_endpoint\|def build_anthropic_client\|def build_anthropic_kwargs' \
  ~/.hermes/hermes-agent/agent/anthropic_adapter.py
```

### Step 2: Apply the patch

```bash
cd ~/.hermes/hermes-agent
git apply /path/to/references/code-patch.diff
```

If `git apply` fails (line drift, local modifications), apply the changes
manually following `references/manual-application-guide.md`.

### Step 3: Restart

```bash
hermes gateway restart
```

### Step 4: Verify

```bash
cd ~/.hermes/hermes-agent
venv/bin/python3 -c '
import httpx
from agent.anthropic_adapter import build_anthropic_client

client = build_anthropic_client(
    api_key="test",
    base_url="https://open.bigmodel.cn/api/anthropic",
)
req = httpx.Request("POST", "https://open.bigmodel.cn/api/anthropic/v1/messages",
                    headers=dict(client.default_headers), json={"model": "GLM-5.2"})
for hook in client._client.event_hooks.get("request", []):
    hook(req)

stainless = any(k.startswith("x-stainless") for k in req.headers.keys())
beta = "anthropic-beta" in req.headers
print("X-Stainless stripped:", "OK" if not stainless else "FAIL")
print("anthropic-beta stripped:", "OK" if not beta else "FAIL")
print("x-query-id present:", "OK" if "x-query-id" in req.headers else "FAIL")
print("x-session-id present:", "OK" if "x-session-id" in req.headers else "FAIL")
'
```

Expected: all four lines print `OK`.

## Code Locations

| Location | Function | What it does |
|----------|----------|-------------|
| ~Line 737 | `build_anthropic_client()` | BigModel header setup + strip hook |
| ~Line 2213 | `build_anthropic_kwargs()` | Model name uppercasing |
| ~Line 2325 | `build_anthropic_kwargs()` | BigModel manual thinking config |

Exact line numbers may shift with Hermes updates. Search for `_is_bigmodel_endpoint`
to find the right spots.

## ZCode Version

The `ZCode/3.1.2` version string and headers were captured from a specific
ZCode release. To check for newer versions:

```bash
# If ZCode is installed locally:
grep -o 'ZCode/[0-9.]+' /Applications/ZCode.app/Contents/Resources/glm/zcode.cjs | head -1

# Or check the app version:
defaults read /Applications/ZCode.app/Contents/Info.plist CFBundleShortVersionString
```

If BigModel changes the fingerprint in a future ZCode release, re-capture the
headers from ZCode's rollout data (see below) and update the code accordingly.

## Troubleshooting

### Still getting error 1305 after applying the fix

1. **Verify `api_mode`**: Check `~/.hermes/config.yaml` — must be
   `anthropic_messages`, not `chat_completions`.
2. **Verify `base_url`**: Must contain `open.bigmodel.cn` or `api.z.ai`.
3. **Check the gateway log** for TypeError or import errors:
   ```bash
   tail -50 ~/.hermes/logs/gateway.error.log
   ```
4. **Run the verification script** (Step 4 above) — if it prints FAIL,
   the code wasn't applied correctly.
5. **Restart the gateway** to load the new code:
   ```bash
   hermes gateway restart
   ```
6. **Check for code conflicts**: If you have other patches to
   `anthropic_adapter.py`, ensure they don't override the BigModel block.

### TypeError: Header value must be str or bytes

This means the old `None`-value code is still active. Ensure you replaced
the `headers["X-Stainless-*"] = None` lines with the httpx event hook.

### Patch doesn't apply (git apply fails)

Your Hermes version may differ. Use `references/manual-application-guide.md`
to apply the changes manually.

## What NOT to Change

- Do not re-add `anthropic-beta` for BigModel — ZCode doesn't send it.
- Do not send `temperature` for BigModel — ZCode doesn't send it.
- Do not use adaptive thinking (`{"type":"adaptive"}`) — use manual thinking.
- Do not lowercase the GLM model name — ZCode sends uppercase.
- Do not set header values to `None` to suppress them — httpx ≥0.80 throws TypeError.

## How ZCode's Fingerprint Was Discovered

ZCode's Electron app ships its minified JS at:
```
/Applications/ZCode.app/Contents/Resources/glm/zcode.cjs
```

Key functions (minified names):
- `Wto()` — builds default headers (User-Agent, X-Title, X-ZCode-*, HTTP-Referer)
- `fct()` — builds per-request attribution headers (x-request-id, x-zcode-trace-id, x-session-id, x-query-id)
- `rIr()` — merges default headers with per-request headers

ZCode's actual request rollouts are at `~/.zcode/cli/rollout/model-io-sess_*.jsonl`,
containing real headers and request body sent to BigModel.

## Files

- `references/code-patch.diff` — Git patch file with all changes
- `references/manual-application-guide.md` — Step-by-step manual application if patch fails
- `references/fingerprint-comparison.md` — Detailed ZCode vs Hermes comparison tables
