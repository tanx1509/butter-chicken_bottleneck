# Bottleneck RL

A reinforcement learning agent that learns to meter traffic at a 3-to-1 lane merge and beats a hand-coded controller by 31 percent on throughput.

**Result:** +36.6% throughput and -14.2% waiting time over the no-control baseline on 15 held-out seeds. The trained agent rediscovered bang-bang control from scratch without being told that such a strategy exists in the traffic engineering literature.

**Live demo:** [butterchicken-bottleneck.netlify.app](https://butterchicken-bottleneck.netlify.app)

**Full technical report:** [Google Docs link](https://docs.google.com/document/d/1CYTFOBWig_5cYO9JtnLQAWumbojER2tBeFj6EHvlXdg/edit?usp=sharing)

---

## Headline numbers

Fifteen-seed held-out evaluation. Same seeds across all three controllers.

| Metric | Baseline | Rule-based | PPO (ours) | Improvement |
|---|---|---|---|---|
| Throughput (veh/step) | 0.388 | 0.404 | **0.530** | **+36.6%** |
| Avg waiting time (steps) | 158.5 | 157.1 | **135.9** | **-14.2%** |
| Max congestion length (cells) | 180.0 | 128.4 | 168.9 | -6.1% |

For context: peer-reviewed ramp-metering deployments in California, Minnesota, and the Netherlands report throughput gains of 5 to 15 percent. This simulator has a harder capacity drop than real roads, so there is more headroom for recovery. The relative improvement is meaningful, the absolute number is simulator-specific.

---

## What the agent learned

The trained policy uses only two of five available actions:

- **Cap velocity at 2** for 316 out of 500 steps (when merge zone is dense)
- **Full stop** for 184 out of 500 steps (when the downstream has cleared)

This is a bang-bang control pattern, a named strategy in traffic engineering literature. When the downstream is loaded, throttle hard. When it clears, release a platoon. The agent discovered this on its own. It was handed a reward function and a simulator, nothing else.

---

## What is in this repo

| File | Purpose |
|---|---|
| `ButterChicken.ipynb` | Self-contained notebook with simulator, EDA, training, evaluation, plots |
| `visualization.html` | Standalone browser animation (also deployed on Netlify) |
| `models/ppo_bottleneck_final.zip` | Trained PPO model (Stable-Baselines3 format) |
| `results/*.png` | Nine plots covering EDA, controller comparison, learning curve, final metrics |
| `results/final_results.json` | All numerical results in machine-readable form |
| `bottleneck_rl_convoke.pptx` | Presentation deck |
| `requirements.txt` | Python dependencies |

---

## The approach

**The problem.** When a multi-lane road narrows, drivers abandon lane discipline and panic-merge late. The interference between merging vehicles drops throughput 15 to 30 percent below the single-lane theoretical capacity. Traffic engineering calls this the capacity drop. It is not a capacity problem, it is a behaviour problem under spatial constraint.

**The simulator.** Custom Nagel-Schreckenberg cellular automaton. 3 lanes by 250 cells. Lanes 0 and 1 end at cell 180. Vehicles have integer positions and velocities, heterogeneous aggressiveness, and obey NS physics with random deceleration. The capacity drop is modeled explicitly: when merge zone density exceeds 0.35, vehicles in the zone are forcibly slowed. Without this mechanic, the bottleneck is geometry-limited and no controller can help.

**The rule-based meter.** A density-aware preemptive throttle. Every step, measures merge zone density and caps vehicle velocities upstream before the critical density is reached. Gives +4.1% throughput and -28.7% congestion length over baseline.

**The PPO agent.** Stable-Baselines3 PPO. 10-dimensional observation (per-lane density and velocity, merge zone density, exit density, recent throughput averages). 5 discrete actions (no cap, cap at 3, 2, 1, full stop). Reward shaped to break the "do nothing" local optimum that collapsed two earlier training runs.

- **Training:** 150,000 timesteps, entropy coefficient 0.05, learning rate 3e-4, CPU only
- **Evaluation:** 15 held-out seeds, deterministic action selection
- **Wall clock:** 7 minutes

---

## Reproduce

```bash
git clone https://github.com/tanx1509/bottleneck-rl.git
cd bottleneck-rl
pip install -r requirements.txt
jupyter notebook ButterChicken.ipynb
```

Runs top to bottom in about 10 minutes on CPU. No GPU required.

Stack: Python 3.10+, NumPy, Matplotlib, Gymnasium, Stable-Baselines3, pandas.

---

## Honest observations

**Congestion length stayed roughly flat for PPO.** The agent uses the upstream road as a buffer: it holds a long queue upstream in exchange for keeping the merge zone clear. The reward weights throughput and waiting time, not congestion length. A different reward would give a different policy.

**Training was sensitive to reward shaping.** Two earlier training runs with a simpler reward collapsed onto a "do nothing" policy. Reward shaping with explicit density bonuses and a passivity penalty was needed to break the local optimum. Standard RL practice but worth stating.

**The learning curve is visibly noisy late in training.** Entropy coefficient held at 0.05 throughout to prevent premature convergence. Deterministic evaluation is stable.

**This is a simulator, not a real system.** Relative improvements are meaningful. The absolute +36.6% is specific to this simulator's capacity drop coefficients.

---

## Context

Built for Convoke 8.0 KnowledgeQuarry, Problem 1, ML Engineering track. University of Delhi, Cluster Innovation Centre.

**Placed 1st.**

---

## License

MIT
