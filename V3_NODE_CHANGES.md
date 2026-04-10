# ComfyUI V3 Node Compatibility Changes

## Problem

ComfyUI V3 nodes (e.g. `LatentFlipBatch`, `easy showAnything`) use a new architecture with `EXECUTE_NORMALIZED` as their FUNCTION. The existing code generator produced broken scripts for these nodes:

1. `cls.hidden is None` crash — V3 nodes need `PREPARE_CLASS_CLONE()` instead of `CLASS()`
2. `unexpected keyword argument 'audioui'` — frontend-only params leaked into `execute()`
3. `unexpected keyword argument 'unique_id'` — V3 nodes handle this internally via `cls.hidden`
4. `INPUT_IS_LIST` nodes received unwrapped arguments
5. Parameter names with colons/emojis caused `SyntaxError`

## Files Modified

### comfyui_to_python_utils.py

Added `prepare_v3_node(cls)`:
```python
def prepare_v3_node(cls):
    if hasattr(cls, 'PREPARE_CLASS_CLONE'):
        return cls.PREPARE_CLASS_CLONE(None)
    return cls
```

### comfyui_to_python.py

| Location | Change |
|----------|--------|
| `generate_workflow()` | Detect V3 via `class_def.FUNCTION in ("EXECUTE_NORMALIZED", "EXECUTE_NORMALIZED_ASYNC")`, track `has_v3_nodes` |
| `get_class_info()` | V3: `prepare_v3_node(CLASS)` instead of `CLASS()` |
| `generate_workflow()` | V3: inspect `execute()` signature instead of `EXECUTE_NORMALIZED(**kwargs)` |
| `generate_workflow()` | V3: skip `unique_id` injection |
| `create_function_call_code()` | New `input_is_list` param, wrap args in `[]` when `INPUT_IS_LIST=True` |
| `format_arg()` | Use `clean_parameter_name()` to sanitize keys |
| `clean_variable_name()` | Handle colons, empty results |
| `clean_parameter_name()` | New method — sanitize parameter names |
| `assemble_python_code()` | Embed `prepare_v3_node` source when `has_v3_nodes=True` |

### README.md

Added V1.4.0 Release Notes section.

## Git Info

- Branch: `fix-v3-node-compatibility`
- Commit: `78e1d67`
- Status: committed, not yet pushed

---

## DynamicCombo Fix (2026-03-06)

### Problem

V3 nodes using `DynamicCombo` inputs (e.g. `TextGenerateLTX2Prompt`'s `sampling_mode`) crashed at runtime:

```
AttributeError: 'str' object has no attribute 'get'
```

**Root cause:** ComfyUI stores `DynamicCombo` inputs as flat dot-separated keys in the workflow API JSON:

```json
{
  "sampling_mode": "on",
  "sampling_mode.temperature": 0.7,
  "sampling_mode.seed": 0,
  "sampling_mode.top_k": 64
}
```

At runtime, ComfyUI's `build_nested_inputs()` converts these into a nested dict before calling `execute()`:

```python
{"sampling_mode": {"sampling_mode": "on", "temperature": 0.7, "seed": 0, "top_k": 64}}
```

The code generator had two problems:
1. **Input filtering dropped dot-separated keys** — `sampling_mode.temperature` didn't match any `execute()` parameter, so it was discarded
2. **No dict literal formatting** — even if kept, `format_arg()` didn't know how to generate proper Python dict literals

This also caused a secondary error (`manual_seed expected a long, but got NoneType`) when `seed` was `None` inside the dict.

### Changes in comfyui_to_python.py

| Location | Change |
|----------|--------|
| `generate_workflow()` | Call `group_dynamic_inputs()` for V3 nodes before parameter filtering |
| `group_dynamic_inputs()` | New static method — groups `{"key": v, "key.child": v2}` into `{"key": {"key": v, "child": v2}}` |
| `format_arg()` | Added `elif isinstance(value, dict)` branch to handle plain dict values (not just `variable_name` refs) |
| `format_dict_value()` | New static method — generates Python dict literals with proper string escaping, `None` handling, and `seed`/`noise_seed` randomization |

### Affected Nodes (known cases)

| Node | DynamicCombo Input | Sub-parameters |
|------|--------------------|----------------|
| `TextGenerateLTX2Prompt` / `TextGenerate` | `sampling_mode` | `temperature`, `top_k`, `top_p`, `min_p`, `seed`, `repetition_penalty` |
| `ResizeImageMaskNode` | `resize_type` | `width`, `height`, `crop` |

Any V3 node using `io.DynamicCombo.Input` will benefit from this fix.

## Push & PR

```bash
cd /mnt/lv2t/Service/ComfyUI/custom_nodes/ComfyUI-to-Python-Extension
git push fork fix-v3-node-compatibility
```

Then create PR: `ALIoneTreE1119:fix-v3-node-compatibility` → `pydn:main`
