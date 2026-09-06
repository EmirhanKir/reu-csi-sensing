# Cross Session Generalization Experiments

WiFi CSI Human Activity Recognition
Emirhan Kir

## Overview

This document tracks the experiments run on the WiFi CSI dataset to improve cross session activity recognition accuracy. The goal set by Dr. Bulut was to reach 70 percent or higher on the cross session benchmark. The activities are Walking, Toe Tap, and Jumping Jacks. Each session follows the same collection protocol: a Walking rep for 10 seconds, a none rep for 10 seconds standing still, a Toe Tap rep for 10 seconds, another none, then Jumping Jacks for 10 seconds, another none. This full cycle repeats 10 times per session.

Data is collected on an ESP32 transmitter and receiver paired with a Raspberry Pi running the CSI Pi system. Raw files come out as ttyUSB0.csv with 29 columns and a matching annotations.csv marking the start timestamp of every segment.

Five sessions were usable for this work: DS2, DS3, DS4, DS5, DS6. DS4 was originally excluded due to a timestamp mismatch but was brought back in for an all sessions run.

## Final Pipeline

- Parse the CSI string into 64 subcarrier amplitudes by taking the magnitude of the interleaved imaginary and real integer pairs
- Drop the 12 null subcarriers, keeping 52 active subcarriers
- Apply rolling mean denoising with a window of 10, three passes
- Subtract a local none baseline for every activity cycle, pairing each activity rep with the none segment that immediately precedes it (this differs from a single global session none mean and was the key step that unlocked most of the accuracy gain)
- Drop the none rows after baseline subtraction
- Apply per session z score normalization
- Fit PCA on the combined training sessions only with 20 components, then apply that same PCA to the held out test session
- Create sliding windows of size 50 with stride 3
- Augment training data with vectorized Gaussian noise, time shift, and amplitude scaling at a multiplier of 2
- Train a 1D CNN (Conv1D layers of 64, 128, 64 filters, GlobalAveragePooling, Dense layers of 128 and 64 with batch normalization and dropout)
- Fine tune the trained model on 20 percent of the held out test session using a contiguous time block split to prevent overlap leakage between calibration and evaluation windows
- Evaluate on the remaining 80 percent of the held out session

## Key experiments tried

**DNN baseline.** Dense 128 / Dense 64 layers, matching the original REU24 Water Sensing notebook. 2 train / 1 test protocol across DS2 through DS6: mean cross session accuracy ~37.3%. Within session accuracy was ~86%, showing the bottleneck was generalization, not the model's ability to learn the activity signal.

**1D CNN.** Three Conv1D layers with GlobalAveragePooling and dense layers. Raised the cross session mean to ~45.4%, with wide fold-to-fold variance.

**Light CNN.** A lighter architecture landed at ~41.6% mean cross session accuracy. None of these model swaps alone were enough to approach the 70% target.

**Per-session z-score normalization.** Normalizing each session against its own mean and standard deviation reduces distribution shift from AGC, hardware warmup, and placement. Contributed to the jump from ~37% to ~57% on a 2-train/1-test protocol.

**Global vs. local none-baseline subtraction.** A single global none-mean per session helped, but left within-session drift in place. Switching to a local baseline, pairing each activity segment only with its immediately preceding none segment, captured that drift and was one of the biggest accuracy levers in the project.

**Vectorized augmentation.** The first augmentation pass used a Python for-loop per window and was slow enough to look like a hang. Rewriting it with vectorized numpy masks cut augmentation time from minutes to seconds per fold with identical math.

**Fine-tuning with calibration data.** After training the base model on the training sessions, 20% of the held-out test session (more aggressively augmented) is used to fine-tune the trained model at a low learning rate (5e-5) for ~20 epochs; the remaining 80% is the actual eval set.

**Window overlap leakage fix.** The first fine-tuning split used a random permutation to assign windows to calibration vs. eval. This produced suspicious 98-99% accuracy with under 1% standard deviation, inconsistent with the difficulty seen everywhere else. With stride 3 and window size 50, adjacent windows share 47 of 50 timesteps, so a random shuffle let near-duplicate windows leak across the split. Switching to a contiguous time-block split (first 20% of the session becomes calibration, remainder becomes eval, any window straddling the boundary is dropped) eliminated shared raw timesteps and dropped accuracy to a believable, honest 64-66%.

**Other approaches tried:** an RF baseline (~58% on DS1, used as a sanity check), Hampel outlier filtering (comparable to rolling mean, not clearly better), CORAL domain adaptation (no clear improvement on its own), and sweeps over PCA component count, stride, window size, and epoch/patience settings.

## Results

### Cross-session mean accuracy by experiment

| Experiment | Protocol | Mean Accuracy |
|---|---|---|
| DNN baseline, no fine-tuning | 2 train / 1 test | 37.3% |
| 1D CNN, no fine-tuning | 2 train / 1 test | 45.4% |
| Light CNN, no fine-tuning | 2 train / 1 test | 41.6% |
| 1D CNN + z-score + global baseline, fine-tuning (window leakage present) | 4 train / 1 test | 98.9% (leaked, not trustworthy) |
| DNN, local baseline, leakage-safe fine-tuning | 4 train / 1 test | 64.0% |
| 1D CNN, local baseline, leakage-safe fine-tuning | 4 train / 1 test | 65.7% |
| 1D CNN, local baseline, leakage-safe fine-tuning, DS4 excluded, epoch cap 100 | 3 train / 1 test | 71.4% |

### Final leave-one-out run (all 5 sessions, epoch cap 70)

| Held-out session | Training sessions | Accuracy |
|---|---|---|
| DS2 | DS3, DS4, DS5, DS6 | 85.7% |
| DS3 | DS2, DS4, DS5, DS6 | 60.6% |
| DS4 | DS2, DS3, DS5, DS6 | 47.8% |
| DS5 | DS2, DS3, DS4, DS6 | 71.5% |
| DS6 | DS2, DS3, DS4, DS5 | 63.1% |
| **Mean** | | **65.7%** |
| Std | | 12.6% |

DS4 pulled the mean down significantly, consistent with the data quality issues that originally excluded it. Total run time: 54.2 minutes on a Colab T4.

### Final run: DS4 excluded, epoch cap raised to 100

Same pipeline (local per-cycle none baseline, leakage-safe contiguous calibration split, 1D CNN, PCA 20 components, window 50, stride 3) with DS4 removed and epoch cap raised from 70 to 100. Total run time: 47.7 minutes on a Colab T4.

| Held-out session | Training sessions | Accuracy |
|---|---|---|
| DS2 | DS3, DS5, DS6 | 84.0% |
| DS3 | DS2, DS5, DS6 | 61.1% |
| DS5 | DS2, DS3, DS6 | 71.4% |
| DS6 | DS2, DS3, DS5 | 69.2% |
| **Mean** | | **71.4%** |
| Std | | 8.2% |

This run clears the 70% benchmark. Standard deviation also dropped from 12.6% to 8.2%, indicating the model became more consistent across sessions, and DS6 improved from 63.1% to 69.2% with the extra training epochs.

## What this showed

Model architecture changes alone did not solve cross-session generalization for this data. The real lever was how the data was prepared and how the train/test split was structured. The local none-baseline was the most physically grounded improvement, since it directly captures AGC and hardware drift that varies within a session. The leakage-safe split was the most important correctness fix, since it caught a result that looked excellent but wasn't real.
