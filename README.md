# Training receipts — results

**A training step that produces its own proof.** A separate verifier checks the proof, and on the hardware we have compared
the proof comes out bit for bit the same. This repository shows what we have measured so far: results and one real artifact,
not the codebase. **The verifier is not included here; it and the full proof files are available on request.**

Every number below was measured on our own runs (October 2026) unless it is marked *estimate*. Hardware: one NVIDIA GH200 (Grace
ARM + H100 96 GB), one H100 80 GB (x86 host), one A100, and CPU-only machines for independent re-checks.

## Results at a glance

| | Result | Where |
|---|---|---|
| 1 | **A whole training run audits from about 2 MB of files**: a 13,400-byte root plus its 1.28 MB proof and 0.68 MB accumulator state. Our verifier checks them in **1.47 s** on 4 CPU cores with 60 MB of memory. A public randomness beacon picks which steps get proven, *after* training, so the steps cannot be chosen in advance. | [§1](#1-a-whole-run-in-about-2-mb), [`results/gold-run-1-root.json`](results/gold-run-1-root.json) |
| 2 | **Proving one GPT-2 124M training step takes 104–106 s on one GH200** on our fastest path (vocabulary 50,304, 124.5 M parameters). The frozen configuration our audited runs use, on their smaller 65-symbol-vocabulary model (85.9 M parameters), takes ~24–25 min of proving (~32–33 min per step end to end). | [§2](#2-proving-time-per-step) |
| 3 | **AdamW is fully checked inside the receipts.** A 64-step AdamW run had 10 beacon-sampled steps proven and accepted, **and 47 of 47 steps replay bit for bit between ARM (GH200) and x86 (H100).** | [§3](#3-adamw-and-bit-for-bit-across-cpu-architectures) |
| 4 | **One GPT-2-small-configuration training step split across two machines (GH200 + H100) verifies as one step.** This is our first proof spanning two GPUs, and a separate machine reproduced it from scratch with independently built verifiers. | [§4](#4-one-step-across-two-machines) |
| 5 | **An exact integer training rule tracks bf16 on real text** (GPT-2 124M on FineWeb): within **0.04 nats** of held-out loss at every checkpoint through 4,000 steps, and 0.019 at the end. *Early result, one seed; a second-seed run is in progress.* | [§5](#5-an-exact-integer-training-rule-on-real-text), [`results/fineweb-integer-rule.csv`](results/fineweb-integer-rule.csv) |
| 6 | **The rule's core kernel is 11× faster** than our current exact rules (matrix multiply plus exact rounding, GPU time: 78.9 µs vs 0.870 ms), with outputs equal. | [§6](#6-kernel-speed) |
| 7 | **A third GPU type proves as it trains:** on an A100, 18 consecutive steps of a fresh AdamW run are each proven, verified by two independent verifiers, and chained to the previous step. | [§7](#7-proving-as-it-trains-a100) |

---

## 1. A whole run in about 2 MB

**Gold run 1.** We trained a GPT-2-small-architecture model (12 layers × 768; 85.9 M parameters at this run's 65-symbol vocabulary)
for 300 steps with Lion and committed every step. Only then did a public drand beacon round (6583616) draw a sample of 31 step indices.
Each sampled step was proven and verified, and the 31 per-step receipts were folded into one accumulator. A final proof over that
accumulator gives the **root**. The root's summary is the file in this repository.

| | |
|---|---|
| Root file | [`results/gold-run-1-root.json`](results/gold-run-1-root.json), 13,400 bytes, sha256 `80d2b4cf7eedf668246f55ccc61167a15ac80b19a28a5a611a5d4daa6f480873` |
| What the check reads | the root, the final proof (1,279,752 bytes), the accumulator state (676,737 bytes), and the run's public header and beacon round. All of these are available on request; only the root is in this repository |
| Verdict | ACCEPT over all 31 sampled steps; every one passed our strongest check level (STRONG, 31 of 31) |
| Checking the root | **1.47 s** wall with our verifier, 60 MB peak memory, 4 pinned CPU cores (ARM Neoverse-N2), 3 runs all equal; the proof check itself is 0.038 s |
| Sample | drawn from the run's committed header hash and the beacon round, so it is fixed only *after* every step is committed |
| Second run | Gold run 2 (AdamW, 64 steps, 10 sampled): root ACCEPT, all 10 sampled steps STRONG |

What the root check re-verifies: the final proof against the accumulator, the run's public header, the sample recomputed from the
beacon (including the beacon's BLS signature), and the chain from the declared initial weights and zero optimizer state to the first step.
What it does *not* re-run: the per-step node checks, which we ran (see "What we do not claim").

## 2. Proving time per step

Two model sizes appear below, both 12 layers × 768, so read each row's model column. Every run passed every check (statement,
layout and tape equal; both our Python and Rust verifiers ACCEPT).

| Path | Model | Prover time per step | Notes |
|---|---|---|---|
| Fastest path, GH200 (4 runs) | vocabulary 50,304, 124.5 M parameters, 8 × 1,024 tokens a step | **104 s, 105 s, 105 s, 106 s** | ~146–152 s end to end including Rust verification *(computed from measured parts)* |
| Audited runs' frozen configuration, GH200 (9 steps of gold run 1) | vocabulary 65, 85.9 M parameters | 1,431–1,483 s proving (~24–25 min) | 1,891–1,986 s per step with witness generation, replay and verification (~32–33 min) |
| Same configuration, H100 (1 step of gold run 1) | vocabulary 65, 85.9 M parameters | ~65 min proving | |

Per-step detail for the audited configuration: [`results/proving-times.csv`](results/proving-times.csv).

## 3. AdamW, and bit for bit across CPU architectures

- **AdamW inside the receipt:** gold run 2 (the same 65-symbol-vocabulary model as gold run 1) used AdamW, the optimizer most training uses. All 10 beacon-sampled steps were proven
  (statement EQUAL to the committed one, verifier ACCEPT), and the run's root is ACCEPT.
- **47 of 47 steps replay bit for bit, ARM vs x86:** steps 0–46 of the AdamW run, replayed on the H100's x86 host, give exactly the
  states committed by the GH200's ARM pass.
- **Whole receipts match too:** proven once on ARM and once on x86, a full Lion step's receipt has every proof-bearing file
  byte-identical. That is 105 of its 345 files: the 34 column files, the 34 openings with their proof-of-work nonces, the points,
  layout, receipt and tape. The other 240 differ only in timing fields and the machine report. The 87 proof-bearing files of an AdamW
  step match the same way.

## 4. One step across two machines

A single GPT-2-small-configuration training step was split into two shards. One was proven on the GH200 and the other on the H100, and the two receipts were
linked and verified as **one step**. Every member passed our strongest check level (STRONG). A separate CPU-only
machine reproduced the verdict from scratch, with independently built verifiers: all 806 verdict rows are identical to the GPU run's.

## 5. An exact integer training rule on real text

**Why it matters:** proofs need exact arithmetic, and today's exact rules make a training step tens of times slower than bf16. We are
building an exact integer training rule that runs on integer tensor cores and stays bit-exact across hardware.

**Setup:** GPT-2 124M (vocabulary 50,304), FineWeb, 16 × 1,024 tokens a step, AdamW, cosine schedule over 4,000 steps, the same seed
and the same batches in every run. The table shows held-out loss minus bf16's at the same step (lower is better; our bar is 0.05):

| Rule | 500 | 1,000 | 1,500 | 2,000 | 2,500 | 3,000 | 3,500 | 4,000 |
|---|---|---|---|---|---|---|---|---|
| first version | +0.675 | +1.068 | +1.265 | — (plateaued) | | | | |
| second version | +0.027 | +0.049 | +0.056 | +0.136 | | | | |
| **current version** | **+0.021** | **+0.040** | **+0.026** | **+0.039** | **+0.021** | **+0.022** | **+0.018** | **+0.019** |

Columns are training steps. bf16 reference held-out loss: 6.094 / 5.582 / 5.273 / 5.003 / 4.824 / 4.705 / 4.632 / 4.590; the current
version ends at 4.609. Data: [`results/fineweb-integer-rule.csv`](results/fineweb-integer-rule.csv).

## 6. Kernel speed

At one MLP GEMM shape of the GPT-2-small configuration (8192 × 3072 × 768), measured as GPU time (CUDA-graph replay):

| | Exact integer rule | Our current exact rules | Speed-up |
|---|---|---|---|
| Matrix multiply + exact rounding | 78.9 µs | 0.870 ms | **11.0×** |
| Same with a bias | 0.130 ms | 0.884 ms | **6.8×** |

Outputs are equal across all 5 linear-layer shapes and against the reference on sampled rows. Data:
[`results/kernel-speed.csv`](results/kernel-speed.csv).

## 7. Proving as it trains (A100)

A fresh AdamW run of the gold runs' model (65-symbol vocabulary, 85.9 M parameters) trains on an A100, and every step is proven as it goes. Each step must pass
five checks: the node's own check, the Python verifier, the Rust verifier, the link to the previous step's receipt, and our strongest
check level (STRONG). Steps 0–17 pass all five, at about 1.5 h per step on this card. Step 0's first settlement
failed on a harness problem and was re-settled; every later step passed on the first attempt.

---

## What we do not claim

- **The verifier is not in this repository.** The 13 KB root file shows the artifact, but checking it needs our verifier plus the
  ~2 MB of proof files it reads. Both are available on request.
- **The root trusts the per-step node checks.** We ran them. With our verifier, anyone can re-run them from the per-step receipt
  files (measured: 62 s and 511 MB per step); those files are large and available on request.
- **Most steps are sampled, not proven.** Gold run 1 proved 31 of its 300 steps. Separately, we have now proven its first 25 steps
  one by one, with no sampling; their combined root is being built.
- **Security level:** the step receipts currently give about 84–88 bits of soundness. Raising it is planned work.
- **The integer training rule is early.** The result above is one seed, 4,000 steps at 124M; a second-seed run is in progress. The
  speed-up is for the kernel, not yet for a full training step; the rest of the step is the next cost.
- **The 1.47 s check** ran on 4 pinned cores of an ARM server with 60 MB of memory (laptop-class resources), not on an actual laptop.

## What more compute would answer

All of the above is at GPT-2-small scale (85.9 M–124.5 M parameters) on three GPUs (GH200, H100, A100). The questions that need more compute are how proving cost and the integer rule's
quality scale to 1B–7B models, and how receipts behave across many GPUs. Our *estimate* is that the proving overhead per step falls
steeply with model size, because the costs that dominate at this scale grow more slowly than the training FLOPs. That is what we want to measure.
