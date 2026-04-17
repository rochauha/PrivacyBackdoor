# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Research code for a *privacy backdoor* attack: manipulate the weights of a pre-trained model (MLP, ViT, BERT) so that after downstream fine-tuning, an attacker can reconstruct individual training examples (images or sentences) from the fine-tuned weights alone. There is also a variant targeting DP-SGD black-box training.

The codebase is split into two halves:
- `src/` — malicious initialization + fine-tuning (produces a fine-tuned `.pth`).
- `analysis/` — consume those `.pth` files to reconstruct training data and measure reconstruction quality (PSNR/SSIM), plus plot gradient concentration for the DP variant.

## Running experiments

All runs go through a single entrypoint driven by a YAML config:

```bash
python src/main.py --mode <MODE> --config_name <CONFIG_NAME>
```

`--mode` selects the pipeline, which then dispatches to a `build_*` function in `run_*.py`:
- `mlpvn` → `run_mlp.build_mlp_model` (toy MLP backdoor)
- `vibkd` → `run_vit.build_vision_transformer` (ViT backdoor; also used for plain fine-tuning)
- `txbkd` → `run_text_classification.build_bert_classifier` (BERT backdoor)
- `stdtr` → `run_dpprv.build_public_model` (standard MLP pre-train, typically a precursor to `dpbkd`)
- `dpbkd` → `run_dpprv.build_dp_model` (DP-SGD black-box attack)

`--config_name` resolves to `./experiments/configs/{config_name}.yml`. **The `experiments/` tree is gitignored** — reference templates live in `materials/configs/` (e.g. `vit_gelu_randhead_caltech_splice.yml`, `bert_relu_randhead_trec50.yml`, `imagenet_small_gelu.yml`, `mlp_epsilon3.yml`). Copy or symlink them into `experiments/configs/` before invoking. `main.py` also *renames the config file in place* after the run, appending a timestamp — running the same config twice requires restoring it.

Typical chained workflow for ViT: first run `vibkd` with `imagenet_small_gelu` to produce a small benign pre-trained checkpoint, then run `vibkd` again with `vit_gelu_randhead_caltech_splice` pointing `MODEL.PATH` at that checkpoint to perform the actual data-stealing attack.

## Analysis commands

After a fine-tuning run writes `./weights/GROUP/NAME.pth`:

```bash
# Reconstructed images vs. ground truth (MLP or ViT)
python analysis/reconstruct_images.py --path ./weights/GROUP/NAME.pth --plot_mode recovery --arch vit --hw 4 8 --inches 4.35 2.15 --scaling 0.229 0.224 0.225 --bias 0.485 0.456 0.406
python analysis/reconstruct_images.py --path ./weights/GROUP/NAME.pth --plot_mode raw      --arch vit --hw 4 8 --inches 4.35 2.15

# Reconstructed sentences (BERT) — note the *_monitor.pth suffix written alongside the main checkpoint
python analysis/analyze_reconstruct_sentences.py --path ./weights/GROUP/NAME_monitor.pth --save_path ./text --verbose True

# DP gradient distribution
python analysis/analyze_diffprv.py --path ./weights/GROUP/NAME_rgs_ex0.pth --save_path ./pic --biconcentration True

# Reconstruction quality (PSNR/SSIM)
python analysis/quality.py --path ./weights/GROUP/NAME.pth --hw 4 8 --step 4
```

Pass `--scaling`/`--bias` only when the dataset was normalized (ImageNet stats: `0.229 0.224 0.225` / `0.485 0.456 0.406`).

There is no test suite, linter config, or build step — this is a research repo.

## Architecture

### The attack in one paragraph

A pre-trained model is surgically rewritten so a chosen set of neurons act as *baits*: during fine-tuning, one bait fires for (ideally) one training example and absorbs that example's signal into its weights. Comparing the bait's weights before vs. after fine-tuning lets the attacker reconstruct the input. The `src/` side builds and fine-tunes the malicious model; the `analysis/` side diffs weights and renders the reconstructions.

### Control flow

`main.py` → `run_*.py` (per-mode builder) → optional `edit_*.py` malicious init → `train.py::train_model` → save checkpoint. `run_*.py` files are thin config-to-call glue; the substance lives in `edit_vit.py`, `edit_bert.py`, and `model_mlp.py`.

### Key modules

- **`src/edit_vit.py`** (1480 lines) — `ViTWrapper` around torchvision's `vit_b_32`. Owns `backdoor_initialize`, `semi_activate_initialize`, `small_model`, records the pre-fine-tuning weights (`save_init_model=True`) so reconstruction later has a reference, and exposes `module_parameters('encoder'|'heads')` so the optimizer can use different LRs for probe vs. encoder (see `run_vit.py`). Also handles the "splice" variant (image stitching).
- **`src/edit_bert.py`** (995 lines) — analogous wrapper over HuggingFace BERT. Produces a `monitor` object saved to `*_monitor.pth` that `analyze_reconstruct_sentences.py` consumes; that file, not the main checkpoint, holds the reconstruction signal.
- **`src/model_mlp.py`** — all MLP variants, including the reconstruction-bait MLP and the gradient-concentration MLP used for the DP attack.
- **`src/model_vnlla.py`, `src/run_vnlla.py`** — vanilla (non-malicious) baselines. Per `src/README.md`, files not listed there ("Unlisted files are not used in the final version") are exploratory / legacy; prefer the listed entrypoints.
- **`src/data.py`** — dataset loading; exposes `INLAID`, `RESIZE`, `SUBSET`, `IS_NORMALIZE`, `IS_AUGMENT` knobs consumed from `config['DATASET']`.
- **`src/train.py`** — single `train_model` loop used by every mode. `src/README.md` explicitly notes it is "redundant on purpose" to keep each mode's path simple; do not try to unify branches without reading the individual callers.
- **`src/tools.py`** — grab-bag; per `src/README.md` contains functions retained from earlier iterations that the final pipeline no longer uses.

### Config shape

YAML configs have top-level `DATASET`, `MODEL`, `TRAIN`, `SAVE_PATH`, plus `TARGET` for `dpbkd`. Inside `MODEL`, `USE_BACKDOOR_INITIALIZATION: true` activates the attack path and requires `WEIGHT_SETTING`, `BAIT_SETTING`, `REGISTRAR`, `NUM_BACKDOORS`, `ARCH.hidden_act`; setting it false (and optionally `USE_SEMI_ACTIVE_INITIALIZATION` / `USE_SMALL_MODEL`) reuses the same pipeline for benign pre-training. See `materials/configs/*.yml` for full shapes.

### Known fragility

Per `README.md`: random-head pre-trained models can *break down* during fine-tuning. Re-running usually recovers. If a run produces degenerate reconstructions, rerun before debugging the code.
