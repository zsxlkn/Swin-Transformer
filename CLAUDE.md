# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment Setup

```bash
conda create -n swin python=3.7 -y && conda activate swin
conda install pytorch==1.8.0 torchvision==0.9.0 cudatoolkit=10.2 -c pytorch
pip install timm==0.4.12
pip install opencv-python==4.4.0.46 termcolor==1.1.0 yacs==0.1.8 pyyaml scipy
```

Optional CUDA-fused window process (enables `--fused_window_process` flag):
```bash
cd kernels/window_process && python setup.py install
```

Swin-MoE additionally requires [Tutel](https://github.com/microsoft/tutel):
```bash
python3 -m pip install --user --upgrade git+https://github.com/microsoft/tutel@main
```

## Common Commands

**Evaluate a checkpoint:**
```bash
python -m torch.distributed.launch --nproc_per_node <gpus> --master_port 12345 main.py \
  --eval --cfg <config-file> --resume <checkpoint> --data-path <imagenet-path>
```

**Train from scratch (e.g., Swin-T with 8 GPUs):**
```bash
python -m torch.distributed.launch --nproc_per_node 8 --master_port 12345 main.py \
  --cfg configs/swin/swin_tiny_patch4_window7_224.yaml --data-path <imagenet-path> --batch-size 128
```

**Override any config key at the command line:**
```bash
--opts TRAIN.EPOCHS 100 TRAIN.WARMUP_EPOCHS 5
```

**Memory-saving flags:** `--accumulation-steps <N>` (gradient accumulation), `--use-checkpoint` (~60% memory saving for large models).

**Measure throughput:**
```bash
python -m torch.distributed.launch --nproc_per_node 1 --master_port 12345 main.py \
  --cfg <config> --data-path <imagenet-path> --batch-size 64 --throughput --disable_amp
```

SimMIM pre-training and fine-tuning use `main_simmim_pt.py` / `main_simmim_ft.py` with configs under `configs/simmim/`. Swin-MoE uses `main_moe.py` with configs under `configs/swinmoe/`.

## Architecture

### Config system (`config.py`)
All hyperparameters live in a `yacs` `CfgNode` (`_C`). YAML files under `configs/` are loaded via `--cfg`; individual keys can be overridden with `--opts`. The config is frozen after construction and passed everywhere as `config`. Per-model sections are `config.MODEL.SWIN`, `config.MODEL.SWINV2`, `config.MODEL.SWIN_MOE`, `config.MODEL.SWIN_MLP`, `config.MODEL.SIMMIM`.

### Model dispatch (`models/build.py`)
`build_model(config, is_pretrain=False)` selects the class from `config.MODEL.TYPE`:
- `"swin"` → `SwinTransformer` (`models/swin_transformer.py`)
- `"swinv2"` → `SwinTransformerV2` (`models/swin_transformer_v2.py`)
- `"swin_moe"` → `SwinTransformerMoE` (`models/swin_transformer_moe.py`)
- `"swin_mlp"` → `SwinMLP` (`models/swin_mlp.py`)
- `is_pretrain=True` → `build_simmim(config)` (`models/simmim.py`)

### Core model (`models/swin_transformer.py`)
Key classes in execution order:
1. `PatchEmbed` — splits image into non-overlapping 4×4 patches and linearly embeds them.
2. `SwinTransformerBlock` — one transformer block with either W-MSA (no shift) or SW-MSA (cyclic shift by `window_size // 2`). Interleaved blocks alternate between the two.
3. `WindowAttention` — multi-head self-attention within each local window; adds a learned relative position bias table indexed by `relative_position_index`.
4. `BasicLayer` — stacks N `SwinTransformerBlock`s and optionally applies `PatchMerging` to downsample 2×.
5. `SwinTransformer` — 4-stage hierarchy; embed dims double at each stage (96→192→384→768 for Swin-T/S/B).

The cyclic-shift + attention-mask trick (`attn_mask` computed in `__init__`, registered as a buffer) is how SW-MSA avoids cross-region attention without extra padding. An optional fused CUDA kernel (`kernels/window_process/`) replaces `torch.roll` + `window_partition` with a single op.

### Swin V2 differences (`models/swin_transformer_v2.py`)
Uses scaled cosine attention (`q·k / (‖q‖‖k‖ · τ)`) instead of dot-product, and a log-spaced continuous relative position bias (`CPB_MLP`) instead of a fixed bias table — this allows transferring weights across window sizes and resolutions via `pretrained_window_sizes`.

### Data pipeline (`data/build.py`)
`build_loader` returns train/val `DataLoader`s. Dataset class is chosen by `config.DATA.DATASET` (`imagenet`, `imagenet22K`). Supports folder datasets, zipped datasets (`ZipReader`), and in-memory caching. SimMIM-specific loaders (`data_simmim_pt.py`, `data_simmim_ft.py`) add mask generation on top of the standard augmentation pipeline.

### Checkpoint handling (`utils.py`)
- `load_checkpoint` — resumes full training state (model + optimizer + scheduler + scaler).
- `load_pretrained` — loads only model weights, automatically removes buffers that are always re-initialized (`relative_position_index`, `attn_mask`) and interpolates position embeddings when image/window sizes differ between pre-training and fine-tuning.
