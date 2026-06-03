# Gemma 4 Multi-Token Prediction (MTP) on MLX — install & usage

This fork of [`ml-explore/mlx-lm`](https://github.com/ml-explore/mlx-lm) (MIT) adds
**Multi-Token-Prediction speculative decoding** for Gemma 4: the `gemma4_assistant`
drafter model class plus the `mtp_speculative_generate_step` loop. It is the same
code proposed upstream in [PR #1276](https://github.com/ml-explore/mlx-lm/pull/1276)
(+ the generation integration), shipped here so it is usable today regardless of
upstream merge status.

## Install

From PyPI (published; coexists with upstream `mlx-lm`, import path stays `mlx_lm`):

```bash
pip install mlx-lm-broomva
```

Or as a drop-in replacement for `mlx-lm` from the fork — pin the validated,
immutable tag (survives rebases on upstream `main`):

```bash
pip install "git+https://github.com/broomva/mlx-lm.git@mtp-v0.1"
```

Or track the moving branch (gets rebased onto upstream as it advances):

```bash
pip install "git+https://github.com/broomva/mlx-lm.git@feat/gemma4-mtp-generate"
```

## Usage

The drafter is auto-detected — pass it as `draft_model` and `stream_generate`
routes to the MTP loop. **Tune `num_draft_tokens` to the acceptance rate**
(see the perf note below — `1` is the sweet spot at batch=1):

```python
from mlx_lm import load, stream_generate
from mlx_lm.sample_utils import make_sampler

model, tok = load("mlx-community/gemma-4-e4b-it-4bit")
drafter, _ = load("mlx-community/gemma-4-E4B-it-assistant-bf16")

prompt = tok.apply_chat_template(
    [{"role": "user", "content": "Explain speculative decoding."}],
    add_generation_prompt=True,
)

for r in stream_generate(
    model, tok, prompt,
    max_tokens=200,
    draft_model=drafter,         # ← MTP drafter; auto-detected
    num_draft_tokens=1,          # ← tune to accept rate (see below)
    sampler=make_sampler(temp=0.0),
):
    print(r.text, end="", flush=True)
```

Output is identical to non-speculative generation (the target verifies every
token) — only latency changes.

## Validated performance (M4 Pro, batch=1 greedy, `gemma-4-e4b-it-4bit`)

| `num_draft_tokens` | speedup vs baseline |
|---|---|
| **1** | **1.11–1.22×** ✅ |
| 2 | 1.06× |
| 4 | 0.78× |
| 8 | 0.44× |

**Why `num_draft=1`:** the drafter forward is only ~5% of a target forward, so it is
cheap — but at the ~30–40% acceptance rate seen here, drafting *wide* wastes the
verification forward on tokens that get rejected. Drafting `1` ahead and verifying
`2` is the sweet spot at batch=1. The 1.7–2.2× headline from Google's announcement
needs batch 4–8 (their stated Apple-Silicon caveat); that batched path is not yet
implemented here.

## What this adds vs upstream mlx-lm

| File | Change |
|---|---|
| `mlx_lm/models/gemma4_assistant.py` | NEW — the MTP drafter model class (PR #1276) |
| `mlx_lm/models/gemma4_text.py` | `return_shared_kv_states` emission (gated; default path byte-identical) |
| `mlx_lm/models/gemma4.py` | wrapper threads the emission flag |
| `mlx_lm/generate.py` | `mtp_speculative_generate_step` + `is_mtp_drafter` + `stream_generate` dispatch |
| `tests/test_models.py` | model + bit-equivalence + accept-path + threading + wrapper tests |

The standard `speculative_generate_step` path is untouched — zero regression risk
to non-MTP drafters.

## License

MIT (inherited from `ml-explore/mlx-lm`, © Apple Inc.). This fork keeps the
copyright notice and adds the MTP changes under the same license. See `LICENSE`.
