# Memory-Aware Nonlinear Active Noise Cancellation

> **Final Year Project — Ongoing**
>
> A research-oriented simulation study of **Active Noise Cancellation (ANC)** under practical challenges such as **time-varying secondary paths, nonlinear system behavior, and actuator saturation**.

---

## 📌 Overview

Active Noise Cancellation (ANC) is a technique used to reduce unwanted noise by generating an anti-noise signal that destructively interferes with the original disturbance.

In practical ANC systems, however, the acoustic environment is not always constant. The **secondary path** between the controller output and the error microphone can change over time due to changes in the acoustic environment, actuator characteristics, microphone position, or other system conditions.

This project investigates a **memory-aware nonlinear ANC system** with an adaptive secondary-path estimation mechanism.

The current implementation focuses on two major areas:

1. **RL-tuned Kalman secondary-path tracking** for time-varying ANC.
2. **Saturating-actuator characterization** to study the effect of real-world actuator limitations on ANC performance and stability.

The project is currently **under development**, and further experimentation and improvements are planned.

---

## 🎯 Project Objectives

The main objectives of the project are:

* Develop a simulation environment for Active Noise Cancellation.
* Model realistic machinery/engine-like noise.
* Incorporate a nonlinear disturbance model with memory.
* Estimate the ANC secondary path using a Kalman filter.
* Track changes in the secondary path over time.
* Develop an innovation-triggered adaptive scheduling mechanism.
* Compare low and high Kalman process-noise configurations.
* Investigate the effect of actuator saturation on ANC performance.
* Compare different FxLMS strategies under actuator saturation.
* Analyze ANC performance, convergence, and stability.

---

## 🧠 System Concept

The overall ANC system can be represented as:

```text
                    Primary Noise
                         |
                         v
                     Primary Path
                        P(z)
                         |
                         v
                   Disturbance d(n)
                         |
                         |
Reference x(n) ----------+------------------+
                         |                  |
                         v                  |
                    Control Filter         |
                       W(z)                |
                         |                  |
                         v                  |
                  Actuator / Speaker       |
                         |                  |
                    Saturation             |
                         |                  |
                         v                  |
                  Secondary Path           |
                       S(z)                |
                         |                  |
                         v                  |
                   Anti-noise y(n)         |
                         |                  |
                         +------(-)---------+
                                |
                                v
                           Error e(n)
                                |
                                v
                             FxLMS
                                |
                                v
                        Update W(z)
```

At the same time, the secondary path is estimated using an auxiliary probe:

```text
             Auxiliary Probe
                    |
                    v
             Secondary Path S(z)
                    |
                    v
              System Response
                    |
                    v
              Kalman Filter
                    |
                    v
             Estimated S_hat(z)
                    |
                    v
             FxLMS Controller
```

---

## 🔬 Part A — RL-Tuned Kalman Secondary-Path Tracker

The first part of the project addresses the problem of a **time-varying secondary path**.

The secondary path is represented as an FIR model whose coefficients are treated as the state of a Kalman filter.

An auxiliary probe signal is injected into the system to provide information about the secondary path.

### Why Kalman Filtering?

The Kalman filter provides a way to estimate the unknown secondary-path coefficients while accounting for uncertainty and measurement noise.

The project investigates the effect of the Kalman **process noise covariance Q**.

Two static configurations are compared:

| Configuration              | Behavior                                                               |
| -------------------------- | ---------------------------------------------------------------------- |
| Low Q                      | Better steady-state model quality but slower adaptation                |
| High Q                     | Faster adaptation but potentially noisier steady state                 |
| Innovation-triggered reset | Conservative during steady state and aggressive after detected changes |

The implementation uses:

```text
Low process noise:
Q = 1e-9

High process noise:
Q = 1e-5
```

The secondary-path estimate contains **48 taps** in the current configuration.

---

## 🔄 Innovation-Triggered Scheduler

Instead of continuously operating with a high process-noise value, the proposed approach normally operates conservatively and monitors the Kalman innovation.

The basic idea is:

```text
Normal operation
      |
      v
Monitor innovation
      |
      v
Innovation increases significantly?
      |
     YES
      |
      v
Covariance reset
      +
Higher probe level
      +
Temporary controller mute
      |
      v
Secondary-path re-lock
      |
      v
Return to normal ANC operation
```

The current implementation uses an innovation-ratio threshold to trigger the covariance reset.

This mechanism is referred to within the project as the **RL scheduler**. The current implementation is an innovation-driven adaptive scheduler rather than a conventional deep-RL agent such as DQN or PPO.

---

## 📈 Time-Varying Secondary Path

To evaluate tracking performance, the simulation introduces a significant secondary-path change during operation.

The current Part A experiment uses a path change at:

```text
t = 3 seconds
```

The purpose is to evaluate whether the estimated secondary path can:

* remain accurate during steady state,
* detect a sudden path change,
* recover after the change,
* and maintain ANC performance.

---

## 🎛️ FxLMS Controller

The ANC controller is based on the **Filtered-x Least Mean Squares (FxLMS)** algorithm.

A conventional LMS controller uses the reference signal directly for adaptation.

In ANC, the reference signal must account for the secondary path:

```text
Reference x(n)
      |
      v
Estimated Secondary Path S_hat(z)
      |
      v
Filtered Reference x'(n)
      |
      v
FxLMS Adaptation
```

The filtered reference is then used to update the adaptive control filter.

The current implementation uses a **64-tap control filter**.

---

# 🔊 Part B — Saturating-Actuator Characterization

The second part of the project investigates the effect of **actuator saturation** on ANC.

In a practical system, a loudspeaker or actuator cannot generate unlimited output.

Therefore:

```text
Requested control output
          |
          v
     Actuator limit
          |
          v
 Actual control output
```

The project evaluates both:

* **Hard saturation**
* **Soft saturation**

### Hard Saturation

The control output is clipped to a specified range.

Conceptually:

```text
if y > limit:
    y = limit

if y < -limit:
    y = -limit
```

### Soft Saturation

A `tanh()`-based nonlinear saturation model is used:

```text
y_out = limit × tanh(y / limit)
```

This provides a smoother transition toward the actuator limit.

---

## 🧪 Experiments

### 1. Saturation Severity Sweep

Different actuator limits are tested to understand how increasing saturation affects noise-reduction performance.

Current test points include:

```text
200%
100%
60%
40%
25%
15%
```

of the measured linear peak control output.

Metrics include:

* Noise Reduction (NR)
* Percentage of clipped samples

---

### 2. FxLMS Method Comparison

The project compares four approaches:

```text
1. Naive FxLMS
2. Leaky FxLMS
3. Saturation-aware FxLMS
4. Output-constrained FxLMS
```

These approaches are evaluated under different actuator saturation levels.

---

### 3. Stability Analysis

The project also performs a raw step-size sweep to study the stability of the adaptive controller.

Current step sizes include:

```text
1e-3
2e-3
3e-3
4e-3
5e-3
7e-3
```

The behavior of the system is compared with:

```text
Linear actuator
vs.
Hard-clipped actuator
```

The objective is to understand how actuator saturation affects the stability and behavior of the adaptive control loop.

---

# 📊 Evaluation Metrics

The project currently uses several metrics.

### Noise Reduction (NR)

Noise reduction is calculated from the ratio between disturbance power and residual noise power.

Higher NR indicates better noise cancellation.

---

### ERLE

**Echo Return Loss Enhancement (ERLE)** is used as a time-varying measure of cancellation performance.

It is calculated from the ratio between disturbance power and residual power over a moving window.

---

### Secondary-Path NMSE

The estimated secondary path is compared against the true simulated secondary path using **Normalized Mean Square Error (NMSE)**.

```text
NMSE =
||S_hat - S||²
----------------
   ||S||²
```

The value is represented in dB in the generated results.

Lower NMSE indicates a more accurate secondary-path estimate.

---

### Correlation

Correlation between the estimated and true secondary-path impulse responses is also calculated.

A value closer to `1` indicates stronger similarity between the two responses.

---

### Reconvergence Time

After a secondary-path change, the project measures how long the system takes to return close to its post-change performance level.

---

# 🧰 Technologies Used

The current simulation is implemented in **Python**.

### Programming Language

* Python 3

### Libraries

* NumPy
* SciPy
* Pandas
* Matplotlib

The project is implemented as a self-contained Python simulation and generates figures, CSV files, and JSON result files.

---

# 📁 Project Structure

A recommended repository structure is:

```text
Memory-Aware-Nonlinear-ANC/
│
├── rl_kalman_and_saturation.py
│
├── README.md
│
├── rl_kalman_saturation/
│   │
│   ├── figures/
│   │   ├── A1_shat_nmse.png
│   │   ├── A2_erle.png
│   │   ├── A3_tradeoff.png
│   │   ├── B1_severity.png
│   │   ├── B2_methods.png
│   │   └── B3_stability.png
│   │
│   └── results/
│       ├── partA_kalman_metrics.csv
│       ├── partA_kalman_summary.json
│       ├── partB_severity_sweep.csv
│       └── partB_saturation.json
│
└── LICENSE
```

The exact generated directory contents may evolve as the project develops.

---

# ⚙️ Installation

Clone the repository:

```bash
git clone <your-repository-url>
cd Memory-Aware-Nonlinear-ANC
```

Install the required Python packages:

```bash
pip install numpy scipy pandas matplotlib
```

---

# ▶️ Running the Project

Run the main simulation using:

```bash
python rl_kalman_and_saturation.py
```

The program performs:

```text
Part A
  ↓
Time-varying ANC simulation
  ↓
Low-Q Kalman
High-Q Kalman
RL covariance-reset scheduler
  ↓
Metrics + plots

Part B
  ↓
Actuator saturation study
  ↓
Severity sweep
Method comparison
Stability analysis
  ↓
Metrics + plots
```

The generated figures and results are stored in:

```text
rl_kalman_saturation/
```

---

# 📌 Current Status

**Project Status: 🚧 Ongoing**

### Completed / Implemented

* [x] ANC simulation environment
* [x] Primary and secondary path modeling
* [x] Machinery/engine-like reference noise generation
* [x] Nonlinear disturbance model with memory
* [x] FxLMS controller
* [x] Kalman-based secondary-path tracking
* [x] Time-varying secondary-path simulation
* [x] Innovation-based change detection
* [x] Covariance-reset scheduling
* [x] Actuator saturation modeling
* [x] Saturation severity experiments
* [x] FxLMS method comparison
* [x] Stability experiments
* [x] Automated metric and figure generation

### 🔨 Work in Progress

* [x] Further validation with additional operating conditions
* [ ] More extensive parameter tuning
* [ ] Evaluation with additional nonlinearities
* [ ] Improved adaptive scheduling strategies
* [ ] Further stability analysis
* [ ] Real-world / hardware validation
* [ ] Final experimental evaluation
* [ ] Final project documentation

---

# 🚀 Future Work

The project is currently being extended toward a more robust and practical ANC framework.

Potential future work includes:

* Testing additional time-varying secondary-path scenarios.
* Improving the adaptive scheduling mechanism.
* Investigating alternative reinforcement-learning-based scheduling strategies.
* Evaluating additional nonlinear actuator models.
* Testing different ANC adaptation algorithms.
* Studying robustness under different noise and SNR conditions.
* Performing real-time implementation.
* Integrating the algorithm with physical audio hardware.
* Comparing simulation results with experimental measurements.

---

# 📚 Research Focus

The project combines concepts from several areas:

```text
Digital Signal Processing
          +
Adaptive Filtering
          +
Active Noise Cancellation
          +
Kalman Filtering
          +
Nonlinear Systems
          +
Adaptive Control
          +
Reinforcement-Learning-inspired Scheduling
          +
Actuator Saturation
```

The primary research focus is on improving ANC robustness when the secondary acoustic path is **time-varying and nonlinear**, while also understanding the limitations introduced by actuator saturation.

---

# 👥 Project

**Final Year Project**

**Project Title:**
**Memory-Aware Nonlinear Active Noise Cancellation**

**Status:** Ongoing

> This repository contains the current research and simulation implementation. Results, algorithms, and system architecture may be updated as the project progresses.

---

## ⚠️ Disclaimer

This project is intended for **academic and research purposes**. The current implementation is primarily a simulation-based study and should not be considered a production-ready ANC controller or a validated real-world acoustic system.

---

## 📄 License

This project is currently intended for academic use.

A formal open-source license can be added when the project is finalized.
