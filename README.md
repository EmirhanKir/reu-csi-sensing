# Communication-Aware Passive WiFi Sensing

Summer 2026 REU research at Virginia Commonwealth University's MoWiNG Lab, on how background network traffic affects passive WiFi CSI (Channel State Information) sensing for human activity and intrusion-behavior recognition.

**Emirhan Kir** (Rowan University), **Nafeez Fahad** and **Dr. Eyuphan Bulut** (VCU MoWiNG Lab)

This is lab research, not a solo project.

## What this is

Passive WiFi sensing infers human activity from ambient Channel State Information without any dedicated sensing hardware. This work builds active and passive sensing pipelines on the same ESP32 platform, in the same physical space, and directly measures how background network traffic (browsing, streaming, downloading) degrades passive sensing accuracy, something no prior work had systematically isolated. The same passive pipeline is then applied to a security-relevant task: detecting intrusion behaviors like door handle jiggling and dragging from ambient WiFi packets alone.

## Key results

- **Active sensing (5 activities, general recognition): 81% accuracy** (macro P/R/F1 all 0.81, 9,289 test windows). Toe taps and squats were most reliable (95% and 89% recall); jumping jacks was the weakest class, most often confused with hand raises due to similar arm motion.
- **Passive sensing (3 classes, intrusion behavior): 75% accuracy** (macro F1 0.73, 11,016 test windows), using only ambient WiFi packets, no dedicated transmitter. Door open/close was most reliable (89% recall); door handle jiggling and dragging were weaker and confused with each other more often.
- **Traffic conditions cause a large, structural drop in passive accuracy.** Leave-one-recording-out accuracy fell from 56.1% (active baseline) to 45.8% (download) to 25.4% (mixed) to 14.2% (streaming). This tracks raw packet counts almost exactly: active sensing captured ~642,000 packets in the same window that streaming captured only ~63,000, an order of magnitude fewer.
- **This is a data starvation problem, not just a modeling problem.** Streaming traffic is bandwidth-heavy and bursty in a way that crowds out the smaller, frequent packets a passive sniffer needs, meaning accuracy can't be recovered by better models alone when too few packets are arriving in the first place.

Full methodology, related work, and discussion are in the paper: [`docs/WiFi_Sensing_Paper.pdf`](docs/WiFi_Sensing_Paper.pdf). The DFRWS USA 2026 poster is in [`docs/REU_Poster.pptx`](docs/REU_Poster.pptx).

## Repository layout

```
notebooks/
  cross_session_loo.ipynb     leave-one-out pipeline for cross-session/traffic-condition activity recognition
  intrusion_detection.ipynb   3-class passive intrusion behavior classifier (75% accuracy result)
docs/
  WiFi_Sensing_Paper.pdf      full paper
  REU_Poster.pptx             DFRWS USA 2026 poster
```

## Pipeline

Both sensing modes share the same first stages, then diverge:

- **Shared:** raw CSI capture, per-subcarrier amplitude extraction, denoising
- **Active sensing:** PCA on the denoised amplitude features, then a DNN/CNN classifier, since the active link's steady, evenly spaced packet stream makes a fixed feature representation reliable
- **Passive sensing:** per-cycle local baseline subtraction and per-session z-score normalization, then time-based windowing (rather than fixed packet count), then an attention-based classifier, which lets the model weight whatever packets actually arrive in a window rather than assuming an evenly spaced sequence

This adjustment matters because passive packet arrival is dictated entirely by surrounding network traffic, not by the sensing system itself.

## Hardware

- Two ESP32 microcontrollers running the ESP32 CSI Tool: one configured as a TX-RX link for active sensing at a fixed, evenly spaced probing rate; a second reflashed into passive sniffer mode, placed near an ordinary router, listening to whatever traffic it happened to be carrying
- Router configured through a scripted scenario generator to reproduce controlled traffic mixes (browsing, uploading, downloading, streaming, idle) across recordings
- Both sensing modes collected in the same room with the same equipment layout, removing location and multipath as a confound

## Literature this work builds on

Hernandez & Bulut (COMST 2023; WoWMoM 2020), McDonough et al. (MobiHoc 2023, Wi-Alert), Fahad & Bulut (INFOCOM Workshops 2025; WoWMoM 2025), Fahad, Touhiduzzaman & Bulut (ICNC 2025), Akpabio & Bulut (PerCom Workshops 2025, PhysiFi), Hu et al. (MobiCom 2024, "What You Need Is a Good CSI"), and Dong, Yang & Srivastava (UniFi, arXiv 2025). Full references in the paper.

## Acknowledgment

This work is partially supported by the National Science Foundation under Grant Award Number 2447453.
