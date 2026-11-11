# How this was made

This repo shows results. It holds the prompts exactly as sent, the answers exactly as received, and the numbers. It
does not hold code, the trained network or the game, so nothing here is meant to be re-run. This page records how
each piece was produced, so that the results can be read with the method in mind.

## Prompts

- **Real weights.** The 18 tensors of the trained encoder and policy head were written as text, rounded to 3 decimals,
  one matrix row per line. A forward pass computed from the rounded text was checked against the original network
  (differences of about 0.01 or less in the actions).
- **Shuffled weights.** Each tensor's values were randomly permuted within that tensor (flattened, permuted with a
  fixed random seed, reshaped). Labels, shapes, line count and the set of values per tensor are unchanged; every line
  of the prompt before the weight block is byte-identical to the real-weights prompt.
- **No weights.** The weight block is replaced by the single line "The weights of this network are not shown."
  Everything before it is byte-identical.
- **Neutral and misleading input definitions.** Only the list that defines the inputs and actions was rewritten.
  Everything from the description of the network onward, including the question, the answer format and the weight
  block, is byte-identical to the full-definitions prompt.
- **Fix prompts.** The failure shown is a recorded episode of the trained network with its planner. The observations
  in the trace were rebuilt from the simulation log and checked against the logged values.
- **Planted bugs.** One tensor of the trained network was changed by one operation, and the corrupted weights were
  written out as above. The exact entries changed are in `lm_prompts/planted_bug/<bug>.planted.json`.

## Answers

Each prompt was sent once per model through OpenRouter as a single user message: no system prompt, no history, no
tools, reasoning effort high, provider-default temperature. The answer and, where the provider returned it, the
reasoning trace were saved unedited. Each answer file starts with a header giving the model, the prompt file and its
SHA-256, token counts, cost and finish reason; `lm_prompts/responses/index.csv` has one row per request.

## Scoring

- **Identify the game.** Whether an answer is correct was judged by hand with the rule in
  `lm_prompts/identify_game/answer_key.md`. The confidence is the number the model wrote.
- **Citation check.** A citation of the form `E1[1,62] = -1.286` was compared with that entry in the prompt the model
  saw and in the real-weights prompt (tolerance 0.0015). Where a model quotes values per input without positions,
  values of at least 0.1 on a line starting "Input k" were looked up in row k of E1 in both prompts.
- **Fix the failure.** The last `edits` block of an answer was applied to the real trained network, in every
  condition, and the edited network was run for 500 episodes with the planner. An episode counts as survived if the
  prey reaches the goal with zero captures. Intervals are Wilson 95 % intervals; comparisons are two-proportion tests.
- **Planted bug.** "Located" means that every corrupted entry is among the entries the model's `edits` block writes.

## `results/summary.csv`

One row per (experiment, condition, model). It is the single source of every number in the README, the docs pages and
the figures. The survival counts in it come from the per-episode evaluation logs, which are not part of this repo.

| column | meaning |
|---|---|
| `experiment`, `task`, `prompt`, `input_definitions`, `weights`, `variant`, `model` | which cell of the study the row is; `prompt` is the prompt file under `lm_prompts/` without `.txt`, `variant` is the bug of a planted-bug row or the trace version of a fix row |
| `outcome`, `confidence_pct`, `answer` | identify and planted-bug rows: the verdict (judged by hand with the answer-key rule), the stated confidence, the model's own wording |
| `citations_*`, `quotes_*` | how many quoted weights were found, and how many match the prompt shown / the real prompt |
| `n_episodes`, `survived`, `survived_pct`, `ci_low_pct`, `ci_high_pct` | planner evaluation and its Wilson 95 % interval |
| `episodes_with_capture`, `mean_captures`, `mean_steps` | further outcome details of the same run |
| `edit_statements`, `edits_entry_or_row`, `edits_column_or_tensor` | size and kind of the `edits` block |
| `location_hit`, `planted_touched`, `planted_total`, `stray_entries` | planted bug: did the edit write the corrupted entries |
| `cost_usd`, `answer_file` | what the answer cost and where it is |
