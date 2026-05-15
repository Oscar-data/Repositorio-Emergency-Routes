# 🚑 Emergency Routes

**Real-Time Route Optimization for Emergency Medical Services**

> An integrated AI-based decision-support system for optimizing emergency medical services in the province of Valencia, developed in collaboration with the Generalitat Valenciana.

[![Python](https://img.shields.io/badge/Python-3.9+-blue.svg)](https://python.org)
[![Whisper](https://img.shields.io/badge/Whisper-small-green.svg)](https://github.com/openai/whisper)
[![mBERT](https://img.shields.io/badge/mBERT-multilingual--cased-orange.svg)](https://huggingface.co/bert-base-multilingual-cased)
[![License](https://img.shields.io/badge/License-Academic-lightgrey.svg)](#license)

---

## Table of Contents

- [Abstract](#abstract)
- [Motivation](#motivation)
- [System Architecture](#system-architecture)
- [Modules](#modules)
  - [Route Optimization](#1-route-optimization-with-closures)
  - [Audio Triage](#2-audio-triage-system)
- [Data](#data)
- [Results](#results)
- [Deployment](#deployment)
- [Setup & Reproduction](#setup--reproduction)
- [Limitations](#limitations)
- [Team & Contributions](#team--contributions)
- [References](#references)

---

## Abstract

Conventional navigation systems do not account for real-time road closures, extreme weather conditions, or large-scale urban events. In emergency situations, this forces ambulances to recalculate routes reactively, losing critical minutes.

**Emergency Routes** addresses this through two complementary modules:

1. **Route Optimization Engine** — Incorporates road restrictions as exclusion polygons in OpenRouteService, with 7 predefined scenarios (Fallas, Marathon, DANA, PATRICOVA, among others) and support for multi-casualty incidents under the START/SHORT doctrine and the MANV dispatch model.

2. **Automated Triage System** — Transcribes emergency calls using Whisper and classifies their severity with a fine-tuned mBERT model (`bert-base-multilingual-cased`), reaching a **macro F1 of 85.4%** and **perfect recall in the Severe category**.

## Motivation

On October 29, 2024, the Valencian Community was hit by one of the most devastating floods in recent European history (DANA). The catastrophe caused massive road closures and communication blackouts. At 17:00, a peak of **2,438 simultaneous calls** was recorded at 112, with **19,821 total calls** and **4,770 incidents** that day — overwhelming normal operational capacity.

This project was born from a simple question: *how many minutes could be saved if the CICU always had the fastest route, the most critical patient identified, and the most precise estimated time of arrival?*

## System Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Citizen Portal  │────▶│  Firebase Cloud   │◀────│ Dispatch Portal  │
│  (Netlify)       │     │  (Firestore DB)   │     │ (Netlify)        │
└────────┬────────┘     └──────────────────┘     └────────┬─────────┘
         │                                                 │
         ▼                                                 ▼
┌─────────────────┐                              ┌──────────────────┐
│  Hugging Face    │                              │ OpenRouteService │
│  Spaces (AI)     │                              │ (Routing Engine) │
│  - Whisper       │                              │ - avoid_polygons │
│  - mBERT triage  │                              │ - Dijkstra / A*  │
└─────────────────┘                              └──────────────────┘
```

- **Frontend**: Static HTML portals hosted on Netlify (citizen + dispatch).
- **Real-time Communication**: Firebase Firestore with `onSnapshot` listeners.
- **AI Pipeline**: Whisper (small) + fine-tuned mBERT on Hugging Face Spaces (Docker).
- **Routing**: OpenRouteService API with dynamic closure polygons.

## Modules

### 1. Route Optimization with Closures

The routing module determines, in real time and under road closure scenarios, which ambulance should be dispatched and which route to follow to minimize **Total Operational Latency** (ambulance → incident + incident → hospital).

**Key features:**
- 7 preloaded restriction scenarios (Fallas, Marathon, DANA, PATRICOVA, Semana Marinera, San Vicente, Ofrenda)
- Custom CSV upload for ad-hoc closures (accidents, demonstrations, roadworks)
- Filters ambulances by operational availability (shift + status)
- Haversine pre-filter (top 8 closest) → ORS travel-time ranking
- Hospital selection filtered by incident severity
- **Multi-Casualty Incident (MCI) mode**: START/SHORT triage + MANV dispatch doctrine
  - Automatic SAMU/SVB role assignment
  - 70% fleet cap to preserve provincial coverage
  - Hospital dispersion when ≥ 5 ambulances dispatched

### 2. Audio Triage System

Processes emergency calls end-to-end: audio → transcription → severity classification.

| Component | Model | Details |
|-----------|-------|---------|
| Speech-to-Text | Whisper `small` (244M params) | Supports Spanish + Valencian, no GPU required |
| Severity Classification | mBERT (`bert-base-multilingual-cased`) | Fine-tuned on 504 synthetic calls, 3 classes |
| Medical Context | Wikipedia API | Automatic clinical term lookup, no API key needed |

**Why mBERT over alternatives?**
- vs. BETO: native Valencian support without translation
- vs. XLM-RoBERTa: equivalent performance (macro F1 0.853) at lower computational cost
- WordPiece tokenization handles uncommon medical terminology well

## Data

| Source | Origin | Status |
|--------|--------|--------|
| Hospital directory | Generalitat Valenciana | 🔒 Confidential |
| Ambulance inventory (SAMU/SVB) | Generalitat Valenciana | 🔒 Confidential |
| Road restriction layers (7 scenarios) | Valencia City Council / GVA / OSM | 📦 [Download from Releases](../../releases) |
| Emergency call dataset (504 calls) | Synthetic | ✅ Included |
| Traffic intensity 2023–2025 | Generalitat Valenciana | ✅ Included |
| River flows and reservoirs | Júcar Hydrographic Confederation | ✅ Included |
| Weather data (AEMET) | AEMET | ⏳ Future development |

> **Note**: Hospital and ambulance data are confidential and provided exclusively for academic use. See [`Data/README.md`](Data/README.md) for details on restriction layer construction.

## Results

### Triage Model Performance (test set, n=76)

| Metric | Value |
|--------|-------|
| Accuracy | 85.5% |
| Macro Precision | 87.4% |
| Macro Recall | 85.3% |
| Macro F1 | **85.4%** |

**Per-class highlights:**
- **Moderate**: Perfect recall (1.00) — no moderate emergency goes undetected.
- **Mild**: Perfect precision (1.00) — when predicted mild, always correct. Conservative errors (mild → severe) are clinically preferable.
- **Severe**: Recall of 0.92, with 2 errors classified as moderate (never as mild). **No severe case is ever classified as mild** — the most dangerous error is fully prevented.

### Route Optimization (Fallas scenario demo)

| Metric | No closures | With Fallas (781 closures) |
|--------|-------------|---------------------------|
| Ambulance → Incident | 6.9 min | 6.8 min |
| Incident → Hospital | 3.9 min | 6.5 min |
| **Total Operational Latency** | **11.0 min** | **13.0 min** |

The system proactively routes around all 781 closure polygons. Without it, an ambulance would discover blocked streets only upon arrival, losing critical minutes.

## Deployment

The system is fully cloud-based and accessible from any device without local installation:

| Component | Service | URL |
|-----------|---------|-----|
| Citizen Portal | Netlify | Hosted static HTML |
| Dispatch Portal | Netlify | Hosted static HTML |
| Real-time DB | Firebase Firestore | `onSnapshot` push updates |
| AI Pipeline | Hugging Face Spaces | Docker container with Whisper + mBERT |

## Setup & Reproduction

### Prerequisites

- Python 3.9+
- `ffmpeg` installed on your system
- API keys for OpenRouteService and Firebase (see [`.env.example`](.env.example))

### Installation

```bash
# Install dependencies (for the triage notebook)
pip install openai-whisper transformers torch requests

# Install ffmpeg (Ubuntu/Debian)
sudo apt install ffmpeg
```

### Running the Triage Model

Open `Code/audio_clasificacion_emergencias.ipynb` in Jupyter or Google Colab (GPU recommended for faster transcription).

### Running the Dispatch Portal Locally

Open `Code/mockup/emergencias_global.html` in a browser. You will need to enter your ORS API key in the interface. Firebase must be configured with your own project credentials.

## Project Structure

```
emergency-routes/
├── README.md
├── .gitignore
├── .env.example
│
├── docs/
│   └── G1_PROYIII_LaTex.pdf          # Full project report
│
├── Code/
│   ├── audio_clasificacion_emergencias.ipynb   # Whisper + mBERT triage pipeline
│   ├── embalses_caudales.ipynb                 # Reservoir/river flow analysis
│   ├── datos_hidrograficos_chj.xlsx            # CHJ hydrographic data
│   ├── llamadas_emergencia.xlsx                # Synthetic emergency call dataset
│   └── mockup/
│       ├── ciudadano_global.html               # Citizen portal (alert submission)
│       └── emergencias_global.html             # Dispatch portal (route optimization)
│
└── Data/
    ├── README.md                               # Data documentation
    └── intensidad_total_23-25.csv              # Traffic intensity (future development)
```

> ⚠️ **Road restriction scenario CSVs** (7 files, ~53 MB) are not included directly in the repository due to GitHub's file size limits. Download them from the [**Releases**](../../releases) section of this repo.

## Limitations

- **ORS API load limits**: The public API cannot process the full DANA exclusion cartography at once. A self-hosted ORS instance would be needed for regional-scale disasters.
- **Synthetic training data**: The triage model was trained on 504 synthetic calls because real 112 recordings are confidential. Performance on real calls may differ.
- **Hugging Face free tier**: The AI container sleeps after inactivity; first request after sleep has additional latency.
- **Flood prediction model deferred**: Only one DANA event exists in the historical record, making it impossible to build a generalizable flood prediction model.

## SDG Alignment

- 🏥 **SDG 3** — Good Health and Well-Being: reducing emergency response latency.
- 🏙️ **SDG 11** — Sustainable Cities and Communities: urban resilience to disasters.
- 🌍 **SDG 13** — Climate Action: preparing emergency systems for extreme weather.

## Team & Contributions

| Name | Contributions |
|------|---------------|
| **Jorge Acín Zurita** | Data collection, audio triage pipeline (Whisper transcription + mBERT classification), model fine-tuning, hyperparameter search, Wikipedia medical context module, SDG alignment analysis |
| **Pablo Arnau Tomás** | Data collection and cleaning (hospitals, ambulances, traffic intensity, AEMET, CHJ hydrographic data), exploratory analysis, synthetic emergency call dataset generation, project report and documentation (LaTeX), results evaluation and visualization |
| **Nacho Castilla Llorca** | Cloud deployment (Netlify, Firebase Firestore, Hugging Face Spaces), real-time communication between portals, citizen and dispatch portal frontend development, route optimization engine, ORS integration, dispatch algorithm (Haversine pre-filter, ambulance selection) |
| **Pablo Huguet Chordi** | Data collection, project report and documentation (LaTeX), results evaluation and visualization, system testing and scenario simulations, SDG alignment analysis |
| **David Llácer Córdoba** | Data collection, geospatial data processing (QGIS), construction of road restriction layers (Fallas, Marathon, DANA, PATRICOVA, Semana Marinera, San Vicente, Ofrenda), coordinate reprojection (UTM ↔ WGS84) |
| **Óscar Muñoz García** | Data collection, audio triage pipeline (Whisper transcription + mBERT classification), model fine-tuning, hyperparameter search, Wikipedia medical context module, SDG alignment analysis, synthetic emergency call dataset generation |

> **Note**: All team members contributed to the overall project design, discussion of results, and iterative improvement across modules.

**Institution**: Polytechnic University of Valencia (UPV) — Bachelor's Degree in Data Science, Project III

**Collaborators**: Yulia Karpova and M.ª Fulgencia Villa

## License

This project was developed for academic purposes as part of the Data Science degree at UPV. The source code is released for educational and research use. Confidential data provided by the Generalitat Valenciana is not included and must not be redistributed.

## References

See the full reference list in [`docs/FinalReport.pdf`](docs/FinalReport.pdf), Section 8.
