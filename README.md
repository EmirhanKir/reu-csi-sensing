# Communication-Aware Passive WiFi Sensing

Summer 2026 REU research at Virginia Commonwealth University's MoWiNG Lab, on how background network traffic and cross-session domain shift affect passive WiFi CSI (Channel State Information) sensing for human activity and intrusion-behavior recognition.

Researcher: Emirhan Kir (Rowan University)
Mentor: Nafeez Fahad
Faculty Advisor: Dr. Eyuphan Bulut, VCU MoWiNG Lab

This is lab research, not a solo project, and the paper built on this work is still in progress.

## What this is

Ordinary WiFi hardware picks up how a room's signal bends and scatters around a moving body, no camera, no wearable, no line of sight needed. This project uses that passive signal two ways: first, to study how background network traffic (streaming, downloads, mixed load) degrades activity-recognition accuracy compared to a clean active-sensing baseline; second, to test whether the same passive approach can reliably detect intrusion-relevant behaviors like door handle jiggling and dragging.

## Key findings

- **Passive sensing accuracy is traffic-dependent.** Streaming traffic starves the passive sniffer of usable CSI packets, capturing roughly 10x fewer packets than active sensing in the same room, same duration, same hardware.
- **Active sensing (5 activities: Walking, Toe_Tap, Jumping_Jacks, Squat, Hand_Raise): 81% accuracy.** Toe_Tap and Squat were the most reliably detected; Jumping_Jacks was the weakest class, most often confused with Hand_Raise due to similar arm motion.
- **Passive intrusion-behavior detection (3 classes: Door_Handle_Jiggle, Door_OpenClose, Dragging): 75% accuracy** using only ambient WiFi packets, no dedicated transmitter. Door_OpenClose was detected most reliably; Door_Handle_Jiggle and Dragging were weaker and more often confused with each other, since both are subtler, sustained motions.
- **Cross-session generalization is the harder problem.** Within-session accuracy on the same pipeline was consistently ~86%, but leave-one-session-out (LOO) accuracy across different data collection sessions started around 35-45%, exposing a real domain-shift problem rather than a modeling one.

## Repository layout

```
notebooks/
  Cross_Session_LOO_No_DS4.ipynb      cross-session leave-one-out pipeline (final, DS4 excluded)
  Thief_S11_Aligned_Corrected.ipynb   3-class intrusion behavior classifier (S11-aligned pipeline)
docs/
  Cross_Session_Experiments_Log.md    full experiment history and results for the LOO work
```

## Cross-session LOO pipeline

The core challenge: within-session accuracy (~86-93%) held up fine, but leave-one-session-out accuracy sat around 35-45% across four data collection sessions (DS2, DS3, DS5, DS6), well below a 70% benchmark set by Dr. Bulut. Getting from there to a passing result took a series of preprocessing fixes more than model changes:

1. **Per-session z-score normalization** to correct for each session's own AGC and hardware drift
2. **Local per-cycle none-baseline subtraction**, pairing each activity segment with the none/rest segment immediately preceding it, rather than a single global session average
3. **A leakage-safe calibration split.** An early version used a random shuffle to split calibration/eval windows, which let near-duplicate overlapping windows (adjacent windows share 47 of 50 timesteps at stride 3) leak across the split and produced an inflated, untrustworthy 98-99% accuracy. Switching to a contiguous time-block split with dropped boundary windows fixed this and dropped accuracy to an honest, real number

Final result: **71.4% mean LOO accuracy** (DS2/DS3/DS5/DS6, DS4 excluded for known data quality issues), clearing the 70% benchmark, with standard deviation improving from 12.6% to 8.2% over the all-sessions run. Full experiment-by-experiment history, including the dead ends, is in [`docs/Cross_Session_Experiments_Log.md`](docs/Cross_Session_Experiments_Log.md).

## Intrusion behavior detection

A parallel, forensics-framed track: can the same passive CSI approach detect intrusion-relevant behaviors specifically? Using a reflashed ESP32 in passive sniffer mode, a dataset was collected across three behaviors (door handle jiggling, door open/close, dragging), each preceded by a none-baseline segment.

Final classification report (11,016 test windows):

| Class | Precision | Recall | F1 |
|---|---|---|---|
| Door_Handle_Jiggle | 0.76 | 0.61 | 0.68 |
| Door_OpenClose | 0.82 | 0.89 | 0.85 |
| Dragging | 0.62 | 0.72 | 0.67 |
| **Overall accuracy** | | | **0.75** |

This result, along with the active-sensing 5-activity baseline (81% accuracy), was presented at the DFRWS USA 2026 REU poster session under the title *"Communication-Aware Passive WiFi Sensing under Dynamic Traffic and Environmental Conditions."*

## Hardware and pipeline

- ESP32 transmitter/receiver pair with a Raspberry Pi running the CSI-Pi system for active sensing; a reflashed ESP32 in passive sniffer mode for passive/intrusion sensing
- Raw CSI parsed into 64 subcarrier amplitudes (magnitude of interleaved imaginary/real pairs), 12 null subcarriers dropped, 52 active subcarriers retained
- Rolling-mean denoising, local baseline subtraction, per-session z-score normalization, PCA (fit on training sessions only), sliding-window segmentation, then a 1D CNN classifier (Conv1D → BatchNorm → Dropout → Dense)
- Pipeline built and iterated in Google Colab against data stored in Google Drive

## Literature this work builds on

Hernandez & Bulut (COMST 2023), Fahad & Bulut (ICNC 2025, WoWMoM 2025, Infocom 2025), Akpabio & Bulut (PerCom 2025, PhysiFi), McDonough et al. (MobiHoc 2023, Wi-Alert), and Hu et al. (MobiCom 2024, "What You Need Is a Good CSI").
