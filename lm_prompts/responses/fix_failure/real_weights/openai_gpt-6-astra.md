---
model: openai/gpt-6-astra
provider: OpenAI
prompt: lm_prompts/fix_failure/real_weights.txt
prompt_sha256: 580b7a9e72af64eac3dcdb5d08f6c2da1c3a859167a7a576b76387aba47a48b0
prompt_chars: 125461
reasoning_effort: high
timestamp: 2026-11-09T15:53:39
elapsed_s: 373.4
finish_reason: stop
prompt_tokens: 73561
completion_tokens: 11420
reasoning_tokens: 10876
cost_usd: 1.490505
---
The visible failure is **not a lack of time to outrun the predator**. At step 3, the policy proposes almost exactly the dangerous goalward route:

- \(m_x=7.81\) saturates `tanh`; even its substantial pre-tanh noise scarcely changes the destination’s x-coordinate.
- \(m_y=0.59\), with standard deviation `0.006`, gives almost no lateral alternatives. The policy proposals therefore reinforce the direct route instead of providing meaningful detours.
- The high peeking output cannot help in that cell.

The output weights also contain a pronounced downward/peeking mode: unit 85 has LayerNorm scale `2.969`, y weight `-1.116`, and peeking weight `0.725`. This can overwhelm the otherwise modest y-output weights, consistent with the pathological initial outputs.

**The planner’s scoring is also implicated:** it accepted the unsafe route despite having independent random candidates. Its hidden models are unavailable, so changing the policy cannot guarantee that safer proposals win. I would preserve the encoder completely rather than invalidate those models, and repair the policy’s proposal distribution: soften saturated forward destinations, remove the dominant downward mode, restore substantial lateral diversity, and stop policy-generated peeking.

```edits
P3[:, 0] *= 0.20
b3[0] = 0.2682
P3[:, 1] *= 0.50
P3[85, 1] = 0
b3[1] = 0.45
P3[:, 2] = 0
b3[2] = 0.10
P3[:, 3:6] = 0
b3[3:6] = [0.60, 0.62, -3.00]
```

These edits leave the entire encoder and both policy hidden layers unchanged.

The x-logit becomes `0.20 * old_x_logit + 0.25`: still strongly goalward, but no longer effectively immovable under sampling. The y readout retains its other learned features while removing unit 85’s extreme downward contribution and recentering ordinary goalward outputs.

Pre-tanh sampling standard deviations become approximately `0.46`, `0.50`, and `0.000047`. This supplies forward-moving lateral alternatives instead of nearly identical direct-goal proposals. Policy-generated peeking actions stay near `0.10`, inside the legal range and below the stopping threshold.

This is an untested proposal-distribution repair, not an enforced capture-radius constraint. The unchanged planner still determines which routes are executed.
