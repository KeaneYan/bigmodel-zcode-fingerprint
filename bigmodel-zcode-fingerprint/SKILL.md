---
name: bigmodel-zcode-fingerprint
description: "Fix BigModel/GLM error 1305 rate-limiting by matching ZCode's exact HTTP fingerprint."
version: 1.0.0
author: KeaneYan
license: MIT
metadata:
  hermes:
    tags: [hermes, bigmodel, glm, zcode, rate-limiting, anthropic-adapter]
---

# BigModel/GLM ZCode Fingerprint Matching

## Problem

BigModel (Zhipu) serves GLM models via an Anthropic-compatible endpoint at
`https://open.bigmodel.cn/api/anthropic`. BigModel's own coding agent (ZCode)
uses this same endpoint.

BigModel uses **request fingerprinting** to distinguish real ZCode clients
from impostors. When it detects non-ZCode traffic, it applies aggressive
rate-limiting — returning **error 1305** ("该模型当前访问量过大") or HTTP 429.

Hermes Agent uses the Anthropic Python SDK (Stainless-generated), which
auto-injects **7 `X-Stainless-*` telemetry headers** and **`anthropic-beta`**
headers that ZCode never sends. BigModel treats these as a non-ZCode signal
and throttles.

Meanwhile, ZCode (built on Vercel AI SDK / TypeScript) sends none of these
Stainless headers, uses **manual thinking** instead of adaptive, and sends
**uppercase model names** (`GLM-5.2` not `glm-5.2`).

## Solution

Three changes in `agent/anthropic_adapter.py`:

1. **Strip `X-Stainless-*` and `anthropic-beta` headers** via an httpx
   `event_hooks` callback on a custom `httpx.Client`.
2. **Switch to manual thinking** (`{"type":"enabled","budget_tokens":N}`) and
   drop `temperature` for BigModel endpoints.
3. **Uppercase the model name** (`glm-5.2` → `GLM-5.2`).
4. **Add `x-query-id` and `x-session-id`** per-request headers.

## How to Apply

### Prerequisites

- Hermes Agent source code at `~/.hermes/hermes-agent/`
- The file `agent/anthropic_adapter.py` must exist
- Python venv with httpx installed

### Steps

1. **Read the current code** to locate the insertion points:
   ```bash
   grep -n '_is_bigmodel_endpoint\|def build_anthropic_client\|def build_anthropic_kwargs' \
     ~/.hermes/hermes-agent/agent/anthropic_adapter.py
   ```

2. **Apply the patch** from `references/code-patch.diff`:
   ```bash
   cd ~/.hermes/hermes-agent
   git apply references/code-patch.diff
   ```

   If `git apply` fails due to line drift, use `patch -p1` instead, or apply
   the changes manually following the instructions in
   `references/manual-application-guide.md`.

3. **Restart the gateway** (or CLI) to load the changes:
   ```bash
   hermes gateway restart
   ```

4. **Verify** the fingerprint matches ZCode:
   ```bash
   cd ~/.hermes/hermes-agent
   venv/bin/python3 << 'PY'
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
   for k in sorted(req.headers.keys()):
       if k not in ("host", "content-length"):
           print(f"  {k}: {req.headers[k]}")
   PY
   ```
   Expected: NO `x-stainless-*` or `anthropic-beta` headers. ZCode headers
   present (`user-agent: ZCode/3.1.2`, `x-zcode-*`, `x-query-id`, `x-session-id`).

## Code Locations (in agent/anthropic_adapter.py)

| Location | Function | What it does |
|----------|----------|-------------|
| ~Line 737 | `build_anthropic_client()` | BigModel header setup + strip hook |
| ~Line 2213 | `build_anthropic_kwargs()` | Model name uppercasing |
| ~Line 2325 | `build_anthropic_kwargs()` | BigModel manual thinking config |

Exact line numbers may shift with Hermes updates. Search for `_is_bigmodel_endpoint`
to find the right spots.

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
