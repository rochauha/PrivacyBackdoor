# SETUP

Environment setup for the PrivacyBackdoor repo. The repo has no dependency manifest (`requirements.txt`, `pyproject.toml`, etc.) and no pinned versions, so the environment must be assembled manually.

## Requirements

- Python 3.9–3.13. On **Python 3.13 you must use the cu124 (or newer) PyTorch index** — see the "Install PyTorch" section below. The cu121 / cu118 indices have no cp313 `torchvision` wheel, and pip will silently fall back to a broken 2017-era sdist (`torchvision==0.2.0` or `0.1.6`) that predates `vit_b_32` and will break every ViT/BERT path.
- A GPU is recommended for ViT/BERT modes (`vibkd`, `txbkd`). The toy MLP mode (`mlpvn`) runs on CPU.
- No API keys or external services are needed. HuggingFace and torchvision will auto-download model weights / datasets on first use (network access required once).

Note: `src/train.py` and `src/data.py` unconditionally import `opacus`, `transformers`, and `datasets`, so these packages are required even if you only plan to run `mlpvn`.

## Create the virtual environment

```bash
cd /path/to/PrivacyBackdoor

python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
```

## Install PyTorch

Check your driver's CUDA version with `nvidia-smi` (top-right of output) and pick the matching wheel index. The PyTorch wheels are backward-compatible within a major CUDA version, so e.g. a driver reporting CUDA 12.6 can use cu124 wheels.

```bash
# CUDA 12.4 (recommended — has cp313 wheels; works on drivers >= 12.4)
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu124

# CUDA 12.1 (only for Python <= 3.12 — cu121 has no cp313 torchvision wheel)
# pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121

# CUDA 11.8
# pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118

# CPU only
# pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
```

See https://pytorch.org/get-started/locally/ for other CUDA wheel tags.

### Verify torch/torchvision were installed correctly

```bash
python -c "import torch, torchvision; from torchvision.models import vit_b_32, ViT_B_32_Weights; print('torch', torch.__version__, 'torchvision', torchvision.__version__, 'cuda', torch.cuda.is_available()); print('vit_b_32 OK')"
```

If you see `torchvision 0.2.0` or `0.1.6`, or an `ImportError: cannot import name 'vit_b_32'`, pip resolved a legacy sdist from PyPI — your Python version has no wheel at the chosen index. Reinstall against the cu124 index (or downgrade Python to 3.12):

```bash
pip uninstall -y torch torchvision
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu124
```

## Install the remaining dependencies

```bash
pip install \
    transformers \
    datasets \
    opacus \
    pyyaml \
    numpy \
    scipy \
    matplotlib \
    scikit-image
```

## Sanity check

```bash
python -c "import torch, torchvision, transformers, datasets, opacus, yaml, numpy, scipy, matplotlib, skimage; print('torch', torch.__version__, 'cuda', torch.cuda.is_available())"
```

## Directory layout before first run

`main.py` reads configs from `./experiments/configs/` (gitignored) and writes logs to `./experiments/logs/`. Create those and the weights output directory:

```bash
mkdir -p experiments/configs experiments/logs weights
```

Reference config templates live in `materials/configs/` — copy or symlink the one you want into `experiments/configs/` before invoking `main.py`. Note that `main.py` renames the config file in place after each run (appending a timestamp), so keep a copy if you plan to rerun.

## Caveats

- The repo has no pinned versions, so you will get the latest releases of every library. If you hit errors such as a missing `ViT_B_32_Weights` symbol, BERT tokenizer API changes, or Opacus signature mismatches, pin older versions as a first debugging step. A known-working baseline to try:
  - `torch==2.6.*` + `torchvision==0.21.*` (Python 3.13, cu124), or `torch==2.1.*` + `torchvision==0.16.*` (Python ≤ 3.12)
  - `transformers==4.35.*`
  - `opacus==1.4.*`
- If pip reports only `torchvision` versions `0.1.6` / `0.2.0` as available, your Python version has no wheel at the index URL you chose. Switch to cu124 (or cpu), or downgrade to Python 3.12.
- Analysis scripts under `analysis/` use `from src.tools import ...`. Run them from the repo root. If you hit `ModuleNotFoundError: src`, create an empty `src/__init__.py`.
- Per the main `README.md`, random-head pre-trained models can break down during fine-tuning. Re-running usually recovers — do not assume a degenerate result is a code bug until you have rerun at least once.
