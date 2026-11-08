# Experiment results

Graded runs from re-encoding experiments on CREST and Steer2Edit. Results only, no running code.

Re-encoding replaces steered state in the KV cache with clean state during generation, so steering at one step does not
carry through every later step.

## Files

| Path | Contents |
|---|---|
| `summary.csv` | One row per setting (and per seed for CREST): n, correct, mean tokens, cap hits |
| `runs.csv` | One row per run, without generated text |
| `crest/math500_level5/` | `paper` and `k1a2`, seeds 64-66, 134 questions each |
| `crest/aime_2022_2025/` | `paper` and `k1a2`, seeds 64-66, 120 problems each |
| `steer2edit/math500_100q/` | Trade-off curve and entropy gate, 100 questions, seed 64 |
| `steer2edit/aime_2022_2024_60q/` | AIME, first 60 of a frozen random order of 2022-2024, seed 64 |

Each `.jsonl.gz` has one JSON record per run, including the generated `text`, `prediction`, `gold` and `correct`.
Token ID lists were removed to save space. `runs.csv` has 3,958 runs. `uid` identifies a run.

`tokens` in the CSVs means `n_gen` for CREST and `reasoning_length` for Steer2Edit, the length metric used in each
project's readout.

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
| 0.65 | 80 / 2,782 | 91 / 2,993 | not included |
| 0.70 | 89 / 2,973 | 92 / 3,069 | not included |
| 0.75 | 91 / 2,918 | 92 / 3,504 | not included |
| 0.80 | 92 / 3,107 | 98 / 3,410 | 97 / 3,215 |
| 1.00 | 96 / 3,439 | not run | |

| rho, alpha 0.20 | `edit` | `edit_re` |
|---:|---:|---:|
| 0.65 | 69 / 2,457 | 59 / 2,213 |
| 0.80 | 84 / 3,726 | 81 / 3,528 |

### AIME: correct / average tokens / cap hits, of 60 problems

Vanilla: 45 / 20,450 / 10.

| rho, alpha 0.05 | `edit` | `edit_re` |
|---:|---:|---:|
| 0.65 | | 20 / 15,928 / 4 (58 problems) |
| 0.70 | 18 / 14,008 / 2 | |
| 0.80 | 30 / 18,124 / 7 | 34 / 17,854 / 8 |
| 1.00 | 40 / 18,557 / 11 | 42 / 18,987 / 10 |

`edit_ent` at rho 1.00 is partial here: 16 of 60 problems, 12 correct. These are not comparable with the 60-problem rows.

## Not included yet

These ran on another cluster, so their files are not in this repo:

- AIME `edit_ent` rho 1.00, remaining 44 problems. Reported as 46/60 at about 18,200 tokens; not checked against run files here.
- MATH-500 `edit_ent` at rho 0.65, 0.70 and 0.75.

## Notes

- Every Steer2Edit table is one seed. Mean-length standard errors on MATH-500 are 300-500 tokens, larger than most length
  differences between settings.
- MATH-500 Level 5 seed 64 combines two batches: 100 questions run first and the other 34 run later, with the same arm
  settings. Seeds 65 and 66 ran all 134 together.
- In `crest/aime_2022_2025/k1a2_seed64`, 90 records have `chunk` = `step_end_only|split_only` and 30 add `|hold_opener`.
  The method is the same; the label was added to the record later.
- Runs used different GPU types and slices, so `seconds` and `peak_mem_gb` are not comparable across batches.
- AIME answers may be in the models' training data.
