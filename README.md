# ระบบตรวจวัดความเครียดโดยใช้เทคโนโลยีปัญญาประดิษฐ์ร่วมกับสัญญาณทางสรีระวิทยา
# An AI-driven Stress Detection System using Physiological Signals

Senior project, B.Eng. Computer Engineering and Artificial Intelligence, Walailak University (academic year 2568, Oct 2024 – Aug 2025).
Silver Medal at I-New Gen Award 2025 and Thailand Research Expo 2025. Human research ethics approval WUEC-24-136-02.

A wearable system that classifies stress into 4 levels (normal, low, moderate, high) in real time from electrodermal activity (EDA) and heart rate variability (HRV), and alerts a caregiver through a web dashboard.

## Problem
About 30% of Thai people experience stress, but only around 15% are diagnosed and cared for, partly because screening still relies on self-report questionnaires. Prior studies on physiological signals mostly report correlations or statistical significance rather than an automatic classifier, and often use bulky or chest-worn devices.

## Data
We built the dataset ourselves.

- **Device:** EmotiBit (EDA at 15 Hz, PPG at 25 Hz) with an ESP32 streaming over Wi-Fi.
- **Participants:** 25 volunteers aged 18–26, balanced by BMI group, in a controlled room (23–25 °C).
- **Protocol (30 min):** setup, adaptation, a stress-inducing task block (Guessing Game, Photo Hunt, Sudoku with short breaks), then recovery.
- **Labels:** 4-level self-reported stress collected during breaks.
- **Size:** about 5.18M raw samples, reduced to 28,285 rows × 115 columns after processing.

Preprocessing pipeline:
1. Signal cleaning, peak detection and feature extraction with `neurokit2` (`eda_process`, `ppg_process`).
2. Outlier removal (|z| > 3) and resampling to 1 Hz.
3. Label encoding and `StandardScaler`.
4. 30-second sliding windows, shape `(N, 30, features)`.
5. Train/test split (70/30, stratified) **by participant ID before windowing** to avoid cross-subject leakage.
6. SMOTE on the training set only, with `k_neighbors` adjusted for very small classes.

Features: EDA (phasic, tonic, SCR amplitude, SCR onsets) and HRV (RMSSD, SDNN, LF, HF, LF/HF), plus gender, BMI, sleep and skin type.

## Model and Training
- Compared **RNN, LSTM and GRU** on three feature sets: EDA only, HRV only, EDA+HRV.
- Hyperparameter search with Keras Tuner (Bayesian Optimization, 30–50 trials, 5-fold CV, early stopping). Search space: 32–256 units, 1–5 layers, dropout 0.05–0.2, Adam/SGD, learning rate 1e-4 to 1e-2.
- Selected model: **GRU on EDA+HRV**, 3 layers × 192 units, dropout 0.15, Adam, batch size 64, 37 epochs.
- Trained on laptop GPUs (RTX 3050 Ti / RTX 3060).

## Challenges and Solutions
| Challenge | What we did |
|---|---|
| Sensors at different sampling rates, unlabeled raw streams | Unified preprocessing and resampling to 1 Hz, aligned with questionnaire labels |
| Class imbalance (few "normal" and "high" labels) | SMOTE on training data only, with dynamic `k_neighbors` |
| Leakage between windows of the same person | Split by participant before creating sequences |
| Unstable learning curves for RNN/LSTM | Compared architectures; GRU trained most stably |

## Evaluation
Test set results (Precision / Recall / F1 / Accuracy, %):

| Model | EDA | HRV | EDA+HRV |
|---|---|---|---|
| RNN | 93.44 / 93.11 / 93.15 / 93.14 | 99.50 / 99.50 / 99.50 / 99.50 | 99.40 / 99.40 / 99.40 / 99.40 |
| LSTM | 95.00 / 95.00 / 95.01 / 94.98 | 93.00 / 93.00 / 92.59 / 92.55 | 97.00 / 97.00 / 97.27 / 97.24 |
| GRU | 89.49 / 96.34 / 92.91 / 92.89 | 95.73 / 98.54 / 97.14 / 97.15 | **99.42 / 99.42 / 99.42 / 99.42** |

F1 was the main selection metric. We also ran repeated hold-out tests (80:20 to 50:50, 10 runs each) to check stability.

**Caveat:** with only 25 participants in a lab setting, scores near 99% likely overestimate real-world performance, and we suspect some overfitting on HRV features. A larger, more diverse cohort and field testing are needed before trusting these numbers.

## System and Deployment
EmotiBit → ESP32 (Wi-Fi, JSON) → central server → preprocessing (same as training) → GRU model → Next.js web app.

- Data stored in MongoDB (device status, personal data, sensor readings, preprocessed data, predictions).
- Backend built with FastAPI and Node.js.
- Dashboard with real-time EDA/HR graphs, stress alerts and PDF reports; a small TFT screen on the device shows HR, skin temperature and battery.

## Limitations and Future Work
- Small sample (25 people, aged 18–26), lab-only data and self-reported labels.
- Field testing in schools, hospitals and workplaces.
- Adding AI-generated suggestions for stress-relief activities.

## Team
- กฤตพร รักนุ้ย
- สุรพัศ แซ่ตั้น (Surapat Saetan)
- สุรพัศ หกหนู

Advisor: ผศ. ดร.มัลลิกา เกลี้ยงเคล้า
