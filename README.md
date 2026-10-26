# Microduck-Training-Smoke-Test-for-Windows

# Training Microduck to Walk with Isaac Lab 3.0 — on Windows

## Introduction

**What is Microduck?**
Microduck is a small, affordable (~US$399), open-source bipedal robot from Pollen Robotics and Hugging Face. It walks, sits, crouches, catches its balance when pushed, and gets back up on its own. Its training code is open, so anyone can teach it new skills in simulation and load them onto the real robot.

**What this tutorial accomplishes**
1. Installs **Isaac Lab 3.0** in its own isolated environment.
2. Confirms the **Microduck tasks** are available.
3. Trains a walking policy with **reinforcement learning (RL)** while watching the ducks learn live.
4. **Plays** the trained policy and **analyzes the results** in TensorBoard.

**Two fixes over NVIDIA's official tutorial**
(*"Train in Simulation, Deploy on a Real Robot: Microduck with Isaac Lab 3.0"*, Hugging Face, Oct 2026)

> **1. It runs on Windows, not Linux.** The official tutorial targets a Linux workstation. Every command here is converted to **Windows PowerShell**, including Windows-only setup (long paths, uv installer).
>
> **2. It fixes the GitHub branch mismatch.** The official tutorial clones `develop`, which **no longer exists** (`fatal: Remote branch develop not found`). Its content now lives in **`main`**. The `release/3.0.0` branch does **not** include Microduck yet.

**Requirements:** Windows 11, NVIDIA RTX GPU (tested on RTX 4090 Laptop, 16 GB), driver **580.88+**, Git.

---

## Before you start

Isaac Lab 3.0 runs **without Isaac Sim** by default: it uses the **Newton** physics engine and is launched with `uv run isaaclab ...`. **uv** is a fast Python environment manager that creates an isolated environment inside the project folder (`IsaacLab3\.venv`), so no conda environment is needed.

> **Already have Isaac Lab 2.x?** It does not contain Microduck, and it can't be upgraded in place. Install Isaac Lab 3.0 in a **separate folder and environment**, as shown below. Your existing 2.x install and projects stay untouched; just don't `git pull` or switch branches inside the old folder.

---

## Step 1 — Prerequisites

**1a. Driver:** run `nvidia-smi` and confirm **Driver Version ≥ 580.88**.

**1b. Enable long file paths** (must be done before cloning). Right-click **Windows PowerShell** → **Run as administrator** (prompt shows `C:\Windows\system32>`). Paste **one command at a time**:

```powershell
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```
```powershell
git config --system core.longpaths true
```

Success = `LongPathsEnabled : 1`. If you see `Requested registry access is not allowed`, the window isn't admin.

**1c. Install uv** in a **normal** PowerShell window, then close and reopen it:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```
```powershell
uv --version
```

---

## Step 2 — Install Isaac Lab 3.0 (with the branch fix)

```powershell
cd C:\Users\<YOUR_USER>
```
```powershell
git clone --branch main https://github.com/isaac-sim/IsaacLab.git IsaacLab3
```
```powershell
cd IsaacLab3
```
```powershell
Test-Path source\isaaclab_tasks\isaaclab_tasks\contrib\microduck
```
```powershell
uv sync
```

`Test-Path` should print `True`. `uv sync` downloads several GB the first time (10–30 min).
To use this environment from another folder: `C:\Users\<YOUR_USER>\IsaacLab3\.venv\Scripts\Activate.ps1`

---

## Step 3 — Confirm the Microduck tasks

```powershell
uv run isaaclab list_envs --keyword MicroDuck --show_presets
```

You should see `IsaacContrib-Velocity-Flat-MicroDuck` (flat floor — start here) and `IsaacContrib-Velocity-Rough-MicroDuck` (stairs, slopes, uneven ground), both using **rsl_rl** with an optional **`backlash`** preset (simulates the "play" in worn gears so policies transfer better to the real robot).

---

## Step 4 — Train

**Smoke test first** (5 iterations, catches errors in a minute):

```powershell
uv run isaaclab train --rl_library rsl_rl --task IsaacContrib-Velocity-Flat-MicroDuck --num_envs 64 --max_iterations 5 presets=backlash
```

**Train and watch live.** Each duck is one practice attempt; all ducks share **one brain** (the policy), which is updated every iteration from everything they experienced.

*With the Newton visualizer (default, nothing extra to install):*

```powershell
uv run isaaclab train --rl_library rsl_rl --task IsaacContrib-Velocity-Flat-MicroDuck --num_envs 100 --viz newton_gl presets=backlash
```

*With the Isaac Sim viewer (one-time extra download of Isaac Sim, several GB):*

```powershell
uv sync --extra isaacsim
```
```powershell
uv run --extra isaacsim isaaclab train --rl_library rsl_rl --task IsaacContrib-Velocity-Flat-MicroDuck --num_envs 100 --viz kit presets=backlash
```

> 🧠 **Newton visualizer vs. Isaac Sim — in simple terms**
> Think of Newton as a **gym** and Isaac Sim as a **film studio**.
>
> | | **Newton visualizer** (gym) | **Isaac Sim** (studio) |
> |---|---|---|
> | What it is | A light window that draws the simulation | A full robotics app with photoreal RTX rendering |
> | Strengths | Fast, light, leaves more GPU for learning | Realistic sensors, labeled synthetic data, ROS 2 bridge, 3D scene editor |
> | **Use it when** the robot must... | **learn how to move** (walk, balance, fly stably) | **see, sense, or connect** to a larger system (cameras, lidar, ROS) or when you want to edit/customize your own environment |
>
> For Microduck, the **physics stays Newton either way** — the task is built for Newton's MuJoCo-Warp solver and its servo models. Isaac Sim only replaces the window. Walking is a gym job, so the Newton visualizer is the better choice.

> 📌 Isaac Lab 3.0 is **headless by default**: no window opens unless you pass `--viz`.

> ⚠️ **This is a demo run, not a proper training.** 100 ducks with a viewer open is great for *understanding*, but slow for *learning*: each iteration collects 20× less experience than 2048 ducks, and drawing the scene takes GPU time from physics. Expect only partial results (standing, wobbly steps). A proper training uses **thousands** of ducks and **no viewer**: the official tutorial uses 4096; on a 16 GB laptop GPU, use 2048 (or 1024 if you run out of memory):
>
> ```powershell
> uv run isaaclab train --rl_library rsl_rl --task IsaacContrib-Velocity-Flat-MicroDuck --num_envs 2048 presets=backlash
> ```
>
> The task's default limit is 50,000 iterations, so stop with **Ctrl+C** when results look good. A checkpoint (saved "brain") is written **every 250 iterations** — stop earlier and nothing is saved.

---

## Step 5 — Play the trained policy

`play` loads your latest checkpoint, runs it without learning, and exports `policy.onnx` (the brain file a real Microduck uses). You can run it in a **second** PowerShell window while training continues, to "peek" at progress.

*Newton visualizer:*

```powershell
uv run isaaclab play --rl_library rsl_rl --task IsaacContrib-Velocity-Flat-MicroDuck --checkpoint latest --num_envs 16 --viz newton_gl presets=backlash
```

*Isaac Sim viewer:*

```powershell
uv run --extra isaacsim isaaclab play --rl_library rsl_rl --task IsaacContrib-Velocity-Flat-MicroDuck --checkpoint latest --num_envs 16 --viz kit presets=backlash
```

> ❗ `--task` and `presets=` must **match what you trained**. Each task has its own log folder (e.g. `logs\rsl_rl\microduck_velocity_flat_backlash`). Playing `Rough` after training only `Flat` fails with `No compatible manifested run with a checkpoint was found`.

**What "good" looks like:** ducks stay upright, walk in their own commanded directions, turn smoothly, use a steady alternating gait, and catch themselves after stumbles. Some stand still on purpose (they were commanded to).
**Red flags:** falling within seconds, every duck doing the same odd motion, jittering legs, hopping or dragging — usually signs of too little training.

Find the exported file with:

```powershell
Get-ChildItem -Recurse -Filter policy.onnx logs
```

---

## Step 6 — Analyze the results in TensorBoard

The viewer shows the **simulation**; it doesn't plot training scores. The **RSL-RL** library writes those scores to log files, and **TensorBoard** turns them into graphs over time. (PowerShell also prints them each iteration — fine for a quick check, but graphs show trends.)

From inside `IsaacLab3`:

```powershell
uvx tensorboard --logdir logs\rsl_rl
```

Open `http://localhost:6006` in your browser. Every run appears as its own colored curve, so you can compare your 100-duck and 2048-duck runs side by side.

**The graphs to look at, and why:**

| Graph | What it means | Good sign |
|---|---|---|
| `Train/mean_reward` | The duck's **overall score** | Rises, then **flattens** (learning has settled) |
| `Train/mean_episode_length` | How long ducks **survive** before an episode ends | Grows toward the maximum |
| `Episode_Reward/track_lin_vel` | Does it **move at the commanded speed and direction**? The key *quality* signal | Rises |
| `Episode_Reward/track_ang_vel` | Does it **turn as commanded**? | Rises |
| `Episode_Reward/upright` | Does the body **stay upright**? | Rises early, then stays high |
| `Episode_Termination/fell_over` | Share of episodes ending in a **fall** | **Drops** |
| `Episode_Termination/time_out` | Share of episodes where the duck **survived the full time** | **Rises** |



