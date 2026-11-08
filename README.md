# W-SENTRY

**Cross-dataset evaluation of wireless and IoT intrusion detectors**

[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![PyTorch](https://img.shields.io/badge/PyTorch-2.6.0-EE4C2C.svg)](https://pytorch.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> **Major Project (2026-27)** — KG Reddy College of Engineering & Technology, Hyderabad  
> *Team:* Riteesh Thiruveedhula, V. Sai Tarun Reddy, M. Vikas Goud, Shaik Gafar  
> *Guide:* Bapu Sreenivas

---

## Overview

W-SENTRY trains wireless and IoT intrusion detectors and measures how well they hold up on a dataset they were not trained on. It uses three public datasets (WSN-DS, CICIoT2023 and Edge-IIoTset), with their attack labels mapped to a shared five-class taxonomy. It includes:

1. **Two-track feature engineering**: native dataset-specific features for same-dataset tests, and protocol-agnostic behavioral features for cross-dataset tests
2. **Log normalization preprocessing** for billion-range protocol fields (tcp.ack_raw, tcp.payload)
3. **Hybrid CNN-BiLSTM-Attention model** for sequential network traffic
4. **Domain Adversarial Neural Network (DANN)** with gradient reversal, trained on pooled CICIoT2023 and Edge-IIoTset data
5. **Random Forest and XGBoost baselines**

The main finding: models that score 91.9% to 99.6% on their own dataset score far lower on the others.

### Key Results

| Result | Value |
|--------|-------|
| **Same-dataset accuracy, hybrid (native features)** | WSN-DS 98.06% · CICIoT2023 97.80% · Edge-IIoTset 94.39% |
| **Same-dataset accuracy, Random Forest** | WSN-DS 99.60% · CICIoT2023 99.42% · Edge-IIoTset 92.06% (macro-F1 only 65.9%) |
| **Cross-dataset accuracy, Random Forest** (full target test set) | 0.18% to 67.64%; all six scores are below a constant prediction of the target's largest class |
| **Cross-dataset accuracy, hybrid** (target restricted to the source's classes) | 0.30% to 80.70% |
| **DANN on pooled CICIoT2023 + Edge-IIoTset** | 96.5% / 96.3% on those two datasets' own test sets (not a transfer test) |

Every number is a single run, without variance estimates, read from `data/processed/phase1_results.json` and `phase2_results.json` or derived from their confusion matrices (macro-F1). Scoring protocols differ between models, as the table notes. The saved results came from earlier versions of the scripts than the ones in this repo, and there is no saved result for a 3-domain DANN.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  DATA SOURCES                                                        │
│  WSN-DS (374K) · CIC-IoT-2023 (1.05M) · Edge-IIoTset (~200K)         │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  FEATURE EXTRACTION                                                  │
│  ┌──────────────────┐  ┌────────────────────┐  ┌─────────────────┐ │
│  │ Native Features  │  │ Behavioral Features│  │ Log Normalization│ │
│  │ (14-48 cols)     │  │ (20 common cols)   │  │ + RobustScaler  │ │
│  └──────────────────┘  └────────────────────┘  └─────────────────┘ │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  DEEP LEARNING MODELS                                                │
│  ┌─────────────────────────────┐  ┌────────────────────────────────┐│
│  │ Hybrid CNN-BiLSTM-Attention │  │ DANN (Domain Adversarial)      ││
│  │ Conv1D(15→32→64) + BN       │  │ Feature Extractor (shared)     ││
│  │ BiLSTM(64→128, 2 layers)    │  │ Threat Classifier (5 classes)  ││
│  │ Self-Attention + Dense      │  │ Gradient Reversal Layer        ││
│  │ Output: 5 threat classes    │  │ Domain Discriminator (2 doms)  ││
│  └─────────────────────────────┘  └────────────────────────────────┘│
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  DETECTION & MITIGATION                                              │
│  Threat Classification · Firewall Rules · Dashboard (Streamlit)     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Installation

### Prerequisites
- Python 3.11+
- CUDA 12.1+ (for GPU training; CPU fallback supported)
- Git
- 8GB RAM (16GB recommended for full dataset processing)

### Setup

Setting up a fresh Windows laptop for the two-laptop demo? Double-click `setup_laptopA.cmd`, or follow [SETUP_NEW_LAPTOPS.md](SETUP_NEW_LAPTOPS.md).

```bash
git clone https://github.com/Riteesh-2k6/w-sentry.git
cd w-sentry

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Download datasets (requires Kaggle API key)
# Set KAGGLE_USERNAME and KAGGLE_KEY environment variables first
python scripts/download_datasets.py
```

---

## Usage

### Phase 1: Baseline ML Models (Random Forest / XGBoost)

```bash
python run_phase1.py
# Output: data/processed/phase1_results.json
# Metrics: per-dataset accuracy + 3×3 transfer matrix
```

### Phase 2: Deep Learning (Hybrid + DANN)

```bash
python run_phase2.py
# Output: data/processed/phase2_results.json
# Trains:
#   - Hybrid CNN-BiLSTM-Attention (native + behavioral features)
#   - DANN 3-domain (WSN+CIC+Edge; no saved result) and 2-domain (CIC+Edge)
#   - Temperature scaling calibration
```

### Phase 3: Dashboard + Live Demo

Full interactive SOC dashboard and two-laptop live attack/mitigation pipeline.
See `docs/DEMO_RUNBOOK.md` for the two-laptop setup and `docs/PHASE3_INTEGRATION.md`
for how Phase 3 was integrated and how to run everything.

```bash
# 0. Install Phase 3 deps
pip install -r requirements_phase3.txt

# 1. Train a real, deployable checkpoint (the bundled one is an untrained scaffold)
python scripts/train_model.py
# -> models/saved/{hybrid_model.pt,scaler.pkl,temp_scaler.pt} + held-out metrics

# 2. Show off the model (metrics + live detection → mitigation, single machine)
scripts/showcase_model.sh

# 3. Launch the Streamlit SOC dashboard (detector machine)
scripts/run_dashboard.sh          # http://localhost:8501
# Tabs: Performance, Live Monitor, Transfer Matrix, ROC/AUC, Per-Class, Mitigation

# 4. Launch simulated attacks (attacker machine, WSL / Scapy) — private LAN only
scripts/run_attack.sh syn-flood --target <DETECTOR_IP> --duration 30
scripts/run_attack.sh port-scan --target <DETECTOR_IP>
scripts/run_demo.sh  --target <DETECTOR_IP> --gateway <GATEWAY_IP>   # full sequence
```

---

## Project Structure

```
w-sentry/
├── data/
│   ├── raw/                      # Downloaded datasets (not in repo, use scripts/download_datasets.py)
│   │   ├── WSN-DS.csv            # 374K samples, 15 routing features
│   │   ├── IoT_Intrusion.csv     # CIC-IoT-2023, 1.05M samples, 48 flow features
│   │   └── ML-EdgeIIoT-dataset.csv  # ~200K stratified sample used, 35 protocol features
│   └── processed/                # Experiment results (JSON)
│       ├── phase1_results.json   # RF/XGBoost baselines
│       └── phase2_results.json   # Hybrid DL + DANN results
├── src/
│   ├── data/
│   │   └── dataset_loader.py     # Two-track loader (native vs behavioral features)
│   ├── features/
│   │   └── preprocessor.py       # Log normalization + RobustScaler + SMOTE + windowing
│   ├── models/
│   │   ├── baseline_ml.py        # RandomForest + XGBoost
│   │   ├── hybrid_dl.py          # CNN-BiLSTM-Attention
│   │   ├── dann.py               # Domain Adversarial Neural Network
│   │   └── calibration.py        # Temperature scaling + ECE
│   ├── dashboard/                # Streamlit SOC interface (Phase 3)
│   ├── live/                     # Scapy capture + real-time inference (Phase 3)
│   ├── attack/                   # hping3/nmap/ARP spoof scripts (Phase 3)
│   └── mitigation/               # Firewall automation (Phase 3)
├── tests/                        # Pytest suite
│   ├── test_phase1.py
│   └── test_phase2.py
├── paper/                        # IEEE conference paper draft
│   ├── w-sentry-paper-draft.md
│   └── figures/
│       ├── fig1_architecture.html  # Interactive SVG architecture diagram
│       ├── fig2_transfer_matrix.png
│       ├── fig3_accuracy_comparison.png
│       └── fig4_log_normalization_ablation.png
├── docs/
│   ├── PHASE3_PLAN.md            # Dashboard implementation plan (700 lines, 9 task blocks)
│   └── ...
├── run_phase1.py                 # Phase 1 experiment runner
├── run_phase2.py                 # Phase 2 experiment runner
├── requirements.txt
└── README.md
```

---

## Research Contribution

The paper, *Same-Dataset Accuracy Does Not Transfer: Cross-Dataset Evaluation of Intrusion Detectors on WSN-DS, CICIoT2023, and Edge-IIoTset*, reports:

1. Same-dataset results for RF, XGBoost, the hybrid model and DANN, and 3×3 transfer matrices for RF and the hybrid model.
2. Accuracy hides class imbalance: RF reaches 92.1% accuracy on Edge-IIoTset but only 65.9% macro-F1.
3. Every RF transfer accuracy is below the share of the largest class in the target test set.
4. DANN, trained on pooled data, scores 1.1 to 1.3 points below the behavioral-feature hybrid models trained on each dataset separately (97.79% on CICIoT2023, 97.42% on Edge-IIoTset). Whether its adversarial term learned domain-invariant features is untested.

It does not name a best model. Its conclusion is that same-dataset accuracy alone is not evidence of transfer, and a transfer claim needs matched scoring protocols, per-class metrics and repeated runs.

**Target venues:**
- IEEE CONECCT 2027 (International Conference on Electronics, Computing and Communication Technologies)
- IEEE ICCCNT 2027 (International Conference on Computing, Communication and Networking Technologies)
- IEEE Access (journal, open access)

**Paper status:** 6-page IEEE conference paper in preparation (`paper/w-sentry-paper-ieee.tex`)

---

## Datasets

| Dataset | Samples | Features | Attack Types | Domain |
|---------|---------|----------|--------------|--------|
| [WSN-DS](https://www.kaggle.com/datasets/ghulamali/wsn-ds) | 374K | 14 routing | Grayhole, Blackhole, Flooding, TDMA, Scheduling | Wireless Sensor Networks |
| [CIC-IoT-2023](https://www.kaggle.com/datasets/madhavmalhotra/iot-intrusion-dataset) | 1.05M | 48 flow | DDoS (HTTP/TCP/UDP), Recon, MITM, Spoofing, Backdoor | IoT Network Traffic |
| [Edge-IIoTset](https://www.kaggle.com/datasets/mohamedamineferrag/edgeiiotset-cyber-security-dataset-of-iot-iiot) | ~200K sampled | 35 protocol | DDoS, SQL injection, XSS, Ransomware, Port scan, Backdoor | Industrial Edge Networks |

**License:** All datasets are publicly available under their respective licenses (citation required).

**Download script:** `scripts/download_datasets.py` (requires Kaggle API credentials)

---

## Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **CPU** | Intel i5 / Ryzen 5 | Intel i7 / Ryzen 7 |
| **RAM** | 8GB | 16GB |
| **GPU** | None (CPU training works) | NVIDIA RTX 3050+ (6GB VRAM) |
| **Storage** | 10GB | 20GB (datasets + models) |
| **OS** | Windows 10/11, Ubuntu 20.04+ | Windows 11 / Ubuntu 22.04 |

**Tested on:**
- Windows 11 with RTX 3050 (6GB), CUDA 13.0, PyTorch 2.6.0+cu128
- Training time: roughly 3 min/epoch (GPU) vs 20 min/epoch (CPU) for 200K sequences (rough observation, not logged)

---

## Citation

The paper is not published yet. Until it is, please cite the repository:

```bibtex
@misc{thiruveedhula2026wsentry,
  title={Same-Dataset Accuracy Does Not Transfer: Cross-Dataset Evaluation of Intrusion Detectors on WSN-DS, CICIoT2023, and Edge-IIoTset},
  author={Thiruveedhula, Riteesh and Reddy, Velagala Sai Tarun and Goud, Mogili Vikas and Gafar, Shaik},
  year={2026},
  howpublished={\url{https://github.com/Riteesh-2k6/w-sentry}},
  note={Paper in preparation}
}
```

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## Acknowledgments

- **Base Papers:**
  - Phalaagae et al., "A Hybrid CNN-LSTM Model With Attention Mechanism for Improved Intrusion Detection in Wireless IoT Sensor Networks," *IEEE Access*, 2025
  - Ganin et al., "Domain-Adversarial Training of Neural Networks," *JMLR*, 2016
- **Datasets:** WSN-DS (Almomani et al.), CIC-IoT-2023 (Canadian Institute for Cybersecurity), Edge-IIoTset (Ferrag et al.)
- **Institution:** KG Reddy College of Engineering & Technology, Hyderabad

---

## Contact

- **Riteesh Thiruveedhula** — [triteesh@outlook.com](mailto:triteesh@outlook.com) | [LinkedIn](https://linkedin.com/in/riteesh2k6) | [GitHub](https://github.com/Riteesh-2k6)
- **Project Repository** — [https://github.com/Riteesh-2k6/w-sentry](https://github.com/Riteesh-2k6/w-sentry)

---

**Status:** ✅ Phase 1 Complete · ✅ Phase 2 Complete · ✅ Phase 3 (Dashboard + Live Demo) Integrated · 📝 Phase 4 (Paper) in preparation
