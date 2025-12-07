# 🚨 Statistical vs ML Intrusion Detection (CIC-IDS 2017)

Side-by-side notebooks that build a simple statistical detector (robust z-score) and an Isolation Forest on two CIC-IDS 2017 traces: the Wednesday DoS variants and Friday DDoS. Both pipelines keep time order, fit only on benign windows to avoid leakage, and compare metrics/visuals/alerts on a held-out tail.

---

## Dataset

* Format: flow-level CSV exported from PCAP
* Traffic includes multiple application-layer DoS attacks
* Labels: `BENIGN` or specific attack types

The notebook synthesizes timestamps to preserve order and supports time-based resampling.
No label information is used during training for anomaly detectors.

---

## Repository Structure

```
├── wednesday_dos_variants.ipynb   # main notebook 1
├── friday_ddos.ipynb              # main notebook 2
├── requirements.txt
├── README.md
```

## Approach

### 1. Preprocessing

* Strip whitespace from feature names
* Synthesize a monotonic timestamp index (10 ms per flow)
* Create per-flow totals:

  * `packets` = Fwd + Bwd packets
  * `bytes` = Fwd + Bwd bytes

### 2. Feature selection

We focus on DoS-relevant indicators:

* **Volume**: packets, bytes
* **Payload structure**: packet length mean/std/var, min/max
* **Handshake manipulation**: SYN, ACK, RST, FIN, PSH flag counts
* **Directionality**: subflow forward/backward packets

These capture low-rate connection exhaustion (slow HTTP) and high-volume floods.

---

## Time-window aggregation

Flows are resampled into **5-second windows**:

* Volume → **sum**
* Size statistics → **mean**
* Port dispersion → **unique destination ports**

Each window is labeled as attack if **>50% of flows are non-benign**.

This produces a robust window-level ground truth while avoiding individual misclassification noise.

---

## Detectors

### Baseline — **Z-Score**

* Build benign mean/std from training windows
* Score = sum of absolute normalized deviations
* Threshold = 99th percentile of benign
* Optimized for low false positives

### Machine Learning — **Isolation Forest**

* Unsupervised tree-based anomaly detector
* Trained **only on benign windows**
* `contamination = 0.02` for DoS variants and `contamination = 0.05` for DDoS
* Produces per-window anomaly predictions

Zero label leakage: labels are used **only for evaluation**.

---

## Evaluation

Train/test split is chronological:

* Find all attack windows
* Anchor test at last 20% of attack windows
* Ensures test contains real positives

Metrics:

* Precision (alert quality)
* Recall (attack coverage)
* F1 (balanced score)
* Confusion matrix (FP/FN tradeoffs)

---

## Visualization

Plots show:

* Packets per window over time
* Test region highlighted
* Detector anomaly markers (z-score vs IF)

These visually reveal:

* Which spikes each detector captures
* Missed slow-rate patterns
* Volumetric floods that Isolation Forest detects reliably

---

## Real-time demo

A simple loop prints alerts for anomalous windows:

```
[ALERT] timestamp | packets=... | SYN=... | attack=0/1 | IF=... | Z=...
```

Useful for live demonstration or SOC console simulation.

---

## Results (high-level)

* **Z-score** outperforms Isolation Forest on DoS because handshake imbalance creates statistically extreme deviations in a few windows.
* LOIC produces extremely uniform, repeated bursts, which **Isolation Forest** isolates very easily because the high-dimensional pattern is consistent and far from benign clusters.
* Both models can miss low-rate attacks that change only a few features slightly

The notebook is designed for **interpretability > accuracy** - methods are transparent, reproducible, and easy to extend.

---

## Requirements

Download data files `Wednesday-workingHours.pcap_ISCX.csv` and `Friday-WorkingHours-Afternoon-DDos.pcap_ISCX.csv` from here: https://www.kaggle.com/datasets/chethuhn/network-intrusion-dataset

Install dependencies:

```bash
pip install -r requirements.txt
```

or minimally:

```
pandas
numpy
scikit-learn
matplotlib
```
