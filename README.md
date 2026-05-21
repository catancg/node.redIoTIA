# Node-RED IoT Oil Well Monitoring with AI

A real-time oil well monitoring system built on Node-RED. Sensor data arrives via MQTT, is classified by a brain.js neural network trained on the [SKAB anomaly detection dataset](https://github.com/waico/SKAB), and an operator report is generated in Spanish by Google Gemini (ARIA).

---

## What it does

1. **Ingests** sensor readings from an oil well (pressure, temperature, vibration, flow) over MQTT.
2. **Buffers** 10 consecutive readings into a fixed window.
3. **Extracts** 20 statistical features (avg, std, min, max, trend × 4 sensors) from the window.
4. **Classifies** the well state as `normal` or `anomaly` using a feedforward neural network (MLP).
5. **Generates** a structured Spanish-language operational report via Gemini when the classification changes.
6. **Tracks** live prediction accuracy against the ground-truth labels embedded in the MQTT data.
7. **Displays** everything on a real-time dashboard at `http://localhost:1880/ui`.

---

## Flow Architecture

Data moves through four sequential stages defined in `flows.json`.

### Stage 1 — Ingestion & Authentication

| Node | Role |
|---|---|
| **MQTT In** | Subscribes to `cursoiot/well1/sensor` on `broker.hivemq.com:1883` |
| **Validar token MQTT** | Validates `msg.payload.token` against the `WELL_TOKEN` flow variable. Extracts ground-truth label from the `anomaly` column if present. Rejects unauthenticated messages. |
| **Ruteo por autenticacion** | Routes authenticated messages forward; drops rejected ones to a debug node. |

The MQTT payload is expected to have this shape:

```json
{
  "well_id": "WELL-01",
  "token": "well01-secret-2024",
  "pressure": 850.0,
  "temperature": 82.5,
  "vibration": 2.1,
  "floww": 52.3,
  "anomaly": 0
}
```

The `anomaly` field (0 or 1) is the ground-truth label from SKAB, used only by the live accuracy tracker. It is stripped before reaching the classifier.

### Stage 2 — Sensor Processing

| Node | Role |
|---|---|
| **Simulador sensores pozo** | Scales raw pressure from ADC units (300–1100) to PSI (200–3000). Passes temperature, vibration, and flow through unchanged. |
| **Change nodes** (×4) | Fan out individual sensor values to the four dashboard gauges on every sample. |
| **Buffer ventana pozo** | Accumulates exactly 10 samples into a fixed, non-overlapping window. Emits when full and **resets to empty** — predictions fire every 10 samples, not every sample. Also computes the majority-vote ground-truth label for the window. |
| **Calculo de features** | Computes avg, std, min, max, and trend for each of the 4 sensors over the 10-sample window — 20 features in total. |

### Stage 3 — Neural Network Classification

| Node | Role |
|---|---|
| **Prepare NN Input** | Normalizes the 20 features to [0, 1] using fixed physical ranges (e.g. pressure: 200–3000 PSI). Blocks inference until `nnReady` is true. |
| **neuralnet** | brain.js feedforward MLP (nntype: 0). Receives either training data (`msg.trainData`) or inference data (`msg.runData`). |
| **NN Ready Gate** | After training completes, sets `nnReady = true` and triggers holdout evaluation. Routes inference messages to `Interpret NN Output`; routes eval messages to `Accumulate Accuracy`. |
| **Interpret NN Output** | Picks the class with the highest activation from `msg.decision` (`normal` or `anomaly`). Assembles the result payload. |
| **function 1** | Trims the payload to `{status, confidence, raw}` for the status switch. |
| **Ruteo por estado** | Routes the result by class to the appropriate debug and dashboard nodes. |

**Training** happens automatically 2 seconds after deploy via an inject node (`Auto-train on deploy`):

```
Auto-train on deploy  →  Build NN Training Dataset  →  neuralnet  →  NN Ready Gate
```

### Stage 4 — Gemini ARIA Operator Report

| Node | Role |
|---|---|
| **Status Change Gate** | Sends a message to Gemini only after 3 consecutive identical predictions that differ from the last sent status. Prevents redundant API calls. |
| **Formatear entrada Gemini** | Maps sensor features and NN output into a structured Spanish-language payload. |
| **Delay 30s** | Rate-limits Gemini calls to one per 30 seconds, dropping excess messages. |
| **gemini-generate-content** | Calls Gemini (`gemini-3.1-flash-lite`) with a detailed system prompt. ARIA produces a structured report with operational status, diagnosis, alerts, and recommended actions. |
| **Recomendaciones ARIA** | `ui_template` widget that displays the report on the dashboard. |

---

## Dashboard (`/ui`)

Single tab **"Pozo Petrolero"** with three groups:

| Group | Widgets |
|---|---|
| **Sensors** | Four gauges: Well Pressure (PSI), Fluid Temp (°C), Pump Vibration (mm/s), Production Flow (bbl/day) |
| **Status Monitoring** | Current well status (normal / anomaly), holdout model accuracy, live accuracy |
| **Recomendaciones del Operador IA** | Full-width panel displaying ARIA's latest Spanish report |

---

## Training Data — `skab_traindata.json`

### Origin

The training data is derived from the [SKAB (Skoltech Anomaly Benchmark)](https://github.com/waico/SKAB) dataset — a public benchmark for time-series anomaly detection on industrial pump data. The source files used are `1.csv` through `4.csv` from the `data/other/` folder, containing pump sensor readings with labeled anomaly events.

### Generation (`gen_traindata.py`)

The script `gen_traindata.py` converts raw SKAB CSV rows into sequences the MLP can train on:

1. **Label extraction** — `anomaly=1` → `anomaly`, `anomaly=0` → `normal`. (The `changepoint`/warning class is not used.)
2. **Sensor remapping** — Raw SKAB values are remapped into realistic oil-well ranges per label using the same conversion constants as `skab_replay.py`, so training features sit in the same value space as live inference features.
3. **Normalization** — All four sensors are normalized to [0, 1] using fixed physical ranges matching `Prepare NN Input`:
   - Pressure: 200–3000 PSI
   - Temperature: 50–130 °C
   - Vibration: 0–40 mm/s
   - Flow: 0–100 bbl/day
4. **Sequence building** — Non-overlapping windows of 10 consecutive timesteps (step = 10). This matches the `Buffer ventana pozo` fixed-window size exactly, ensuring training and inference see the same feature distributions for stable periods.
5. **Sampling** — All anomaly sequences are kept; normal sequences are randomly downsampled to produce a balanced dataset. Total: 384 sequences (248 normal / 136 anomaly).

### Format

Each entry in `skab_traindata.json` is:

```json
{
  "input": [
    { "pressure": 0.75, "temperature": 0.38, "vibration": 0.05, "flow": 0.52 },
    ...
  ],
  "output": { "normal": 1, "anomaly": 0 }
}
```

`input` is an array of 10 timesteps. `output` is a one-hot label.

### How it feeds the MLP

At deploy time, `Build NN Training Dataset` reads `skab_traindata.json`, computes the 20 statistical features (avg, std, min, max, trend) over each 10-timestep sequence, applies an 80/20 train/holdout split, and passes the training portion directly to the `neuralnet` node. The holdout portion is used immediately after training to compute the **Model Accuracy** metric shown on the dashboard.

---

## Setup

### Prerequisites

- [Node.js](https://nodejs.org/) 18+
- Node-RED: `npm install -g node-red`
- A Google Gemini API key

### Install

```bash
cd ~/.node-red        # or wherever this repo is cloned
npm install
node-red
```

### Configure credentials

The Gemini API key is stored encrypted in `flows_cred.json` (not committed). Set it through the Node-RED editor:

1. Open `http://localhost:1880`
2. Double-click the `gemini-api-key` node
3. Enter your Gemini API key and deploy

### WELL_TOKEN

The MQTT authentication token defaults to `well01-secret-2024`. To change it, open the **Flow nuevo** tab properties in the Node-RED editor and edit the `WELL_TOKEN` environment variable.

---

## Replaying SKAB data via MQTT

Use `skab_replay.py` to publish SKAB CSV rows to the MQTT broker, simulating a live well:

```bash
pip install pandas paho-mqtt
python skab_replay.py --file SKAB/data/other/1.csv
python skab_replay.py --folder SKAB/data/other/ --delay 0.5
python skab_replay.py --file SKAB/data/other/1.csv --verbose
```

Each message published includes the `anomaly` ground-truth column so the **Live Accuracy** tracker can evaluate predictions in real time.

---

## Key design decisions

**Why a fixed non-overlapping buffer?**
The MLP is trained on non-overlapping 10-sample windows from SKAB. Using a sliding window at inference would expose the model to mixed-class windows (transitions) it never saw during training, degrading accuracy. A fixed buffer that resets after each prediction keeps the inference feature distribution aligned with training.

**Why pre-computed statistics instead of raw sequences?**
brain.js feedforward networks do not handle sequences natively. Computing avg, std, min, max, and trend over the 10-sample window compresses the temporal information into 20 scalar features, which a standard MLP can classify effectively.

**Why majority-vote ground-truth labels?**
`_trueLabel` is computed from the majority class across the 10 samples in each fixed window. This is more accurate than using only the latest sample's label, particularly for windows that straddle a normal-to-anomaly transition.
