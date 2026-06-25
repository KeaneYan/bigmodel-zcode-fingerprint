# BigModel GLM ZCode Fingerprint Matching

Fix for Hermes Agent's BigModel/GLM error 1305 rate-limiting by matching ZCode's exact HTTP fingerprint.

## Problem

BigModel (Zhipu) serves GLM models via an Anthropic-compatible endpoint at `https://open.bigmodel.cn/api/anthropic`. BigModel's own coding agent ZCode uses this same endpoint, and BigModel uses **request fingerprinting** to distinguish real ZCode clients from impostors — applying aggressive rate-limiting (error 1305 / HTTP 429) to non-ZCode traffic.

Hermes Agent uses the Anthropic Python SDK which auto-injects 7 `X-Stainless-*` telemetry headers and `anthropic-beta` headers that ZCode never sends. Additionally, Hermes was using adaptive thinking (vs ZCode's manual thinking) and lowercase model names.

## Solution

Three changes in `agent/anthropic_adapter.py`:

1. **Strip `X-Stainless-*` and `anthropic-beta` headers** via an httpx event hook
2. **Switch to manual thinking** and drop `temperature` for BigModel endpoints  
3. **Uppercase the model name** (`glm-5.2` → `GLM-5.2`)
4. **Add `x-query-id` and `x-session-id`** per-request headers

## Installation

```bash
# Install via Hermes skills
hermes skills install https://github.com/KeaneYan/bigmodel-zcode-fingerprint/blob/main/bigmodel-zcode-fingerprint/SKILL.md

# Or tap the repo
hermes skills tap add KeaneYan/bigmodel-zcode-fingerprint
```

## Usage

After installing the skill, ask your Hermes Agent to apply the fix:

> Apply the bigmodel-zcode-fingerprint skill to fix GLM error 1305 rate-limiting.

The skill contains:
- `references/code-patch.diff` — Git patch file
- `references/manual-application-guide.md` — Step-by-step manual instructions
- `references/fingerprint-comparison.md` — Detailed ZCode vs Hermes comparison

## Requirements

- Hermes Agent (any recent version with `agent/anthropic_adapter.py`)
- BigModel/GLM provider configured (`open.bigmodel.cn/api/anthropic` or `api.z.ai/api/anthropic`)

## License

MIT
