---
layout: default
title: "Reaper Abliteration README"
description: "Recovered README.md from reaper-abliteration PyPI package"
---

# Reaper Abliteration README.md file

*This is the README.md file recovered from `dist-info/METADATA`.*

[![PyPI](https://img.shields.io/pypi/v/reaper-abliteration)](https://pypi.org/project/reaper-abliteration/)
[![License: PolyForm Noncommercial](https://img.shields.io/badge/License-PolyForm%20NC-purple.svg)](https://polyformproject.org/licenses/noncommercial/1.0.0/)

Remove censorship from LLMs. No training, no fine-tuning, no manual parameter selection.

Reaper finds and removes the internal "refusal subspace" of transformer models using directional ablation with automatic parameter optimization. Point it at a model, walk away, come back to an uncensored version that retains the original model's capabilities.

```bash
pip install -U reaper-abliteration
abliterate Qwen/Qwen3-4B-Instruct-2507
```

That's it. Reaper handles batch size detection, refusal direction computation, multi-objective optimization, and Pareto-optimal trial selection automatically.

## What Makes Reaper Different

Most abliteration tools remove a single direction from a few weight matrices and call it done. Reaper goes further:

**Supervised direction extraction** — Instead of naive mean-difference, Reaper can extract refusal directions via Fisher LDA (`--lda-direction`) or LEACE (`--leace-direction`, provably optimal linear concept erasure). These methods separate the refusal signal from topic shift, producing cleaner directions that ablate refusal without collateral damage. Directions are then gradient-optimized against actual model behavior with RDO (`--rdo-refine`).

**Subspace-level ablation** — Instead of removing one direction (rank-1), Reaper extracts a k-dimensional refusal subspace via SVD and removes the entire thing. Refusal isn't a single vector — it's a manifold, and rank-k ablation captures more of it. With `--som-directions`, Reaper clusters harmful activations to discover distinct refusal modes (hard refusal, safety lectures, consequence warnings) and targets each independently.

**MoE-native ablation** — Reaper handles Mixture-of-Experts architectures natively, including fused expert parameters that can't use LoRA. Hook-based ablation operates in activation space post-routing. Automatic super expert detection protects the 2-3 critical experts per layer whose removal causes model collapse, and expert-aware scaling lets the optimizer target refusal-mediating experts without damaging benign ones.

**Capability-aware ablation** — The weight-SVD stability guard identifies the most important dimensions of each weight matrix (via truncated SVD) and projects refusal directions away from them before ablating. This means Reaper actively avoids damaging what the model is good at, rather than hoping for the best.

**Sparse, targeted modification** — After computing what to change, Reaper zeros out low-impact entries, so the actual weight modification touches as few dimensions as necessary. Less collateral damage, better capability retention.

**Multi-layered refusal detection** — Keyword matching, lecture pattern detection, semantic embedding similarity, and Minos-v1 neural classifier (`--minos-refusal-detection`) work together to accurately classify model responses. The classifier overrides keyword matching when they disagree, eliminating false positives from words like "sorry" appearing in compliant responses.

**Multi-token capability measurement** — Single-token KL divergence misses a lot. Reaper measures divergence over multiple autoregressive tokens using teacher-forcing, catching capability damage that superficial metrics miss.

**Automatic everything** — Up to 1000 trials of TPE optimization with live Pareto front tracking, progressive evaluation for 2-4x speedup, seed parameter persistence across runs, multi-GPU data parallelism with proportional batch splitting for asymmetric GPU setups, 4-bit quantization support, and a real-time dashboard with clipboard copy.

## Installation

Requires Python 3.10+ with PyTorch 2.2+ configured for your hardware.

```bash
pip install -U reaper-abliteration
```

## Usage

### Minimal

```bash
abliterate <model_id>
```

Reaper will:
1. Benchmark your hardware and pick an optimal batch size
2. Compute per-layer refusal directions from harmless/harmful prompt datasets
3. Run 200 optimization trials with a live terminal dashboard
4. Present Pareto-optimal results (refusals vs KL divergence)
5. Let you save, upload to HuggingFace, or chat-test the result

### With advanced features

```bash
abliterate --ablation-rank 3 --weight-svd-guard --sparsity-threshold 0.1 \
  --norm-preserve --partial-projection --kl-tokens 3 <model>
```

### Multi-GPU

**Data parallelism** — model fits on each GPU, batches split across them (fastest):

```bash
abliterate --multi-gpu <model>
```

**Pipeline parallelism** — model too large for one GPU, layers split across devices:

```bash
CUDA_VISIBLE_DEVICES=0,1 abliterate <large_model>
```

**Asymmetric GPUs** — different VRAM sizes (e.g. 96GB + 48GB):

```bash
CUDA_VISIBLE_DEVICES=0,1 abliterate --max-memory '{"0": "90GB", "1": "42GB"}' <large_model>
```

### Reasoning models

Models with `leshooting` blocks (DeepSeek-R1, QwQ, o1-style):

```bash
abliterate --model-type reasoning deepseek-ai/DeepSeek-R1-Distill-Qwen-7B
```

### Configuration

All options work as CLI flags, environment variables (`REAPER_` prefix), or `reaper.toml`:

```toml
model = "Qwen/Qwen3-4B-Instruct-2507"
trials = 300
ablation_rank = 3
weight_svd_guard = true
norm_preserve = true
partial_projection = true
kl_tokens = 3
multi_gpu = true
```

Run `abliterate --help` for the full option list.

## Core Concepts

### The Optimization Loop

Each trial samples a set of ablation parameters (weight kernels, direction indices, layer selection thresholds), applies them to the model via LoRA adapters, then evaluates refusal count and KL divergence. Optuna's TPE sampler learns which regions of parameter space produce good tradeoffs.

LoRA adapters make trials fast — resetting between trials zeros the adapter weights and removes activation hooks instead of reloading the model.

### Refusal Directions

Reaper computes per-layer "refusal directions" from residual activations. The default mean-difference approach works well for most models. For harder targets, `--lda-direction` uses Fisher LDA to separate refusal signal from topic shift, and `--leace-direction` uses provably optimal linear concept erasure (requires `pip install reaper-abliteration[leace]`). For hybrid architectures (e.g. DeltaNet + attention), Reaper automatically extracts separate directions per layer type.

With `--use-pca-directions`, Reaper extracts additional directions via contrastive PCA — a generalized eigenvalue decomposition that finds axes maximizing harmful variance relative to harmless variance (Cholesky-whitened). This captures refusal-specific signal rather than general activation variance.

With `--ablation-rank > 1`, Reaper extracts the top-k directions via SVD of the centered difference matrix, capturing the full refusal subspace rather than just its centroid.

With `--som-directions`, Reaper clusters harmful residual deviations per-layer via KMeans (inspired by [Zhang et al. 2025](https://arxiv.org/abs/2511.08379)) to discover multiple refusal modes — keyword refusal, safety lectures, consequence warnings, crisis hotline redirects — as distinct directions. A greedy diversity selection ensures the chosen directions are maximally different from each other.

With `--iterative-refinement`, Reaper runs a two-pass pipeline: first optimizing rank-1 ablation to remove the primary refusal circuit, then re-extracting residuals through the ablated model to discover secondary refusal circuits that the mean direction misses. The stacked directions are then jointly optimized at rank-2.

With `--rdo-refine`, Reaper gradient-optimizes the refusal direction against actual model behavior before the main optimization loop. Starting from the statistical mean direction, it runs projected gradient descent on the unit sphere — minimizing refusal token probability on harmful prompts while preserving KL divergence on harmless prompts.

### Weight Modification

For each transformer component (attention out-projection, MLP down-projection), Reaper orthogonalizes the weight matrix against the refusal direction(s):

```
delta_W = -lambda * V^T (V @ W)    # rank-k projection removal
```

This is applied via LoRA adapters (`lora_A = V @ W`, `lora_B = -lambda * V^T`), so the base weights are never modified until you export.

### Progressive Evaluation

Most trials are bad. Reaper detects this early: a quick KL check on a subset of prompts prunes obviously-damaged trials before running the full (expensive) refusal evaluation. This gives 2-4x speedup with no quality loss.

## Feature Reference

### Direction Refinement

| Flag | What it does |
|------|-------------|
| `--use-pca-directions` | Contrastive PCA: extract directions maximizing harmful vs harmless variance |
| `--combine-directions` | Let optimizer weight-combine mean + PCA directions |
| `--projected-direction` | Gram-Schmidt orthogonalize refusal against harmless direction |
| `--concept-atoms` | SRA: orthogonalize against capability-probing directions (math, code, reasoning) |
| `--concept-atom-ridge F` | Ridge regularization for SRA (default: 0.1) |
| `--som-directions` | Cluster harmful residuals to find multiple refusal modes (keyword, lecture, redirect) |
| `--som-clusters N` | Number of KMeans clusters for SOM direction finding (default: 16) |
| `--iterative-refinement` | Two-pass: find secondary refusal circuits after initial rank-1 ablation |
| `--deep-ablation` | Filtered iterative refinement — re-extract only from still-refusing prompts |
| `--rdo-refine` | Gradient-optimize direction via first-token logit loss (projected GD on unit sphere) |
| `--winsorize-residuals F` | Clamp residual magnitudes at this quantile to reduce outlier influence (default: 0.0 = off) |
| `--rdo-steps N` | Number of RDO gradient steps (default: 100) |
| `--rdo-lr F` | RDO learning rate (default: 0.01) |
| `--rdo-kl-weight F` | KL penalty weight for RDO (default: 1.0) |
| `--lda-direction` | Fisher LDA supervised direction estimation |
| `--leace-direction` | LEACE provably optimal linear concept erasure (requires `[leace]` extra) |

### Ablation Control

| Flag | What it does |
|------|-------------|
| `--ablation-rank N` | LoRA rank / number of refusal directions to remove (default: 1) |
| `--weight-svd-guard` | Protect top singular vectors of each weight matrix from ablation |
| `--svd-guard-rank N` | Number of singular vectors to protect (default: 3) |
| `--sparsity-threshold F` | Zero low-magnitude ablation entries; Optuna tunes 0.01-0.5 (default: 0.0 = off) |
| `--norm-preserve` | Rescale weight rows to original L2 norms after ablation |
| `--partial-projection` | Let Optuna tune removal strength 0.1-1.0 instead of full removal |
| `--adaptive-layer-selection` | COSMIC layer scoring: skip layers with low causal refusal signal |
| `--per-head-ablation` | Target specific attention heads (optimizer picks which) |

### Evaluation

| Flag | What it does |
|------|-------------|
| `--kl-tokens N` | Multi-token KL divergence via teacher-forcing (default: 1) |
| `--semantic-refusal-detection` | Embedding similarity instead of keyword matching |
| `--lecture-refusal-detection` | Detect compliant refusals (safety lectures, consequence warnings) |
| `--minos-refusal-detection` | Minos-v1 neural refusal classifier (overrides keyword matching) |
| `--harmful-dataset-preset P` | Override harmful prompts with a preset (`mlabonne`, `strongreject`) |

### Hardware & Performance

| Flag | What it does |
|------|-------------|
| `--multi-gpu` | Data-parallel inference across GPUs (requires model fits on each GPU) |
| `--gpu-devices 0,1` | Restrict to specific GPUs |
| `--max-memory '{"0":"90GB","1":"42GB"}'` | Per-GPU VRAM budget for asymmetric setups (e.g. mixing 96GB + 48GB cards) |
| `--quantization bnb_4bit` | 4-bit quantization for large models |
| `--use-gradient-checkpointing` | Trade compute for 60-80% VRAM reduction |
| `--batch-size N` | Manual batch size (0 = auto) |

### Optimization

| Flag | What it does |
|------|-------------|
| `--trials N` | Number of optimization trials (default: 200) |
| `--warmup-trials N` | Random exploration before TPE kicks in (default: 60) |
| `--use-seed-params / --no-use-seed-params` | Load/skip Pareto-optimal seeds from prior runs |
| `--dashboard / --no-dashboard` | Live terminal dashboard |
| `--routing-analysis` | MoE expert routing diagnostic with super expert detection |

## Research Tools

Included in the base install.

**Residual plots** (`--save-plots`): PaCMAP 2D projections of per-layer residual vectors for harmful vs harmless prompts. Generates per-layer PNGs and an animated GIF.

**Residual geometry** (`--show-geometry`): Quantitative table of cosine similarities, norms, and silhouette scores between harmful/harmless residuals at each layer.

## Technical Details

### Ablation Pipeline

```
1. Extract residuals for harmful + harmless prompts
1b.[optional] Winsorize: clamp residual magnitudes at quantile threshold
2. Compute per-layer refusal directions (mean diff, LDA, or LEACE)
2b.[optional] Layer-type-aware extraction for hybrid architectures (DeltaNet + attention)
2c.[optional] SOM clustering: KMeans on harmful deviations → diverse refusal modes
3. [optional] Refine: project out harmless, orthogonalize against capability atoms
4. [optional] RDO: gradient-optimize direction via first-token logit loss
5. [optional] COSMIC layer scoring: causal cosine-similarity scores per layer
6. [optional] Iterative refinement: rank-1 pass → re-extract residuals → secondary direction
7. For each trial:
   a. Zero LoRA adapters + remove activation hooks (fast reset)
   b. Sample parameters: weight kernels, direction scope, layer thresholds
   c. For each weight matrix:
      i.   [optional] SVD guard: project directions away from top singular vectors
      ii.  Compute LoRA: lora_A = V @ W, lora_B = -lambda * V^T
      iii. [optional] Sparsity: zero entries below threshold * max(|lora_A|)
      iv.  [optional] Norm preserve: adjust lora_B so row norms match original W
      v.   Write LoRA adapter weights
   d. Evaluate: KL divergence + refusal count (progressive: KL-skip then subset scan)
   e. Record to Pareto front
8. Present Pareto-optimal trials for selection
9. Export: merge LoRA into base weights, save/upload
```

### Supported Architectures

- Dense transformers (Llama, Mistral, Qwen, Gemma, Phi, etc.)
- Mixture-of-experts (Qwen-MoE, Phi-MoE, Granite-MoE, gpt-oss style fused experts)
- Hybrid architectures (DeltaNet + attention layers, e.g. Qwen3.5)
- Multimodal models (vision-language with `AutoModelForImageTextToText`)
- Reasoning models with thinking blocks (DeepSeek-R1, QwQ)

Not yet supported: pure SSMs (Mamba, RWKV without attention layers).

## References

This project builds on research by:

- Arditi et al. 2024 — [Refusal in Language Models Is Mediated by a Single Direction](https://arxiv.org/abs/2406.11717)
- Labonne 2024 — [Uncensor any LLM with abliteration](https://huggingface.co/blog/mlabonne/abliteration)
- Lai 2024 — [Projected Abliteration](https://huggingface.co/blog/grimjim/projected-abliteration)
- Pres et al. 2025 — [Surgical Refusal Ablation](https://arxiv.org/abs/2601.08489)
- Ibrahim 2025 — [Gabliteration: Norm-Preserving Biprojected Abliteration](https://arxiv.org/abs/2512.18901)
- Li et al. 2025 — [COSMIC: Causal Layer Scoring](https://arxiv.org/abs/2506.00085)
- Chen et al. 2025 — [The Geometry of Refusal in LLMs (RDO)](https://arxiv.org/abs/2502.17420)
- Zhang et al. 2025 — [SOM-Based Multi-Directional Refusal Suppression](https://arxiv.org/abs/2511.08379)
- Belrose et al. 2023 — [LEACE: Perfect Linear Concept Erasure](https://arxiv.org/abs/2306.03819)

## License

Copyright 2025-2026 HauhauCS (hauhaut901@gmail.com)

PolyForm Noncommercial 1.0.0 — free for personal, research, and non-commercial use. Commercial use requires a separate license. See [LICENSE](LICENSE) for details. Contact hauhaut901@gmail.com for commercial licensing.
