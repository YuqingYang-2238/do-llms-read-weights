# Answer key for lm_prompts/identify_game/full_inputs__real_weights.txt (do NOT show this to the LLM)

Ground truth for the network whose weights are in `prompt.txt`.

| Field | Value |
|---|---|
| Checkpoint | `logs/tdmpc2/123/peeking/models/final.pt` (TD-MPC2, `{"model": state_dict}`) |
| What is shown | encoder (`_encoder.state.*`) + policy prior (`_pi.*`) only, 16,678 parameters, 3 decimals. Dynamics, reward and Q heads are NOT shown. |
| Game | cellworld "BotEvade": a mouse-like prey agent must cross a hexagonal-cell arena with occlusions (world `clump01_05`) from the entrance at (0.05, 0.5) to the goal at (1.0, 0.5) while avoiding a robot predator that chases it when it has line of sight. Getting within 0.1 of the predator while both see each other is a capture ("puff", `obs.puffed`). This is the simulated version of the MacIver / Dombeck-lab mouse-vs-robot "cellworld" experiments. |
| Extra mechanic | "Peeking": action[2] > 0.5 while next to a wall or occlusion makes the prey stop for a step (`is_peeking`); input 11 is the AIV (information-gain) score of the current cell/heading from `data/AIV_alex_metric.npy`. |
| Reward (`reward.py::custom_reward`) | -2 on capture; +2 when goal distance < 0.1; -0.01*tanh(max(0, 0.5 - d)) when predator visible and closer than 0.5; peeking: +0.01*tanh(aiv/0.18) if aiv > 0.027 else -0.02; -0.001 per step. |
| Training config | `outputs/2025-10-10/16-12-41/.hydra/config.yaml`: seed 123, 100k steps, `model_size: 1` (enc_dim 64, latent 32, mlp_dim 96), mpc=false in config (but `act()` still runs the MPPI planner), predator/prey speed ratio 0.15, episode length 300, time_step 0.25. |
| Performance | `logs/tdmpc2/123/peeking/eval.csv`, last evals: episode reward ~587, success / survival rate 0.3-0.7. |
| Input names in code | prey_x, prey_y, prey_direction, predator_x, predator_y, predator_direction, prey_goal_distance, puffed, puff_cooled_down, finished, is_peeking, aiv, near_wall, near_occlusion |
| Action names in code | (destination_x, destination_y, peek) |

Scoring suggestion (as in the reference repo): map the `Game:` line to the truth by keywords.
Count as correct: predator-prey / pursuit-evasion / escape-to-goal / hide-and-seek style answers where a
controlled agent reaches a target while avoiding a chasing agent that can only see it with line of sight.
