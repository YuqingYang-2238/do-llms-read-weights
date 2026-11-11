# Experiment `identify_game`: identify the game

**Question put to the model:** "The weights below belong to one policy network that was trained with reinforcement
learning to play a game in this arena. Which game was it trained on? Work from the weights."

The prompt describes the 14 inputs, the 3 actions and the network, never names the game, and ends with all 16,678
weights. Ground truth and the scoring rule are in [`lm_prompts/identify_game/answer_key.md`](../lm_prompts/identify_game/answer_key.md):
an answer is correct if it describes a controlled agent that reaches a target while avoiding a chasing agent that sees
it only with line of sight (predator-prey, pursuit-evasion, escape-to-goal, hide-and-seek as the evader).

## 1a. Seven models on `full_inputs__real_weights`

| model | game named | confidence | verdict |
|---|---|---|---|
| Kimi K3.1 Pro | goal-directed hide-and-seek / pursuit-evasion, agent as the evader | 85 % | correct |
| DeepSeek V4 Pro | predator-prey / cat-and-mouse pursuit-evasion, agent is the prey | 80 % | correct |
| Grok 4.7 | hide-and-seek (line-of-sight tag), agent as the **seeker** | 60 % | wrong role |
| GPT-6 Astra | goal-directed cat-and-mouse escape, predator-prey reach-avoid | 40 % | correct |
| Claude Opus 5.5 | tag, played as the evader against an autonomous chaser | 35 % | correct |
| Claude Haiku 5.5 | pursuit-evasion in an obstacle arena, reach a target while avoiding the chaser | 30 % | correct |
| Gemini 4 Argon | none: the whole 32,000-token output budget went to reasoning | - | no answer |

Five of seven correct. Taken alone, this looks like weight reading.

## 1b. Weight ablation: `real_weights`, `no_weights`, `shuffled_weights`

Same prompt, three versions of the weight block (every line before the block is byte-identical):

| condition | weight block |
|---|---|
| real | the 18 tensors |
| none | one line: "The weights of this network are not shown." |
| shuffled | each tensor's values randomly permuted within that tensor: same labels, shapes, line count and set of values, no structure |

| weights | model | game named | confidence | verdict | numbers quoted, and where they are found |
|---|---|---|---|---|---|
| real | Astra | predator-prey reach-avoid escape | 40 % | correct | 11 positional citations: 11 match the real block |
| real | Kimi | hide-and-seek, agent as evader | 85 % | correct | 43 values quoted per input: 35 are in the named row |
| none | Astra | "cannot be determined" | 0 % | no answer | - |
| none | Kimi | "not identifiable" | 0 % | no answer | - |
| shuffled | Astra | predator-prey evasion, hider side | 25 % | correct | 17 positional citations: **17 match the shuffled block, 0 the real one** |
| shuffled | Kimi | hide-and-seek, agent as evader | 74 % | correct | 57 values quoted: **40 in the shuffled row named, 4 in the real row** |

How the citation check works: a positional citation is text of the form
`E1[1,62] = -1.286`; it is compared with that entry in the prompt the model saw and in the real prompt (tolerance
0.0015). Kimi quotes values per input without positions, so values of at least 0.1 on a line starting "Input k" are
looked up in row k of E1 in both prompts.

In the shuffled block the structure the models describe does not exist. Norms of the 14 rows of E1 (one per input):

```
real      0.49 0.70 0.17 0.53 0.58 0.16 0.72 0.43 1.68 0.54 0.26 1.19 0.27 0.25
shuffled  0.71 1.46 0.52 0.62 0.69 0.53 0.50 0.57 0.57 0.51 0.58 0.66 0.49 0.76
```

Yet both shuffled answers tell an input-by-input story. Astra: "Input 1, own y: largest magnitude 1.286, the
outstanding E1 coefficient: `E1[1,62] = -1.286` ... exceptionally strong, concentrated dependence on vertical position."
Kimi: "Obstacle adjacency is especially striking: -0.497 and 0.324 in neighboring encoder units." Those values sit
where they do by chance.

**Reading:** the models transcribe the numbers faithfully; without a block they refuse; with a block of noise they give
the same answer as with the real weights.

## 1c. Input-definition ablation: `neutral_inputs` and `misleading_inputs`

Same weight block, question and answer format; only the list that defines the 14 inputs and 3 actions changes
(everything from "THE NETWORK" on is byte-identical).

| input definitions | what the model is told |
|---|---|
| `full_inputs` | arena, occlusions, line of sight, one other agent, the meaning of every input and action |
| `neutral_inputs` | "input k: a number from a to b" (ranges only); "what the environment does with the 3 action values is not stated"; no arena, no other agent |
| `misleading_inputs` | the full definitions, except: the controlled agent "is the pursuer"; input 7 is "the controlled agent caught the other agent"; input 6 is the distance to "its fixed home base". The weights are still those of the evading prey. |

Each cell: the game named, the confidence the model stated, and the verdict.

| input definitions | weights | GPT-6 Astra | Kimi K3.1 Pro |
|---|---|---|---|
| full | real | predator-prey escape, confidence 40 %, correct | hide-and-seek as evader, confidence 85 %, correct |
| full | shuffled | predator-prey evasion, confidence 25 %, correct | hide-and-seek as evader, confidence 74 %, correct |
| full | none | declines | declines |
| neutral | real | **"not identifiable from the supplied information"** | **robot sumo, confidence 58 %, wrong** |
| neutral | shuffled | "not identifiable" | **robot soccer, confidence 48 %, wrong** |
| neutral | none | declines | declines |
| misleading | real | **seeker side of hide-and-seek, confidence 35 %, wrong** | **pursuer in tag / cops-and-robbers, confidence 88 %, wrong** |
| misleading | shuffled | seeker / tagger, confidence 35 %, wrong | pursuer in predator-prey pursuit, confidence 87 %, wrong |

Quoted numbers stay accurate throughout: with `neutral_inputs` Astra gives 23 positional citations, all 23
correct, and Kimi quotes 48 values of which 42 are in the row it names.

**Reading:** with the meanings removed, the real weights give neither model the game. With the role inverted in the
text, both models follow the text against the weights, Kimi at 88 % confidence (85 % with the full definitions), and neither mentions any
conflict. Shuffling the weights changes none of these answers.

## Files

| what | where |
|---|---|
| prompts | `lm_prompts/identify_game/<input definitions>__<weights>.txt` with input definitions `full_inputs`, `neutral_inputs` or `misleading_inputs` and weights `real_weights`, `shuffled_weights` or `no_weights` (8 files; the misleading definitions were not run without weights) |
| answers | `lm_prompts/responses/identify_game/<prompt name>/<model>.md` and `<model>.reasoning.txt` |
| numbers | rows with `task = identify` in `results/summary.csv` |
