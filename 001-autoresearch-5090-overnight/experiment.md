# 001 — Autoresearch Overnight on RTX 5090

**Date:** March 10–11, 2026
**Type:** Autonomous AI-driven LLM training research
**Duration:** ~10 hours overnight
**Cost:** ~$4–5 RunPod GPU rental + Claude Max subscription for agent loop

## Background

[Autoresearch](https://github.com/karpathy/autoresearch) is a repo by Andrej Karpathy that gives an AI agent a small but real LLM training setup and lets it experiment autonomously. The agent modifies `train.py`, trains for a fixed 5-minute window, checks if validation loss improved, keeps or discards the change, and repeats. The training code is a simplified single-GPU version of [nanochat](https://github.com/karpathy/nanochat).

This was a first-time run — both first time using cloud GPUs and first time running autonomous AI research. The goal was to learn the workflow and see what the agent could discover overnight.

## Infrastructure

| Detail | Value |
|--------|-------|
| **Provider** | RunPod (Community Cloud, on-demand) |
| **GPU** | NVIDIA GeForce RTX 5090 (32GB GDDR7) |
| **Compute capability** | 12.0 (Blackwell) |
| **CUDA / Driver** | 12.9 / 575.57.08 |
| **Container** | RunPod PyTorch 2.x template, 50GB disk |
| **Agent** | Claude Code CLI v2.x (Max subscription, `--dangerously-skip-permissions`) |
| **Session** | tmux (for SSH disconnect resilience) |

## Setup Challenges

Running autoresearch on an RTX 5090 required several workarounds not documented in the repo (which targets H100):

**Flash Attention 3 incompatibility.** The precompiled FA3 kernels (`kernels-community/flash-attn3`) have no binaries for compute capability 12.0. Error: `CUDA error: no kernel image is available for execution on the device`. Fixed by replacing the `fa3.flash_attn_func()` call on line 92 of `train.py` with `torch.nn.functional.scaled_dot_product_attention`. This loses sliding window support and is slower than FA3, but works on any architecture.

**OOM at default settings.** The H100-tuned defaults (`DEVICE_BATCH_SIZE=64`) immediately OOM'd because SDPA uses more memory than FA3, and `torch.compile` (inductor) adds large intermediate buffers. Initial fix: drop `DEVICE_BATCH_SIZE` to 8, `TOTAL_BATCH_SIZE` to 2^16, disable `torch.compile`. The agent later scaled these back up after finding the right combination.

**Root user restriction.** Claude Code's `--dangerously-skip-permissions` refuses to run as root. RunPod defaults to root. Fixed by creating a non-root `researcher` user and reinstalling all tooling (`uv`, Node.js, Claude Code) under that user.

**Missing tooling.** The RunPod container required manual installation of `uv`, Node.js (via nvm), Claude Code, and `tmux`. A setup script would save 20–30 minutes on future runs.

## Results Summary

**147 experiments** over ~10 hours. Val_bpb improved from **1.089** (baseline) to **1.028** (best), a **5.6% reduction**. 19 changes were kept, 127 discarded, 1 crashed. See `summary.md` for the full experiment log, progression of kept improvements, detailed analysis of what worked and what didn't, and the final best configuration.

## What I Learned

**This experiment didn't produce a usable LLM.** It produced an optimized *training recipe* — a `best_train.py` tuned specifically for the RTX 5090 without Flash Attention 3. That recipe doesn't exist anywhere else. If I (or anyone with a 5090) wanted to train a nanochat model, this file is the fastest path to a good result on this hardware.

**Where this sits in the LLM creation pipeline:** Autoresearch optimizes step 1 (find the best training config). The nanochat speedrun is step 2 (actually train a model using that config). Then SFT and RL turn the base model into something you can chat with. Autoresearch finds the recipe; you still have to bake the bread.

**Baking analogy that made it click:** The original autoresearch repo is a cake recipe written for a 9" round pan (H100). My 5090 is a 6"x6" square pan — different shape, different material, different heat distribution. The fundamentals are the same (flour, sugar, eggs / gradients, attention, backpropagation), but the specifics that make it come out right are different. Some things transferred fine (core architecture stayed at depth 8), but the temperatures and timings all changed (batch sizes, learning rates, compilation settings). One key ingredient didn't even work in the new pan (Flash Attention 3), so the agent found a substitute (SDPA) and re-optimized everything around it. And autoresearch didn't adapt the recipe once — it baked 147 cakes overnight, tweaking one thing each time, tasting the result, keeping or tossing the change.

**The counterintuitive finding:** When you only have 5 minutes to train, a small model trained well beats a big model trained poorly. The agent tried going bigger multiple times and it always lost, because fewer training steps mattered more than model capacity. This is a real ML principle (compute-optimal scaling / "Chinchilla" research) that the agent independently rediscovered.

**Most ideas don't work, and that's normal.** 127 of 147 experiments were discarded. Only 13% improved things. The value isn't any single experiment — it's systematic elimination of bad ideas and accumulation of small wins. The architecture never changed; all gains came from "boring" stuff like optimizer tuning and batch sizing.

## Files

| File | Description |
|------|-------------|
| `experiment.md` | This file — context, infrastructure, and setup notes |
| `summary.md` | The agent's detailed technical report: all 147 experiments, best config, key insights, what worked/didn't, full experiment log |
| `results.tsv` | Machine-readable log: commit hash, val_bpb, VRAM, keep/discard, description for every experiment |
| `best_train.py` | The final optimized `train.py` with all winning changes applied (5090/SDPA-compatible) |

## What to Try Next

- **Setup script.** Automate the RunPod + 5090 workarounds (user creation, tooling install, FA3→SDPA patch, conservative batch defaults) into a single `bash setup.sh`
- **Full nanochat GPT-2 speedrun.** Rent an 8×H100 node and train a chat-capable model end-to-end (separate experiment)
- **Longer time budget.** Autoresearch's 5-min window heavily favors throughput over model size. A 30-min or 60-min budget might let deeper models win
- **SFT on the trained model.** Take a nanochat checkpoint and fine-tune on domain-specific data
- **Compare platforms.** Run the same autoresearch session on an H100 (with FA3) and compare what the agent discovers — the optimal configs will likely differ significantly