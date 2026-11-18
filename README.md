**English** | [中文](README.zh-CN.md)

# JAKA RL Lab

JAKA RL Lab is an Isaac Lab reinforcement-learning project for the **JAKA Pi 27-DoF humanoid robot**. It includes the robot USD and MuJoCo assets, locomotion and motion-imitation tasks, a local RSL-RL extension, training and evaluation scripts, and a MuJoCo sim-to-sim deployment program that can be run directly.

The current release migrates robot assets and task identifiers from the previous Khan mini / K1L names to **JAKA Pi**, and updates the joint configuration, control gains, observations and rewards, domain randomization, sensor delay, symmetry augmentation, AMP data, and deployment policies accordingly.

## Features

- 27-DoF JAKA Pi USD, URDF, and MuJoCo models
- PPO locomotion training from velocity commands
- Left-right mirror data augmentation and mirror loss
- AMP locomotion from reference motion
- Dance motion imitation (Mimic) training
- Action delay, IMU delay, and randomization of PD gains, mass, center of mass, and friction
- TorchScript and ONNX policy export
- MuJoCo simulation that switches between locomotion and mimic policies

## Repository layout

```text
jaka_rl_lab/
├── deploy/
│   ├── Jaka_Pi_mujoco.py          # MuJoCo sim-to-sim entry point
│   ├── Jaka_Pi_config.yaml        # Model, control, policy, and mimic configuration
│   ├── Jaka_Pi.xml                # Full MuJoCo model
│   ├── Jaka_Pi_simplified.xml     # Simplified MuJoCo model
│   ├── meshes/Jaka_Pi.urdf        # JAKA Pi URDF
│   ├── mimic/                      # Mimic state machine, inference, and motion data
│   ├── policy_loco.pt              # Default locomotion TorchScript policy
│   └── policy_moxingwu.pt          # Default dance TorchScript policy
├── project/
│   ├── config/                     # Isaac Lab extension metadata
│   └── jaka_rl_lab/
│       ├── assets/jaka/
│       │   ├── jaka.py             # JAKA Pi joints, actuators, and initial state
│       │   └── jaka_pi_simplified/ # JAKA Pi USD assets
│       └── tasks/
│           ├── locomotion/         # Velocity tracking, symmetry, and AMP tasks
│           └── mimic/              # Motion imitation tasks
├── rsl_rl/                         # In-repo RSL-RL implementation (includes the AMP runner)
├── scripts/
│   ├── list_envs.py                # List JAKA environments
│   ├── rsl_rl/train.py             # Training entry point
│   ├── rsl_rl/play.py              # Evaluation and policy export entry point
│   └── mimic/                      # CSV/NPZ/AMP motion processing tools
└── README.md
```

> The directory name `tasks/.../envs/Khan_mini_27dof/` is kept for now so existing module paths remain valid. The tasks and asset configuration inside that directory already use JAKA Pi.

## Requirements and installation

Install and activate a working Isaac Lab environment first. The extension requires Python 3.10 or later. Project metadata currently targets Isaac Sim 4.5.0.

From the repository root:

```bash
python -m pip install -e ./project
python -m pip install -e ./rsl_rl
python -m pip install prettytable argcomplete
```

You can also install through the Isaac Lab launcher:

```bash
/path/to/IsaacLab/isaaclab.sh -p -m pip install -e ./project
/path/to/IsaacLab/isaaclab.sh -p -m pip install -e ./rsl_rl
/path/to/IsaacLab/isaaclab.sh -p -m pip install prettytable argcomplete
```

## Available tasks

List registered environments:

```bash
python scripts/list_envs.py
```

Or:

```bash
/path/to/IsaacLab/isaaclab.sh -p scripts/list_envs.py
```

The current tasks are:

| Task ID | Runner / algorithm | Purpose |
| --- | --- | --- |
| `Jaka-Pi-Loco` | `OnPolicyRunner` / PPO | Base velocity tracking |
| `Jaka-Pi-Loco-Sym` | `OnPolicyRunner` / PPO | Velocity tracking with left-right symmetry augmentation |
| `Jaka-Pi-Loco-SymAmp` | `AmpOnPolicyRunner` / AMPPPO | Symmetry augmentation plus an AMP motion prior |
| `Jaka-Pi-Mimic-Dance` | `OnPolicyRunner` / PPO | Dance reference-motion imitation |

### Locomotion

The three locomotion tasks share:

```text
Environment config: project/jaka_rl_lab/tasks/locomotion/envs/Khan_mini_27dof/velocity_env_cfg.py
PPO config:         project/jaka_rl_lab/tasks/locomotion/agents/rsl_rl_ppo_cfg.py
```

Important settings in the current environment:

- Velocity ranges: forward/backward `[-1.0, 2.0] m/s`, lateral `[-0.5, 0.5] m/s`, yaw rate `[-1.57, 1.57] rad/s`
- Commands are sampled uniformly inside an ellipsoid, with limits on linear and angular acceleration
- The actor uses 10 frames of observation history; the critic also receives base linear velocity, foot contact and force, and joint torque
- Action delay ranges from 0 to 4 control steps, and angular-velocity and orientation observation delay is simulated
- Randomization covers PD gains, base mass and center of mass, friction, and external pushes
- Episodes terminate early on abnormal orientation or persistent velocity-tracking error

`Jaka-Pi-Loco-Sym` uses `SymPPORunnerCfg` and `sym_augmentation_callback` to enable left-right mirror augmentation and a mirror loss (coefficient `0.5`).

`Jaka-Pi-Loco-SymAmp` uses `SymAmpPPORunnerCfg` and adds AMP on top of symmetry augmentation:

```text
AMP data:              project/jaka_rl_lab/tasks/locomotion/envs/Khan_mini_27dof/data.txt
AMP reward:            0.15
mirror loss:           2.0
preloaded transitions: 200000
```

### Mimic Dance

```text
Environment config: project/jaka_rl_lab/tasks/mimic/envs/Khan_mini_27dof/dance/tracking_env_cfg.py
Reference motion:   project/jaka_rl_lab/tasks/mimic/envs/Khan_mini_27dof/dance/moxingwu_edit.npz
Task ID:            Jaka-Pi-Mimic-Dance
```

The policy observation is composed of reference joint positions and velocities, reference anchor orientation, base angular velocity, projected gravity, joint state, and the previous action. Training adds joint and action smoothness penalties, plus position, orientation, and velocity tracking rewards for the global anchor and each body link. Physics randomization and action/IMU delay are used to improve deployment robustness.

## Training

Basic locomotion example:

```bash
python scripts/rsl_rl/train.py \
  --task Jaka-Pi-Loco \
  --num_envs 4096 \
  --max_iterations 1000
```

To train the symmetry AMP task or the dance task, change only the task name:

```bash
python scripts/rsl_rl/train.py --task Jaka-Pi-Loco-SymAmp --num_envs 4096
python scripts/rsl_rl/train.py --task Jaka-Pi-Mimic-Dance --num_envs 4096
```

Using the Isaac Lab launcher:

```bash
/path/to/IsaacLab/isaaclab.sh -p scripts/rsl_rl/train.py \
  --task Jaka-Pi-Loco \
  --num_envs 4096 \
  --max_iterations 1000
```

Common arguments:

```text
--task               Registered task name
--num_envs           Number of parallel environments
--max_iterations     Maximum number of training iterations
--seed               Random seed
--video              Record a training video
--run_name           Suffix for the current run name
--experiment_name    Override the experiment directory name
--resume             Resume from an existing checkpoint
--load_run           Run directory to load
--checkpoint         Checkpoint file to load
--distributed        Enable multi-GPU / multi-node training
```

If `--experiment_name` is omitted, the task name is converted to lowercase with underscores. Logs are written by default to:

```text
logs/rsl_rl/<experiment_name>/<timestamp>_<run_name>/
```

`logs/` and `outputs/` are listed in `.gitignore`.

## Evaluation and export

Run a trained policy:

```bash
python scripts/rsl_rl/play.py --task Jaka-Pi-Loco --num_envs 1
```

Specify a checkpoint:

```bash
python scripts/rsl_rl/play.py \
  --task Jaka-Pi-Loco \
  --checkpoint /path/to/model.pt \
  --num_envs 1
```

`play.py` automatically exports the policy to:

```text
<checkpoint_dir>/exported/policy.pt
<checkpoint_dir>/exported/policy.onnx
```

Common arguments:

```text
--task          Registered task name
--num_envs      Number of simulation environments
--checkpoint    Path to the checkpoint
--load_run      Run to load when no checkpoint is given
--video         Record an evaluation video
--real-time     Run as close to real time as possible
```

## MuJoCo sim-to-sim deployment

Install the extra dependencies:

```bash
python -m pip install mujoco pynput scipy pyyaml torch
```

The deployment script resolves the configuration and model through relative paths. Run it from the `deploy/` directory:

```bash
cd deploy
python Jaka_Pi_mujoco.py
```

Default files:

```text
Configuration:        deploy/Jaka_Pi_config.yaml
MuJoCo model:         deploy/Jaka_Pi.xml
Locomotion policy:    deploy/policy_loco.pt
Mimic policy:         deploy/policy_moxingwu.pt
Mimic motion data:    deploy/mimic/data_in/moxingwu_edit_50Hz.csv
```

The default simulation step is `0.001 s`. The policy and controller run every 20 simulation steps, so the control rate is 50 Hz. Locomotion uses a 90-dimensional single-frame observation and 10 frames of history. Mimic uses a 147-dimensional observation.

Keyboard controls:

| Key | Action |
| --- | --- |
| `↑` / `↓` | Forward / backward |
| `←` / `→` | Left / right |
| `End` / `Page Down` | Positive / negative yaw |
| `v` | Switch between the locomotion and mimic policies |

To replace a deployed policy, copy the `policy.pt` exported by `play.py` into `deploy/`, or edit `Jaka_Pi_config.yaml`:

```yaml
policy_path: "./policy_loco.pt"
xml_path: "./Jaka_Pi.xml"

mimic:
  policy_path: policy_moxingwu.pt
  motion_file: moxingwu_edit_50Hz.csv
```

Joint order, action scale, PD parameters, default pose, observation dimension, and history length in the deployment configuration must match the exported policy. Otherwise the policy will not infer correctly.

## Motion data

- Mimic training reads `moxingwu_edit.npz`.
- AMP training reads `data.txt`.
- MuJoCo Mimic deployment reads the motion file configured under `deploy/mimic/data_in/`.
- `scripts/mimic/` provides tools to convert CSV to NPZ, convert CSV to AMP, and play back NPZ. When converting, keep the input frame rate, output frame rate, joint order, and quaternion format consistent with the JAKA Pi configuration.
