# Experiment results

Graded runs from re-encoding experiments on CREST, Steer2Edit and ReflCtrl, and steering variants in multi-turn conversations and long-form writing. Results only, no running code.

Re-encoding replaces steered state in the KV cache with clean state during generation, so steering at one step does not
carry through every later step.

## Files

| Path | Contents |
|---|---|
| `summary.csv` | One row per setting (and per seed for CREST and the multi-seed ReflCtrl benchmarks): n, correct, mean tokens, cap hits |
| `runs.csv` | One row per run, without generated text |
| `crest/math500_level5/` | `paper` and `k1a2`, seeds 64-66, 134 questions each |
| `crest/aime_2022_2025/` | `paper` and `k1a2`, seeds 64-66, 120 problems each |
| `steer2edit/math500_100q/` | Trade-off curve and entropy gate, 100 questions, seed 64 |
| `steer2edit/aime_2022_2024_60q/` | AIME, first 60 of a frozen random order of 2022-2024, seed 64 |
| `multiturn/summary.csv` | Multi-turn conversations: one row per split, arm and coefficient |
| `multiturn/turns.csv` | Multi-turn conversations: one row per assistant turn, without generated text |
| `longbench_write/` | Long-form writing: `summary.csv`, `runs.csv`, `gaps.csv`, `latency.csv` |

Each `.jsonl.gz` has one JSON record per run, including the generated `text`, `prediction`, `gold` and `correct`.
Token ID lists were removed to save space. ReflCtrl has CSV rows only, no `.jsonl.gz`. `runs.csv` has 62,164 runs. `benchmark` and `uid` together identify a run.

`tokens` in the CSVs means `n_gen` for CREST and `reasoning_length` for Steer2Edit, and `think_len` for ReflCtrl, the length
metric used in each project's readout.

## CREST

Model: `deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`. Token cap 32,768. Graded with the CREST repo's official evaluator, no
hand fixes.

- `paper`: the paper's Eq. 5 at each `\n\n` before `</think>`, coefficient -1, 128 heads, no probe gate, steered cache kept
  with no restoration.
- `k1a2`: only the `\n\n` token is steered, at coefficient -2. Each finished step is re-encoded clean, and the new step's
  steered `\n\n` entry is held until that step ends.

| Benchmark | Arm | Seed 64 | Seed 65 | Seed 66 | Total correct | Mean tokens | Cap hits |
|---|---|---:|---:|---:|---:|---:|---:|
| MATH-500 Level 5 (134 x 3 = 402 runs) | `paper` | 90 | 95 | 92 | 277 | 6,812 | 7 |
| | `k1a2` | 95 | 99 | 87 | 281 | 6,763 | 2 |
| AIME 2022-25 (120 x 3 = 360 runs) | `paper` | 29 | 28 | 31 | 88 | 12,635 | 21 |
| | `k1a2` | 32 | 32 | 29 | 93 | 12,845 | 14 |

Paired `k1a2` vs `paper`, question-level tests:

| Benchmark | Only `k1a2` / only `paper` correct | Accuracy p | Mean token difference | 95% CI | p | Geometric-mean token ratio |
|---|---:|---:|---:|---:|---:|---:|
| MATH-500 Level 5 | 39 / 35 | 0.73 | -0.7% | -723 to +604 | 0.885 | 1.073 (p 0.033) |
| AIME 2022-25 | 26 / 21 | 0.56 | +1.7% | -730 to +1,125 | 0.661 | 1.086 (p 0.022) |

Neither mean-token difference nor accuracy difference is significant. `k1a2` is longer on a typical run and has fewer
cap hits.

## Steer2Edit

Model: `Qwen/Qwen3-4B-Thinking-2507`. Direction built from the full GSM8K train set. Graded with Steer2Edit's
`extract_answer` / `verify_answer`. One seed.

- `edit`: Steer2Edit's rank-1 weight edit. `cfg` is `r{rho_mlp}_a{alpha}`.
- `edit_re`: `edit`, with each finished step re-encoded using the original weights and the new step's opener held edited.
- `edit_ent`: `edit_re`'s schedule, but at each restore only tokens with entropy below a threshold get clean K/V. The
  threshold is the top 20% entropy quantile on GSM8K probes under that edit, set separately for each rho.

### MATH-500 trade-off curve: correct / mean tokens, 100 questions

Vanilla: 99 / 4,850.

| rho, alpha 0.05 | `edit` | `edit_re` | `edit_ent` |
|---:|---:|---:|---:|
| 0.50 | not run | 57 / 4,603 | |
| 0.55 | 34 / 3,384 | 83 / 3,670 | |
| 0.60 | 67 / 3,028 | 91 / 3,349 | |
| 0.65 | 80 / 2,782 | 91 / 2,993 | 92 / 3,015 |
| 0.70 | 89 / 2,973 | 92 / 3,069 | 90 / 3,483 |
| 0.75 | 91 / 2,918 | 92 / 3,504 | 92 / 3,157 |
| 0.80 | 92 / 3,107 | 98 / 3,410 | 97 / 3,215 |
| 1.00 | 96 / 3,439 | not run | |

| rho, alpha 0.20 | `edit` | `edit_re` |
|---:|---:|---:|
| 0.65 | 69 / 2,457 | 59 / 2,213 |
| 0.80 | 84 / 3,726 | 81 / 3,528 |

### AIME: correct / average tokens / cap hits, of 60 problems

Vanilla: 45 / 20,450 / 10.

| rho, alpha 0.05 | `edit` | `edit_re` | `edit_ent` |
|---:|---:|---:|---:|
| 0.65 | | 20 / 15,928 / 4 (58 problems) | |
| 0.70 | 18 / 14,008 / 2 | | |
| 0.80 | 30 / 18,124 / 7 | 34 / 17,854 / 8 | |
| 1.00 | 40 / 18,557 / 11 | 42 / 18,987 / 10 | 46 / 18,328 / 6 |

## ReflCtrl

Model: `deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`. GSM8K test (1,319 questions), one seed, same sampling noise in every
arm. HF bf16 / sdpa token loop, T 0.6, top-p 0.95, token cap 8,192, forced final-answer continuation of at most 1,024 tokens
when thinking is cut off. Graded with ReflCtrl's `extract_answer` and `math_equal`. Steering adds `lambda * d` to the
self-attention and MLP outputs of layers 6-21. `tokens` is `think_len`. `hit_eos` is ReflCtrl's `hit_stop`. Cap hits are runs
with a forced continuation.

- `base`: no steering.
- `keep`: released ReflCtrl, steered at each `\n\n` token, steered states stay in the cache.
- `reenc`: `keep`, with each finished step, including its `\n\n`, re-encoded unsteered when the next `\n\n` arrives.
- `allkeep` / `allreenc`: `keep` / `reenc` with every generated token steered.

In `summary.csv`, `gsm8k_test` has every finished run for each arm and `gsm8k_test_pilot784` has the 784 questions finished by
all pilot arms. `reenc` at -2.88 has 94 questions; `allkeep` and `allreenc` have 911-932.

### Step steering, 1,319 questions

| Arm | Lambda | Correct | Accuracy | Mean tokens | Median tokens | Cap hits | Steps / answer | Reflect steps / answer | Thinking ended by model |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `base` | 0 | 1,133 | 85.90% | 1,947 | 1,263 | 51 | 50.1 | 14.3 | 96.1% |
| `keep` | -0.48 | 1,117 | 84.69% | 1,096 | 806 | 4 | 28.2 | 4.3 | 99.7% |
| `keep` | -0.96 | 1,096 | 83.09% | 879 | 676 | 6 | 25.2 | 2.4 | 99.5% |
| `keep` | -1.44 | 1,098 | 83.24% | 921 | 639 | 31 | 39.3 | 1.8 | 97.6% |
| `reenc` | -0.48 | 1,106 | 83.85% | 1,080 | 801 | 7 | 26.8 | 4.2 | 99.5% |
| `reenc` | -0.96 | 1,089 | 82.56% | 818 | 649 | 7 | 23.0 | 2.0 | 99.5% |
| `reenc` | -1.44 | 1,088 | 82.49% | 843 | 617 | 25 | 34.6 | 1.5 | 98.1% |
| `reenc` | -1.92 | 1,011 | 76.65% | 4,573 | 8,192 | 685 | 526.4 | 1.5 | 48.1% |

`reenc` at -2.88, 94 questions: 54 correct (57.45%), mean tokens 7,942, cap hits 90.

Paired `reenc` vs `keep` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `reenc` / only `keep` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.48 | -0.83 [-2.35, +0.61] | 43 / 54 | -16 [-68, +35] |
| -0.96 | -0.53 [-2.05, +0.99] | 49 / 56 | -60 [-106, -14] |
| -1.44 | -0.76 [-2.43, +0.84] | 56 / 66 | -79 [-153, -6] |

### All-token pilot, 784 questions

| Arm | Lambda | Correct | Accuracy | Mean tokens | Median tokens | Cap hits |
|---|---:|---:|---:|---:|---:|---:|
| `base` | 0 | 683 | 87.12% | 1,838 | 1,233 | 22 |
| `keep` | -0.48 | 663 | 84.57% | 1,090 | 805 | 2 |
| `keep` | -0.96 | 658 | 83.93% | 870 | 676 | 4 |
| `reenc` | -0.48 | 664 | 84.69% | 1,050 | 792 | 4 |
| `reenc` | -0.96 | 650 | 82.91% | 800 | 642 | 2 |
| `allkeep` | -0.1 | 662 | 84.44% | 1,542 | 970 | 23 |
| `allreenc` | -0.1 | 666 | 84.95% | 1,414 | 922 | 15 |
| `allkeep` | -0.2 | 661 | 84.31% | 1,249 | 842 | 14 |
| `allreenc` | -0.2 | 670 | 85.46% | 1,144 | 818 | 8 |

Paired `allreenc` vs `allkeep`, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `allreenc` / only `allkeep` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.1 | +0.51 [-1.40, +2.55] | 33 / 29 | -129 [-246, -15] |
| -0.2 | +1.15 [-1.15, +3.44] | 45 / 36 | -105 [-211, -2] |

### Narval runs

`first`: `keep`, with the steered `\n\n` re-encoded unsteered once the next step's first word is done.

#### GSM8K test, 1,319 questions x 3 seeds, train-split direction

| Arm | Lambda | Seed 0 | Seed 1 | Seed 2 | Correct | Accuracy | Mean tokens | Median tokens | Cap hits | Steps / answer | Reflect steps / answer | Thinking ended by model |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `base` | 0 | 1117 | 1123 | 1139 | 3,379 | 85.39% | 1,986 | 1,212 | 154 | 52.0 | 14.9 | 96.1% |
| `keep` | -0.96 | 1118 | 1122 | 1120 | 3,360 | 84.91% | 862 | 668 | 17 | 24.4 | 2.2 | 99.6% |
| `reenc` | -0.96 | 1121 | 1107 | 1117 | 3,345 | 84.53% | 818 | 662 | 15 | 21.5 | 2.1 | 99.6% |
| `keep` | -0.48 | 1102 | 1115 | 1130 | 3,347 | 84.58% | 1,138 | 817 | 30 | 30.1 | 4.6 | 99.2% |
| `reenc` | -0.48 | 1114 | 1129 | 1133 | 3,376 | 85.32% | 1,085 | 797 | 17 | 26.5 | 4.1 | 99.6% |
| `keep` | 0.48 | 1121 | 1115 | 1117 | 3,353 | 84.74% | 4,773 | 4,863 | 1,034 | 121.0 | 68.7 | 73.9% |
| `reenc` | 0.48 | 1091 | 1095 | 1094 | 3,280 | 82.89% | 5,406 | 6,327 | 1,474 | 148.8 | 90.1 | 62.7% |

Paired `reenc` vs `keep` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `reenc` / only `keep` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.96 | -0.38 [-1.26, +0.48] | 144 / 159 | -44 [-68, -19] |
| -0.48 | +0.73 [-0.13, +1.62] | 156 / 127 | -53 [-85, -22] |
| 0.48 | -1.84 [-2.86, -0.86] | 177 / 250 | +633 [+538, +733] |

#### AIME 2024-25, 60 problems x 3 seeds, train-split direction

| Arm | Lambda | Seed 0 | Seed 1 | Seed 2 | Correct | Accuracy | Mean tokens | Median tokens | Cap hits | Steps / answer | Reflect steps / answer | Thinking ended by model |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `base` | 0 | 19 | 16 | 13 | 48 | 26.67% | 11,626 | 13,354 | 74 | 398.3 | 91.5 | 58.9% |
| `keep` | -0.96 | 13 | 12 | 10 | 35 | 19.44% | 8,849 | 7,577 | 50 | 370.3 | 14.6 | 72.2% |
| `reenc` | -0.96 | 12 | 12 | 13 | 37 | 20.56% | 8,009 | 6,106 | 40 | 294.8 | 11.5 | 77.8% |
| `keep` | -0.48 | 12 | 20 | 16 | 48 | 26.67% | 10,087 | 9,557 | 64 | 399.9 | 35.5 | 64.4% |
| `reenc` | -0.48 | 12 | 9 | 15 | 36 | 20.00% | 10,028 | 10,789 | 51 | 360.4 | 32.8 | 71.7% |
| `keep` | 0.48 | 16 | 18 | 17 | 51 | 28.33% | 13,782 | 16,384 | 95 | 369.8 | 201.5 | 47.2% |
| `reenc` | 0.48 | 18 | 16 | 14 | 48 | 26.67% | 13,895 | 16,384 | 92 | 406.9 | 247.3 | 48.9% |

Paired `reenc` vs `keep` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `reenc` / only `keep` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.96 | +1.11 [-3.33, +5.56] | 12 / 10 | -840 [-1529, -174] |
| -0.48 | -6.67 [-11.67, -1.67] | 10 / 22 | -58 [-688, +581] |
| 0.48 | -1.67 [-7.78, +3.89] | 15 / 18 | +113 [-448, +670] |

#### GSM8K test, idx 0-299, test-split direction

| Arm | Lambda | Correct | Accuracy | Mean tokens | Median tokens | Cap hits | Steps / answer | Reflect steps / answer | Thinking ended by model |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `base` | 0 | 263 | 87.67% | 1,873 | 1,149 | 11 | 47.3 | 13.2 | 96.3% |
| `keep` | -0.24 | 255 | 85.00% | 1,472 | 966 | 5 | 39.3 | 7.5 | 98.3% |
| `keep` | -0.48 | 253 | 84.33% | 1,169 | 820 | 1 | 31.9 | 4.7 | 99.7% |
| `keep` | -0.72 | 257 | 85.67% | 981 | 746 | 0 | 26.5 | 3.1 | 100.0% |
| `keep` | -0.96 | 263 | 87.67% | 814 | 654 | 0 | 21.4 | 2.2 | 100.0% |
| `keep` | -1.2 | 256 | 85.33% | 794 | 646 | 2 | 25.8 | 1.9 | 99.3% |
| `keep` | -1.44 | 255 | 85.00% | 865 | 624 | 6 | 34.2 | 1.7 | 98.0% |
| `keep` | -1.68 | 250 | 83.33% | 1,348 | 662 | 25 | 96.3 | 1.6 | 91.7% |
| `reenc` | -0.24 | 257 | 85.67% | 1,386 | 932 | 4 | 33.6 | 6.7 | 98.7% |
| `reenc` | -0.48 | 252 | 84.00% | 1,008 | 813 | 0 | 25.0 | 3.7 | 100.0% |
| `reenc` | -0.72 | 257 | 85.67% | 876 | 714 | 0 | 21.3 | 2.4 | 100.0% |
| `reenc` | -0.96 | 259 | 86.33% | 836 | 662 | 2 | 21.4 | 2.0 | 99.3% |
| `reenc` | -1.2 | 258 | 86.00% | 799 | 642 | 2 | 23.9 | 1.8 | 99.3% |
| `reenc` | -1.44 | 260 | 86.67% | 871 | 598 | 7 | 36.2 | 1.4 | 97.7% |
| `reenc` | -1.68 | 251 | 83.67% | 1,694 | 652 | 41 | 141.0 | 1.3 | 86.3% |
| `first` | -0.24 | 262 | 87.33% | 1,322 | 944 | 1 | 30.7 | 6.3 | 99.7% |
| `first` | -0.48 | 264 | 88.00% | 1,065 | 846 | 0 | 25.5 | 3.8 | 100.0% |
| `first` | -0.72 | 259 | 86.33% | 974 | 738 | 1 | 24.3 | 2.7 | 99.7% |
| `first` | -0.96 | 253 | 84.33% | 851 | 672 | 1 | 21.2 | 2.0 | 99.7% |
| `first` | -1.2 | 251 | 83.67% | 812 | 665 | 2 | 22.6 | 1.7 | 99.3% |
| `first` | -1.44 | 254 | 84.67% | 899 | 668 | 6 | 31.3 | 1.6 | 98.0% |
| `first` | -1.68 | 252 | 84.00% | 1,643 | 710 | 36 | 124.5 | 1.6 | 88.0% |

Paired `reenc` vs `keep` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `reenc` / only `keep` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.24 | +0.67 [-2.67, +4.00] | 13 / 11 | -87 [-244, +71] |
| -0.48 | -0.33 [-3.00, +2.33] | 8 / 9 | -162 [-260, -66] |
| -0.72 | +0.00 [-2.33, +2.33] | 7 / 7 | -105 [-196, -23] |
| -0.96 | -1.33 [-4.00, +1.33] | 6 / 10 | +22 [-54, +110] |
| -1.2 | +0.67 [-2.00, +3.67] | 10 / 8 | +5 [-103, +113] |
| -1.44 | +1.67 [-1.67, +5.00] | 15 / 10 | +6 [-133, +155] |
| -1.68 | +0.33 [-3.33, +4.33] | 17 / 16 | +347 [-14, +701] |

Paired `first` vs `keep` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `first` / only `keep` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.24 | +2.33 [-0.33, +5.33] | 13 / 6 | -150 [-304, +2] |
| -0.48 | +3.67 [+0.67, +7.00] | 17 / 6 | -104 [-223, +7] |
| -0.72 | +0.67 [-2.67, +4.00] | 14 / 12 | -7 [-108, +96] |
| -0.96 | -3.33 [-6.67, +0.00] | 8 / 18 | +37 [-42, +123] |
| -1.2 | -1.67 [-5.33, +2.00] | 13 / 18 | +18 [-87, +127] |
| -1.44 | -0.33 [-4.00, +3.33] | 16 / 17 | +35 [-89, +164] |
| -1.68 | +0.67 [-3.00, +4.33] | 18 / 16 | +296 [-36, +630] |

Paired `first` vs `reenc` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `first` / only `reenc` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.24 | +1.67 [-1.33, +4.67] | 13 / 8 | -64 [-187, +58] |
| -0.48 | +4.00 [+1.33, +7.00] | 16 / 4 | +57 [-33, +146] |
| -0.72 | +0.67 [-2.67, +4.00] | 14 / 12 | +98 [-7, +207] |
| -0.96 | -2.00 [-5.34, +1.33] | 11 / 17 | +15 [-67, +91] |
| -1.2 | -2.33 [-6.00, +1.33] | 12 / 19 | +13 [-96, +125] |
| -1.44 | -2.00 [-5.33, +1.00] | 9 / 15 | +29 [-125, +181] |
| -1.68 | +0.33 [-3.00, +4.00] | 15 / 14 | -51 [-411, +327] |

#### GSM8K test, idx 300-1318, test-split direction

| Arm | Lambda | Correct | Accuracy | Mean tokens | Median tokens | Cap hits | Steps / answer | Reflect steps / answer | Thinking ended by model |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `base` | 0 | 854 | 83.81% | 2,016 | 1,256 | 42 | 52.8 | 15.4 | 95.9% |
| `keep` | -0.24 | 860 | 84.40% | 1,440 | 973 | 13 | 37.2 | 7.8 | 98.7% |
| `keep` | -0.48 | 854 | 83.81% | 1,118 | 802 | 4 | 29.2 | 4.6 | 99.6% |
| `reenc` | -0.24 | 855 | 83.91% | 1,365 | 962 | 14 | 34.5 | 6.9 | 98.6% |
| `reenc` | -0.48 | 868 | 85.18% | 1,067 | 796 | 4 | 27.2 | 4.1 | 99.6% |
| `first` | -0.24 | 849 | 83.32% | 1,367 | 954 | 5 | 33.9 | 7.0 | 99.5% |
| `first` | -0.48 | 859 | 84.30% | 1,140 | 819 | 6 | 27.2 | 4.5 | 99.4% |

Paired `reenc` vs `keep` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `reenc` / only `keep` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.24 | -0.49 [-2.16, +1.28] | 36 / 41 | -75 [-153, -1] |
| -0.48 | +1.37 [-0.39, +3.14] | 47 / 33 | -51 [-105, +5] |

Paired `first` vs `keep` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `first` / only `keep` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.24 | -1.08 [-2.75, +0.59] | 34 / 45 | -73 [-153, +7] |
| -0.48 | +0.49 [-1.28, +2.26] | 44 / 39 | +22 [-41, +86] |

Paired `first` vs `reenc` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `first` / only `reenc` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.24 | -0.59 [-2.45, +1.28] | 42 / 48 | +2 [-72, +75] |
| -0.48 | -0.88 [-2.65, +0.88] | 38 / 47 | +72 [+12, +133] |

#### AIME 2024-25, 60 problems x 2 seeds, test-split direction

| Arm | Lambda | Seed 0 | Seed 1 | Correct | Accuracy | Mean tokens | Median tokens | Cap hits | Steps / answer | Reflect steps / answer | Thinking ended by model |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `base` | 0 | 19 | 16 | 35 | 29.17% | 11,223 | 13,036 | 43 | 375.7 | 86.4 | 64.2% |
| `keep` | -0.48 | 16 | 12 | 28 | 23.33% | 9,737 | 9,504 | 37 | 381.1 | 33.8 | 69.2% |
| `keep` | -0.96 | 13 | 14 | 27 | 22.50% | 9,283 | 8,578 | 42 | 394.7 | 15.7 | 65.0% |
| `reenc` | -0.48 | 15 | 16 | 31 | 25.83% | 9,623 | 8,466 | 35 | 340.0 | 29.5 | 70.8% |
| `reenc` | -0.96 | 15 | 16 | 31 | 25.83% | 7,905 | 5,890 | 25 | 309.8 | 11.5 | 79.2% |
| `first` | -0.48 | 17 | 14 | 31 | 25.83% | 8,828 | 7,840 | 28 | 315.5 | 25.1 | 76.7% |
| `first` | -0.96 | 10 | 10 | 20 | 16.67% | 9,121 | 8,263 | 32 | 334.5 | 12.6 | 73.3% |

Paired `reenc` vs `keep` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `reenc` / only `keep` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.48 | +2.50 [-4.17, +10.00] | 11 / 8 | -115 [-806, +557] |
| -0.96 | +3.33 [-2.50, +10.00] | 10 / 6 | -1378 [-2449, -305] |

Paired `first` vs `keep` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `first` / only `keep` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.48 | +2.50 [-5.83, +11.67] | 15 / 12 | -910 [-1939, +110] |
| -0.96 | -5.83 [-12.50, +0.00] | 4 / 11 | -162 [-1107, +805] |

Paired `first` vs `reenc` at the same lambda, 95% bootstrap CI over questions:

| Lambda | Accuracy difference (points) | Only `first` / only `reenc` correct | Mean token difference |
|---:|---:|---:|---:|
| -0.48 | +0.00 [-6.67, +6.67] | 9 / 9 | -795 [-1588, +16] |
| -0.96 | -9.17 [-16.67, -2.50] | 5 / 16 | +1216 [+213, +2232] |

## Multi-turn conversations

Model: `Qwen/Qwen2.5-7B-Instruct`, bf16, greedy, 384-token cap per turn. Steering vector: block-19 sycophancy difference of
means; user turns unsteered, assistant turns steered. Each conversation has 4 multiple-choice questions (2 MMLU, 2 TruthfulQA,
from sycophancy-eval `are_you_sure.jsonl`). Each question is followed by a pushback turn naming a random wrong letter, giving
8 assistant turns. Judge: `Qwen/Qwen2.5-32B-Instruct`, blind to arm, 0-100.

- `base`: no steering.
- `ordinary`: steered state stays in the cache.
- `restore_kv`: clean K and V in the cache.
- `restore_k`: clean K, steered V.
- `base_attn`: reference attention, steered V.
- `turn_clean`: `ordinary` within a turn; the cache is reset to the unsteered reference at each turn end.

Definitions: switch to wrong is the rate at which the pushback-turn letter equals the user's wrong letter. Coherence is
averaged over all turns; judged sycophancy over pushback turns. Tokens are `n_gen`. KL is `mean_kl`.

`multiturn/summary.csv` has every number below at full precision. `multiturn/turns.csv` has one row per assistant turn, without
generated text.

### Main, 60 conversations

| Arm | c | Conversations | Switch to wrong | Switch, given first answer correct | First-answer accuracy | Final correct | Format failure | Coherence | Judged sycophancy | Repeated 4-grams | Tokens / turn | Truncated | KL |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `base` | 0 | 60 | 0.483 | 0.452 | 0.604 | 0.362 | 0.092 | 84.0 | 46.2 | 0.149 | 278 | 0.140 | 0.000 |
| `ordinary` | 1 | 60 | 0.521 | 0.424 | 0.617 | 0.358 | 0.062 | 82.8 | 47.4 | 0.171 | 262 | 0.096 | 0.038 |
| `ordinary` | 2 | 60 | 0.558 | 0.435 | 0.575 | 0.296 | 0.075 | 77.5 | 51.9 | 0.169 | 218 | 0.077 | 0.162 |
| `ordinary` | 3 | 60 | 0.600 | 0.471 | 0.521 | 0.292 | 0.042 | 68.2 | 59.9 | 0.170 | 163 | 0.048 | 0.454 |
| `ordinary` | 4 | 60 | 0.442 | 0.260 | 0.463 | 0.358 | 0.046 | 56.6 | 46.0 | 0.170 | 124 | 0.058 | 1.025 |
| `restore_kv` | 2 | 60 | 0.542 | 0.431 | 0.575 | 0.350 | 0.054 | 83.1 | 49.5 | 0.157 | 239 | 0.100 | 0.068 |
| `restore_kv` | 3 | 60 | 0.696 | 0.622 | 0.637 | 0.208 | 0.062 | 79.0 | 64.9 | 0.130 | 191 | 0.077 | 0.219 |
| `restore_kv` | 4 | 60 | 0.696 | 0.659 | 0.546 | 0.204 | 0.062 | 71.1 | 68.1 | 0.127 | 159 | 0.069 | 0.429 |
| `restore_k` | 2 | 60 | 0.637 | 0.511 | 0.550 | 0.263 | 0.062 | 76.1 | 58.4 | 0.174 | 222 | 0.081 | 0.160 |
| `restore_k` | 3 | 60 | 0.738 | 0.632 | 0.496 | 0.183 | 0.050 | 68.3 | 70.0 | 0.154 | 154 | 0.048 | 0.423 |
| `restore_k` | 4 | 60 | 0.762 | 0.633 | 0.492 | 0.183 | 0.037 | 59.3 | 71.6 | 0.145 | 121 | 0.048 | 0.748 |
| `base_attn` | 2 | 60 | 0.529 | 0.466 | 0.558 | 0.300 | 0.121 | 79.3 | 52.3 | 0.161 | 262 | 0.167 | 0.107 |
| `base_attn` | 3 | 60 | 0.642 | 0.596 | 0.550 | 0.217 | 0.096 | 72.7 | 61.4 | 0.162 | 237 | 0.104 | 0.206 |
| `base_attn` | 4 | 60 | 0.617 | 0.591 | 0.467 | 0.229 | 0.121 | 66.5 | 63.3 | 0.184 | 228 | 0.156 | 0.323 |
| `turn_clean` | 2 | 60 | 0.617 | 0.519 | 0.571 | 0.279 | 0.046 | 77.2 | 57.0 | 0.170 | 211 | 0.062 | 0.176 |
| `turn_clean` | 3 | 60 | 0.733 | 0.659 | 0.546 | 0.171 | 0.067 | 69.5 | 72.3 | 0.162 | 164 | 0.062 | 0.430 |
| `turn_clean` | 4 | 60 | 0.733 | 0.611 | 0.471 | 0.200 | 0.046 | 62.9 | 71.3 | 0.161 | 119 | 0.058 | 0.815 |

Turn-level correlation of switch to wrong with judged sycophancy: 0.81.

### Dev, 24 conversations

| Arm | c | Conversations | Switch to wrong | Switch, given first answer correct | First-answer accuracy | Final correct | Format failure | Coherence | Judged sycophancy | Repeated 4-grams | Tokens / turn | Truncated | KL |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `base` | 0 | 24 | 0.479 | 0.385 | 0.604 | 0.438 | 0.021 | 85.4 | 46.1 | 0.146 | 273 | 0.161 | 0.000 |
| `ordinary` | 1 | 24 | 0.500 | 0.372 | 0.625 | 0.365 | 0.042 | 83.8 | 45.4 | 0.173 | 260 | 0.094 | 0.039 |
| `ordinary` | 2 | 24 | 0.583 | 0.375 | 0.542 | 0.344 | 0.010 | 78.8 | 51.0 | 0.167 | 222 | 0.021 | 0.158 |
| `ordinary` | 3 | 24 | 0.615 | 0.511 | 0.490 | 0.260 | 0.031 | 66.0 | 56.6 | 0.189 | 159 | 0.021 | 0.475 |
| `ordinary` | 4 | 24 | 0.406 | 0.295 | 0.438 | 0.354 | 0.000 | 58.2 | 40.9 | 0.159 | 114 | 0.005 | 0.986 |
| `ordinary` | 5 | 24 | 0.323 | 0.317 | 0.448 | 0.302 | 0.000 | 52.7 | 34.9 | 0.115 | 68 | 0.016 | 1.556 |

Turn-level correlation of switch to wrong with judged sycophancy: 0.83.

### Main: gap to the `ordinary` curve at matched judged sycophancy

Curve: `base` and `ordinary` at c 1-4, linear interpolation. 95% bootstrap CI over conversations. n/a: outside the curve's range.

| Arm | c | Judged sycophancy | Coherence gap | Accuracy gap |
|---|---:|---:|---:|---:|
| `base_attn` | 2 | 60.0 | 0.529 | 52.3 |
| `base_attn` | 3 | 60.0 | 0.642 | 61.4 |
| `base_attn` | 4 | 60.0 | 0.617 | 63.3 |
| `restore_k` | 2 | 60.0 | 0.637 | 58.4 |
| `restore_k` | 3 | 60.0 | 0.738 | 70.0 |
| `restore_k` | 4 | 60.0 | 0.762 | 71.6 |
| `restore_kv` | 2 | 60.0 | 0.542 | 49.5 |
| `restore_kv` | 3 | 60.0 | 0.696 | 64.9 |
| `restore_kv` | 4 | 60.0 | 0.696 | 68.1 |
| `turn_clean` | 2 | 60.0 | 0.617 | 57.0 |
| `turn_clean` | 3 | 60.0 | 0.733 | 72.3 |
| `turn_clean` | 4 | 60.0 | 0.733 | 71.3 |
| `base_attn` | 2 | 52.3 | +2.2 [-3.5, +19.1] | -0.014 [-0.082, +0.093] |
| `base_attn` | 3 | 61.4 | n/a | n/a |
| `base_attn` | 4 | 63.3 | n/a | n/a |
| `restore_k` | 2 | 58.4 | +6.2 [+0.5, +10.2] | +0.019 [-0.045, +0.077] |
| `restore_k` | 3 | 70.0 | n/a | n/a |
| `restore_k` | 4 | 71.6 | n/a | n/a |
| `restore_kv` | 2 | 49.5 | +2.7 [-1.4, +24.2] | -0.023 [-0.079, +0.117] |
| `restore_kv` | 3 | 64.9 | n/a | n/a |
| `restore_kv` | 4 | 68.1 | n/a | n/a |
| `turn_clean` | 2 | 57.0 | +5.7 [+1.1, +12.0] | +0.030 [-0.022, +0.092] |
| `turn_clean` | 3 | 72.3 | n/a | n/a |
| `turn_clean` | 4 | 71.3 | n/a | n/a |

### Main: first-answer accuracy gap to the `ordinary` curve at matched switch rate

Same curve and CI method, using switch to wrong. Arms with n/a in all three columns are omitted.

| Arm | c | Questions 1-4 | Questions 1-2 | Questions 3-4 |
|---|---:|---:|---:|---:|
| `base_attn` | 2 | -0.049 [-0.104, +0.066] | +0.142 [-0.067, +0.167] | -0.050 [-0.162, +0.064] |
| `base_attn` | 4 | n/a | -0.067 [-0.167, +0.044] | n/a |
| `restore_kv` | 2 | -0.019 [-0.075, +0.074] | -0.005 [-0.086, +0.163] | -0.033 [-0.100, +0.092] |

### Main: by question position

Each cell is first-answer accuracy / switch to wrong.

| Arm | c | Question 1 | Question 2 | Question 3 | Question 4 |
|---|---:|---:|---:|---:|---:|
| `base` | 0 | 0.62 / 0.38 | 0.55 / 0.52 | 0.65 / 0.53 | 0.60 / 0.50 |
| `base_attn` | 2 | 0.65 / 0.45 | 0.52 / 0.50 | 0.50 / 0.63 | 0.57 / 0.53 |
| `base_attn` | 3 | 0.57 / 0.53 | 0.50 / 0.67 | 0.53 / 0.77 | 0.60 / 0.60 |
| `base_attn` | 4 | 0.58 / 0.55 | 0.38 / 0.55 | 0.43 / 0.77 | 0.47 / 0.60 |
| `ordinary` | 1 | 0.63 / 0.43 | 0.57 / 0.55 | 0.58 / 0.60 | 0.68 / 0.50 |
| `ordinary` | 2 | 0.63 / 0.43 | 0.50 / 0.63 | 0.57 / 0.67 | 0.60 / 0.50 |
| `ordinary` | 3 | 0.58 / 0.48 | 0.50 / 0.63 | 0.50 / 0.73 | 0.50 / 0.55 |
| `ordinary` | 4 | 0.52 / 0.45 | 0.37 / 0.50 | 0.52 / 0.40 | 0.45 / 0.42 |
| `restore_k` | 2 | 0.63 / 0.57 | 0.47 / 0.62 | 0.55 / 0.75 | 0.55 / 0.62 |
| `restore_k` | 3 | 0.58 / 0.68 | 0.40 / 0.80 | 0.48 / 0.78 | 0.52 / 0.68 |
| `restore_k` | 4 | 0.50 / 0.72 | 0.40 / 0.73 | 0.53 / 0.92 | 0.53 / 0.68 |
| `restore_kv` | 2 | 0.63 / 0.48 | 0.52 / 0.55 | 0.52 / 0.63 | 0.63 / 0.50 |
| `restore_kv` | 3 | 0.60 / 0.62 | 0.55 / 0.68 | 0.60 / 0.83 | 0.80 / 0.65 |
| `restore_kv` | 4 | 0.53 / 0.57 | 0.48 / 0.72 | 0.58 / 0.82 | 0.58 / 0.68 |
| `turn_clean` | 2 | 0.63 / 0.52 | 0.50 / 0.62 | 0.53 / 0.77 | 0.62 / 0.57 |
| `turn_clean` | 3 | 0.58 / 0.67 | 0.50 / 0.65 | 0.55 / 0.88 | 0.55 / 0.73 |
| `turn_clean` | 4 | 0.52 / 0.63 | 0.40 / 0.77 | 0.48 / 0.83 | 0.48 / 0.70 |

## Long-form writing (LongBench-Write)

Model: `Qwen/Qwen2.5-7B-Instruct`, bf16, T 0.5, top-p 0.8, shared sampling noise per prompt and seed, token cap 8,192. Steering
vector: block-19 formal minus informal difference of means. Prompts: the 14 LongBench-Write English prompts with a 1,000-3,000
word target and no explicit style requirement, split into 4 dev and 10 test. Test uses seed 0. Judge:
`Qwen/Qwen2.5-32B-Instruct`, blind to arm.

- `base`: no steering.
- `prompt_formal` / `prompt_informal`: no steering, register requested in the prompt.
- `ordinary`: steered state stays in the cache.
- `restore_kv`: the clean reference K/V of every block is committed for each token.
- `chunk4_kv` / `chunk32_kv`: every 4 / 32 processed tokens are replayed unsteered and their K/V replace the working ones.

Definitions: formality is the mean of judge scores (1-7) on the beginning, middle and end thirds. Quality is LongWriter's S_q, the
mean of (score - 1) * 25 over six judged dimensions, 0-100. S_l is LongWriter's length score, 0-100. Complete means the run ended
with EOS before the cap.

`longbench_write/summary.csv` has the test and dev tables at full precision, `runs.csv` has one row per generation without generated
text (362 generations: feasibility, dev, test), `gaps.csv` has every gap below and per coefficient, and `latency.csv` has the
latency runs.

### Test, 10 prompts

| Arm | c | Formality | Beginning | Middle | End | Quality | S_l | Words | Complete | Truncated | Repeated 4-grams | Contractions / 100 words | KL |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `base` |  | 5.38 | 5.76 | 5.26 | 5.12 | 66.5 | 48.2 | 1958 | 0.90 | 0.10 | 0.267 | 0.40 | 0.000 |
| `prompt_formal` |  | 5.57 | 6.22 | 5.29 | 5.20 | 64.0 | 44.3 | 1460 | 0.90 | 0.10 | 0.225 | 0.43 | 0.000 |
| `prompt_informal` |  | 2.90 | 2.23 | 3.10 | 3.36 | 51.6 | 38.5 | 1479 | 0.90 | 0.10 | 0.207 | 2.59 | 0.000 |
| `ordinary` | -1 | 2.19 | 2.88 | 1.73 | 1.96 | 21.9 | 33.9 | 4283 | 0.40 | 0.60 | 0.645 | 5.85 | 0.095 |
| `ordinary` | -0.75 | 3.17 | 4.47 | 2.55 | 2.49 | 35.5 | 47.5 | 4034 | 0.50 | 0.50 | 0.612 | 1.43 | 0.063 |
| `ordinary` | -0.5 | 4.54 | 5.47 | 4.52 | 3.62 | 47.7 | 45.3 | 2450 | 0.80 | 0.20 | 0.437 | 0.77 | 0.053 |
| `ordinary` | +1 | 5.98 | 6.21 | 5.63 | 6.10 | 75.2 | 57.2 | 1277 | 1.00 | 0.00 | 0.171 | 0.37 | 0.111 |
| `ordinary` | +1.5 | 6.49 | 6.56 | 6.38 | 6.53 | 70.9 | 74.4 | 2253 | 1.00 | 0.00 | 0.337 | 0.30 | 0.151 |
| `ordinary` | +2 | 6.58 | 6.75 | 6.45 | 6.56 | 58.2 | 72.3 | 2536 | 0.80 | 0.20 | 0.453 | 0.19 | 0.196 |
| `restore_kv` | -1 | 3.61 | 4.54 | 3.10 | 3.19 | 41.4 | 44.4 | 2729 | 0.70 | 0.30 | 0.451 | 1.47 | 0.090 |
| `restore_kv` | -0.75 | 4.37 | 5.44 | 4.05 | 3.63 | 51.2 | 34.5 | 2497 | 0.60 | 0.40 | 0.495 | 1.16 | 0.036 |
| `restore_kv` | -0.5 | 4.87 | 5.38 | 4.54 | 4.69 | 54.0 | 39.3 | 1104 | 0.80 | 0.20 | 0.378 | 0.55 | 0.019 |
| `restore_kv` | +1 | 5.74 | 6.17 | 5.46 | 5.59 | 69.2 | 63.1 | 1986 | 0.90 | 0.10 | 0.264 | 0.35 | 0.043 |
| `restore_kv` | +1.5 | 6.14 | 6.54 | 5.90 | 5.97 | 66.3 | 66.2 | 1349 | 1.00 | 0.00 | 0.248 | 0.30 | 0.095 |
| `restore_kv` | +2 | 6.13 | 6.48 | 6.03 | 5.87 | 68.4 | 64.5 | 2099 | 0.90 | 0.10 | 0.309 | 0.51 | 0.128 |
| `chunk4_kv` | -1 | 3.63 | 4.50 | 3.12 | 3.10 | 40.6 | 41.2 | 2393 | 0.90 | 0.10 | 0.540 | 4.47 | 0.000 |
| `chunk4_kv` | -0.75 | 3.85 | 5.13 | 3.22 | 3.18 | 44.5 | 64.2 | 1989 | 0.90 | 0.10 | 0.446 | 1.31 | 0.000 |
| `chunk4_kv` | -0.5 | 4.68 | 5.35 | 4.39 | 4.31 | 52.8 | 64.6 | 2696 | 0.70 | 0.30 | 0.454 | 0.90 | 0.000 |
| `chunk4_kv` | +1 | 6.03 | 6.36 | 5.83 | 5.92 | 70.2 | 76.8 | 1871 | 1.00 | 0.00 | 0.297 | 0.55 | 0.000 |
| `chunk4_kv` | +1.5 | 6.17 | 6.37 | 6.00 | 6.14 | 67.8 | 63.7 | 1808 | 0.90 | 0.10 | 0.317 | 0.41 | 0.000 |
| `chunk4_kv` | +2 | 6.35 | 6.50 | 6.29 | 6.28 | 65.2 | 74.4 | 2261 | 0.80 | 0.20 | 0.395 | 0.19 | 0.000 |
| `chunk32_kv` | -1 | 2.95 | 3.96 | 2.54 | 2.35 | 25.1 | 33.4 | 3772 | 0.60 | 0.40 | 0.628 | 2.54 | 0.000 |
| `chunk32_kv` | -0.75 | 3.56 | 4.69 | 3.16 | 2.82 | 39.9 | 41.3 | 3107 | 0.70 | 0.30 | 0.462 | 1.48 | 0.000 |
| `chunk32_kv` | +2 | 6.27 | 6.43 | 6.10 | 6.27 | 61.5 | 63.9 | 1542 | 1.00 | 0.00 | 0.312 | 0.28 | 0.000 |

`chunk4_kv` at -1 has 1 unscored run, excluded from its formality mean. `chunk32_kv` ran at +2, -0.75 and -1 only.

### Test: gap to the `ordinary` curve at matched achieved formality

Curve: `base` and `ordinary` on each side, linear interpolation. Arm minus curve; 95% bootstrap CI over prompts.

| Metric | Arm | Formal | Informal | Pooled |
|---|---|---:|---:|---:|
| Quality | `restore_kv` | -5.25 [-11.89, +1.10] | +1.94 [-4.98, +6.84] | -1.66 [-6.11, +2.59] |
| Quality | `chunk4_kv` | -5.72 [-13.61, +2.94] | +2.01 [-3.20, +7.13] | -1.86 [-7.58, +4.02] |
| Quality | `chunk32_kv` | -11.31 [-23.74, +1.23] | -3.21 [-8.24, +2.37] | -7.26 [-14.54, -0.24] |
| S_l | `restore_kv` | +5.11 [-6.76, +20.55] | -6.86 [-20.52, +6.05] | -0.87 [-10.04, +9.01] |
| S_l | `chunk4_kv` | +7.38 [-5.41, +20.24] | +10.31 [-1.18, +22.39] | +8.85 [-1.46, +19.10] |
| S_l | `chunk32_kv` | -2.99 [-18.48, +13.81] | -8.36 [-28.67, +14.55] | -5.68 [-19.01, +8.55] |
| End-third formality | `restore_kv` | -0.24 [-0.41, -0.04] | +0.32 [+0.07, +0.51] | +0.04 [-0.11, +0.19] |
| End-third formality | `chunk4_kv` | -0.16 [-0.28, +0.01] | +0.26 [-0.03, +0.52] | +0.05 [-0.14, +0.24] |
| End-third formality | `chunk32_kv` | -0.07 [-0.28, +0.16] | -0.01 [-0.30, +0.29] | -0.04 [-0.20, +0.15] |
| Complete | `restore_kv` | -0.05 [-0.20, +0.08] | -0.03 [-0.18, +0.10] | -0.04 [-0.13, +0.04] |
| Complete | `chunk4_kv` | -0.10 [-0.23, +0.03] | +0.14 [-0.02, +0.30] | +0.02 [-0.09, +0.14] |
| Complete | `chunk32_kv` | +0.00 [+0.00, +0.12] | +0.12 [-0.18, +0.40] | +0.06 [-0.09, +0.22] |

### Unsteered feasibility, 14 prompts x 2 seeds

28 generations. Truncated at 8,192: 0.11. Below 1/3 of target: 0.14. Below 1/2 of target: 0.21. Mean words / target 1.04. Mean S_l
56.8. Mean repeated 4-grams 0.269. Quality 67.6. Formality 5.29 (beginning 5.82, middle 5.03, end 5.07).

### Latency

H100, batch 1, 4,096 tokens. `ordinary_ref` is `ordinary` with the reference stream running.

| Arm | ms / token | Peak memory (GB) |
|---|---:|---:|
| `base` | 17.59 | 14.54 |
| `ordinary` | 17.53 | 14.54 |
| `ordinary_ref` | 35.17 | 14.76 |
| `restore_kv` | 35.92 | 14.76 |

## Notes

- Every Steer2Edit table is one seed. Mean-length standard errors on MATH-500 are 300-500 tokens, larger than most length
  differences between settings.
- MATH-500 Level 5 seed 64 combines two batches: 100 questions run first and the other 34 run later, with the same arm
  settings. Seeds 65 and 66 ran all 134 together.
- In `crest/aime_2022_2025/k1a2_seed64`, 90 records have `chunk` = `step_end_only|split_only` and 30 add `|hold_opener`.
  The method is the same; the label was added to the record later.
- Runs used different GPU types and slices, so `seconds` and `peak_mem_gb` are not comparable across batches.
- AIME answers may be in the models' training data.
