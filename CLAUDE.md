# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running Node-RED

```powershell
# Start Node-RED (uses settings.js in this directory automatically)
node-red

# Start with debug logging
node-red --verbose

# Access the editor at http://localhost:1880
# Access the dashboard at http://localhost:1880/ui
```

Port defaults to `1880` or the `PORT` environment variable ([settings.js](settings.js#L146)).

## Project Overview

This is a Node-RED IoT monitoring system for oil well operations. All flow logic lives in [flows.json](flows.json); [settings.js](settings.js) controls the runtime. **[flows_2.json](flows_2.json) is a manual backup** — do not treat it as an active or alternate flow.

## Installed Packages ([package.json](package.json))

- `node-red-contrib-brain-js` — provides the `neuralnet` node (LSTM via brain.js)
- `node-red-contrib-gemini` — provides `gemini-generate-content` and `gemini-api-key` nodes
- `node-red-dashboard` — provides `ui_gauge`, `ui_text`, `ui_template`, `ui_tab`, `ui_group` nodes

## Flow Architecture (`flows.json` — "Flow nuevo")

Data flows through four sequential stages:

### 1. Ingestion & Auth
- **MQTT In** listens on topic `cursoiot/well1/sensor` via HiveMQ public broker (`broker.hivemq.com:1883`)
- **Simulador** inject node fires every second as a local test source (bypasses MQTT)
- **Validar token MQTT** checks `msg.payload.token` against the `WELL_TOKEN` flow environment variable (default: `well01-secret-2024`); strips the token from payload on success
- **Ruteo por autenticacion** switch routes authenticated vs. rejected messages

### 2. Sensor Processing
- Authenticated messages go to **Simulador sensores pozo**, which scales raw pressure (0–1100 ADC units → 200–3000 PSI) and normalizes vibration
- Four parallel **change** nodes extract individual sensor values (`pressure`, `temperature`, `vibration`, `floww`) to feed the dashboard gauges
- **Buffer ventana pozo** accumulates a 30-sample sliding window in `flow` context; passes downstream only when full
- **Calculo de features** computes avg, std, min, max, and trend for all four sensors over the window

### 3. Dual AI Classification
Both run in parallel after features are computed:

**Rule-based agent** (`Agente de AI`):
- Scores alerts based on thresholds (e.g., `pressure_avg < 2400`, `vibration_avg > 4.0`)
- Applies bonus scores for combined patterns (e.g., pressure drop + vibration rise)
- Classifies as `normal` (score < 4), `warning` (4–7), or `anomaly` (≥ 8)

**LSTM neural network** (`neuralnet` node + brain.js):
- Auto-trains on deploy via an inject node that fires 2 s after startup (`Build NN Training Dataset`)
- Training data: 800 synthetic sequences of 10 timesteps each (240 normal, 280 warning, 280 anomaly)
- Inference input: last 10 samples from the 30-sample buffer, normalized to [0,1] (`Prepare NN Input`)
- Output interpreted by `Interpret NN Output` → picks highest-confidence class
- Result flows through `function 1` → `Ruteo por estado` switch (normal / warning / anomaly)

### 4. Gemini AI Operator Report (ARIA)
- `Formatear entrada Gemini` maps features + NN output into a structured Spanish-language payload
- `Delay 30s` rate-limits calls to Gemini to 1 per 30 seconds (drops excess messages)
- `gemini-generate-content` calls Gemini with a detailed system prompt; ARIA generates a structured operational report in Spanish
- Report displayed in a `ui_template` dashboard widget ("Recomendaciones del Operador IA")
- Model configured: `gemini-3.1-flash-lite`

## Dashboard Layout (`/ui`)

Single tab "Pozo Petrolero" with three groups:
- **Sensors** — four gauges: Well Pressure (PSI), Fluid Temp (°C), Pump Vibration (mm/s), Production Flow (bbl/day)
- **Status Monitoring** — `ui_text` showing current well status from NN output
- **Recomendaciones del Operador IA** — full-width `ui_template` displaying ARIA's Spanish report

## Key Configuration Notes

- `WELL_TOKEN` is a **flow-level environment variable** set on the "Flow nuevo" tab (not a system env var). Change it in the flow tab's properties panel in the editor.
- Credentials are encrypted in [flows_cred.json](flows_cred.json) using the `_credentialSecret` in [.config.runtime.json](.config.runtime.json). Do not change `credentialSecret` in settings — it will break decryption of existing credentials (Gemini API key).
- `flowFilePretty: true` keeps `flows.json` human-readable and diff-friendly.
- `functionExternalModules: true` allows Function nodes to `require()` npm packages directly.
- The Projects feature is disabled (`editorTheme.projects.enabled: false`).

## Sensor Thresholds (for rule-based agent)

| Sensor | Normal | Warning | Anomaly |
|---|---|---|---|
| Pressure (PSI) | ≥ 2400 | < 2400 | < 2000 |
| Temperature (°C) | 70–95 | 95–110 | > 110 |
| Vibration (mm/s) | < 4 | 4–8 | > 8 |
| Flow (bbl/day) | > 40 | 20–40 | < 20 |
