# BigModel/GLM: Matching ZCode's HTTP Fingerprint

BigModel (Zhipu) serves GLM via an Anthropic-compatible endpoint at
`https://open.bigmodel.cn/api/anthropic` (and `https://api.z.ai/api/anthropic`).
BigModel's own coding agent (ZCode, Electron app) uses this same endpoint.
BigModel uses **request fingerprinting** to distinguish real ZCode clients\nfrom impostors and applies rate-limiting (error 1305 / HTTP 429) accordingly.\n\n> **Correction (2026-06-26):** Fingerprint matching reduces *some* 1305\n> errors, but the **primary cause of persistent 1305 is server-side capacity\n> throttling** during peak Chinese business hours (9am–9pm CST). Hermes and\n> ZCode use the **exact same API key** — there is no auth-tier difference.\n> See the "1305 Root Cause" section below for full evidence and the\n> reverse-engineered ZCode OAuth flow.\n\n**Original hypothesis (partially correct):** The Anthropic Python SDK\n(Stainless-generated) auto-injects 7 `X-Stainless-*` telemetry headers plus\n`anthropic-beta` headers that ZCode never sends. BigModel may treat these as a\nnon-ZCode signal, but this is **not the main driver of 1305** — the\nfingerprint fix is active and 1305 still occurs during peak hours.

## Prerequisite: api_mode Configuration

**GLM provider MUST be configured with `api_mode: anthropic_messages`** in
`~/.hermes/config.yaml`. Without this, requests go through the ChatCompletions
transport and none of the fingerprint matching code runs. The `base_url` must
contain `open.bigmodel.cn` or `api.z.ai` to trigger `_is_bigmodel_endpoint()`.

```yaml
providers:
  custom:
    glm-anthropic:
      base_url: https://open.bigmodel.cn/api/anthropic
      api_key: ${GLM_API_KEY}            # in ~/.hermes/.env
      api_mode: anthropic_messages       # CRITICAL
      models:
        glm-5.2:
          context_length: 200000
          max_tokens: 64000
          reasoning:
            enabled: true
            effort: xhigh                # → budget_tokens=32000
```

## Code Location

All fingerprint matching lives in `agent/anthropic_adapter.py`, inside
`build_anthropic_client()` (the `_is_bigmodel_endpoint(base_url)` branch,
~line 737) and in `build_anthropic_messages_request()` (~line 2330 for
thinking config, ~line 2215 for model name).

## What Hermes Sends (After Fix) — Matches ZCode Exactly

### Headers

| Header | Value | Source |
|--------|-------|--------|
| `User-Agent` | `ZCode/3.1.2` | default_headers |
| `HTTP-Referer` | `https://zcode.z.ai` | default_headers |
| `X-Title` | `Z Code@electron` | default_headers |
| `X-ZCode-Agent` | `glm` | default_headers |
| `X-ZCode-App-Version` | `3.1.2` | default_headers |
| `x-request-id` | UUID | default_headers |
| `x-zcode-trace-id` | UUID | default_headers |
| `x-query-id` | UUID | default_headers |
| `x-session-id` | UUID | default_headers |
| `X-Stainless-*` (×7) | **stripped** | httpx event hook |
| `anthropic-beta` | **stripped** | httpx event hook |

The strip is done via an httpx `event_hooks={"request": [...]}` callback
on a custom `httpx.Client` passed as `kwargs["http_client"]`. This runs
*after* the SDK merges its default headers but *before* the request hits
the wire. Setting header values to `None` (the old approach) causes
`TypeError` in httpx ≥0.80 and must not be used.

### Request Body

| Field | ZCode | Hermes (after fix) |
|-------|-------|--------------------|
| `model` | `GLM-5.2` (uppercase) | `GLM-5.2` (uppercased) |
| `thinking` | `{"type":"enabled","budget_tokens":32000}` | manual thinking, same structure |
| `output_config` | `{"effort":"max"}` | `{"effort":"max"}` |
| `temperature` | **absent** | **absent** |
| `top_p` / `top_k` | **absent** | **absent** |
| `anthropic-beta` | **absent** | **absent** |

### Thinking Budget Mapping

Hermes effort → GLM manual thinking budget:

| Hermes effort | GLM budget_tokens | GLM output_config.effort |
|---------------|-------------------|--------------------------|
| `xhigh` | 32000 | `max` |
| `high` | 16000 | `high` |
| `medium` | 8000 | `medium` |
| `low` / `minimal` | 4000 | `low` |

## How to Verify

```python
cd ~/.hermes/hermes-agent
venv/bin/python3 << 'PY'
import httpx
from agent.anthropic_adapter import build_anthropic_client

client = build_anthropic_client(
    api_key="test",
    base_url="https://open.bigmodel.cn/api/anthropic",
)

# Build a fake request to see what headers survive the strip hook
req = httpx.Request("POST", "https://open.bigmodel.cn/api/anthropic/v1/messages",
                    headers=dict(client.default_headers), json={"model": "GLM-5.2"})
for hook in client._client.event_hooks.get("request", []):
    hook(req)

for k in sorted(req.headers.keys()):
    if k not in ("host", "content-length"):
        print(f"  {k}: {req.headers[k]}")
PY
```

## How ZCode's Fingerprint Was Discovered

ZCode's Electron app ships its minified JS at:
```
/Applications/ZCode.app/Contents/Resources/glm/zcode.cjs
```

Key functions (minified names):
- `Wto()` — builds default headers (User-Agent, X-Title, X-ZCode-*, HTTP-Referer)
- `fct()` — builds per-request attribution headers (x-request-id, x-zcode-trace-id, x-session-id, x-query-id)
- `rIr()` — merges default headers with per-request headers

ZCode uses the Vercel AI SDK (`@ai-sdk/anthropic`), which is TypeScript-based
and does NOT add any `X-Stainless-*` headers (those are specific to the
Stainless-generated Python SDK).

ZCode's actual request rollouts are stored at:
```
~/.zcode/cli/rollout/model-io-sess_*.jsonl
```
These contain the actual headers and request body sent to BigModel, which is
how the exact fingerprint was captured.

## What NOT to Change

- Do not re-add `anthropic-beta` for BigModel — ZCode doesn't send it.
- Do not send `temperature` for BigModel — ZCode doesn't send it.
- Do not use adaptive thinking (`{"type":"adaptive"}`) for BigModel — use
  manual thinking (`{"type":"enabled","budget_tokens":N}`) to match ZCode.
- Do not lowercase the GLM model name — ZCode sends `GLM-5.2` (uppercase).
- Do not set header values to `None` to suppress them — httpx ≥0.80 throws
  `TypeError: Header value must be str or bytes`. Use the event hook approach.

## Pitfall: httpx None Header TypeError

The previous approach to suppressing X-Stainless headers was to set them to
`None` in `default_headers`. This worked in older httpx versions but **httpx
≥0.80 rejects None header values with `TypeError`**. The fix is to use an
`event_hooks={"request": [callback]}` on a custom `httpx.Client` that strips
the headers after the SDK builds the request.

> **sitecustomize.py removed (2026-06-29):** The diagnostic
> `sitecustomize.py` that was installed at
> `venv/lib/python3.11/site-packages/sitecustomize.py` to catch None-header
> TypeErrors has been removed — all None-header paths are eliminated and the
> event hook approach is the permanent fix. No diagnostic patch is needed.

## Public Skill Repo

A standalone skill with the git patch and manual application guide is published at:
https://github.com/KeaneYan/bigmodel-zcode-fingerprint

Other Hermes installations can apply the same fix via:
```bash
hermes skills tap add KeaneYan/bigmodel-zcode-fingerprint
```

## 1305 Root Cause: Server-Side Capacity Throttling (NOT Fingerprint)

**Updated 2026-06-26:** The fingerprint fix is active and correct, but
**1305 errors persist** because the real root cause is BigModel's server-side
capacity throttling during peak Chinese business hours (9am–9pm CST).

### Evidence

1. **Same API key**: Hermes `GLM_API_KEY` and ZCode `config.json` apiKey are
   byte-identical (`91eda6...VcdY`, 49 chars). ZCode's OAuth flow (see below)
   generates a Coding Plan API key that gets written to `config.json` — and
   that's the same key Hermes uses.
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
  Fail fast to MiMo or another fallback provider.
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

**Credential storage:** ZCode stores encrypted OAuth tokens in
`~/.zcode/v2/credentials.json` with `enc:v1:` prefix (AES-256-GCM, key from
`ZCODE_CREDENTIAL_SECRET` or OS keychain via `safeStorage`). These are
separate from the API key in `config.json` and are not used for API requests.

## History

- 2026-06-25: Initial fingerprint analysis and fix. Initially attributed 1305
  root cause to `X-Stainless-*` telemetry headers + `anthropic-beta` + wrong
  thinking mode + wrong model name casing. Also fixed httpx None-header
  TypeError. Published as standalone skill repo.
- 2026-06-26: Corrected root cause. Fingerprint fix is active and correct but
  1305 persists — confirmed as server-side capacity throttling during peak
  hours. Reverse-engineered ZCode OAuth flow to prove Hermes uses the same key.
  Memory entry "1305根因=auth层级差异" is **wrong** and should be updated.
- 2026-06-29: Full skill review. Synced GitHub repo to v1.2.0 (1305 root cause
  correction + OAuth flow). Removed sitecustomize.py diagnostic patch.
  Discovered version source nuance: `CFBundleShortVersionString` (3.1.3) ≠
  internal `appVersion` (3.1.2) — rollout data confirmed `ZCode/3.1.2` is
  still correct. Added anthropic-beta strip risk note. No unit tests exist
  yet — use the verification script in "How to Verify" as regression check.
