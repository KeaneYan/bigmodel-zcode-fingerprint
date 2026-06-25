# Manual Application Guide

If `git apply` fails (due to line drift from Hermes updates or local patches),
apply the three changes manually.

## Change 1: Headers + Strip Hook (in `build_anthropic_client`)

Find the `_is_bigmodel_endpoint(base_url)` branch inside `build_anthropic_client`
(search for `if _is_bigmodel_endpoint`). Replace the header block.

### Before (upstream code)

```python
        kwargs["api_key"] = api_key
        headers = {
            "User-Agent": "ZCode/3.1.2",
            "HTTP-Referer": "https://zcode.z.ai",
            "X-Title": "Z Code@electron",
            "X-ZCode-Agent": "glm",
            "X-ZCode-App-Version": "3.1.2",
            "x-request-id": str(uuid4()),
            "x-zcode-trace-id": str(uuid4()),
        }
        if common_betas:
            headers["anthropic-beta"] = ",".join(common_betas)
        # Suppress X-Stainless-* headers that the SDK injects by default.
        # The SDK respects None values to skip a header.
        headers["X-Stainless-Lang"] = None
        headers["X-Stainless-Package-Version"] = None
        headers["X-Stainless-OS"] = None
        headers["X-Stainless-Arch"] = None
        headers["X-Stainless-Runtime"] = None
        headers["X-Stainless-Runtime-Version"] = None
        headers["X-Stainless-Async"] = None
        kwargs["default_headers"] = headers
```

### After

```python
        kwargs["api_key"] = api_key
        _bm_session_id = str(uuid4())
        headers = {
            "User-Agent": "ZCode/3.1.2",
            "HTTP-Referer": "https://zcode.z.ai",
            "X-Title": "Z Code@electron",
            "X-ZCode-Agent": "glm",
            "X-ZCode-App-Version": "3.1.2",
            "x-request-id": str(uuid4()),
            "x-zcode-trace-id": str(uuid4()),
            "x-query-id": str(uuid4()),
            "x-session-id": _bm_session_id,
        }
        # Do NOT send anthropic-beta — ZCode doesn't, and BigModel may
        # use its presence as a non-ZCode signal.
        kwargs["default_headers"] = headers

        # httpx request hook: strip X-Stainless-* and anthropic-beta
        # headers that the SDK injects after default_headers are merged.
        from httpx import Client as _HttpxClient, Timeout as _HttpxTimeout

        def _strip_sdk_telemetry(request):
            """Remove X-Stainless-* and anthropic-beta from outbound requests."""
            for key in list(request.headers.keys()):
                if key.lower().startswith("x-stainless-") or key.lower() == "anthropic-beta":
                    del request.headers[key]

        _read_timeout_bm = timeout if (isinstance(timeout, (int, float)) and timeout > 0) else 900.0
        kwargs["http_client"] = _HttpxClient(
            timeout=_HttpxTimeout(timeout=float(_read_timeout_bm), connect=10.0),
            event_hooks={"request": [_strip_sdk_telemetry]},
        )
```

**Key points:**
- `x-query-id` and `x-session-id` are NEW headers that ZCode sends.
- The httpx event hook strips `X-Stainless-*` and `anthropic-beta` after the
  SDK merges its default headers but before the request goes on the wire.
- Do NOT use `None` header values — httpx ≥0.80 throws TypeError. The old
  code set them to `None` which caused crashes; the event hook is the fix.

> **If your code doesn't have the `headers["X-Stainless-*"] = None` lines:**
> Your Hermes version may already handle this differently. Just delete any
> remaining `X-Stainless-*` suppression code and add the event hook + new
> headers as shown above.

---

## Change 2: Model Name Uppercasing (in `build_anthropic_kwargs`)

Find `model = normalize_model_name(model, preserve_dots=preserve_dots)` in
`build_anthropic_kwargs`. Add immediately after:

```python
    model = normalize_model_name(model, preserve_dots=preserve_dots)
    # BigModel/GLM: ZCode sends uppercase model names (e.g. "GLM-5.2").
    # normalize_model_name lowercases, so restore the canonical BigModel form
    # to match ZCode's fingerprint exactly.
    if _is_bigmodel_endpoint(base_url) and model.lower().startswith("glm-"):
        model = model.upper()
```

---

## Change 3: Manual Thinking for BigModel (in `build_anthropic_kwargs`)

Find `_is_kimi_coding = _is_kimi_family_endpoint(base_url, model)` and the
reasoning_config block. Add a `_is_bigmodel` check BEFORE `_supports_adaptive_thinking`:

### Before

```python
    _is_kimi_coding = _is_kimi_family_endpoint(base_url, model)
    if reasoning_config and isinstance(reasoning_config, dict) and not _is_kimi_coding:
        if reasoning_config.get("enabled") is not False and "haiku" not in model.lower():
            effort = str(reasoning_config.get("effort", "medium")).lower()
            budget = THINKING_BUDGET.get(effort, 8000)
            if _supports_adaptive_thinking(model):
```

### After

```python
    _is_kimi_coding = _is_kimi_family_endpoint(base_url, model)
    _is_bigmodel = _is_bigmodel_endpoint(base_url)
    if reasoning_config and isinstance(reasoning_config, dict) and not _is_kimi_coding:
        if reasoning_config.get("enabled") is not False and "haiku" not in model.lower():
            effort = str(reasoning_config.get("effort", "medium")).lower()
            budget = THINKING_BUDGET.get(effort, 8000)
            if _is_bigmodel:
                # BigModel/GLM: use manual thinking (matching ZCode's fingerprint)
                _glm_budget_map = {"xhigh": 32000, "high": 16000, "medium": 8000, "low": 4000, "minimal": 4000}
                _glm_budget = _glm_budget_map.get(effort, 8000)
                kwargs["thinking"] = {"type": "enabled", "budget_tokens": _glm_budget}
                _glm_effort_map = {"xhigh": "max", "high": "high", "medium": "medium", "low": "low", "minimal": "low"}
                kwargs["output_config"] = {"effort": _glm_effort_map.get(effort, "medium")}
                # Do NOT send temperature — ZCode doesn't.
            elif _supports_adaptive_thinking(model):
```

**Key points:**
- BigModel goes into manual thinking, NOT adaptive.
- No `temperature` is set for BigModel (ZCode doesn't send it).
- The `_is_bigmodel` check must come BEFORE `_supports_adaptive_thinking`.

---

## Verification

After applying all three changes, restart and verify:

```bash
hermes gateway restart

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
query_id = "x-query-id" in req.headers
session_id = "x-session-id" in req.headers

print("X-Stainless stripped:", "OK" if not stainless else "FAIL")
print("anthropic-beta stripped:", "OK" if not beta else "FAIL")
print("x-query-id present:", "OK" if query_id else "FAIL")
print("x-session-id present:", "OK" if session_id else "FAIL")
'
```
