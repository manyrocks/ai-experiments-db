# Autoresearch Experiment Summary

**Platform:** RTX 5090 (32GB VRAM, Blackwell, compute capability 12.0)
**Date:** March 10-11, 2026
**Branch:** `autoresearch/mar10`
**Total experiments:** 147
**Baseline val_bpb:** 1.089249
**Best val_bpb:** 1.027893 (commit `146cd36`, exp139)
**Total improvement:** 5.6% reduction in validation bits per byte

---

## Best Configuration

| Parameter | Value |
|---|---|
| DEPTH | 8 |
| ASPECT_RATIO | 64 (model_dim=512) |
| HEAD_DIM | 128 (4 heads) |
| WINDOW_PATTERN | "L" (all full attention) |
| MLP | 4x expansion, ReluSquared activation |
| TOTAL_BATCH_SIZE | 2^17 (131072 tokens) |
| DEVICE_BATCH_SIZE | 64 |
| EMBEDDING_LR | 0.6 |
| UNEMBEDDING_LR | 0.004 |
| MATRIX_LR | 0.032 |
| SCALAR_LR | 0.5 |
| WEIGHT_DECAY | 0.12 (cautious, decaying schedule) |
| ADAM_BETAS | (0.7, 0.95) |
| Adam eps | 1e-10 |
| WARMUP_RATIO | 0.0 (no warmup) |
| WARMDOWN_RATIO | 0.65 |
| FINAL_LR_FRAC | 0.02 |
| softcap | 12 |
| embed_init_std | 0.7 |
| VE gate channels | 16 |
| VE LR | 0.5 * embedding_lr |
| Muon ns_steps | 6 |
| Muon beta2 | 0.95 |
| Muon momentum | 0.85 -> 0.95 over 300 steps |
| torch.compile | mode="max-autotune" |
| Peak VRAM | 22.1 GB |
| Training steps | ~1588 |

---

## Progression of Kept Improvements

| Exp | val_bpb | Change | Description |
|-----|---------|--------|-------------|
| 1 | 1.089249 | -- | Baseline (SDPA, depth=8, batch=8) |
| 4 | 1.048707 | -0.041 | torch.compile + batch=32 |
| 6 | 1.037002 | -0.012 | batch=64, total=2^17 |
| 11 | 1.036587 | -0.000 | WINDOW_PATTERN=L (all full attention) |
| 34 | 1.036551 | -0.000 | WEIGHT_DECAY=0.1 |
| 56 | 1.036430 | -0.000 | Adam beta1=0.7 |
| 58 | 1.035787 | -0.001 | VE gate channels 16 |
| 65 | 1.035715 | -0.000 | softcap=12 |
| 69 | 1.034286 | -0.001 | VE LR = 0.5 * embedding_lr |
| 77 | 1.034123 | -0.000 | embedding init std=0.7 |
| 108 | 1.030806 | -0.003 | torch.compile mode=max-autotune |
| 110 | 1.030036 | -0.001 | WARMDOWN_RATIO=0.55 |
| 111 | 1.029742 | -0.000 | WARMDOWN_RATIO=0.6 |
| 112 | 1.029534 | -0.000 | WARMDOWN_RATIO=0.65 |
| 116 | 1.028958 | -0.001 | MATRIX_LR=0.035 |
| 118 | 1.028672 | -0.000 | WEIGHT_DECAY=0.12 |
| 130 | 1.028231 | -0.000 | FINAL_LR_FRAC=0.02 |
| 133 | 1.028203 | -0.000 | MATRIX_LR=0.032 |
| 139 | 1.027893 | -0.000 | Muon ns_steps=6 |

---

## Key Insights

### 1. Throughput is king in fixed time budgets
The single most important factor was maximizing the number of optimizer steps in 5 minutes. Every experiment that increased model size (depth=9/10/16, wider models, larger MLP) resulted in worse val_bpb because fewer steps outweighed better per-step quality.

### 2. Big wins came from batch scaling and compilation
- **torch.compile** (default mode): gave immediate 2x throughput boost
- **torch.compile mode=max-autotune**: another +7% throughput on top
- **Batch size 8->64 with total_batch 2^16->2^17**: doubled tokens seen per run

### 3. Hyperparameter landscape shifts after structural changes
After switching to max-autotune (exp108), several previously-optimal hyperparameters shifted:
- WARMDOWN_RATIO: 0.5 -> 0.65
- MATRIX_LR: 0.04 -> 0.035 -> 0.032
- WEIGHT_DECAY: 0.1 -> 0.12
- FINAL_LR_FRAC: 0.0 -> 0.02

This demonstrates that hyperparameter optima are interdependent and should be re-tested after major changes.

### 4. Diminishing returns on hyperparameter tuning
After ~100 experiments, most single-parameter changes yielded <0.001 improvement. The last 50 experiments produced only ~0.006 total improvement combined.

---

## What Worked

| Category | Best Value | Runner-up | Notes |
|---|---|---|---|
| Activation | ReluSquared | -- | GELU -0.015, SwiGLU -0.007 worse |
| Attention | Full (WINDOW_PATTERN=L) | SSSL | Sliding window adds overhead |
| Softcap | 12 | 10, 15 | Removing softcap: -0.005 |
| Adam beta1 | 0.7 | 0.8 | Lower momentum helps |
| Adam eps | 1e-10 | 1e-8 | Smaller eps: -0.003 better |
| Weight decay | 0.12, decaying | 0.1 | Cautious mask important |
| Embed init | std=0.7 | std=1.0 | Moderate init helps |
| VE gate | 16 channels, sigmoid | 32 channels | Fewer channels = less overfitting |
| VE LR | 0.5x embed_lr | 1.0x | Lower VE LR prevents overfitting |
| Muon ns_steps | 6 | 5 | Better orthogonalization |
| Compile mode | max-autotune | default | +7% throughput |
| Warmdown | 0.65 | 0.5, 0.6 | Longer cooldown helps |

## What Didn't Work

| Experiment | val_bpb | Delta | Why |
|---|---|---|---|
| Label smoothing 0.1 | 1.374 | +0.285 | Completely distorts BPB evaluation |
| Multi-token 2-ahead | 1.106 | +0.072 | Single lm_head can't predict 2-ahead |
| depth=16 batch=16 | 1.141 | +0.052 | Way too few steps |
| Remove QK-norm | 1.053 | +0.019 | QK-norm is critical for stability |
| Disable VE | 1.052 | +0.018 | Value embeddings are essential |
| GELU activation | 1.051 | +0.016 | Much worse than ReluSquared |
| Parallel attn+MLP | 1.049 | +0.015 | Quality loss from parallelism |
| depth=10 | 1.120 | +0.031 | Fewer steps dominate |
| total_batch=2^18 | 1.072 | +0.035 | Too few optimizer steps |
| GQA n_kv_head=2 | 1.044 | +0.008 | Quality loss not worth memory savings |
| Stochastic depth | crash | -- | Breaks torch.compile |

## Extensively Tested (Near-Optimal, No Gain)

These parameters were tested at multiple values and found to be at or very near their optimum:

- **Learning rates:** EMBEDDING_LR (0.4-1.0), UNEMBEDDING_LR (0.003-0.008), MATRIX_LR (0.02-0.06), SCALAR_LR (0.3-1.0)
- **Adam betas:** beta1 (0.6-0.9), beta2 (0.9-0.99)
- **Weight decay:** 0.0-0.4, constant vs decaying schedule
- **Warmdown ratios:** 0.3-0.7
- **Warmup:** 0%, 1%, 2%, 5%
- **FINAL_LR_FRAC:** 0.0, 0.01, 0.02, 0.03, 0.05, 0.1
- **VE configs:** gate channels (8-32), gate scale (1.5-3), frequency (alternating/every/every-3rd/first-half), shared VE, linear gate, normalized VE, weight decay on VE
- **RoPE base:** 5000, 10000, 50000
- **Initialization:** embed std (0.5-1.0), lm_head std (0.001-0.01), transformer scale
- **Architecture:** GQA, model widths (AR=64-80), MLP widths (3x-6x), depths (6-10), HEAD_DIM (64, 128)
- **Muon optimizer:** ns_steps (4-7), beta2 (0.9-0.99), momentum endpoints, warmup schedules
- **Other:** cosine LR, z-loss, seeds, gradient clipping, SDPA scale, dmodel_lr_scale

---

## Full Experiment Log

```
commit	val_bpb	memory_gb	status	description
e20deae	1.089249	6.1	keep	baseline (SDPA, depth=8, batch=8, total_batch=2^16)
c64f2fd	1.140920	23.5	discard	depth=16 batch=16 total=2^17 torch.compile (too big, not enough steps)
96de742	1.120010	9.2	discard	depth=10 batch=16 torch.compile (still too big for time budget)
57566ea	1.048707	11.3	keep	depth=8 batch=32 torch.compile grad_accum=1 (2x throughput)
8f0ce6b	1.054888	15.6	discard	depth=9 (bigger model but fewer steps, net worse)
7235402	1.037002	22.1	keep	batch=64 total=2^17 (2x tokens, 22.7GB VRAM)
1b3e608	1.072441	22.3	discard	total_batch=2^18 (too few optimizer steps)
a1c3232	1.043438	22.1	discard	MATRIX_LR=0.06 (worse than 0.04)
654fd00	1.039437	22.1	discard	MATRIX_LR=0.02 (also worse than 0.04)
b4543a1	1.040070	22.1	discard	5% warmup (no benefit)
1ea8ad8	1.036587	22.1	keep	WINDOW_PATTERN=L (all full attention)
31274c2	1.037301	22.1	discard	WARMDOWN_RATIO=0.7 (slightly worse)
18130b6	1.040836	22.1	discard	softcap=30 (worse than 15)
fe466f5	1.038094	22.1	discard	EMBEDDING_LR=1.0 (worse than 0.6)
dd245e1	1.043396	22.4	discard	SwiGLU activation (worse than ReluSquared)
85018b6	1.048191	27.2	discard	ASPECT_RATIO=80 wider model (too slow)
8550656	1.042473	22.1	discard	WEIGHT_DECAY=0.0 (worse without decay)
837a85b	1.043433	22.1	discard	WEIGHT_DECAY=0.4 (too much decay)
f59f0b3	1.043161	26.2	discard	MLP 6x expansion (fewer steps)
4bdc560	1.045369	22.2	discard	HEAD_DIM=64 (more heads but worse quality)
cee5822	1.042035	22.1	discard	WARMDOWN_RATIO=0.3 (worse)
194a06a	1.037090	22.1	discard	FINAL_LR_FRAC=0.1 (marginal worse)
3c7a935	1.041471	22.1	discard	Adam beta1=0.9 (worse than 0.8)
92eb6e7	1.039715	22.1	discard	MATRIX_LR=0.03 (worse than 0.04)
bcf6a71	1.039398	22.1	discard	UNEMBEDDING_LR=0.008 (worse than 0.004)
6569329	1.039536	22.1	discard	softcap=20 (worse than 15)
d931728	1.041479	24.7	discard	depth=9 AR=56 same width (still too slow)
0834c3b	1.054724	13.3	discard	depth=6 (too small, not enough capacity)
ca1ea9d	1.039866	22.1	discard	EMBEDDING_LR=0.4 (worse than 0.6)
e9262af	1.373986	26.1	discard	label smoothing 0.1 (terrible, distorts eval)
e184a1b	1.041771	22.1	discard	MATRIX_LR=0.05 (worse than 0.04)
9afc9f8	1.037649	20.1	discard	MLP 3x expansion (close but worse)
e97aa77	1.040641	22.1	discard	z-loss regularization (worse)
f6e4493	1.036551	22.1	keep	WEIGHT_DECAY=0.1 (marginal improvement)
ba4b82f	1.038033	22.1	discard	WEIGHT_DECAY=0.05 (worse, 0.1 is sweet spot)
046c06c	1.038865	24.2	discard	MLP 5x expansion (slower, worse)
edd2f45	1.040496	22.1	discard	cosine LR decay (worse than linear)
caedcf9	1.041804	22.1	discard	remove softcap (worse, softcap helps)
29361cb	1.039278	22.1	discard	constant weight decay (decaying schedule better)
56bbd50	1.037381	22.1	discard	Adam beta2=0.99 (slightly worse)
4950194	1.036946	22.1	discard	WARMDOWN_RATIO=0.6 (slightly worse)
acb1ecf	1.037534	22.1	discard	faster Muon momentum warmup (worse)
f461607	1.041540	22.1	discard	SCALAR_LR=1.0 (worse than 0.5)
dd02c20	1.050506	22.1	discard	GELU activation (much worse than ReluSquared)
a3af3f1	1.039086	22.1	discard	x0_lambdas init 0.2 (worse than 0.1)
998126e	0.000000	0.0	crash	stochastic depth (breaks torch.compile)
7549857	1.036821	22.1	discard	WEIGHT_DECAY=0.15 (worse than 0.1)
5b7ba4a	1.037063	22.1	discard	remove cautious WD mask (worse)
e433dad	1.048699	20.1	discard	parallel attn+MLP (faster but much worse quality)
b62ab92	1.038625	22.1	discard	seed 137 (worse than seed 42)
aa56f86	1.039330	22.1	discard	EMBEDDING_LR=0.8 (worse than 0.6)
05ba93a	1.038080	22.1	discard	RoPE base 50000 (worse than 10000)
e4bee4d	1.037753	22.1	discard	WARMDOWN_RATIO=0.4 (worse than 0.5)
2a3ed13	1.052738	20.1	discard	remove QK-norm (much worse)
5669562	1.043711	21.3	discard	GQA n_kv_head=2 (quality loss too much)
ec4cbac	1.036430	22.1	keep	Adam beta1=0.7 (marginal improvement)
79933f7	1.037030	22.1	discard	Adam beta1=0.6 (worse, 0.7 is optimal)
5a21ca3	1.035787	22.1	keep	VE gate channels 32->16 (less overfitting)
83a98bd	1.036582	22.1	discard	VE gate channels 8 (too few, 16 optimal)
4257bd1	1.038816	22.1	discard	UNEMBEDDING_LR=0.006 (worse)
97c2754	1.037612	22.7	discard	VE every layer (too many params)
35b1ac6	1.052294	21.5	discard	disable VE (much worse, VE is critical)
274cd51	1.041992	22.1	discard	SCALAR_LR=0.3 (worse than 0.5)
24c74aa	1.038370	22.1	discard	gradient clipping max_norm=1.0 (worse)
0966b6e	1.035715	22.1	keep	softcap=12 (marginal improvement over 15)
25f04ad	1.036358	22.1	discard	softcap=10 (too low, 12 is optimal)
a359968	1.038063	22.1	discard	Muon ns_steps=4 (worse quality outweighs speed)
981b502	1.037471	22.1	discard	x0_lambdas init 0.05 (worse, 0.1 optimal)
262c0f2	1.034286	22.1	keep	VE LR = 0.5 * embedding_lr (lower VE LR helps)
66b2c90	1.035059	22.1	discard	VE LR = 0.3 * embedding_lr (too low, 0.5 optimal)
6a5349a	1.045642	22.1	discard	weight decay 0.1 on VE (much worse)
40d7ab9	1.039208	22.1	discard	resid_lambdas init 0.9 (worse than 1.0)
a393646	1.037635	22.1	discard	embedding weight decay 0.01 (worse)
3c2464b	1.035202	22.1	discard	VE LR = 0.7 * embedding_lr (0.5 is optimal)
a1c269a	1.039500	22.1	discard	MATRIX_LR=0.045 (worse, 0.04 optimal)
ede9b6f	1.034522	22.1	discard	embedding init std=0.5 (close but worse than 1.0)
26d93c4	1.034123	22.1	keep	embedding init std=0.7 (improvement over 1.0)
357b5e4	1.035450	22.1	discard	embedding init std=0.6 (worse, 0.7 optimal)
405d139	1.036564	22.1	discard	flat Muon momentum 0.95 (warmup from 0.85 is better)
8759cfc	1.035498	22.1	discard	Muon beta2=0.9 (worse, 0.95 optimal)
484c2f8	1.038405	22.1	discard	transformer init s=1/sqrt(d) (worse, sqrt(3)/sqrt(d) better)
7eab348	1.040506	22.1	discard	lm_head init std=0.01 (worse, 0.001 better)
bdc4679	1.036963	22.1	discard	Muon momentum warmup 500 steps (300 better)
35eb6d6	1.037522	22.1	discard	WARMUP_RATIO=0.02 (worse, no warmup better)
734f297	1.042061	21.7	discard	shared single VE across layers (too much quality loss)
e495eb6	1.037048	22.1	discard	Adam beta2=0.99 for embeddings (worse, 0.95 better)
8d27b8a	1.051436	22.1	discard	VE first half only (much worse, alternating better)
5c1040f	1.038593	22.1	discard	SDPA scale=1/d^0.25 (worse, default scale better)
d966437	1.034541	22.1	discard	FINAL_LR_FRAC=0.05 (close but worse than 0.0)
9e82c37	1.034978	22.1	discard	VE gate channels 24 (worse, 16 optimal)
2cca575	1.038468	22.1	discard	x0_lambdas beta1=0.9 (worse, 0.96 optimal)
2562e61	1.038295	22.1	discard	linear VE gate (sigmoid is important)
0be1c39	1.043185	22.1	discard	normalize VE before adding (worse)
95e629d	1.036365	22.1	discard	VE gate scale 1.5 (worse, 2.0 better)
5df577e	1.035804	22.1	discard	resid_lambdas LR *0.02 (worse, *0.01 better)
63cfc90	1.035803	22.1	discard	embedding init std=0.8 (worse, 0.7 optimal)
abb3efa	1.038391	22.1	discard	UNEMBEDDING_LR=0.003 (worse, 0.004 optimal)
e282de9	1.037669	22.1	discard	remove dmodel_lr_scale (scaling helps)
00bbb0c	1.035011	22.1	discard	strided VE gate inputs (close but worse)
4efdd3a	1.037180	22.1	discard	EMBEDDING_LR=0.5 (still worse with new config)
a305f3c	1.034651	22.1	discard	VE gate scale 3 (worse, 2 optimal)
aa68078	1.036391	22.1	discard	VE LR = 0.4 * embedding_lr (0.5 optimal)
ec40e5c	1.036550	22.1	discard	enable flash SDP backend (no benefit)
e7585d3	1.106335	26.1	discard	multi-token 2-ahead loss 0.3 (much worse, distorts training)
415c350	1.039529	22.0	discard	VE every 3rd layer (too few VE layers)
1df0ceb	1.052302	11.4	discard	batch=32 grad_accum=2 (slower, much worse)
7806c95	1.036379	22.1	discard	EMBEDDING_LR=0.7 (worse, 0.6 optimal)
fb3623e	1.030806	24.1	keep	torch.compile mode=max-autotune (+7% steps, significant improvement)
8c91000	1.031095	22.1	discard	max-autotune-no-cudagraphs (similar but slightly worse)
8b8335b	1.030036	22.1	keep	WARMDOWN_RATIO=0.55 (improvement with max-autotune)
07bccca	1.029742	22.1	keep	WARMDOWN_RATIO=0.6 (further improvement)
a52b1ed	1.029534	22.1	keep	WARMDOWN_RATIO=0.65 (still improving)
4b6fe42	1.030257	22.1	discard	WARMDOWN_RATIO=0.7 (worse, 0.65 is optimal)
54e3590	1.031523	22.1	discard	WEIGHT_DECAY=0.08 (worse, 0.1 still optimal)
08d0038	1.031702	22.1	discard	UNEMBEDDING_LR=0.005 (worse, 0.004 still optimal)
3157bf8	1.028958	22.1	keep	MATRIX_LR=0.035 (improvement with new config)
a827dc5	1.029491	22.1	discard	MATRIX_LR=0.03 (worse, 0.035 optimal)
00da62d	1.028672	22.1	keep	WEIGHT_DECAY=0.12 (improvement)
c24f0ba	1.029223	22.1	discard	WEIGHT_DECAY=0.15 (worse, 0.12 optimal)
e5f10c3	1.030547	22.1	discard	Adam beta1=0.65 (worse, 0.7 still optimal)
cef1dfb	1.028919	22.1	discard	softcap=10 (close but 12 still better)
9a31993	1.028752	22.1	discard	softcap=11 (close but 12 still better)
1ccdaa8	1.029391	22.1	discard	embedding init std=0.5 (still worse, 0.7 optimal)
272b61e	1.029387	22.1	discard	VE LR = 0.6 * embedding_lr (0.5 still optimal)
4ce6d73	1.030919	22.1	discard	WARMUP_RATIO=0.01 (still worse, no warmup better)
e6f1757	1.032194	22.1	discard	SCALAR_LR=0.4 (worse, 0.5 optimal)
61a3541	1.029103	22.1	discard	fullgraph=True (slightly worse)
6aff352	1.028989	22.1	discard	VE gate channels=12 (close but worse, 16 optimal)
f264995	1.030007	22.1	discard	EMBEDDING_LR=0.55 (worse, 0.6 optimal)
44b7a56	1.028231	22.1	keep	FINAL_LR_FRAC=0.02 (improvement!)
6064e08	1.028728	22.1	discard	FINAL_LR_FRAC=0.03 (worse, 0.02 optimal)
ebf3aaf	1.028531	22.1	discard	FINAL_LR_FRAC=0.01 (worse, 0.02 optimal)
99946f6	1.028203	22.1	keep	MATRIX_LR=0.032 (marginal improvement)
47a0818	1.029700	22.1	discard	SCALAR_LR=0.6 (worse, 0.5 optimal)
0a2bced	1.028724	22.1	discard	softcap=11 (still worse, 12 optimal)
11795f2	1.029524	22.1	discard	RoPE base=5000 (worse, 10000 optimal)
36961f5	1.030713	22.1	discard	x0_lambdas init=0.15 (worse, 0.1 optimal)
146cd36	1.027893	22.1	keep	Muon ns_steps=6 (better orthogonalization)
2f54102	1.028544	22.1	discard	Muon ns_steps=7 (too many, 6 optimal)
a657488	1.029149	22.1	discard	MATRIX_LR=0.03 (worse, 0.032 still optimal)
1d59c8e	1.027908	22.1	discard	MATRIX_LR=0.035 (identical to 0.032, keep simpler)
aedfcce	1.028648	22.1	discard	WEIGHT_DECAY=0.14 (worse, 0.12 still optimal)
628caff	1.028078	22.1	discard	Muon beta2=0.99 (worse, 0.95 optimal)
ca62429	1.028460	22.1	discard	Muon momentum end=0.96 (worse, 0.95 optimal)
7d9152d	1.030855	22.1	discard	Adam eps=1e-8 (much worse, 1e-10 optimal)
f78914a	1.028156	22.1	discard	WARMDOWN_RATIO=0.7 (still worse, 0.65 optimal)
```

---

## Statistics

- **Experiments kept:** 19 / 147 (13%)
- **Experiments discarded:** 127 / 147 (86%)
- **Crashes:** 1 / 147 (1%)
- **Biggest single improvement:** torch.compile + batch scaling (exp4: -0.041)
- **Biggest single regression:** label smoothing (exp30: +0.285)
- **VRAM range:** 6.1 GB (baseline) to 27.2 GB (wider model)
- **Final VRAM usage:** 22.1 GB out of 32 GB available
