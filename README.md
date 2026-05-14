<div align="center">
  <img src="docs/assets/Oct 2, 2025, 03_56_13 AM.png" alt="AnomAI" width="600"/>
  
  ### Intelligence That Never Sleeps
  
  An AI-powered system monitoring platform that detects anomalies in real-time 
  using an LSTM Autoencoder. Deploy a lightweight agent on any machine and 
  monitor your entire device fleet from a central dashboard.
</div>

---

## Preview

**Device Fleet** — Monitor all machines, anomaly alerts, and health status at a glance
![Device Fleet](docs/assets/device fleet2.png)

**Per-Device Dashboard** — Live CPU, Memory, Disk, and Network metrics with anomaly history
![Dashboard](docs/assets/dashboard2.png)

**Offline Device State** — Auto-detected after 3 minutes without metrics
![Offline Device](docs/assets/offline2.png)

**Resilient to Backend Outages** — Dashboard shows last known state when backend is unreachable
![Backend Offline](docs/assets/offline.png)

**Auto Device Registration** — Agents register themselves; no manual setup
![Add Device](docs/assets/Add device.png)

**Daily PDF Report** — Auto-generated every 24 hours  
📄 [View sample report](docs/assets/anomai-report(2).pdf)

---

## Features

- **Real-time monitoring** — CPU, Memory, Disk, and Network I/O collected every 10 seconds
- **LSTM Autoencoder** — unsupervised anomaly detection trained on each machine's own behavior
- **Two-tier classification**
  - `PERFORMANCE ISSUE` — values genuinely high (CPU > 85%, Memory > 90%)
  - `STATISTICAL DEVIATION` — unusual pattern but values within safe ranges (logged locally only)
- **Auto device registration** — agent registers itself on first run, no manual setup
- **Online / Offline detection** — device marked offline if no metric received in 3 minutes
- **Daily PDF report** — auto-generated every 24 hours and uploaded to the dashboard
- **Offline buffering** — anomalies stored locally in SQLite when backend is unreachable, synced when connection restores
- **Fleet dashboard** — monitor all devices, active alerts, and health scores from one place

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Monitored Machine                       │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                     Python Agent                        │   │
│   │                                                         │   │
│   │  psutil → PowerTransform → MinMaxScale → LSTM Window   │   │
│   │                    ↓                                    │   │
│   │           Reconstruction Error > Threshold?            │   │
│   │           ↙ yes                        ↘ no            │   │
│   │   classify_anomaly()               [NORMAL] log        │   │
│   │   ↙ performance    ↘ deviation                         │   │
│   │  Send to backend   Log locally only                    │   │
│   └──────────────────────────┬──────────────────────────────┘   │
└──────────────────────────────│──────────────────────────────────┘
                               │ HTTPS (JWT)
                               ▼
              ┌────────────────────────────────┐
              │       FastAPI Backend          │
              │   SQLite · JWT Auth            │
              │   /devices  /metrics           │
              │   /anomalies  /reports         │
              └────────────────┬───────────────┘
                               │
                               ▼
              ┌────────────────────────────────┐
              │      Laravel Dashboard         │
              │   Device Fleet · Live Charts   │
              │   Anomaly Alerts · PDF Reports │
              └────────────────────────────────┘
```

---

## How the ML Works

1. **Data collection** — `collect_training_data.py` records 1,440 samples (4 hours) of normal behavior on the target machine
2. **Preprocessing** — Yeo-Johnson PowerTransform → sliding windows of 10 readings → MinMaxScaler
3. **Model** — LSTM Autoencoder (32→16→8→16→32 units) trained to reconstruct normal patterns
4. **Threshold** — set at the 98.5th percentile of validation reconstruction errors (MSE)
5. **Inference** — at runtime, if `reconstruction_error > threshold` → anomaly flagged
6. **Classification** — checks current values: genuinely high → `PERFORMANCE`, statistically unusual → `DEVIATION`

```
Normal behavior  →  low reconstruction error  →  NORMAL
Unusual pattern  →  high reconstruction error →  ANOMALY
  ├── CPU/Mem/Disk above safe thresholds  →  PERFORMANCE ISSUE  →  sent to dashboard
  └── Values within safe ranges          →  STATISTICAL DEVIATION  →  local log only
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Agent | Python, psutil, TensorFlow/Keras, scikit-learn, NumPy, reportlab |
| ML Model | LSTM Autoencoder (unsupervised) |
| Preprocessing | PowerTransformer (Yeo-Johnson), MinMaxScaler |
| Backend | FastAPI, SQLModel, SQLite, JWT |
| Frontend | Laravel, Blade, Chart.js |
| Local Storage | SQLite (offline buffer + analysis history) |

---

## Project Structure

```
AnomAI/
├── backend/                        # FastAPI backend
│   ├── app/
│   │   ├── main.py
│   │   ├── routers/
│   │   │   ├── devices.py
│   │   │   ├── metrics.py
│   │   │   ├── anomalies.py
│   │   │   └── auth.py
│   │   └── models/
│   └── requirements.txt
│
├── frontend/anomai-dashboard/      # Laravel dashboard
│   ├── app/Http/Controllers/Admin/
│   │   └── DeviceController.php
│   └── resources/views/admin/
│
└── system metrics Model/
    └── system_metrics_agent/       # Python agent
        ├── agent.py                # Main agent loop
        ├── collect_training_data.py
        ├── retrain.py              # Full retraining pipeline
        ├── generate_report_now.py  # Force-generate PDF report
        ├── simulate_anomaly.py     # CPU stress test for demo
        └── requirements.txt
```

---

## Getting Started

### 1. Backend

```bash
cd backend
pip install -r requirements.txt
python -m uvicorn app.main:app --port 9000 --reload
```

### 2. Frontend

```bash
cd frontend/anomai-dashboard
composer install
cp .env.example .env
php artisan key:generate
php artisan serve --port=8001
```

### 3. Agent — first-time setup on a new machine

```bash
cd "system metrics Model/system_metrics_agent"
pip install -r requirements.txt

# Step 1: Collect normal behavior data (runs for ~4 hours)
python collect_training_data.py

# Step 2: Train the LSTM model on collected data
python retrain.py

# Step 3: Run the agent
python agent.py
```

The agent automatically registers the device with the backend on first run. No manual configuration needed.

### Force-generate today's report

```bash
python generate_report_now.py
```

### Simulate a CPU anomaly (for testing)

```bash
python simulate_anomaly.py
```

---

## Agent Behavior

| Event | Terminal | Dashboard |
|---|---|---|
| Normal reading | `[NORMAL] score=0.012` | — |
| Statistical deviation | `[DEVIATION] score=0.045` | — |
| Performance issue | `[ANOMALY] PERFORMANCE ISSUE: CPU 94.2%` | Alert shown |
| Device offline | — | Status → OFFLINE after 3 min |
| Backend unreachable | `[Anomaly Buffered]` | Synced when back online |

---

## Requirements

- Python 3.9+
- PHP 8.1+ with Composer
- TensorFlow 2.13+

---

## Author

**Razan Mohammad** — [GitHub](https://github.com/Razanmohamad25)
