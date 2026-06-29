# Fingerprint Comparison: ZCode vs Hermes

## Headers

Captured from ZCode's actual rollout data (`~/.zcode/cli/rollout/model-io-sess_*.jsonl`)
and Hermes's runtime (via httpx request inspection).

### ZCode Headers (Electron app, Vercel AI SDK)

| Header | Value |
|--------|-------|
| `HTTP-Referer` | `https://zcode.z.ai` |
| `User-Agent` | `ZCode/3.1.2` |
| `X-Title` | `Z Code@electron` |
| `X-ZCode-Agent` | `glm` |
| `X-ZCode-App-Version` | `3.1.2` |
| `x-request-id` | UUID (per-request) |
| `x-zcode-trace-id` | UUID (per-request) |
| `x-query-id` | UUID (per-request) |
| `x-session-id` | Session UUID |
| `x-api-key` | API key (injected by AI SDK) |
| `anthropic-version` | `2023-06-01` (injected by AI SDK) |
| `accept` | `application/json` (injected by AI SDK) |
| `content-type` | `application/json` |

**NOT sent by ZCode:**
- `X-Stainless-Lang`, `X-Stainless-Package-Version`, `X-Stainless-OS`, `X-Stainless-Arch`, `X-Stainless-Runtime`, `X-Stainless-Runtime-Version`, `X-Stainless-Async`
- `anthropic-beta`

### Hermes Headers (Before Fix)

Same ZCode headers PLUS 7 `X-Stainless-*` headers + `anthropic-beta`, MINUS
`x-query-id` and `x-session-id`. Additionally, the old code tried to suppress
`X-Stainless-*` by setting header values to `None`, which caused
`TypeError: Header value must be str or bytes` on httpx ≥0.80.

### Hermes Headers (After Fix)

**Identical to ZCode.** The httpx event hook strips `X-Stainless-*` and
`anthropic-beta` after the SDK merges them but before the request goes out.
The `None`-value approach was replaced with the event hook.

---

## Request Body

### ZCode Body (from rollout)

```json
{
  "model": "GLM-5.2",
  "max_tokens": 64000,
  "thinking": {"type": "enabled", "budget_tokens": 32000},
  "output_config": {"effort": "max"},
  "system": [...],
  "tools": [...],
  "tool_choice": {"type": "auto"},
  "stream": true
}
```

**Notable absences:** `temperature`, `top_p`, `top_k`, `metadata`, `stop_sequences`.

### Hermes Body (After Fix)

| Field | ZCode | Hermes (after fix) |
|-------|-------|--------------------|
| `model` | `GLM-5.2` (uppercase) | `GLM-5.2` (uppercased) |
| `thinking` | `{"type":"enabled","budget_tokens":32000}` | Same structure, budget from effort mapping |
| `output_config` | `{"effort":"max"}` | `{"effort":"max"}` (for xhigh effort) |
| `temperature` | absent | absent |
| `top_p` / `top_k` | absent | absent |
| `anthropic-beta` | absent | absent |

### Thinking Budget Mapping

| Hermes effort | GLM `budget_tokens` | GLM `output_config.effort` |
|---------------|---------------------|----------------------------|
| `xhigh` | 32000 | `max` |
| `high` | 16000 | `high` |
| `medium` | 8000 | `medium` |
| `low` / `minimal` | 4000 | `low` |

---

## Why the Differences Matter

BigModel's endpoint at `open.bigmodel.cn/api/anthropic` is shared between
ZCode (their official coding agent) and third-party clients using API keys.
BigModel uses request fingerprinting — specifically the presence of
`X-Stainless-*` headers, `anthropic-beta`, the thinking mode, and model name
casing — to classify traffic and apply different rate-limit thresholds:

- **ZCode fingerprint** → matches official client, avoids additional throttling
- **Non-ZCode fingerprint** → may trigger more aggressive rate-limiting (error 1305 / 429)

> **⚠️ Correction (2026-06-26):** Fingerprint matching reduces *some* 1305 errors
> but does **not** eliminate them. The primary cause of persistent 1305 is
> **server-side capacity throttling** during peak Chinese business hours
> (9am–9pm CST), which affects all clients regardless of fingerprint. Hermes
> and ZCode use the exact same API key — there is no auth-tier difference.
> See the SKILL.md "1305 Root Cause" section for full evidence.
