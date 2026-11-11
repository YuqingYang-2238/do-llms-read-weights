# Experiment `planted_bug`: find the planted bug

The first two experiments left the models a way to answer without the weights: the input definitions in the identify task,
the episode trace in the fix task. This experiment removes every such route.

**Design.** Take the trained network, corrupt exactly one tensor with one simple operation, and show the model the
game, the network, the planner, the edit format and the 18 tensors. The prompt says only:

> ... it worked: the prey survived 313 of 500 evaluation episodes (63 %). After training, exactly ONE of its 18 tensors
> was deliberately corrupted: a simple operation (such as a sign flip, a swap, a rescaling or a reset to zero) was
> applied to one row, one column, a few rows or columns, or the whole of that tensor. Nothing else was changed.
> ... find the corruption from the weights and undo it.

No episode trace, no network output, no result of the corrupted network, nothing that names the tensor or the
operation. The control is the same prompt with the corrupted values shuffled within each tensor.
The ground truth of each bug is in `lm_prompts/planted_bug/<bug>.planted.json`.

## Which corruptions were built

Six corruptions were built and the corrupted networks evaluated for 100 planner episodes (original: 63 %).

| bug | corruption | survived / 100 | used |
|---|---|---|---|
| `e1_row6_zero` | E1 row 6 (distance to the goal) set to 0 | 42 | yes |
| `e1_swap34` | E1 rows 3 and 4 (predator x and y) swapped | 53 | yes |
| `e1_row0_sign` | E1 row 0 (prey x) negated | 1 of 10, then stopped: every episode ran to the 300-step limit | no, too slow to evaluate |
| `p3_col0_sign` | P3 column 0 (x destination) negated | 63 | no, does not hurt |
| `p3_col1_sign` | P3 column 1 (y destination) negated | 67 | no, does not hurt |
| `p3_swap01` | P3 columns 0 and 1 swapped | 74 | no, does not hurt |

Corrupting the policy head does not hurt at all: the planner's own samples and the hidden Q models find the goal
without the policy's proposals. Corrupting the encoder does, because every hidden model then receives latents it was
not trained on. The two bugs used differ in kind: a zeroed row is visible at a glance (64 entries reading `0.000`);
a swap leaves every value in place and only exchanges two rows.

## Results: was the corruption located?

"Located" means the model's `edits` block writes the planted entries. Prompts for all six bugs are in the repo; only
the two bugs above were sent to the models (8 answers, $8.45).

| bug | weights | model | what the model claimed (its stated confidence) | corrupted weights its edit rewrote | other weights its edit rewrote | located |
|---|---|---|---|---|---|---|
| zeroed row | real | Astra | "E1, row 6, columns 0-63: the entire row was reset to zero" (99 %) | **64 of 64** | 0 | **yes** |
| zeroed row | real | Kimi | "E1[6, :], all 64 weights, were reset to zero" (99 %) | **64 of 64** | 0 | **yes** |
| zeroed row | shuffled | Astra | "scattered entries of E1 appear to have been reset to zero" (40 %) | 6 of 64 | 2 | no |
| zeroed row | shuffled | Kimi | "P3, column 1 was rescaled by approximately 2" (58 %) | 0 of 64 | 96 | no |
| swapped rows | real | Astra | "E1, row 1 (prey y) was multiplied by -1" (40 %) | 0 of 128 | 64 | no |
| swapped rows | real | Kimi | "P2, the entire column 5: its signs were flipped" (32 %) | 0 of 128 | 96 | no |
| swapped rows | shuffled | Astra | "Undetermined" (0 %), empty edits block | 0 of 128 | 0 | no |
| swapped rows | shuffled | Kimi | "E1[1, 62] was amplified approximately tenfold" (82 %) | 0 of 128 | 1 | no |

Notes on individual answers:

- **Zeroed row, real weights.** Both models point at the row immediately. Astra: "All 64 entries in this row are
  `0.000`, unlike every other E1 row." The original values cannot be recovered from a zeroed row, and both say so; each
  refills the row with a guess built from the prey-x row (Astra `1.5*e1 - E1[0,:]`, Kimi `-E1[0,:]`).
- **Zeroed row, shuffled weights.** The 64 zeros are now scattered over E1. Astra notices the excess ("I count 76
  entries displayed as `0.000` in E1") and edits 8 single entries in rows 0 and 6, which it picks from the meaning of
  the inputs; 6 of them happen to be planted entries. Kimi leaves E1 and blames P3.
- **Swapped rows.** Nobody finds it, with real or shuffled weights. The guesses (a negated row, a sign-flipped column,
  one large entry) are of the same "this looks unusual" kind in both conditions. The 82 % that Kimi assigns to
  `E1[1,62]` in the shuffled block is the entry -1.286, the same number both models singled out in the shuffled
  identify prompt.

**Reading.** This is the one experiment where the real weights made a difference: a zeroed row is found in the real
block and not in the shuffled one. But it is found by looking, not by computing. The corruption that requires relating
the weights to what the inputs mean, the swap, is not found at all.

## Not reported: survival after the repair

Whether the repaired networks survive as often as the trained one was meant to be measured on 500 planner episodes.
Those runs were stopped before completion and are left out of this write-up. The location results above do not depend
on them.

## Files

| what | where |
|---|---|
| prompts and ground truth | `lm_prompts/planted_bug/<bug>__real_weights.txt`, the shuffled control `<bug>__shuffled_weights.txt`, and `<bug>.planted.json` |
| answers | `lm_prompts/responses/planted_bug/<bug>__real_weights/` and `<bug>__shuffled_weights/` for `e1_row6_zero` and `e1_swap34` |
| numbers | rows with `task = planted bug` in `results/summary.csv` |
