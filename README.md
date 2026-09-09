# Adaptive ANC with RL-Tuned Kalman Secondary-Path Tracking for ECG Artifact Reduction

> **Final Year Project — Active Noise Control Framework**
>
> A research-oriented implementation and comparative evaluation of adaptive **Active Noise Cancellation (ANC)** pipelines for ECG artifact reduction, focusing on **reinforcement-learning-assisted secondary-path identification**, **online Kalman tracking**, and robustness to **time-varying measurement paths**.

[![Python](https://img.shields.io/badge/Python-3.x-blue?logo=python)](https://www.python.org/)
[![Signal Processing](https://img.shields.io/badge/Domain-Digital%20Signal%20Processing-orange)](#)
[![ANC](https://img.shields.io/badge/ANC-FxLMS-green)](#)
[![Research](https://img.shields.io/badge/Project-Research-purple)](#)

---

## 📌 Overview

**Active Noise Control (ANC)** reduces an unwanted disturbance by generating an anti-signal that passes through a **secondary path** before cancelling the disturbance at an error sensor.

A critical challenge in practical ANC systems is that the secondary path is not necessarily stationary. Changes in sensor placement, coupling, electrode contact, actuator characteristics, or the surrounding environment can cause the secondary path to change over time.

This project investigates whether an ANC system can remain stable and effective when this secondary path changes during operation.

Two adaptive ANC pipelines are implemented and compared:

### Pipeline A — RL-Identified FxLMS

A reinforcement-learning-tuned **leaky NLMS** identifier estimates the secondary path offline. The resulting secondary-path estimate is then fixed and used by a conventional **Filtered-x LMS (FxLMS)** controller.

### Pipeline B — RL-Tuned Kalman Secondary-Path Tracking

A **Kalman filter** continuously estimates the secondary path online. A reinforcement-learning-style innovation supervisor detects significant changes and triggers a covariance reset, allowing the estimator to rapidly re-lock to the new path.

The two approaches are evaluated under identical controlled measurement-path changes using biomedical ECG datasets.

The central research question is:

> **Can online Kalman secondary-path tracking provide greater robustness to changing measurement paths than a highly accurate but fixed RL-identified secondary-path model?**

---

## 🎯 Objectives

The main objectives of this project are:

* Develop an adaptive ANC framework for ECG artifact reduction.
* Implement a conventional FxLMS controller.
* Develop an RL-tuned offline secondary-path identification method.
* Develop an online Kalman secondary-path tracking method.
* Detect secondary-path changes using Kalman innovation.
* Introduce covariance-reset based re-locking after path changes.
* Compare stationary and time-varying secondary-path scenarios.
* Evaluate robustness using real ECG datasets.
* Measure output SNR, waveform correlation, recovery, PRD, and R-peak F1.
* Analyze computational cost and practical deployment trade-offs.
* Identify failure modes and limitations of both approaches.

---

## 🧠 System Architecture

The overall comparison can be represented as:

```text
                    ECG + Artifact
                         |
                         v
                  Disturbed Signal
                         |
                         v
              +----------------------+
              |      ANC System      |
              +----------------------+
                         |
              +----------+----------+
              |                     |
              v                     v
       Pipeline A              Pipeline B
       RL-FxLMS                RL-Kalman
              |                     |
              v                     v
      Offline Secondary       Online Secondary
      Path Identification     Path Tracking
              |                     |
              v                     v
        Fixed S_hat(z)       Kalman S_hat(z)
              |                     |
              v                     v
            FxLMS                 FxLMS
              |                     |
              +----------+----------+
                         |
                         v
                   Recovered ECG
                         |
                         v
                 Performance Metrics
```

Both pipelines operate through the **same time-varying true secondary path**. The primary difference is how the secondary-path estimate is obtained.

---

## 🔬 Pipeline A — RL-Identified FxLMS

Pipeline A represents the conventional **offline secondary-path identification** approach.

During an offline phase:

1. The secondary path is excited using white noise.
2. A leaky NLMS estimator identifies the secondary path.
3. A tabular Q-learning agent tunes the identifier parameters.
4. The resulting secondary-path estimate is fixed.
5. The fixed estimate is used by the FxLMS controller.

The Q-learning agent tunes parameters such as:

* Step size
* Filter order
* Leakage

The RL identifier is trained for **60 episodes** with the objective of minimizing identification NMSE.

### Pipeline A

```text
White-Noise Excitation
          |
          v
   Secondary Path
          |
          v
    Leaky NLMS
          |
          v
    Q-Learning
   Hyperparameter
     Selection
          |
          v
   Fixed S_hat(z)
          |
          v
        FxLMS
```

### Strength

Pipeline A provides an extremely accurate secondary-path model when the path is stationary.

### Limitation

Once the actual secondary path changes, the offline estimate becomes stale.

A sufficiently large phase mismatch can rotate the FxLMS gradient in the wrong direction and cause divergence.

---

## 🧮 Pipeline B — RL-Tuned Kalman Secondary-Path Tracker

Pipeline B is the proposed adaptive tracking approach.

The secondary-path coefficients are represented as a state in a Kalman filter.

A low-level auxiliary probe continuously provides information about the current secondary path.

The Kalman filter therefore updates the secondary-path estimate online.

```text
              Auxiliary Probe
                    |
                    v
              True S(z)
                    |
                    v
            Measurement
                    |
                    v
             Kalman Filter
                    |
                    v
             S_hat(z)
                    |
                    v
                  FxLMS
```

The system continuously monitors the **Kalman innovation**.

When the innovation remains significantly elevated, it indicates that the current secondary-path estimate may no longer represent the actual path.

The system then:

1. Detects the change.
2. Inflates/reset the covariance.
3. Temporarily mutes the controller.
4. Allows the Kalman filter to re-lock.
5. Returns to normal ANC operation.

---

## 🔄 Innovation-Triggered Adaptation

The key idea behind Pipeline B is to avoid operating the Kalman filter permanently at a high process-noise level.

Instead:

```text
              Normal Operation
                     |
                     v
             Monitor Innovation
                     |
                     v
          Innovation remains high?
                /           \
              No             Yes
              |               |
              v               v
        Continue ANC     Trigger Reset
                              |
                    +---------+---------+
                    |                   |
                    v                   v
              Inflate P            Increase
                                  Adaptation
                    |                   |
                    +---------+---------+
                              |
                              v
                        Re-lock S_hat
                              |
                              v
                         Resume ANC
```

Two refinements are incorporated into Pipeline B:

### Probe-Echo Cancellation

The estimated echo of the identification probe is removed from the control error.

### Persistence-Gated Trigger

The innovation must remain elevated over a dwell window before a reset is triggered.

This reduces false detections caused by transient ECG events or artifact spikes.

---

## ⚙️ Why Secondary-Path Tracking Matters

In FxLMS, the reference signal is filtered using the estimated secondary path:

```text
x(n)
 |
 v
S_hat(z)
 |
 v
x'(n)
 |
 v
FxLMS
 |
 v
Controller W(z)
```

If the secondary-path estimate is accurate, the adaptation gradient points toward reduced error.

If the estimate becomes sufficiently incorrect, the gradient can become destabilizing.

Therefore:

> **A stale secondary-path estimate is not simply inaccurate — it can destabilize the ANC loop.**

This is the central motivation for continuously tracking the secondary path.

---

## 🧪 Experimental Methodology

The comparison is designed so that both pipelines experience the same experimental conditions.

Shared conditions include:

* Same ECG record
* Same sampling rate
* Same controller length
* Same initial controller
* Same secondary-path model
* Same path-change instants
* Same random seed
* Same true time-varying secondary path

The only major difference is:

```text
Pipeline A
Offline + Fixed S_hat(z)

          VS.

Pipeline B
Online + Continuously Tracked S_hat(z)
```

Three controlled measurement-path changes are introduced during the experiments.

---

## ❤️ Dataset 1 — MIT-BIH Record 118

The first experiment uses:

* **MIT-BIH record 118**
* Lead: MLII
* Sampling rate: 360 Hz
* Electrode-motion (EM) artifact
* Muscle-artifact (MA) noise

The artifact is scaled to produce a target **0 dB input SNR**.

Two scenarios are evaluated:

```text
1. Static secondary path
2. Three controlled path changes
```

The static experiment evaluates pure identification/control quality, while the changing-path experiment evaluates robustness.

---

## 📊 MIT-BIH Results

On a static path, both pipelines operate close to the theoretical performance ceiling.

| Scenario       | Pipeline A |   Pipeline B | Winner |
| -------------- | ---------: | -----------: | ------ |
| Static — EM    |    7.94 dB |      7.58 dB | A      |
| Static — MA    |    6.64 dB |      6.01 dB | A      |
| 3 Changes — EM |  -22.60 dB | **-1.19 dB** | **B**  |
| 3 Changes — MA |   -0.26 dB | **+0.97 dB** | **B**  |

The most significant result is the **EM three-change experiment**, where Pipeline B provides approximately a **21 dB advantage** over Pipeline A.

### Interpretation

```text
Static Path
     |
     +--> Pipeline A ≈ Pipeline B
             |
             v
       Both near ceiling


Changing Path
     |
     +--> Pipeline A
     |       |
     |       v
     |   Model becomes stale
     |       |
     |       v
     |    Divergence
     |
     +--> Pipeline B
             |
             v
       Detects change
             |
             v
       Kalman re-lock
             |
             v
       Bounded operation
```

---

## 🧬 Dataset 2 — PhysioNet ECG-ID

The second experiment uses **five distinct ECG-ID recordings**.

For each recording:

* Sampling rate: **500 Hz**
* Length: **20 seconds**
* 10,000 × 2 samples
* Raw/noisy ECG used as the contaminated signal
* Database-filtered ECG used as the clean reference

The provided recordings span an input SNR range from:

```text
+6.85 dB
       ↓
−11.63 dB
```

Three recordings have negative input SNR, meaning the noise is stronger than the ECG signal.

---

## 📈 ECG-ID Results

Pipeline B wins the output-SNR comparison on **all five recordings**.

| Record | Input SNR | Pipeline A SNR | Pipeline B SNR | Winner |
| ------ | --------: | -------------: | -------------: | ------ |
| P1     |  +6.85 dB |       -1.84 dB |   **+1.13 dB** | B      |
| P2     |  -4.65 dB |      -44.90 dB |   **-5.26 dB** | B      |
| P3     |  -4.75 dB |      -34.05 dB |   **-7.49 dB** | B      |
| P4     |  +0.50 dB |       -6.49 dB |   **-5.05 dB** | B      |
| P5     | -11.63 dB |      -61.40 dB |   **-4.42 dB** | B      |

Pipeline A experiences catastrophic divergence on the noisier recordings, while Pipeline B remains bounded.

---

## 📊 Aggregate ECG-ID Performance

Mean ± standard deviation across the five ECG-ID recordings:

| Metric      | Pipeline A — RL-FxLMS | Pipeline B — RL-Kalman |
| ----------- | --------------------: | ---------------------: |
| Output SNR  |       -29.7 ± 22.7 dB |      **-4.2 ± 2.9 dB** |
| Correlation |           0.21 ± 0.24 |        **0.51 ± 0.12** |
| PRD         |     28,099 ± 45,177 % |         **171 ± 48 %** |
| Records Won |                 0 / 5 |              **5 / 5** |

The key advantage of Pipeline B is not perfect denoising. Its main advantage is **stability and graceful degradation under changing-path conditions**.

---

## 🔍 Record-Level Observations

### P1 — Cleanest Record

Input SNR: **+6.85 dB**

Pipeline B achieves:

```text
Output SNR = +1.13 dB
Correlation = 0.74
```

This is the only ECG-ID record where clear positive absolute output SNR is achieved.

### P2 and P3

With input SNR around **-4.7 dB**, Pipeline A diverges to approximately:

```text
-34 dB to -45 dB
```

Pipeline B remains bounded around:

```text
-5 dB to -7 dB
```

and retains positive waveform correlation.

### P5 — Hardest Case

Input SNR:

```text
-11.63 dB
```

Pipeline A:

```text
Output SNR = -61.40 dB
Correlation = -0.01
```

Pipeline B:

```text
Output SNR = -4.42 dB
Correlation = 0.47
```

Pipeline B does not produce positive SNR, but it remains bounded instead of catastrophically diverging.

---

## 🛠️ Pipeline Improvements

Several improvements were evaluated during the study.

### Controller Length

Increasing the biomedical controller from:

```text
32 taps → 64 taps
```

improved ECG-ID rec_1 performance.

| Metric      |  32 taps |      64 taps |
| ----------- | -------: | -----------: |
| Output SNR  | +0.69 dB | **+1.13 dB** |
| Recovery    |    14.7% |    **23.0%** |
| Correlation |    0.729 |    **0.743** |
| PRD         |      92% |      **88%** |

The same controller length was applied to both pipelines to maintain a fair comparison.

---

## ❌ What Did Not Work

The study also documents approaches that produced poor results.

### Freezing the Controller During Re-Lock

Freezing the control filter instead of muting it resulted in divergence.

**Conclusion:** temporary controller muting is necessary during re-lock.

### Larger FxLMS Step Size

Increasing the step size to approximately:

```text
μ = 0.02 – 0.03
```

worsened performance on broadband noise because of increased misadjustment.

### Post-Change Step-Size Boost

A temporary increase in step size after a path change did not provide measurable improvement.

### Shorter Mute Window

Reducing the mute duration prevented the secondary-path estimator from fully re-locking.

These negative results help define the engineering constraints of the proposed approach.

---

## 💻 Computational Cost

The two approaches distribute computational cost differently.

| Metric                    | Pipeline A | Pipeline B |
| ------------------------- | ---------: | ---------: |
| Offline RL identification |    ~3.05 s |       None |
| Online cost/sample        | **7.9 μs** |    26.9 μs |
| Total time to result      |     3.11 s | **0.19 s** |
| Complexity                |       O(L) |  O(L + L²) |

Pipeline B costs approximately **3.4× more per sample** because of the Kalman covariance update.

However, at the tested biomedical sampling rate, the measured 26.9 μs/sample remains comfortably below the 500 Hz sample period.

---

## 📏 Evaluation Metrics

The project evaluates the pipelines using multiple complementary metrics.

### Output SNR

Measures the quality of the recovered signal relative to the residual artifact.

**Higher is better.**

### Waveform Correlation

Measures similarity between the recovered ECG and clean reference.

**Higher is better.**

### Recovery Percentage

Defined using the recovered-vs-clean error energy.

**Higher is better.**

### PRD

**Percentage Root-Mean-Square Difference**

**Lower is better.**

### R-Peak F1

Measures preservation of ECG beat annotations and R-peak detection performance.

### Stability

The project also examines whether the adaptive controller remains bounded following secondary-path changes.

The use of multiple metrics is important because a single SNR value does not fully describe ECG morphology preservation.

---

## 🏆 Main Findings

The experiments lead to several important conclusions.

### Static Secondary Path

Pipeline A is highly competitive and reaches the theoretical ceiling because its offline secondary-path estimate is extremely accurate.

### Changing Secondary Path

Pipeline B is significantly more robust because it continuously tracks the secondary path.

### Noisy ECG

Pipeline A can experience catastrophic divergence when the path changes and the fixed secondary-path estimate becomes incorrect.

Pipeline B remains bounded and preserves substantially better waveform correlation.

### Overall

```text
Pipeline A
    ↓
Highly accurate when static
    ↓
Fragile under path changes


Pipeline B
    ↓
Slightly higher computational cost
    ↓
Online path tracking
    ↓
Change detection
    ↓
Covariance reset
    ↓
Re-lock
    ↓
Graceful degradation
```

The report concludes that Pipeline B is the more robust approach under non-stationary conditions.

---

## 🔬 Research Contribution

The project explores the combination of:

```text
Active Noise Control
        +
Filtered-x LMS
        +
Reinforcement Learning
        +
Kalman Filtering
        +
Online Secondary-Path Tracking
        +
Innovation-Based Change Detection
        +
Adaptive Control
        +
Biomedical Signal Validation
```

The primary contribution is the investigation of an **online RL-tuned Kalman secondary-path tracker** that can detect and adapt to controlled measurement-path changes while preventing the catastrophic divergence observed with a fixed secondary-path model.

---

## 🚧 Limitations

The current study has several important limitations.

### Not a Clinical Device

The ECG experiments are controlled signal-processing validations.

The secondary path, probe, and path changes are modelling constructs.

No clinical or diagnostic claims are made.

### Not Perfect Denoising

Pipeline B improves robustness but does not transform extremely noisy ECG recordings into clean signals.

On four of the five ECG-ID recordings, the absolute output SNR remains negative.

### Reference Limitation

The ECG-ID reference uses:

```text
Noise = Raw ECG − Database-filtered ECG
```

This is possible because the dataset provides a raw/filtered pair.

A practical system would generally have an imperfect reference sensor.

### Limited Dataset

The study currently uses:

* One MIT-BIH record
* Five ECG-ID recordings

A larger cohort and paired statistical testing are identified as future work.

---

## 🚀 Future Work

The report identifies several directions for further development.

### 1. Noise-Robust Change Detection

The most important remaining limitation is reliable change detection under very noisy ECG conditions.

A proposed next step is a:

> **Noise-robust, QRS-gated change detector**

This would reduce false or missed triggers caused by noisy Kalman innovation.

### 2. Hybrid Architecture

A promising architecture is:

```text
RL-Identified FxLMS
        |
        v
Excellent Initial S_hat
        |
        v
Kalman Tracker
        |
        v
Online Adaptation
```

This would use Pipeline A's high-quality offline estimate as the initial state for Pipeline B.

The goal is to combine:

* High initial accuracy
* Online adaptation
* Robustness to path changes
* Stable operation

The hybrid architecture is a natural next design direction.

### 3. Larger Dataset

Future experiments should include:

* More ECG records
* Larger cohorts
* Statistical significance testing
* Additional artifact types
* More path-change scenarios

### 4. Real-Time / Hardware Validation

The algorithms can eventually be evaluated on physical audio/sensor hardware to determine how simulation results translate into real-world ANC systems.

---

## 📌 Project Status

**Status: 🚧 Final Year Project / Research in Progress**

### Implemented

* [x] FxLMS controller
* [x] RL-tuned secondary-path identification
* [x] Kalman secondary-path tracking
* [x] Innovation-triggered covariance reset
* [x] Probe-echo cancellation
* [x] Persistence-gated change detection
* [x] MIT-BIH validation
* [x] PhysioNet ECG-ID validation
* [x] Controlled secondary-path changes
* [x] Output SNR analysis
* [x] Correlation analysis
* [x] Recovery and PRD analysis
* [x] R-peak evaluation framework
* [x] Computational-cost comparison
* [x] Failure-mode analysis

### 🔨 Future Development

* [ ] Noise-robust QRS-gated change detection
* [ ] Hybrid RL-FxLMS + Kalman architecture
* [ ] Larger ECG dataset evaluation
* [ ] Statistical significance testing
* [ ] Additional artifact types
* [ ] Additional path-change conditions
* [ ] Real-time implementation
* [ ] Hardware validation
* [ ] Final project documentation

---

## 🏁 Conclusion

The experimental results show that the **RL-tuned Kalman secondary-path tracker** provides substantially better robustness than a fixed RL-identified FxLMS pipeline when the secondary measurement path changes.

On stationary paths, the offline RL-identified pipeline reaches the theoretical performance ceiling.

Under changing paths, however, the fixed secondary-path estimate can become destabilizing, resulting in catastrophic divergence.

The Kalman-based pipeline addresses this through:

```text
Online secondary-path estimation
            +
Innovation monitoring
            +
Persistence-gated detection
            +
Covariance reset
            +
Temporary controller muting
            =
Robust adaptation to path changes
```

The main contribution is therefore **not universal ECG denoising**, but improved **stability, robustness, and graceful degradation under non-stationary measurement paths**.

The clearest next step is to improve the change detector using a **noise-robust, QRS-gated strategy**, followed by evaluation of the proposed hybrid offline-RL + online-Kalman architecture.

---

## ⚠️ Disclaimer

This repository is intended for **academic and research purposes**.

The biomedical experiments are controlled signal-processing validations and **do not constitute a clinical device, diagnostic system, or medical recommendation**.

The secondary path, auxiliary probe, and measurement-path changes used in the experiments are modelling constructs. Results should not be interpreted as evidence of clinical efficacy.

---

## 👥 Project

**Final Year Project**

**Research Area:** Active Noise Control / Adaptive Signal Processing / Biomedical Signal Processing

**Primary Focus:**

> **RL-Tuned Kalman Secondary-Path Tracking for Robust Adaptive ANC**

**Application:** ECG artifact reduction

**Project Status:** Ongoing

---

## 📄 License

This project is currently intended for academic and research use.

A formal open-source license will be added when the project is finalized.

---

## 📖 References & Reproducibility

The experiments documented in this repository correspond to the accompanying technical report:

> **FxLMS+RL vs Kalman+RL — MIT-BIH & ECG-ID Comparison**
> *Comparative Evaluation of Two Adaptive ANC Pipelines*

The report documents the datasets, methodology, controlled path perturbations, evaluation metrics, experimental results, computational cost, limitations, and future work.
