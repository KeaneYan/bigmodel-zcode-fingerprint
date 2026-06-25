# BigModel GLM ZCode Fingerprint Matching

Fix for Hermes Agent's BigModel/GLM **error 1305 rate-limiting** by matching ZCode's exact HTTP fingerprint.

## Problem

BigModel (Zhipu) serves GLM models via an Anthropic-compatible endpoint at `https://open.bigmodel.cn/api/anthropic`. BigModel uses **request fingerprinting** to distinguish real ZCode clients from impostors — applying aggressive rate-limiting (error 1305 / HTTP 429) to non-ZCode traffic.

Hermes Agent uses the Anthropic Python SDK which auto-injects 7 `X-Stainless-*` telemetry headers and `anthropic-beta` that ZCode never sends. Additionally, Hermes was using adaptive thinking (vs manual), sending `temperature=1`, missing `x-query-id`/`x-session-id` headers, and sending lowercase model names.

## Solution

Four changes in `agent/anthropic_adapter.py`:

1. **Strip `X-Stainless-*` and `anthropic-beta` headers** via httpx event hook
2. **Switch to manual thinking** and drop `temperature` for BigModel endpoints
3. **Uppercase the model name** (`glm-5.2` → `GLM-5.2`)
4. **Add `x-query-id` and `x-session-id`** per-request headers

## Requirements

- Hermes Agent with `agent/anthropic_adapter.py`
- GLM provider configured with `api_mode: anthropic_messages`
- `base_url` containing `open.bigmodel.cn` or `api.z.ai`

## Installation

```bash
# Install the skill
hermes skills install https://github.com/KeaneYan/bigmodel-zcode-fingerprint

# Then ask Hermes to apply it
> Apply the bigmodel-zcode-fingerprint skill to fix GLM error 1305.
```

## Files

- `bigmodel-zcode-fingerprint/SKILL.md` — Full instructions
- `bigmodel-zcode-fingerprint/references/code-patch.diff` — Git patch
- `bigmodel-zcode-fingerprint/references/manual-application-guide.md` — Manual steps
- `bigmodel-zcode-fingerprint/references/fingerprint-comparison.md` — Comparison tables

## License

MIT
