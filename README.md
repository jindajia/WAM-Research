# Acting on moving objects with world-action models

## The problem
Large generative robot policies take close to a second to produce an action chunk. If the target keeps moving, the actions come from an observation that is already stale. A policy that tracks the object's motion could compensate; one that doesn't will aim at where the object *was*. We study this with **Cosmos3-Nano-Policy-DROID**, a world-action model that generates future video and a 32-step action chunk together. Its policy post-training uses real-robot DROID data. Everything below uses simulated scenes, a few seeds, and one scene per test.

## 1. The policy's predicted video largely ignores observed motion
A red ball rolls across a table. Two models receive the **same five observed frames** (15 fps, about 0.27 s) and predict the next 1.6 s of video:
- **Cosmos3-Nano**, a general video model from the same family;
- **Cosmos3-Nano-Policy-DROID**, the robot-policy checkpoint.

**Result:**
- **Base Nano follows the motion.** Reversing the history direction shifted its predicted ball by **+74 to +88 px**, against about +80 px in reality (3 seeds × 2 camera views).
- **DROID barely responds.** The same reversal shifted its prediction by only **−9 to +17 px**.

![Base video model vs robot policy vs ground truth, ball rolling right and left](media/rolling_ball_comparison.gif)

*Same five-frame input to both models; cyan ring = the ball's last observed position. Matched video-comparison settings: 35 denoising steps, guidance 3, video only. These differ from the four-step deployment settings used below.*

## 2. The motion is still readable inside the policy
Linear probes on the frozen DROID checkpoint recovered the ball's velocity during its normal four-step request.
- **Held-out data:** the probes worked on held-out ball positions and on an **unseen speed** (±0.30 m/s, correct sign every time).
- **Control:** they failed on a repeated-frame control, as they should.
- **Timing:** the features are available about **0.2–0.3 s** into a roughly one-second five-frame request.
- **Limits:** this shows the information is *decodable*, not that the policy *uses* it. It does not demonstrate physical understanding or identify a causal mechanism. Layer-by-layer comparisons and an activation-patching test are in the detailed report.

## 3. Five-frame input and what it costs
The official path takes a **single observation**. We extended it to accept **five observation frames** through the framework's existing conditioning machinery, without retraining. This is an experimental input extension, not a demonstrated improvement: five frames are outside the policy's fine-tuning setting.

| | single frame (official) | five frames (ours) |
|---|---|---|
| warm policy call, until actions are on the CPU | 0.80 s | 1.03 s |
| VAE encoding of the observation | 9.6 ms | 152 ms |
| output | 32 × 8 action chunk | 32 × 8 action chunk |

*One H100, four denoising steps, guidance 3, shift 5. Median of 20 warm in-process calls; video decoding and network round trip excluded. VAE time is the synchronized stage time.*

**Profiling:** denoising takes about 92% of a single-frame request. Turning video decoding off still leaves the video latents being denoised alongside the actions; they are 99% of the generated tokens.

## 4. Executing the actions in simulation
We also ran the policy's actions in the official RoboLab / Isaac Sim `BananaInBowlTask`, our evaluation environment.
- **Static banana (initial benchmark):** the official setup succeeded **3/3**.
- **Anticipation test:** the banana moves in a single pass at 0.05 m/s, so reaching for its last observed position misses.
  - **Policy:** single-frame and five-frame inputs both grasped **0/6**. Neither showed consistent anticipatory positioning: the gripper came down where the banana had last been seen.
  - **Same start, static banana:** the policy succeeded 2/3 (single-frame) and 3/3 (five-frame).
  - **Scripted controller:** one using ground-truth motion to aim ahead grasped in both directions.
- **Limits:** the tests are small (3 seeds per condition). They do **not** show that inference latency caused the failures.

[![Predicted video vs executed actions, banana moving](media/moving_banana_pred_vs_exec.gif)](media/moving_banana_pred_vs_exec.mp4)

*Each row compares the policy's first predicted video (left) with what actually happened when its actions were executed (right). Top: single-frame input; bottom: five-frame input. Both rows share the same latest image (cyan circle = banana at the first observation). In both predictions the banana stays put while the gripper descends. In reality it slides away. Click for the MP4.*

## Research direction
**Can motion information that becomes available early inside a world-action model be used to update or correct actions that are still being generated, or not yet executed, as fresh observations arrive?**

Our results make this a concrete question, not a solved one:
- **Available early:** the motion signal appears early in the request.
- **Cheap to add:** a single-frame encode costs about 10 ms of a 0.8 s call.
- **Not used:** the policy currently neither predicts nor acts on that motion.

Next steps are to test whether that signal can steer pending actions, first in simulation and then on a real arm with a slowly moving object.

## Collaboration
I am looking for **access to a robot-arm setup** for a small experiment with a slowly moving object, and I would welcome collaboration where interests overlap.

*Detailed reports (motion study, profiling, grasping experiments) are in a private repository and available on request:
[motion study](https://github.com/jindajia/WAM-Latency-Research/blob/main/REPORT.md) ·
[profiling](https://github.com/jindajia/WAM-Latency-Research/blob/main/PROFILING.md) ·
[grasping](https://github.com/jindajia/WAM-Latency-Research/blob/main/reports/8_droid_grasp_pilot/REPORT_droid_grasp_pilot.md).*
