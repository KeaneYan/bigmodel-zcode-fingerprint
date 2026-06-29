---
name: bigmodel-zcode-fingerprint
description: "Match ZCode's HTTP fingerprint for BigModel/GLM requests. Reduces some rate-limiting; does NOT eliminate error 1305 (server-side capacity throttling)."
version: 1.2.0
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
from impostors. When it detects non-ZCode traffic, it may apply more aggressive
rate-limiting — returning **error 1305** ("该模型当前访问量过大") or HTTP 429.

Hermes Agent uses the Anthropic Python SDK (Stainless-generated), which
auto-injects **7 `X-Stainless-*` telemetry headers** and **`anthropic-beta`**
headers that ZCode never sends. BigModel treats these as a non-ZCode signal.

Additionally, Hermes was using adaptive thinking (vs ZCode's manual thinking),
was sending `temperature=1` (ZCode doesn't), was missing `x-query-id` and
`x-session-id` headers, and was sending lowercase model names (`glm-5.2`
instead of ZCode's `GLM-5.2`).

> **⚠️ Important correction (2026-06-26):** Fingerprint matching reduces *some*
> 1305 errors, but the **primary cause of persistent 1305 is server-side
> capacity throttling** during peak Chinese business hours (9am–9pm CST).
> Hermes and ZCode use the **exact same API key** — there is no auth-tier
> difference. The fingerprint fix is active and correct, but 1305 will still
> occur during capacity peaks. See "1305 Root Cause" below for full evidence.

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
defaults read /Applications/ZCode.app/Contents/Info.plist CFBundleShortVersionString

# Or grep the version string in the bundled JS:
grep -o 'ZCode/[0-9.]+' /Applications/ZCode.app/Contents/Resources/glm/zcode.cjs | head -1
```

If BigModel changes the fingerprint in a future ZCode release, re-capture the
headers from ZCode's rollout data and update the code accordingly.

## 1305 Root Cause: Server-Side Capacity Throttling (NOT Fingerprint)

**Updated 2026-06-26:** The fingerprint fix is active and correct, but
**1305 errors persist** because the real root cause is BigModel's server-side
capacity throttling during peak Chinese business hours (9am–9pm CST).

### Evidence

1. **Same API key**: Hermes `GLM_API_KEY` and ZCode `config.json` apiKey are
   byte-identical. ZCode's OAuth flow generates a Coding Plan API key that
   gets written to `config.json` — and that's the same key Hermes uses.
2. **Fingerprint fix verified active**: Code inspection confirms all 6 checks
   pass (`_is_bigmodel_endpoint`, ZCode UA, X-Stainless strip, event_hooks,
   anthropic-beta strip, GLM uppercase).
3. **Direct testing**: 20 sequential requests (with and without fingerprint,
   with and without large context + thinking) all returned 200 at off-peak.
4. **1305 pattern**: Strongly correlated with Chinese business hours. Nearly
   zero outside 7am–10pm CST. Bursts of 3–18 errors per hour during peaks.

### Why ZCode appears not to hit 1305

ZCode is interactive (low request rate, single user). Hermes has cron jobs,
curator reviews, long sessions — far higher concurrency against the same
shared Coding Plan endpoint pool.

### What helps

- **Fast-fallback on 1305**: Don't waste time on 5 retries (2–20s each).
  Fail fast to a fallback provider (e.g. MiMo, OpenRouter).
- **Stagger cron jobs**: Avoid scheduling at :00 — everyone hits the API then.
- **Reduce context size**: Smaller requests = less server load = less likely
  to trip capacity limits.

## ZCode OAuth → API Key Flow (Reverse-Engineered)

ZCode's OAuth login generates the same Coding Plan API key that Hermes uses.
Flow (from `zcode.cjs`, function names are minified):

```
Browser login → authCode
→ POST {BIGMODEL_BASE}/api/auth/tokenByAuthCode
   body: { appId, appSecret, authCode }
→ { accessToken, refreshToken? }
→ GET {BIGMODEL_BASE}/api/biz/customer/getCustomerInfo
   Authorization: <accessToken>
→ { organizationId, projectId }
→ GET {BIGMODEL_BASE}/api/biz/v1/organization/{orgId}/projects/{projectId}/api_keys
→ Find or create key named "zcode-api-key"
→ GET {BIGMODEL_BASE}/api/biz/v1/.../api_keys/copy/{keyId}
→ { apiKey, secretKey }
→ Final key = "{apiKey}.{secretKey}" (or just apiKey for bigmodel)
→ Written to ~/.zcode/v2/config.json as provider apiKey
```

**Key insight:** The OAuth token does NOT get used directly for API requests.
It's exchanged for a standard Coding Plan API key via BigModel's biz API.
This means implementing OAuth in Hermes would produce the **exact same key**
— there is no auth-tier difference to exploit.

## Troubleshooting

### Still getting error 1305 after applying the fix

**This is expected during peak hours.** The fingerprint fix reduces
non-ZCode-signal throttling but cannot prevent server-side capacity limits.
Mitigation:

1. **Fast-fallback**: Configure a fallback model (e.g. MiMo) so 1305 triggers
   an immediate switch rather than retrying.
2. **Stagger cron jobs**: Offset schedules by a few minutes from :00.
3. **Reduce context**: Lower `context_length` or use compression to send
   smaller requests.

### Verify the fix is active

1. **Verify `api_mode`**: Check `~/.hermes/config.yaml` — must be
   `anthropic_messages`, not `chat_completions`.
2. **Verify `base_url`**: Must contain `open.bigmodel.cn` or `api.z.ai`.
3. **Run the verification script** (Step 4 above) — if it prints FAIL,
   the code wasn't applied correctly.
4. **Check for code conflicts**: If you have other patches to
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

## History

- 2026-06-25: Initial fingerprint analysis and fix. Initially attributed 1305
  root cause to X-Stainless-* telemetry headers + anthropic-beta + wrong
  thinking mode + wrong model name casing. Also fixed httpx None-header
  TypeError. Published as standalone skill repo.
- 2026-06-26: Corrected root cause. Fingerprint fix is active and correct but
  1305 persists — confirmed as server-side capacity throttling during peak
  hours. Reverse-engineered ZCode OAuth flow to prove Hermes uses the same key.
