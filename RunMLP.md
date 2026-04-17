# Running the MLP Reconstruction Experiment (`mlpvn`)

End-to-end walkthrough for the toy-MLP privacy backdoor: malicious init → fine-tune on CIFAR-10 → reconstruct training images from the checkpoint.

## Prerequisites

- A configured virtual environment with all dependencies installed. See `SETUP.md` if you have not done this yet.
- The repo-provided config template at `materials/configs/mlp_recon_cifar10.yml`.

## Config template

The template `materials/configs/mlp_recon_cifar10.yml` targets CIFAR-10 with a 2-hidden-layer MLP and 32 baits:

```yaml
DATASET:
  NAME: 'cifar10'
  ROOT: './data'
  IS_NORMALIZE: False
  SUBSET: null


MODEL:
  HIDDEN_SIZE: [64, 32]
  ACTIVATION: 'ReLU'
  USE_BACKDOOR: True
  NUM_BACKDOORS: 32
  PREPROCESS: null

  BAIT_SETTING:
    NUM_TRIALS: 4096
    APPROACH: 'gaussian'
    MULTIPLIER: 1.0
    QUANTILE: 0.9998
    DETAILS:
      is_normalize: True
    SELECTION_DICT:
      largest_correlation: 0.3

  WEIGHT_SETTING:
    INTERMEDIATE:
      multiplier: 1.0
      noise_threshold: 0.0
    OUTPUT:
      approach: 'random_connect'
      multiplier: 1.0


TRAIN:
  BATCH_SIZE: 64
  LR: 0.01
  EPOCHS: 5
  DEVICE: 'cuda'
  NUM_WORKERS: 2


SAVE_PATH: './weights/mlp_recon_cifar10.pth'
```

## Usage

```bash
# 0. Activate venv
source .venv/bin/activate

# 1. CIFAR-10 must be pre-downloaded — src/data.py calls CIFAR10(..., download=False)
python -c "import torchvision.datasets as d; d.CIFAR10('./data', train=True, download=True); d.CIFAR10('./data', train=False, download=True)"

# 2. Copy the template into experiments/configs/ (main.py reads from there)
mkdir -p experiments/configs experiments/logs weights
cp materials/configs/mlp_recon_cifar10.yml experiments/configs/mlp_recon_cifar10.yml

# 3. Run the attack (malicious init + fine-tune). Writes ./weights/mlp_recon_cifar10.pth
python src/main.py --mode mlpvn --config_name mlp_recon_cifar10

# 4. Reconstruct images from the checkpoint (4 rows x 8 cols = 32 baits)
python analysis/reconstruct_images.py \
    --path ./weights/mlp_recon_cifar10.pth \
    --arch toy \
    --plot_mode recovery \
    --chw 3 32 32 \
    --hw 4 8 \
    --inches 4.35 2.15 \
    --save_path ./mlp_recon.png

# Optionally, show the raw activating images instead of reconstructions:
# python analysis/reconstruct_images.py --path ./weights/mlp_recon_cifar10.pth \
#     --arch toy --plot_mode raw --chw 3 32 32 --hw 4 8 --inches 4.35 2.15 \
#     --save_path ./mlp_raw.png
```

## Template notes

- `HIDDEN_SIZE: [64, 32]` — the first-layer width must be ≥ `NUM_BACKDOORS` (32), since baits occupy the first 32 neurons. Both entries required (there's an `assert len(hidden_size) == 2` in `NativeMLP`).
- `IS_NORMALIZE: False` — keeps raw `[0,1]` pixels, so the analysis script needs no `--scaling` / `--bias`. Flip to `True` and pass the CIFAR-10 (or ImageNet) stats if you prefer normalized inputs.
- `QUANTILE: 0.9998` — bait fires for roughly the top 0.02% of training inputs (~10 samples out of 50k). Higher → fewer activations per bait (cleaner reconstruction but more chance of zero activations). Tune if reconstructions look muddy.
- `SELECTION_DICT.largest_correlation: 0.3` — forces selected baits to be roughly orthogonal so different baits latch onto different images. Remove it to fall back to raw candidates.
- `OUTPUT.approach: 'random_connect'` — wires each bait into a random output class. Alternatives in `model_mlp.py:72-115`: `'wrong_class'`, `'random_gaussian'`, or anything else to leave the output layer untouched.

## Reruns and fragility

- `main.py` renames the config file in place after each run (appending a timestamp), so the original copy in `experiments/configs/` will be gone. The reference lives in `materials/configs/` — recopy it to rerun.
- If reconstructions come out degenerate on the first try, rerun before debugging. Per the main `README.md`, random-head pre-trained models can break down during fine-tuning, and re-running usually recovers.

## Analysis script flags quick reference

| Flag            | Meaning                                                                                     |
|-----------------|---------------------------------------------------------------------------------------------|
| `--arch`        | `toy` for MLP, `vit` for ViT. (MLP uses `toy`, not `mlp`.)                                  |
| `--plot_mode`   | `recovery` (reconstructed), `raw` (activating training images), `single` (per-bait dump).   |
| `--chw`         | Channels, height, width of the image — required for `--arch toy` in `recovery` mode.         |
| `--hw`          | Grid rows × cols for the plot (e.g. `4 8` for 32 baits).                                    |
| `--inches`      | Matplotlib figure size.                                                                      |
| `--scaling`     | Per-channel std for denormalizing; pass only if `DATASET.IS_NORMALIZE: True`.               |
| `--bias`        | Per-channel mean for denormalizing; pass only if `DATASET.IS_NORMALIZE: True`.              |
| `--save_path`   | Output image path. Omit to display interactively.                                           |
