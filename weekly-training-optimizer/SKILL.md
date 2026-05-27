---
name: weekly-training-optimizer
description: >
  Weekly CADAssembliesCrafter training optimization on Scaleway. SSHes into the
  Scaleway dev machine, pulls the latest WandB metrics (md, cd, iou across configs),
  backs up the current training code and config, stops the running training process,
  applies hyperparameter or architectural improvements based on metric trends, and
  launches a new training run. Use this skill whenever the user mentions "optimize
  training", "weekly training", "check training metrics", "restart training",
  "training optimizer", "tune hyperparameters on scaleway", or any variation of
  analyzing WandB metrics and adjusting an ongoing training run.
---

# Weekly Training Optimizer

Analyze WandB training metrics for CADAssembliesCrafter, decide on improvements, and restart training on Scaleway with updated configuration.

## Overview

Dimi trains a CADAssembliesCrafter model on a Scaleway GPU machine. The model uses Rectified Flow Matching with a PartCrafter DiT architecture to generate CAD assembly parts conditioned on parent images, locations, and per-part images. Training runs are tracked in WandB. This skill automates the weekly review cycle: pull metrics, diagnose what's working and what isn't, back up the current state, apply config or code changes, and kick off a fresh run.

## Machine and Project Configuration

### Scaleway Machine

| Field     | Value                        |
|-----------|------------------------------|
| SSH Host  | `scaleway`                   |
| HostName  | `51.159.148.226`             |
| Repo Path | `/root/LeoCAD`               |
| User      | `root`                       |
| GPUs      | 2x NVIDIA H100 PCIe (81GB)  |
| Conda env | `part_crafter`               |
| Python    | `/root/miniconda3/envs/part_crafter/bin/python` |

### WandB

| Field    | Value                                                           |
|----------|-----------------------------------------------------------------|
| URL      | https://wandb.ai/leoai/CADAssembliesCrafterVisualization        |
| Entity   | `leoai`                                                         |
| Project  | `CADAssembliesCrafterVisualization`                              |
| API Key  | Stored in `Desktop/.secrets/wandb_api_key`                      |

### Key Metrics

Metrics appear for multiple guidance-scale config levels (e.g., cfg3.0, cfg5.0, cfg7.0):

- **`metrics/md_cfg<X>.<Y>`** — Minimum Distance (lower is better)
- **`metrics/cd_cfg<X>.<Y>`** — Chamfer Distance (lower is better)
- **`metrics/iou_cfg<X>.<Y>`** — Intersection over Union (higher is better)
- **`metrics/md_pred_to_gt_cfg<X>.<Y>`** — One-directional MD, pred->gt
- **`metrics/md_gt_to_pred_cfg<X>.<Y>`** — One-directional MD, gt->pred

If md_pred_to_gt and md_gt_to_pred diverge, it signals the model is either missing coverage (high gt->pred) or hallucinating geometry (high pred->gt).

## Codebase Map

Understanding the code structure is essential for making meaningful changes. All paths are relative to `/root/LeoCAD`:

### Training Entry Point
- `src/CADAssembliesCrafter/main.py` — Hydra-based entry point. Launched via:
  ```bash
  PYTHONPATH="/root/LeoCAD/algo_prod:/root/LeoCAD:..." \
    /root/miniconda3/envs/part_crafter/bin/python main.py
  ```

### Hydra Config (all under `src/CADAssembliesCrafter/config/`)
- `diffusion_image_per_part_training.yaml` — Main training config (checkpoint path, name, phase)
- `optimizer/adamw.yaml` — AdamW config (lr, weight_decay, betas)
- `scheduler/cosine_warmup_transformers.yaml` — Cosine schedule (warmup steps, total steps)
- `pl_trainer/diffusion_trainer.yaml` — PL Trainer (devices, accumulate_grad_batches, precision, strategy)
- `pl_module/diffusion_image_per_part_pl_module.yaml` — Module config (condition_type, cfg_drop_prob, part_drop_prob, location_drop_prob, EMA, grad_checkpoint)
- `models/diffusion_image_per_part_model.yaml` — PartCrafter DiT model (global_attn_block_ids, cross_attn settings, multi_location_cond)
- `dataset/train_diffusion.yaml` — Train dataset augmentations (rotating_degree, crop_image_ratio, augment_pc_translation_shift, augment_pc_scale_factor_range)
- `dataset/base_dataset_diffusion.yaml` — Core dataset config (max_num_children, n_query_points, unipart_ratio, shuffle_children_pc)
- `dataset/base_dataset_mesh_assembly.yaml` — Mesh sampling (epsilon_near_points, near_points_frac, epsilon_bb, n_query_points)
- `data_module/diffusion_data_module.yaml` — DataLoader (batch_size, num_workers, persistent_workers)
- `callbacks/ema_save_callback.yaml` — EMA checkpoint saving
- `callbacks/validation_metrics_callback.yaml` — Validation with IOU/CD/MD metrics
- `callbacks/evaluation_callback.yaml` — Full eval with visualizations

### Core Python Files
- `pl_module/RectifiedFlowMatchingModule.py` (978 lines) — Training logic, loss computation, CFG dropout, condition encoding, freeze scheduling
- `models/cadac_transformer.py` — CADACTransformer with part identity embeddings, local/global attention masking
- `dataset/part_crafter_collate.py` — Custom collate for variable-length part sequences
- `callbacks/validation_metrics_cb.py` — Validation metrics computation
- `callbacks/evaluation_cb.py` — Full evaluation with rendering
- `utils/conditioner_utils.py` — Hunyuan conditioner/feature extractor wrappers

### Data Pipeline
- `src/GenericDataset/GenericDataModule.py` — PL DataModule (spawn multiprocessing)
- `src/GenericDataset/GenericComposedDataset.py` — Multi-DB dataset with retry logic

## Step-by-Step Workflow

### Step 0: Bootstrap SSH and Environment

SSH keys are stored in `Desktop/.secrets/`. Before any SSH commands, set up `~/.ssh/`:

```bash
# Find and mount secrets
SECRETS_DIR=""
for candidate in \
  "$HOME/Desktop/.secrets" \
  /sessions/*/mnt/.secrets; do
  if [ -d "$candidate" ] 2>/dev/null; then
    SECRETS_DIR="$(echo $candidate)"
    break
  fi
done

if [ -z "$SECRETS_DIR" ]; then
  echo "ERROR: .secrets directory not found. Use request_cowork_directory to mount ~/Desktop/.secrets"
  exit 1
fi

mkdir -p ~/.ssh && chmod 700 ~/.ssh
cp "$SECRETS_DIR"/scaleway_ed25519 ~/.ssh/scaleway_ed25519
chmod 600 ~/.ssh/scaleway_ed25519

# Also read the user's SSH config if available (may be mounted at /sessions/*/mnt/.ssh)
USER_SSH=""
for candidate in /sessions/*/mnt/.ssh/config; do
  if [ -f "$candidate" ] 2>/dev/null; then
    USER_SSH="$candidate"
    break
  fi
done

if [ -n "$USER_SSH" ]; then
  cp "$USER_SSH" ~/.ssh/config
  chmod 644 ~/.ssh/config
else
  # Create minimal config
  cat > ~/.ssh/config << 'SSHEOF'
Host scaleway
    HostName 51.159.148.226
    User root
    IdentityFile ~/.ssh/scaleway_ed25519
    StrictHostKeyChecking accept-new
    ConnectTimeout 15
SSHEOF
  chmod 644 ~/.ssh/config
fi

# Read WandB API key for later use
WANDB_KEY=$(cat "$SECRETS_DIR/wandb_api_key" 2>/dev/null)

# Test connection
ssh scaleway "echo 'Connected' && hostname && uptime"
```

If the secrets directory is not found, use `request_cowork_directory` to ask the user to mount `~/Desktop/.secrets` and `~/.ssh`.

### Step 1: Discover Current Training State

```bash
ssh scaleway "cd /root/LeoCAD && \
  echo '=== RUNNING TRAINING ===' && \
  ps aux | grep -E 'python.*main.py' | grep -v grep; \
  echo '=== CURRENT EPOCH ===' && \
  tail -1 training_*.log 2>/dev/null | head -c 200; \
  echo && echo '=== GIT STATUS ===' && \
  git branch --show-current && git log --oneline -5; \
  echo '=== GPU ===' && \
  nvidia-smi --query-gpu=name,memory.used,memory.total,utilization.gpu --format=csv,noheader"
```

From the output, identify:
- The **main process PID** (the one running `main.py`, not child spawns)
- The **current epoch and step** from the latest log line
- The **git branch** and any uncommitted changes
- **GPU utilization** and memory

### Step 2: Pull and Analyze WandB Metrics

```bash
ssh scaleway "WANDB_API_KEY=<key> /root/miniconda3/envs/part_crafter/bin/python3 -c \"
import wandb, pandas as pd
api = wandb.Api()
runs = api.runs('leoai/CADAssembliesCrafterVisualization', order='-created_at', per_page=3)
for run in runs:
    print(f'Run: {run.name} | State: {run.state} | Created: {run.created_at}')
    history = run.history(samples=100)
    metric_cols = [c for c in history.columns if c.startswith('metrics/')]
    if len(metric_cols) > 0:
        valid = history[metric_cols].dropna(how='all')
        n = len(valid)
        if n > 0:
            last = valid.iloc[-1]
            print('Latest metrics:')
            for col in sorted(metric_cols):
                if not pd.isna(last[col]):
                    print(f'  {col}: {last[col]:.6f}')
        if n >= 4:
            q = max(1, n // 4)
            early, late = valid.head(q).mean(), valid.tail(q).mean()
            print(f'Trend (last 25% vs first 25%, n={n}):')
            for col in sorted(metric_cols):
                if col in early.index and col in late.index:
                    if not pd.isna(early[col]) and not pd.isna(late[col]) and early[col] != 0:
                        delta = late[col] - early[col]
                        pct = delta / abs(early[col]) * 100
                        print(f'  {col}: {chr(8593) if delta > 0 else chr(8595)} {abs(delta):.6f} ({pct:+.1f}%)')
    print('---')
\""
```

Also read ALL current configs to understand the baseline:

```bash
ssh scaleway "echo '=== OPTIMIZER ===' && cat /root/LeoCAD/src/CADAssembliesCrafter/config/optimizer/adamw.yaml && \
  echo '=== SCHEDULER ===' && cat /root/LeoCAD/src/CADAssembliesCrafter/config/scheduler/cosine_warmup_transformers.yaml && \
  echo '=== TRAINER ===' && cat /root/LeoCAD/src/CADAssembliesCrafter/config/pl_trainer/diffusion_trainer.yaml && \
  echo '=== PL_MODULE ===' && cat /root/LeoCAD/src/CADAssembliesCrafter/config/pl_module/diffusion_image_per_part_pl_module.yaml && \
  echo '=== DATASET ===' && cat /root/LeoCAD/src/CADAssembliesCrafter/config/dataset/train_diffusion.yaml && \
  echo '=== BASE DATASET ===' && cat /root/LeoCAD/src/CADAssembliesCrafter/config/dataset/base_dataset_diffusion.yaml && \
  echo '=== MODEL ===' && cat /root/LeoCAD/src/CADAssembliesCrafter/config/models/diffusion_image_per_part_model.yaml && \
  echo '=== MAIN CONFIG ===' && cat /root/LeoCAD/src/CADAssembliesCrafter/config/diffusion_image_per_part_training.yaml"
```

Form a diagnosis based on the metrics and config:
- **Plateau**: Changes <1% across all metrics over the last 25% of evaluation samples.
- **Regression**: md/cd trending up, or iou trending down by >2%.
- **Coverage imbalance**: md_pred_to_gt and md_gt_to_pred diverging (model favoring one direction).
- **Healthy**: Steady improvement >1% in key metrics. Recommend continuing.

## Improvement Strategies

This section is the core intelligence of the skill. Improvements are ordered by risk level — always prefer lower-risk changes. The right change depends on diagnosing *why* metrics are not improving, not just that they are not.

### Tier 1: Config-Only Changes (Low Risk)

These only edit YAML files — no code changes. Safest to apply.

**Learning rate and schedule:**
- The cosine scheduler has `num_training_steps`. If the current global step exceeds this, the LR has bottomed out and training is effectively stalled. Fix: set `num_training_steps` to current_step + desired_remaining_steps.
- For a resume from checkpoint, the LR may be too aggressive. Try reducing `lr` by 2-5x (e.g., 1e-4 to 5e-5 or 2e-5).
- Increase `num_warmup_steps` (e.g., 500 to 1000-2000) for smoother resume.

**Data augmentation tuning:**
- `crop_image_ratio` (currently 0.5): Controls how much of the parent image is cropped. Increasing it (to 0.7) forces the model to work with less visual context; decreasing it (to 0.3) gives more context.
- `rotating_degree` (currently 30) and `rotating_ratio` (currently 0.5): Point cloud rotation augmentation. Increasing degree or ratio adds more geometric diversity.
- `augment_pc_translation_shift` (currently 0.05): Random translation jitter. Increasing (to 0.1) can improve robustness.
- `augment_pc_scale_factor_range` (currently [0.9, 1.1]): Scale augmentation. Widening (to [0.8, 1.2]) adds more scale variation.

**CFG dropout rates:**
- `cfg_drop_prob` (currently 0.1): Classifier-free guidance dropout. Increasing to 0.15-0.2 can improve CFG quality at inference.
- `part_drop_prob` (currently 0.1): Per-part dropout. Increasing teaches the model to handle missing parts.
- `location_drop_prob` (currently 0.1): Location condition dropout.

**Batch and accumulation:**
- `batch_size` (currently 8) x `accumulate_grad_batches` (currently 8) x 2 GPUs = effective 128.
- Can try reducing accumulation (8 to 4) for more frequent updates, or increasing (8 to 16) for smoother gradients.

**Dataset composition:**
- `unipart_ratio` (currently 0.3): Fraction of samples with a single part. Adjusting this changes the distribution between simple and complex assemblies.
- `max_num_children` (currently 8): Maximum parts per assembly.

**Callbacks:**
- The current config uses `ema_save_callback` only. Switch to `validation_metrics_callback` to get periodic IOU/CD/MD evaluation during training (not just at dedicated eval runs). This adds overhead but gives much better visibility into training progress.

### Tier 2: Config + Minor Code Changes (Medium Risk)

These may touch Python files in addition to YAML. Changes should be small, targeted, and isolated.

**Timestep sampling distribution:**
- In `RectifiedFlowMatchingModule.general_step()`, the `weighting_scheme` and `logit_mean`/`logit_std`/`mode_scale` are hardcoded (marked with `# TODO: make configurable`):
  ```python
  u = compute_density_for_timestep_sampling(
      weighting_scheme="logit_normal",
      batch_size=B,
      logit_mean=0.0,
      logit_std=1.0,
      mode_scale=1.29,
  )
  ```
  These control which timesteps get more training weight. Increasing `logit_std` (to 1.2-1.5) broadens the distribution; changing `mode_scale` shifts emphasis toward noisier timesteps.

**Loss weighting:**
- The loss uses `compute_loss_weighting_for_sd3("logit_normal", sigmas)`. The weighting scheme is also hardcoded. Alternative: `"sigma_sqrt"` or `"uniform"` weighting can change which noise levels contribute most to the loss.

**EMA decay:**
- `ema_config.decay` (currently 0.9999). For a plateaued model, try lowering to 0.999 to make EMA more responsive to recent updates.

**Freeze scheduling:**
- The module supports `freeze_schedule` for staged unfreezing. If certain layers (e.g., the cross-attention layers or the location embeddings) are undertrained, a freeze schedule can focus optimization on them.

### Tier 3: Meaningful Architectural / Data Pipeline Changes (Higher Risk)

These involve substantive code changes. Apply only with clear evidence and always with a rollback plan.

**Attention pattern changes:**
- `global_attn_block_ids` (currently [0,2,4,6,8,10,12,14,16,18,20] — every even layer): Controls which transformer layers use global (cross-part) attention vs local (within-part) attention. If cd is high but iou is ok, the model may need more global context for geometric coherence. Try adding more global layers or making all layers global.

**Point sampling strategy:**
- `epsilon_near_points` (currently [0.05, 0.005]): Surface proximity bands for query point sampling. Tightening the inner band (0.005 to 0.002) focuses training on fine surface detail but may hurt coarse structure.
- `near_points_frac` (currently 0.9): Ratio of near-surface vs volumetric query points. If iou is low, increasing volumetric sampling (0.9 to 0.8) may help learn interior structure.
- `n_query_points` (currently 81920): Total query points. Can reduce for faster training or increase for finer supervision.

**Conditioning pipeline changes:**
- The condition type `"parent_image"` uses both per-child image features (cond) and parent assembly image features (cond2). If md_gt_to_pred is much worse than md_pred_to_gt, the model may not be encoding parent context well enough. Consider:
  - Unfreezing the DINOv2 image encoder (currently frozen)
  - Adding more cross-attention layers between parent and child features
  - Changing `cross_attn_always_local: true` to `false` to allow cross-attention between parts

**Multi-location conditioning:**
- `multi_location_cond: true` means location info is per-touchpoint rather than per-part. If location metrics are noisy, try simplifying to single-location conditioning to reduce complexity.

## Safety Mechanisms

Code changes on a remote training machine carry real risk. These mechanisms ensure changes can always be rolled back and are verified before training starts.

### Step 3: Full Backup (MANDATORY before any changes)

```bash
ssh scaleway "cd /root/LeoCAD && \
  BACKUP_DIR=\"backups/\$(date +%Y-%m-%d_%H%M%S)\" && \
  mkdir -p \"\$BACKUP_DIR\" && \
  # Back up ALL config files (not just some)
  cp -r src/CADAssembliesCrafter/config/ \"\$BACKUP_DIR/config/\" && \
  # Back up Python files that might be modified
  cp src/CADAssembliesCrafter/pl_module/RectifiedFlowMatchingModule.py \"\$BACKUP_DIR/\" 2>/dev/null; \
  cp src/CADAssembliesCrafter/models/cadac_transformer.py \"\$BACKUP_DIR/\" 2>/dev/null; \
  cp src/CADAssembliesCrafter/dataset/part_crafter_collate.py \"\$BACKUP_DIR/\" 2>/dev/null; \
  cp src/CADAssembliesCrafter/callbacks/*.py \"\$BACKUP_DIR/\" 2>/dev/null; \
  cp src/GenericDataset/GenericDataModule.py \"\$BACKUP_DIR/\" 2>/dev/null; \
  cp src/GenericDataset/GenericComposedDataset.py \"\$BACKUP_DIR/\" 2>/dev/null; \
  # Save full git state
  git diff > \"\$BACKUP_DIR/uncommitted.patch\" 2>/dev/null; \
  git diff --staged > \"\$BACKUP_DIR/staged.patch\" 2>/dev/null; \
  git log --oneline -20 > \"\$BACKUP_DIR/git_log.txt\"; \
  git rev-parse HEAD > \"\$BACKUP_DIR/git_head.txt\"; \
  echo \"Backup at: \$BACKUP_DIR\" && ls -la \"\$BACKUP_DIR/\""
```

### Step 4: Stop Current Training (if running)

```bash
# Find the main process (the bash wrapper), not child processes
ssh scaleway "ps aux | grep 'python main.py' | grep -v grep"
```

Kill with SIGTERM first (allows checkpoint save), verify, then SIGKILL if needed:

```bash
ssh scaleway "kill -SIGTERM <PID> && sleep 10 && \
  (ps -p <PID> > /dev/null 2>&1 && kill -9 <PID> || echo 'Process stopped cleanly')"
```

Wait for GPU memory to fully release:
```bash
ssh scaleway "sleep 5 && nvidia-smi --query-gpu=memory.used --format=csv,noheader"
```

### Step 5: Apply Changes with Verification

**For YAML config changes**, use Hydra-aware Python editing rather than sed (which can break YAML):

```bash
ssh scaleway "cd /root/LeoCAD && /root/miniconda3/envs/part_crafter/bin/python3 -c \"
import yaml, sys

# Read the config
config_path = 'src/CADAssembliesCrafter/config/<specific_file>.yaml'
with open(config_path) as f:
    original = f.read()

cfg = yaml.safe_load(original)

# Apply specific changes (example: update lr)
cfg['lr'] = 5e-5

# Write back
with open(config_path, 'w') as f:
    yaml.dump(cfg, f, default_flow_style=False, sort_keys=False)

# Verify by reading back
with open(config_path) as f:
    updated = yaml.safe_load(f)

# Print diff
print('Changes applied:')
for k in set(list(cfg.keys())):
    old_val = yaml.safe_load(original).get(k)
    new_val = updated.get(k)
    if old_val != new_val:
        print(f'  {k}: {old_val} -> {new_val}')
\""
```

**For Python code changes**, follow this strict protocol:

1. **Read the current file** to verify the exact lines being changed:
   ```bash
   ssh scaleway "sed -n '<start>,<end>p' /root/LeoCAD/<path>"
   ```

2. **Apply a targeted sed or Python-based patch** — never rewrite entire files:
   ```bash
   ssh scaleway "sed -i 's/exact_old_line/exact_new_line/' /root/LeoCAD/<path>"
   ```

3. **Verify the change** by reading back the modified lines:
   ```bash
   ssh scaleway "sed -n '<start>,<end>p' /root/LeoCAD/<path>"
   ```

4. **Run a syntax check** to catch broken Python:
   ```bash
   ssh scaleway "/root/miniconda3/envs/part_crafter/bin/python3 -c 'import py_compile; py_compile.compile(\"/root/LeoCAD/<path>\", doraise=True)'"
   ```

5. **Run an import test** to verify the module loads without errors:
   ```bash
   ssh scaleway "cd /root/LeoCAD && PYTHONPATH='/root/LeoCAD/algo_prod:/root/LeoCAD:/root/LeoCAD/src/CADAssembliesCrafter/PartCrafter:/root/LeoCAD/src/CADAssembliesCrafter/Hunyuan3D_2_1:/root/LeoCAD/src/CADAssembliesCrafter/Hunyuan3D_2_1/hy3dshape:/root/LeoCAD/src/CADAssembliesCrafter/Hunyuan3D_Part:/root/LeoCAD/src/CADAssembliesCrafter/Hunyuan3D_Part/XPart:/root/LeoCAD/src/CADAssembliesCrafter/UltraShape' /root/miniconda3/envs/part_crafter/bin/python3 -c 'from src.CADAssembliesCrafter.pl_module.RectifiedFlowMatchingModule import RectifiedFlowMatchingModule; print(\"Import OK\")'"
   ```

If any verification step fails, **immediately restore from backup**:
```bash
ssh scaleway "cd /root/LeoCAD && cp backups/<timestamp>/<file> <original_path>"
```

### Step 6: Start New Training Run

The training is launched with a complex PYTHONPATH. Use the exact same launch pattern found in the process listing from Step 1:

```bash
ssh scaleway "cd /root/LeoCAD/src/CADAssembliesCrafter && \
  export PYTHONPATH='/root/LeoCAD/algo_prod:/root/LeoCAD:/root/LeoCAD/src/CADAssembliesCrafter/PartCrafter:/root/LeoCAD/src/CADAssembliesCrafter/Hunyuan3D_2_1:/root/LeoCAD/src/CADAssembliesCrafter/Hunyuan3D_2_1/hy3dshape:/root/LeoCAD/src/CADAssembliesCrafter/Hunyuan3D_Part:/root/LeoCAD/src/CADAssembliesCrafter/Hunyuan3D_Part/XPart:/root/LeoCAD/src/CADAssembliesCrafter/UltraShape' && \
  nohup /root/miniconda3/envs/part_crafter/bin/python main.py \
  2>&1 | tee /root/LeoCAD/training_\$(date +%Y-%m-%d_%H-%M-%S).log &
  echo \"PID: \$!\""
```

**Post-launch verification** (critical — do not skip):

```bash
# Wait for training to initialize
ssh scaleway "sleep 30 && \
  echo '=== PROCESS ===' && \
  ps aux | grep 'python main.py' | grep -v grep && \
  echo '=== GPU ===' && \
  nvidia-smi --query-gpu=memory.used,utilization.gpu --format=csv,noheader && \
  echo '=== LOG TAIL ===' && \
  tail -20 /root/LeoCAD/training_*.log 2>/dev/null | tail -20"
```

Look for:
- Process is running (not crashed)
- GPU memory is being used (model loaded)
- Log shows training steps progressing (not stuck on imports or errors)
- No CUDA OOM errors, no `RuntimeError`, no `KeyError`

If training crashes within the first 30 seconds, **restore from backup and report the error**. Do not retry automatically.

### Step 7: Generate Summary Report

Save a detailed report as a markdown file. Include:

- Previous run name, duration, final epoch/step, final metrics
- Diagnosis (plateau / regression / healthy / coverage imbalance) with reasoning
- Each change applied with before/after values, tier level, and risk assessment
- New run PID, start time, checkpoint resumed from
- Backup location and list of backed up files
- Link to WandB dashboard: https://wandb.ai/leoai/CADAssembliesCrafterVisualization

If running as a scheduled task, save to the outputs directory and post to Slack if available.

## Decision Rules for Scheduled (Unattended) Runs

When running without user interaction, be conservative:

1. **Only apply Tier 1 changes** (config-only). Never modify Python code unattended.
2. **Only act on clear signals**: plateau (>20 eval samples, <1% change) or regression (>2% degradation).
3. **If metrics are ambiguous or healthy**: generate a report but do not stop training or change anything.
4. **Always verify** the new run starts successfully before considering the task done.
5. **If anything goes wrong** (SSH fails, backup fails, launch fails): stop immediately, restore backups if needed, and report the failure.

## Edge Cases

- **SSH connection fails**: Verify Step 0 completed. If secrets not found, use `request_cowork_directory`. If keys exist but connection fails, report and stop.
- **Multiple training processes**: List all, identify the main one (running `main.py`), report all PIDs. In unattended mode, do not kill — just report.
- **GPU memory not released**: After killing training, if GPU memory is still high, try `nvidia-smi --gpu-reset` or report that a manual reboot may be needed.
- **Hydra output directory conflict**: Each run creates a new output dir with a timestamp. If the old dir still has a lock file, the new run may fail. Check and clean up if needed.
- **Uncommitted git changes**: Note them in the report but never commit, stash, or discard them.
- **Checkpoint not found**: If the checkpoint path in the config does not exist (e.g., it was moved), the run will crash. Verify the checkpoint exists before launching.
