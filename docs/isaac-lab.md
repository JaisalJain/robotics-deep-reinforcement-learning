# NVIDIA Isaac Lab

## Introduction

**NVIDIA Isaac Lab** is an open-source robotics reinforcement learning framework built on top of **NVIDIA Isaac Sim**.

It provides ready-to-use environments, robot assets, reinforcement learning integrations, and GPU-accelerated simulation for developing intelligent robotic systems.

Instead of writing a simulator, physics engine, robot loader, reward functions, and training infrastructure from scratch, Isaac Lab provides these components out of the box, allowing researchers and developers to focus on robot behavior and learning.

In this project, Isaac Lab is used to train reinforcement learning policies for:

- UR10 robotic manipulation
- ANYmal C quadruped locomotion
- H1 humanoid locomotion

---

# Isaac Sim vs Isaac Lab

Although the names are similar, they serve different purposes.

| Isaac Sim               | Isaac Lab                        |
| ----------------------- | -------------------------------- |
| Robotics simulator      | Reinforcement learning framework |
| GPU-accelerated physics | RL environments and training     |
| USD-based simulation    | Robot task definitions           |
| Robot visualization     | Reward functions                 |
| Sensor simulation       | PPO integration                  |
| RTX rendering           | Parallel training                |

In simple terms:

- **Isaac Sim** provides the virtual world.
- **Isaac Lab** teaches robots how to behave inside that world.

---

# Overall Architecture

The complete training pipeline can be visualized as follows:

```text
                 Isaac Sim
                      │
                      ▼
             Physics Simulation
                      │
                      ▼
          Isaac Lab Environment
                      │
      ┌───────────────┼───────────────┐
      ▼               ▼               ▼
 Observation      Reward        Termination
   Manager        Manager          Manager
      │               │               │
      └───────────────┼───────────────┘
                      ▼
                PPO Algorithm
                      │
                      ▼
               Neural Network
                      │
                      ▼
              Robot Controller
```

Each component has a dedicated responsibility, making environments modular and easy to customize.

---

# Manager-Based Environments

One of Isaac Lab's core design principles is the use of **Managers**.

Instead of placing all environment logic inside one file, responsibilities are separated into independent modules.

Common managers include:

- Observation Manager
- Action Manager
- Reward Manager
- Event Manager
- Termination Manager
- Curriculum Manager

This modular architecture makes environments easier to understand, extend, and maintain.

---

# Observation Manager

The Observation Manager determines what information is available to the policy.

Typical observations include:

- Joint positions
- Joint velocities
- Base orientation
- End-effector pose
- Desired velocity
- Target position
- Contact information

These observations become the input to the neural network.

---

# Action Manager

The Action Manager converts policy outputs into robot commands.

Depending on the task, actions may represent:

- Joint positions
- Joint velocities
- Joint torques
- Base velocity commands

The neural network predicts continuous values that are applied to the robot every simulation step.

---

# Reward Manager

The Reward Manager computes how well the robot is performing.

Example reward terms include:

Positive rewards

- Reaching a goal
- Tracking commanded velocity
- Maintaining balance

Penalty terms

- Excessive joint velocity
- Large control effort
- Falling
- Unstable motion

The reward function defines the behavior the robot should learn.

---

# Termination Manager

Episodes end when specific conditions are met.

Examples include:

- Maximum episode length reached
- Robot falls
- Collision occurs
- Task successfully completed

After termination, the environment resets automatically.

---

# Curriculum Learning

Some tasks become progressively more difficult during training.

Instead of exposing the robot to the hardest scenario immediately, Isaac Lab gradually increases task difficulty.

Examples include:

- Faster walking speeds
- More difficult terrain
- Larger disturbances

Curriculum learning often improves training stability and convergence.

---

# Domain Randomization

Domain randomization improves policy robustness by introducing variability during simulation.

Examples include:

- Surface friction
- Robot mass
- Sensor noise
- Initial robot pose
- External disturbances

Policies trained under varying conditions are generally more robust than policies trained in perfectly consistent environments.

---

# Parallel GPU Simulation

One of Isaac Lab's biggest advantages is large-scale parallel simulation.

Instead of training a single robot:

```text
Robot 1

Robot 2

Robot 3

...

Robot N
```

all environments execute simultaneously on the GPU.

This dramatically increases sample collection speed and significantly reduces training time.

Across this project, more than **540 million simulation steps** were executed using parallel simulation.

---

# Repository Structure

The Isaac Lab source tree used throughout this project is organized as follows:

```text
IsaacLab/

├── apps/
├── docker/
├── docs/
├── scripts/
│
├── source/
│
├── logs/
│
└── isaaclab.sh
```

Important directories:

| Directory   | Purpose                                |
| ----------- | -------------------------------------- |
| scripts/    | Training and evaluation scripts        |
| source/     | Environment and task definitions       |
| docker/     | Docker configuration                   |
| logs/       | Saved checkpoints and TensorBoard logs |
| isaaclab.sh | Main launcher script                   |

---

# Running Isaac Lab

Start the Docker container:

```bash
cd ~/IsaacLab

./docker/container.py start ros2
```

Enter the running container:

```bash
docker exec -it isaac-lab-ros2 bash
```

Inside the container:

```bash
cd /workspace/isaaclab
```

All training and evaluation commands are executed from this directory.

---

# Training a Policy

General command:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
--task <TASK_NAME> \
--headless
```

Examples from this project:

UR10

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
--task Isaac-Reach-UR10-v0 \
--headless
```

ANYmal C

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
--task Isaac-Velocity-Rough-Anymal-C-v0 \
--headless
```

H1

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
--task Isaac-Velocity-Rough-H1-v0 \
--headless
```

## Headless Training

Training commands in this project use the `--headless` flag:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
--task Isaac-Reach-UR10-v0 \
--headless
```

The `--headless` option disables the graphical user interface and runs the simulation without rendering a window. This significantly reduces GPU rendering overhead, allowing more computational resources to be dedicated to physics simulation and reinforcement learning. For long training runs, headless mode is generally preferred.

---

# Evaluating a Trained Policy

To evaluate the most recent checkpoint:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/play.py \
--task <TASK_NAME> \
--use_pretrained_checkpoint
```

To load a specific checkpoint:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/play.py \
--task <TASK_NAME> \
--load_run RUN_DIRECTORY \
--checkpoint model_999.pt
```

---

# Training Outputs

During training, Isaac Lab automatically creates an experiment directory under:

```text
/workspace/isaaclab/logs/
```

A typical run contains:

```text
logs/

model_0.pt
model_50.pt
...
model_999.pt

events.out.tfevents...

params/

git/
```

The important outputs are:

- PyTorch checkpoints (`.pt`)
- TensorBoard logs
- Training configuration
- Experiment metadata

---

# Workflow Used in This Project

The complete workflow followed in this repository is summarized below.

```text
Start Docker
        │
        ▼
Enter Container
        │
        ▼
Launch Training
        │
        ▼
Parallel Simulation
        │
        ▼
PPO Optimization
        │
        ▼
Save Checkpoints
        │
        ▼
Evaluate Policy
        │
        ▼
Record Images & Videos
        │
        ▼
Document Results
```

---

# Key Takeaways

- Isaac Sim provides high-fidelity GPU-accelerated simulation.
- Isaac Lab builds reinforcement learning workflows on top of Isaac Sim.
- Manager-based environments separate observations, rewards, actions, and termination logic.
- Thousands of environments can be simulated in parallel.
- Training automatically generates checkpoints and TensorBoard logs.
- The workflow demonstrated in this project was used to train manipulation, quadruped locomotion, and humanoid control policies.
