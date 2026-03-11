# 001 — Autoresearch Overnight on RTX 5090

**Date:** March 10–11, 2026
**Type:** Autonomous AI-driven LLM training research
**Duration:** ~10 hours overnight
**Cost:** ~$4–5 GPU rental + Claude Max subscription for agent loop

## Background

[Autoresearch](https://github.com/karpathy/autoresearch) is a repo by Andrej Karpathy that gives an AI agent (Claude Code) a small but real LLM training setup and lets it experiment autonomously. The agent modifies `train.py`, trains for a fixed 5-minute window, checks if validation loss improved, keeps or discards the change, and repeats. The training code is a simplified single-GPU version of [nanochat](https://github.com/karpathy/nanochat).

The goal was to run this overnight on a consumer GPU and see what the agent discovers — both as a learning exercise in LLM training dynamics and as a first test of autonomous AI research workflows.

## Infrastructure

| Detail | Value |
|--------|-------|
| **Provider** | RunPod (Community Cloud) |
| **GPU** | NVIDIA GeForce RTX 5090 (32GB GDDR7) |
| **CUDA** | 12.9, Compute Capability 12.0 (Blackwell) |
| **Driver** | 575.57.08 |
| **Instance type** | On-demand |
| **Agent** | Claude Code CLI (Claude Max subscription, `--dangerously-skip-permissions`) |
| **Session management** | tmux |

## Setup Challenges

The RTX 5090 (Blackwell consumer architecture) required several workarounds that don't apply to the H100 reference platform:

**Flash Attention 3 incompatibility.** The precompiled FA3 kernels (`kernels-community/flash-attn3`) don't have binaries for compute capability 12.0. The error was `CUDA error: no kernel image is available for execution on the device`. Fixed by replacing the `fa3.flash_attn_func()` call on line 92 of `train.py` with `torch.nn.functional.scaled_dot_product_attention` (PyTorch's built-in SDPA). This loses sliding window attention support and is slower than FA3, but works on any GPU architecture.

**OOM at default batch size.** The default `DEVICE_BATCH_SIZE=64` was tuned for H100 80GB with FA3's memory efficiency. SDPA uses more memory. Combined with `torch.compile` (inductor) generating large intermediate buffers, the initial run OOM'd at 32GB. Fixed by dropping `DEVICE_BATCH_SIZE` to 8 and `TOTAL_BATCH_SIZE` to 2^16. The agent later scaled these back up after disabling/re-enabling compile with better settings.

**Root user restriction.** Claude Code's `--dangerously-skip-permissions` flag refuses to run as root. RunPod containers default to root. Fixed by creating a non-root `researcher` user.

**Missing tooling.** The RunPod container (PyTorch 2.x template) didn't include `uv`, Node.js, or `screen`/`tmux` for the researcher user. Each had to be installed manually. A setup script would save 20–30 minutes on future runs.

## Baseline

Before handing off to the agent, a manual baseline was established:

| Metric | Value |
|--------|-------|
| val_bpb | 1.089415 |
| Peak VRAM | 6.2 GB / 32 GB |
| MFU | 7.53% |
| Tokens processed | 94.2M |
| Training steps | 1,437 |
| Parameters | 50.3M |
| Config | depth=8, dim=512, 4 heads, DEVICE_BATCH_SIZE=8, TOTAL_BATCH_SIZE=2^16 |

The extremely low VRAM usage (6.2 / 32 GB) meant the agent had massive headroom to scale up.

## Results

The agent ran experiments overnight, systematically exploring batch sizes, model depth, learning rates, attention patterns, optimizer settings, and compilation modes. See `results.tsv` for the complete log of every experiment with keep/discard decisions.

### Best Configuration Found

| Component | Setting |
|-----------|---------|
| Model | depth=8, dim=512, 4 heads, MLP 4x, ReluSquared |
| Batch | DEVICE_BATCH_SIZE=64, TOTAL_BATCH_SIZE=2^17 |
| Attention | Full attention (no sliding window), SDPA, softcap=12 |
| Optimizer | Muon (ns_steps=6, LR=0.032, WD=0.12) + AdamW (beta1=0.7, eps=1e-10) |
| Schedule | No warmup, 65% warmdown, FINAL_LR_FRAC=0.02 |
| Embeddings | init std=0.7, VE on alternating layers, VE LR=0.5x, gate channels=16 |
| Compile | torch.compile(mode="max-autotune") |

### Best val_bpb

Started at **1.089** → improved to approximately **1.037** (and likely further by end of run).

### Biggest Wins (ranked by impact)

1. **torch.compile max-autotune** — +7% throughput, biggest single improvement
2. **Batch size scaling** (8→64, total 2^16→2^17) — 2x tokens processed in same time budget
3. **Full attention (WINDOW_PATTERN=L)** — removed sliding window overhead on SDPA
4. **Muon ns_steps=6** — better gradient orthogonalization

### Key Insight

In a fixed 5-minute time budget, throughput is king. Every experiment that increased model size or compute per step was worse, because fewer optimizer steps dominated. The winning strategy was maximizing steps while tuning hyperparameters carefully within the existing architecture. Depth increases (8→16, 8→10, 8→9) all lost because the bigger model couldn't complete enough training steps to converge.

This is a real and somewhat counterintuitive ML lesson: at small compute budgets, a well-tuned small model beats a larger model that's undertrained.

## Files in This Folder

| File | Description |
|------|-------------|
| `experiment.md` | This file — overview and context |
| `results.tsv` | Complete log of every experiment: commit hash, val_bpb, VRAM usage, keep/discard, description |
| `summary.md` | The agent's own summary of findings (generated by Claude Code at end of run) |
| `best_train.py` | The final optimized train.py with all winning changes applied |

## What to Try Next

- **Run on an H100** for direct comparison with the autoresearch leaderboard (FA3 would work there)
- **Increase time budget** beyond 5 minutes to see if deeper models become viable
- **Try the TinyStories dataset** for a more constrained domain where small models can produce readable output
- **Build a setup script** to automate the RunPod + 5090 workarounds for faster iteration
- **Run the full nanochat GPT-2 speedrun** on an 8×H100 node (separate experiment)
