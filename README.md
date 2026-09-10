# XR Teardown — Unitree `xr_teleoperate`

**5054241001 — Ustu Bina Syahdiba** · S1 AI Engineering · Individual Assignment 1

|                          |                                                                                                                                        |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| Subject                  | `xr_teleoperate` XR teleoperation framework for humanoid robots formerly known as `avp_teleoperate`                                         |
| Publisher / manufacturer | Unitree Robotics                                                                                                                       |
| Release or major update  | `v1.6` July 2026 (release note dated `2026.7.29`). Project first published 2024                                                          |
| Platform(s)              | Meta Quest 3 / 3S, PICO 4 Ultra Enterprise, Apple Vision Pro; robot side Unitree G1, H1, H2, R1   |
| How I examined it        | Documentation and source code, plus my own static measurement of the code at a pinned commit. No headset or robot was used.              |
| Hands-on date(s)         | Documentation and source only. Measurement run 8 Sep 2026 against commit `817fb00` (7 Sep 2026).                                         |

> **My claim in one sentence.** `xr_teleoperate` looks like a teleoperation tool but it is really a data-collection tool for training robot AI and I thought this was a teleoperation tool for a moment.

---

## 1. Device class

It targets 6-DoF standalone inside-out headsets with hand tracking and a WebXR browser (Quest 3 and 3S, PICO 4 Ultra Enterprise, Apple Vision Pro). The headset runs no robotics code at all. Connection is made by opening a URL in the headset browser and the entire control stack lives on a separate x86-64 Linux host plus the robot's onboard PC2. The headset contributes exactly three things:

1. Head and hand pose
2. A stereo display surface 
3. Wi-Fi

![Caption](assets/fig1.png)

![Caption](assets/fig3.jpeg)

There are three display modes: `immersive`, `ego`, and `pass-through`. They differ in how much of the operator's view is taken up by the robot's camera feed versus their own real room.

- **`immersive`**: The operator sees only the robot's stereo camera feed, filling their whole view. It looks just like VR. The cost is that the operator cannot see their own surroundings at all.
- **`ego`**: The operator sees their real room with the robot's camera feed shown only as a small window instead of filling their view.
- **`pass-through`**: The operator sees only their real room and the robot's camera feed is not shown.

The hardware document notes a stereo head camera "provides more immersion", and depth judgement for grasping depends on it. Because gaze is unavailable, v1.6 changed the default to a "head-yaw-relative arm reference" the operator's head yaw substitutes for where they are looking. This works, but the operator must turn their head to reorient the arm frame, and cannot glance. On Apple Vision Pro where eye tracking exists, the framework still does not use it because the WebXR path is the easiest path across all three vendors.

## 2. Input modality

Hand tracking is the default (`--input-mode=hand`); controller tracking is the alternative (`--input-mode=controller`). Head yaw sets the arm reference frame. Finger joints are mapped onto the robot's end-effector for the Dex3-1 hand, a thumb with 3 active DoF and index and middle fingers with 2 each. With `--motion`, locomotion is delegated to a separate device: an R3 remote in hand mode, or the controller joysticks. Session states (begin, record, stop, quit) is driven by pressing **r**, **s**, and **q** on the host computer's keyboard.

For a system whose purpose is recording dexterous manipulation demonstrations, controllers would cap the achievable dataset at pinch-and-release. The designers trusted hand tracking with the robot's fingers but not with stopping the robot. That is an accurate assessment of hand tracking, and it produces the failure below.

Three concrete modes, in order of how much they would cost:
1. The operator is blind to their own control panel because the recording is started and stopped with **s** on a terminal the operator cannot see while wearing a headset in `immersive` mode.
2. Under joint limits the solver will deliver the wrist position the operator asked for and quietly rotate the wrist to whatever is convenient. Nothing reports this. The operator sees the hand arrive and does not see that it arrived at the wrong angle until the grasp fails.
3. Latency is added for filtering the noise from the sensor.

I still don't what to change for this particular program cause I'm not so sure what changes I need to make.

![Caption](assets/fig2.png)

## 3. Use of AI

I searched the repository and its three runtime submodules for text-to-3D, upscaling, and texture synthesis and found nothing, which is consistent with a system whose visual content is a live camera.

| Where                     | What it does                                                                                                              | On-device or cloud                          | Cost it carries                                                                       | Source + the line I am relying on                                                                                                                                                                        |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Perception                | Head, wrist and hand skeleton tracking, performed by the headset's own runtime and read over WebXR                          | On-device, inside the headset (vendor)      | Vendor-locked accuracy; occlusion and lighting failures the framework cannot fix; biometric hand geometry leaves the headset every frame | Meta WebXR docs [2] — "The API exposes the poses of the 25 skeleton joints (shown in the figure below) in each of the user's hands"; repo codebase overview [1] — `television.py` "Captures head, wrist, and hand/controller data from XR devices using Vuer" |
| Rendering & delivery      | H.264 stereo video from the robot's head camera to the headset over WebRTC. No learned foveation, super-resolution or frame interpolation found | On-device (robot PC2 encoder) → local Wi-Fi | Bandwidth and encode latency in the operator's control loop; requires Wi-Fi 6 router   | Repo README [1] — "Configure SSL certificates for the televuer module so that XR devices (e.g., Pico / Quest / Apple Vision Pro) can securely connect via HTTPS / WebRTC"                                  |
| Interaction               | **No** speech, LLM or translation ships. A hook is provided for one: the IPC mode is documented as being for agent control  | n/a (not shipped)                           | n/a — but it marks the intended direction                                              | Repo README [1], `--ipc` parameter — "Allows controlling the xr_teleoperate program's state via IPC. **Suitable for interaction with agent programs.**"                                                    |
| Downstream learning (the actual payload) | Recorded episodes are converted to LeRobot format and used to train ACT, Diffusion Policy, and Pi0 vision-language-action policies | Cloud / offline GPU training, after the session | Operator time; disk; bystander imagery in published datasets                            | Repo codebase overview [1] — `episode_writer.py` "Used to record data for imitation learning"; `unitree_IL_lerobot` [3] — "`Train Pi0 Policy`" with `--policy.type=pi0` and `--dataset.repo_id=unitreerobotics/G1_Dex3_ToastedBread_Dataset` |

There is no AI in this system cause this system is used for dataset collecting.

## 4. Impact

Lower the cost of collecting real robot manipulation demonstrations on a humanoid, so that imitation-learning policies can be trained on hardware most labs cannot afford to instrument.

The robot's head camera records the whole room, the whole time a session runs, and everything it sees is saved as training data. Anyone who walks past ends up in a dataset that may later be published without being asked and without knowing a recording is happening. There is no setting that asks people nearby for consent. The headset tracks 25 points on each hand with two hands at the default 30 times per second, that's 1,500 hand-position readings every second, plus head position.

By default, the system assumes the operator has two hands that can move all their fingers because it maps human finger joints straight onto the robot's finger joints. Someone with one hand, or limited finger movement, has to switch to `controller` mode instead.

## 5. What I take from this

This system is not built so a human can be somewhere else but it is built so a human can be a labelled training example.

---

## References

1. Unitree Robotics. (2026). *xr_teleoperate — An Open-Source Teleoperation Framework and Data Collection Toolkit for Embodied Intelligence*. Source code and documentation at commit `817fb00`. https://github.com/unitreerobotics/xr_teleoperate, accessed 8 Sep 2026.
2. Meta Platforms. *WebXR Hand Tracking*. Meta Horizon developer documentation. https://developers.meta.com/horizon/documentation/web/webxr-hands/, accessed 8 Sep 2026.
3. Unitree Robotics. *unitree_IL_lerobot, LeRobot training validation and Unitree data conversion*. https://github.com/unitreerobotics/unitree_IL_lerobot, accessed 8 Sep 2026.
4. Qin, Y., Yang, W., Huang, B., Van Wyk, K., Su, H., Wang, X., Chao, Y.-W., & Fox, D. (2023). *AnyTeleop: A General Vision-Based Dexterous Robot Arm-Hand Teleoperation System*. Robotics: Science and Systems., origin of the `dex-retargeting` optimisers. https://yzqin.github.io/anyteleop/, accessed 8 Sep 2026.
5. Cheng, X., Li, J., Yang, S., Yang, G., & Wang, X. (2024). *Open-TeleVision: Teleoperation with Immersive Active Visual Feedback*. arXiv:2407.01512. https://arxiv.org/abs/2407.01512, accessed 8 Sep 2026. The upstream project `xr_teleoperate` acknowledges first.

## Figure credits

- Fig. 1 — System and wiring diagram. © Unitree Robotics, reproduced from the `xr_teleoperate` repository documentation [1] for academic commentary.![Caption](assets/fig1.png)
- Fig. 2 — System and wiring diagram. © Unitree Robotics, reproduced from the `xr_teleoperate` repository documentation [1] for academic commentary.![Caption](assets/fig2.jpg)
- Fig. 3 — A photo of me trying the VR headset.![Caption](assets/fig3.jpeg)

## AI-assistance disclosure

The title were chosen by me because I like humanoid robots a lot and I see the future of it. I used Claude to search for the references but did not use it to write the whole documentation. I used Claude to help me make the documentation easier to understand.

### What I disagreed with my AI assistant about

I need to double check the AI generation for helping me with this documentation because sometimes the AI halucinated and did a crucial mistake.