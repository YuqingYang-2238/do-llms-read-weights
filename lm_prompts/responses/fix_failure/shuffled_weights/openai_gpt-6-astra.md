---
model: openai/gpt-6-astra
provider: OpenAI
prompt: lm_prompts/fix_failure/shuffled_weights.txt
prompt_sha256: 35f291bbfab02ef741797efcfed1cc130f14a22cd6aac3141aa9adc0c26d52a3
prompt_chars: 125461
reasoning_effort: high
timestamp: 2026-11-10T14:03:49
elapsed_s: 390.8
finish_reason: stop
prompt_tokens: 73575
completion_tokens: 10940
reasoning_tokens: 10358
cost_usd: 1.46668
---
The visible failure is **goal-directed action collapse without adequate lateral alternatives**, not an inability to outrun the predator.

- Predator coordinates have nonzero encoder weights; they are not disconnected. Nevertheless, at the critical decision the policy produces \(m=(7.81,0.59,4.69)\): essentially maximum destination x, destination y \(0.527\), and peeking. That destination takes the prey through the capture region.
- The y standard deviation is only \(0.006\). Consequently, the policy’s 24 proposed sequences offer almost no lateral variation at precisely this decision. Large opposing y-readout weights—including `P3[83,1] = -1.222` and `P3[87,1] = -1.116`—also accompany the observed, severely saturated negative-y outputs earlier in the episode. Merely increasing sampling variance modestly would not overcome those earlier means.
- Peeking cannot protect the prey at the critical position, and peek-heavy proposals can waste escape opportunities near obstacles. The navigation system avoids solid cells, **not the predator’s capture circle**.

The planner also selected the unsafe route, so these observations do not establish that proposal collapse is the only fault: its unshown reward, dynamics, or Q models may misrank trajectories. I would not rewrite their encoder to manufacture a danger signal. Instead, I preserve the encoder, both policy hidden layers, and the entire x-output behavior, while repairing lateral proposal coverage and removing stationary policy proposals.

```edits
P3[:, 1] *= 0.01
b3[1] = 0.80111
P3[:, 4] = 0
b3[4] = 0.595
P3[:, 2] = 0
b3[2] = 0.1
P3[:, 5] = 0
b3[5] = -1
```

The new lateral mean is exactly `0.8 + 0.01 * old_m_y`, before tanh. This removes the extreme negative-y saturation while retaining a small learned adjustment. At the critical observation, its mean destination y becomes approximately **0.667**, with pre-tanh standard deviation approximately **0.45**. Thus, proposals include substantial upper and lower detours, as well as destinations near goal height, rather than repeatedly proposing the same collision course.

The peek output becomes approximately **0.10**, with negligible noise, so policy proposals keep moving. The planner’s 488 Gaussian candidates remain unrestricted, including their ability to finish at the goal or select peeking independently.

These are untested edits, not a guaranteed clearance controller. Their intended benefit is better maneuver sequences without disrupting the latent representation on which the working planner depends; an erroneous hidden trajectory scorer can still reject them.
