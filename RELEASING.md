# Releasing the Broomva MLX-MTP distribution

This branch (`dist/broomva`) carries the **distribution** artifacts for the
Gemma 4 MTP work — kept off the upstream-PR branches (`feat/gemma4-assistant`,
`feat/gemma4-mtp-generate`) so those stay clean for `ml-explore/mlx-lm` review.

Three distribution channels, in order of preference:

## 1. Upstream merge (preferred)
PR [#1276](https://github.com/ml-explore/mlx-lm/pull/1276) (model class) →
follow-up PR (generation). If merged, MTP ships in `pip install mlx-lm` and this
branch is retired. **This is the goal; the channels below are insurance.**

## 2. Git install from the fork (zero extra work — use this now)
The `feat/gemma4-mtp-generate` branch + the immutable `mtp-v0.1` tag are directly
pip-installable. This is what Spec E's `inference-mlx` backend uses today:

```bash
pip install "git+https://github.com/broomva/mlx-lm.git@mtp-v0.1"
```

Installs **as `mlx-lm`** (replaces any upstream mlx-lm in the env). Correct for a
dedicated inference runtime. See `MTP-USAGE.md`.

## 3. PyPI derivative (`mlx-lm-broomva`) — ready, not yet published
For external users who want a `pip install <name>` that coexists with the option
of upstream mlx-lm. This branch renames the package metadata to `mlx-lm-broomva`.

**Why a renamed fork and not a runtime-monkeypatch extension package:** MTP requires
modifying `gemma4_text`'s forward to emit `shared_kv_states` (the per-layer K/V is
local to the forward and discarded at the norm — unreachable from outside the model
file). A monkeypatch would have to replace that method wholesale and re-break on every
upstream change. The renamed fork carries the change in-tree, which is strictly more
robust. Its only cost is periodic rebase on upstream `main` (same as channel 2).

### Publishing — Trusted Publishing (OIDC), the default

**No token or secret.** `.github/workflows/publish-pypi.yml` publishes via GitHub
OIDC against a PyPI pending-publisher config (owner `broomva`, repo `mlx-lm`,
workflow `publish-pypi.yml`, environment `pypi`). To cut a release:

```bash
git checkout dist/broomva
git tag pypi-v<version> && git push origin pypi-v<version>   # or Actions → publish-pypi → Run
```

Then consumers:

```bash
pip install mlx-lm-broomva
# import path stays `mlx_lm` (intentional — drop-in), so:
#   from mlx_lm import load, stream_generate
```

### Fallback — manual upload (only if not using Actions)

```bash
python -m build && python -m twine check dist/*
python -m twine upload dist/*        # needs a fresh project-scoped PyPI token
```

### Keeping the derivative current
```bash
git fetch upstream main
git checkout feat/gemma4-mtp-generate && git rebase upstream/main   # resolve any conflicts
git checkout dist/broomva && git rebase feat/gemma4-mtp-generate    # re-apply the rename on top
# bump version, rebuild, re-upload
```

## Decision log
- **2026-06-03** — fork made installable (`mtp-v0.1` tag + channel 2); PyPI rename
  prepared on `dist/broomva` (channel 3) and verified to build (`python -m build`).
- **2026-06-03** — **`mlx-lm-broomva` 0.31.3 published to PyPI** via Trusted
  Publishing (OIDC, no token). Validated: `pip install mlx-lm-broomva` in a fresh
  venv ships the MTP code and imports as `mlx_lm`. Channel 3 is live.
  https://pypi.org/project/mlx-lm-broomva/ — tracked in BRO-1350.
