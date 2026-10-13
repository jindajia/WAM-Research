# Acting on moving objects with world-action models

## The problem
Generative robot policies take close to a second to produce actions, so a moving target has already moved by the time they act. We study **Cosmos3-Nano-Policy-DROID**, a world-action model that generates future video and a 32-step action chunk together; its policy post-training uses real-robot DROID data. All tests below are simulated, with one scene each and a few seeds.

## 1. The policy's predicted video largely ignores observed motion
A ball rolls across a table. The base video model **Cosmos3-Nano** and the policy checkpoint receive the **same five observed frames** (15 fps, 0.27 s) and predict the next 1.6 s. Reversing the history direction shifts base Nano's predicted ball by **+74 to +88 px** (reality: about +80 px), but DROID's by only **−9 to +17 px** (3 seeds × 2 camera views).

![Base video model vs robot policy vs ground truth, ball rolling right and left](media/rolling_ball_comparison.gif)

*Cyan ring = last observed ball position. Matched comparison settings (35 denoising steps, guidance 3, video only), not the four-step deployment settings used below.*

## 2. The motion is still readable inside the policy
Linear probes on the frozen DROID checkpoint recover the ball's velocity during its normal four-step request, on held-out positions and an **unseen speed**, while a repeated-frame control fails. The features are available about **0.2–0.3 s** into a roughly one-second five-frame request. This shows the information is *decodable*, not that the policy *uses* it; it is not evidence of physical understanding or a causal mechanism.

## 3. Five-frame input and what it costs
We extended the official single-observation path to accept **five frames** through the framework's existing conditioning machinery, without retraining. This is an experimental input extension outside the policy's fine-tuning setting, not a demonstrated improvement.

| | single frame (official) | five frames (ours) |
|---|---|---|
| warm policy call, until actions are on the CPU | 0.80 s | 1.03 s |
| VAE encoding of the observation | 9.6 ms | 152 ms |
| output | 32 × 8 action chunk | 32 × 8 action chunk |

*One H100, four denoising steps, guidance 3, shift 5. Median of 20 warm in-process calls; video decoding and network round trip excluded. VAE time is the synchronized stage time.*

Profiling shows denoising takes about 92% of a single-frame request, and disabling video decoding still leaves the video latents being denoised with the actions.

## 4. Executing the actions in simulation
We executed the policy's actions in the official RoboLab / Isaac Sim `BananaInBowlTask` (our evaluation environment).
- **Initial static benchmark:** 3/3 successes.
- **Anticipation test** (banana in a single pass at 0.05 m/s): single-frame and five-frame inputs both grasped **0/6**, with no consistent anticipatory positioning. From the same start, a static banana was grasped 2/3 and 3/3, and a scripted controller using ground-truth motion grasped the moving banana in both directions.

These tests are small (3 seeds per condition) and do **not** show that inference latency caused the failures.

[![Predicted video vs executed actions, banana moving](media/moving_banana_pred_vs_exec.gif)](media/moving_banana_pred_vs_exec.mp4)

*First predicted video (left) vs executed actions (right); top single-frame, bottom five-frame, same latest image. Cyan circle = banana at the first observation: it stays put in both predictions but slides away in reality. Click for the MP4.*

## Research direction
**Can motion information that becomes available early inside a world-action model be used to update or correct actions that are still being generated, or not yet executed, as fresh observations arrive?**

This is a direction to investigate, not an implemented method: the motion signal appears early, a fresh single-frame encode costs about 10 ms, yet the policy currently neither predicts nor acts on the motion.

## Collaboration
I am looking for **access to a robot arm** for a small experiment with a slowly moving object, and welcome collaboration where interests overlap.

*Detailed reports (motion study, profiling, grasping experiments) are in a private repository and available on request:
[motion study](https://github.com/jindajia/WAM-Latency-Research/blob/main/REPORT.md) ·
[profiling](https://github.com/jindajia/WAM-Latency-Research/blob/main/PROFILING.md) ·
[grasping](https://github.com/jindajia/WAM-Latency-Research/blob/main/reports/8_droid_grasp_pilot/REPORT_droid_grasp_pilot.md).*
