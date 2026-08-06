# Network Traffic Classification using ML

A comparative analysis of **Decision Tree**, **Random Forest**, and **Neural Network** models for
classifying network traffic by device category — with the explicit constraint that the resulting
model has to run on an **edge gateway with limited memory and compute**.

Most traffic-classification work reports accuracy and stops there. This project measures accuracy
*alongside* model size, peak memory usage, and per-flow prediction time, so the models can be judged
on whether they would actually fit on a Raspberry-Pi-class device.

This is the code and report for my bachelor assignment (Advanced Technology, Faculty of Science &
Technology, University of Twente, April 2024).

---

## Motivation

As IoT deployments grow, network operators need to know *what kind of device* is behind a flow in
order to apply the right Quality of Service (QoS) policy. The traditional approaches don't hold up:

- **Deep Packet Inspection (DPI)** is accurate but computationally expensive, and becomes a
  bottleneck on high-traffic networks.
- **Port-based classification** is cheap but increasingly useless, since modern applications use
  dynamic and non-standard ports.

Machine learning on *flow statistics* avoids both problems — it never touches the payload and is
cheap enough to run inline. The open question this project addresses is whether such a model is
small and fast enough for an edge gateway. The reference target is a lightweight Edge
Gateway-as-a-Service device (Raspberry Pi 3 class: ~1 GB RAM, 16 GB storage), which sets the budget
that every model here is measured against.

## Dataset

Traffic traces collected by UNSW Sydney researchers (Sivanathan et al., 2019), captured at a network
access point in a lab environment instrumented with 28 IoT devices (Nest Dropcam, Samsung SmartCam,
iHome, TP-Link Smart Plug, NEST Protect, Netatmo, and others) plus non-IoT devices. Roughly 800k
packets per day were captured; this project uses the days from 2016-09-23 to 2016-10-12.

The CSVs contain one packet per row and do **not** carry device labels, so devices were labelled by
mapping the `eth.src` MAC address against the device list published by the dataset authors. Each
device maps to one of six categories:

| Device Category   | Packets   | Share |
|-------------------|-----------|-------|
| Cameras           | 2,527,215 | 28.6% |
| Non-IoT           | 2,311,906 | 26.2% |
| Health-Monitor    |   565,079 |  6.4% |
| Hubs/Controllers  |   522,818 |  5.9% |
| Energy Management |   318,764 |  3.6% |
| Appliances        |   162,238 |  1.8% |

The category is the prediction target — it is what determines the QoS requirements of the device.

> The traffic CSVs are **not included in this repository.** They are available from the UNSW Sydney
> IoT Analytics group; see the [paper][unsw-paper] the dataset accompanies.

## Feature engineering

Raw per-packet attributes (packet ID, timestamp) carry little signal on their own, so packets are
aggregated into **flows** — grouped by `(IP.src, IP.dst, eth.src)` — within a fixed time window.
Each flow contributes one training example described by:

`total_packets` · `total_bytes` · `mean_packet_size` · `max_packet_size` · `packet_rate` ·
`byte_rate` · `mean_inter_arrival` · `max_inter_arrival` · `IP.src` · `IP.dst` · `eth.src`

Packet and byte rates are the totals divided by flow duration; inter-arrival time is the gap between
consecutive packets in a flow. All of these are cheap to compute incrementally, which matters for a
model meant to run in real time.

The **time window is treated as an experimental variable**, since it trades off two things: longer
windows produce more accurate flow statistics, but aggregate more packets per example and therefore
yield fewer training examples. Four windows were evaluated: **15 s, 30 s, 60 s, and 300 s**.

Training and test sets are split **by day**, not by random sampling, so the models are evaluated on
traffic they have genuinely never seen.

## Models

| Model | Library | Notes |
|-------|---------|-------|
| Decision Tree  | scikit-learn | Hyperparameters tuned with `GridSearchCV` |
| Random Forest  | scikit-learn | Hyperparameters tuned with `GridSearchCV` |
| Neural Network | TensorFlow/Keras | 6 hidden layers, ReLU, dropout between layers, Adam, early stopping (batch size 64, up to 50 epochs) |

SVM was considered and deliberately excluded: at O(n²)–O(n³) training complexity it would not scale
to this dataset, for likely no accuracy gain.

## Results

All models were trained on an ASUS Vivobook 15 Pro (11th-gen Intel i9 @ 2.5 GHz, 16 GB RAM, Windows
11), with random state fixed to 100 for reproducibility.

### Accuracy and F1 score

| Model | 15 s | 30 s | 60 s | 300 s |
|-------|------|------|------|-------|
| Decision Tree  | 0.79 / 0.79 | 0.80 / 0.81 | 0.79 / 0.79 | 0.72 / 0.72 |
| Random Forest  | 0.93 / 0.92 | 0.95 / 0.95 | **0.96 / 0.96** | 0.91 / 0.91 |
| Neural Network | 0.91 / 0.92 | 0.95 / 0.95 | **0.96 / 0.96** | 0.95 / 0.94 |

<sub>accuracy / F1 score</sub>

### Prediction time per flow (µs)

| Model | 15 s | 30 s | 60 s | 300 s |
|-------|------|------|------|-------|
| Decision Tree  |  3 |  5 |  5 | 25 |
| Random Forest  |  8 | 18 | 10 | 21 |
| Neural Network | 34 | 50 | 45 | 69 |

### Model size and peak memory during prediction

| Interval | Model | Size (MB) | Peak memory (MB) |
|----------|-------|-----------|------------------|
| 15 s  | DT | 0.010 | 223.28 |
|       | RF | 0.757 | 290.90 |
|       | NN | 0.580 | 2461.05 |
| 30 s  | DT | 0.010 | 233.00 |
|       | RF | 1.859 | 282.23 |
|       | NN | 0.580 | 2121.80 |
| 60 s  | DT | 0.010 | 225.12 |
|       | RF | 1.002 | 270.44 |
|       | NN | 0.580 | 1288.58 |
| 300 s | DT | 0.010 | 236.39 |
|       | RF | 1.910 | 249.45 |
|       | NN | 0.580 | 185.06 |

## Key findings

- **Random Forest at a 60-second window is the best all-round choice** — 0.96 accuracy and F1, 10 µs
  per flow, ~1 MB on disk, and ~270 MB peak memory. It offers the best balance of the three metrics.
- **Neural Networks match RF on accuracy but not on cost.** They tie at 0.96 / 0.96 at 60 s, but are
  4–5× slower per prediction and — at short intervals — dramatically more memory-hungry (2.4 GB peak
  at 15 s, which exceeds the target device's RAM entirely). Their peak memory falls sharply as the
  interval grows, since there are fewer examples to hold at once.
- **Decision Trees are the cheapest and the least accurate.** A consistent 0.01 MB and as little as
  3 µs per flow, but accuracy caps out around 0.80 and degrades to 0.72 at 300 s. Worth it only when
  resources are the binding constraint and precision isn't critical.
- **Intermediate time windows win.** 60 s is the sweet spot: long enough for flow statistics to be
  descriptive, short enough to leave a sufficient number of training examples. Performance falls off
  at 300 s for the tree-based models.
- **All three model families fit within the edge device's hardware budget** — with the caveat that
  the NN's peak memory at short intervals rules those configurations out in practice.

The full analysis, including related work and methodology, is in
[`Bachelor_Assignment__AT_final.pdf`](Bachelor_Assignment__AT_final.pdf).

## Repository contents

```
Network_traffic_classification_withMemoryAndSpeed.ipynb   Flow extraction, training, and
                                                          memory/speed instrumentation
Bachelor_Assignment__AT_final.pdf                         Full report with all results
```

## Running the notebook

There is no `requirements.txt`; the notebook was developed against Python 3.11 with:

```
pandas  numpy  scikit-learn  tensorflow  memory_profiler  joblib
```

To reproduce:

1. Obtain the UNSW traffic CSVs and place them next to the notebook, named by capture date
   (`16-09-23.csv`, `16-09-24.csv`, …). Training uses 09-23, 09-24, 09-25, 09-26, 09-27, 09-29,
   10-04, and 10-12; testing uses 09-30, 10-01, 10-02, 10-03, and 10-05.
2. Run the cells in order — later cells depend on dataframes and models built by earlier ones.

### A note on the committed notebook

The notebook is the **working version** used to instrument memory and speed, not a cleaned-up
reproduction script. Two things worth knowing:

- It covers the **15 s and 60 s** intervals only. The 30 s and 300 s figures reported above come from
  the report.
- It uses different Random Forest and Neural Network configurations than the report's final tuned
  ones (the notebook's network is a 128 → 64 → 32 stack with batch size 32), so its printed numbers
  differ somewhat from the tables above. The report's tables are the authoritative results.

## Reference

Dataset from: A. Sivanathan et al., "Classifying IoT Devices in Smart Environments Using Network
Traffic Characteristics," *IEEE Transactions on Mobile Computing*, vol. 18, no. 8, pp. 1745–1759,
2019. [doi:10.1109/TMC.2018.2866249][unsw-paper]

[unsw-paper]: https://doi.org/10.1109/TMC.2018.2866249
