# GK-03 Synthetic Fraud-Screen Machine Learning Replication
## Round 4 Final Technical Report & System Reconstruction

---

### Executive Summary
This report presents the final engineering architecture, empirical evaluation, and surrogate machine learning reconstruction for the *GK-03 Blackbox Fraud-Screening System*. The primary objective was to reverse-engineer and replicate the decision mechanics and scoring outputs of the GK-03 engine across its strict 10-dimensional input feature space while preserving zero-fabrication integrity, strict schema validation, and high fidelity fraud discrimination.

---

## 1. Project Objective

The GK-03 system is a synthetic AI engine that evaluates transactional requests and simultaneously outputs:
1. *score*: A continuous non-fraud approval probability in the bounded range $[0.0, 1.0]$ ($P(\text{isFraud}=0)$).
2. *decision*: A discrete binary classification label:
   $$\text{decision} = \begin{cases} \text{APPROVE}, & \text{if score} \ge 0.50 \\ \text{DECLINE}, & \text{if score} < 0.50 \end{cases}$$

The objective of Round 4 was to construct a robust, production-grade surrogate model pipeline capable of:
- Ingesting strictly validated 10-parameter transaction payloads.
- Handling severe financial class imbalance (~0.13% fraud prevalence).
- Achieving superior discrimination (ROC-AUC > 99.5%) and high fraud recall (> 80%).
- Providing end-to-end standalone reproducibility across CLI, automated test suites, interactive Web UI, and Google Colab environments.

---

## 2. The 10 Mandatory Input Parameters

The GK-03 specification mandates exactly 10 input features per screening query. All features are strictly validated against defined data types and boundary intervals:

| # | Feature Name | Data Type | Valid Range / Specification | Domain Interpretation |
|---|:---|:---|:---|:---|
| 1 | account_age_days | Float / Int | [18.0, 75.0] | Age of the originating account in days |
| 2 | account_balance | Float | [0.0, 100.0] | Normalized starting account balance |
| 3 | amount | Float | [0.0, 100.0] | Normalized transaction amount requested |
| 4 | beneficiaries | Int | [0, 6] | Number of pre-registered recipient accounts |
| 5 | channel | Categorical String | Configurable ("D", "P", "T", "C", "I") | Transaction payment channel / modality |
| 6 | linked_cards | Int | [0, 20] | Count of payment cards attached to the account |
| 7 | months_active | Float / Int | [0.0, 40.0] | Cumulative active account tenure in months |
| 8 | recent_chargebacks | Int | [0, 5] | Count of historical dispute / chargeback events |
| 9 | trust_score | Float / Int | [300.0, 900.0] | Customer credit / historical trust rating |
| 10 | utilisation | Float | [0.0, 1.0] | Ratio of transaction depletion to available funds |

---

## 3. Dataset Used & Working Split

### Auxiliary Benchmark Dataset (PaySim)
- *Source Dataset*: PaySim Synthetic Financial Dataset (~6,362,620 raw records, 11 columns, 8,213 fraudulent events).
- *Class Imbalance in Raw Data*: 0.1290% fraudulent transactions (~774:1 imbalance ratio).
- *Auxiliary Isolation Principle*: PaySim data was utilized as an auxiliary empirical foundation to simulate real-world transaction distributions, ratio interactions, and class imbalance without fabricating synthetic labels.

### Stratified Working Sample
To enable deterministic, reproducible training and validation:
- *Sample Size*: 200,000 transactions (data/processed/paysim_sample.csv).
- *Fraud Instances*: 260 fraud events (exact 0.1300% prevalence preserved via stratified random sampling with seed 42).
- *Train/Test Partition*: Stratified 80/20 train-test split:
  - *Training Set*: 160,000 transactions (208 fraud cases).
  - *Held-out Test Set*: 40,000 transactions (52 fraud cases, 39,948 legitimate cases).

---

## 4. Feature Engineering

A deterministic, leakage-free transformation pipeline was implemented in src/feature_engineering.py to map transaction variables into the 10 GK-03 bounded features:

1. *amount*: Log-scaled robust compression to handle extreme skewness:
   $$\text{amount}_{\text{GK03}} = \text{clip}\left(\frac{\ln(1 + \text{raw\_amount})}{\ln(1 + 10^6)} \times 100.0, \, 0.0, \, 100.0\right)$$
2. *account_balance*: Log-scaled normalization of originator's starting balance (oldbalanceOrg):
   $$\text{balance}_{\text{GK03}} = \text{clip}\left(\frac{\ln(1 + \text{oldbalanceOrg})}{\ln(1 + 10^7)} \times 100.0, \, 0.0, \, 100.0\right)$$
3. *utilisation*: Depletion ratio capturing complete fund extraction risk:
   $$\text{utilisation} = \text{clip}\left(\frac{\text{raw\_amount}}{\max(\text{oldbalanceOrg}, \, \text{raw\_amount}, \, 10^{-3})}, \, 0.0, \, 1.0\right)$$
4. *channel*: Categorical mapping of transaction types (DEBIT $\to$ "D", PAYMENT $\to$ "P", TRANSFER $\to$ "T", CASH_OUT $\to$ "C", CASH_IN $\to$ "I").
5. *account_age_days*: Time evolution mapping across simulation steps:
   $$\text{account\_age\_days} = \text{clip}\left(18.0 + \frac{\text{step}}{744.0} \times (75.0 - 18.0), \, 18.0, \, 75.0\right)$$
6. *months_active*: Cumulative activity mapping:
   $$\text{months\_active} = \text{clip}\left(\frac{\text{step}}{744.0} \times 40.0, \, 0.0, \, 40.0\right)$$
7. *beneficiaries*: Deterministic recipient hash: $\sum(\text{ASCII}(\text{nameDest})) \pmod 7 \in [0, 6]$.
8. *linked_cards*: Deterministic originator card hash: $\sum(\text{ASCII}(\text{nameOrig})) \pmod{21} \in [0, 20]$.
9. *recent_chargebacks*: Structural risk proxy combining destination zero-balance flags and account depletion:
   $$\text{chargebacks} = \text{clip}(2 \cdot \mathbb{I}{\text{zero\_dest}} + 2 \cdot \mathbb{I}{\text{depleted}} + \mathbb{I}_{\text{amount} > 200k}, \, 0, \, 5)$$
10. *trust_score*: Composite trust metric:
    $$\text{trust\score} = \text{clip}(600 + 1.5 \cdot \text{balance} - 200 \cdot \text{utilisation} + 50 \cdot \mathbb{I}{\text{payment}} - 40 \cdot \text{chargebacks}, \, 300.0, \, 900.0)$$

---

## 5. Model Architecture & Selection

Two competitive machine learning architectures were trained and benchmarked using a unified ColumnTransformer (OneHotEncoder with handle_unknown='ignore' for channel and passthrough for numeric features):

1. *Random Forest Classifier (Selected)*:
   - Estimators: 100 trees
   - Max Depth: 12
   - Class Weighting: balanced (automatically adjusts weights inversely proportional to class frequencies)
   - Parallelism: n_jobs=-1, random_state=42
2. *Histogram-based Gradient Boosting Classifier*:
   - Max Iterations: 100
   - Class Weighting: balanced
   - random_state=42

### Model Selection Result
The *Random Forest Classifier* achieved higher ranking discrimination (ROC-AUC: 0.99792 vs 0.99741) and superior precision-recall trade-off, making it the primary serialized surrogate model (models/gk03_model.joblib).

---

## 6. Training Approach

- *Imbalance Mitigation*: Rather than generating artificial synthetic observations (e.g. SMOTE) which can distort delicate decision boundaries, cost-sensitive balanced class weighting was utilized:
  $$w_j = \frac{N}{2 \cdot n_j}$$
- *Data Preprocessing*: Scikit-Learn ColumnTransformer pipeline fitted exclusively on the training partition to prevent data snooping and serialized independently as models/gk03_preprocessor.joblib.
- *Reproducibility*: Global random seed 42 enforced across sampling, train/test splitting, and model initialization.

---

## 7. Testing & Quality Assurance

The system was verified with a comprehensive suite of *41 automated pytest unit and integration tests* spanning 5 test suites:

- tests/test_data_loader.py (25 tests): Verified schema enforcement, mandatory column presence, null value rejections, out-of-bound parameter rejections across all 10 features, and target range validation.
- tests/test_model.py (5 tests): Verified artifact loading, score bounds ($[0.0, 1.0]$), decision rule mapping, extreme min/max boundary validity, and out-of-range exception raising.
- tests/test_paysim_tools.py (3 tests): Verified diagnostic reporting, stratified sampling integrity, and missing file exception handling.
- tests/test_preprocessing.py (4 tests): Verified transformer matrix dimensions, unseen categorical channel handling, and label encoding/decoding.
- tests/test_ui_and_e2e.py (4 tests): Verified end-to-end inference across low-risk, high-risk, medium-risk, and edge boundary scenarios.

*Test Execution Result*: 41 passed in 2.39 seconds (100% pass rate).

---

## 8. Empirical Performance Metrics

Evaluated on the *40,000 held-out test transactions* (52 Ground Truth Frauds, 39,948 Legitimate Transactions):

| Metric | Score | Performance Interpretation |
|:---|:---|:---|
| *Accuracy* | *99.733%* (0.99733) | Correct classification on 39,893 out of 40,000 cases |
| *ROC-AUC* | *99.792%* (0.99792) | Exceptional ranking discrimination between risk classes |
| *Fraud Recall* | *80.769%* (0.80769) | Caught 42 out of 52 fraudulent transactions |
| *Fraud Precision* | *30.216%* (0.30216) | 42 True Positives against 97 False Positives |
| *Fraud F1-Score* | *0.43979* | Balanced harmonic mean under 0.13% extreme imbalance |
| *Legitimate Precision* | *99.975%* (0.99975) | Highly reliable approval precision |
| *Legitimate Recall* | *99.757%* (0.99757) | 39,851 legitimate transactions approved |

### Detailed Confusion Matrix
$$\begin{pmatrix} \text{TN} = 39,851 & \text{FP} = 97 \\ \text{FN} = 10 & \text{TP} = 42 \end{pmatrix}$$

- *True Negatives (TN = 39,851)*: Legitimate transactions correctly approved.
- *False Positives (FP = 97)*: Legitimate transactions flagged for review (false alarms).
- *False Negatives (FN = 10)*: Frauds missed (approval error rate < 0.025% of total volume).
- *True Positives (TP = 42)*: Frauds successfully intercepted.

---

## 9. Limitations & Operating Constraints

1. *Auxiliary Domain Gap*: PaySim was generated from a 2017 mobile money financial simulation. While transaction ratios and depletion dynamics generalize well, proprietary metadata (e.g. external credit bureau scores) required synthetic proxy mapping.
2. *Precision-Recall Tradeoff*: Operating with balanced class weighting prioritizes recall (80.77%) over precision (30.22%), yielding ~97 false positives per 40,000 transactions.
3. *Decision Threshold Rigidity*: The current architecture uses a static threshold $\tau = 0.50$ on the approval score. For low-friction retail environments, adjusting $\tau$ to $0.40$ or $0.60$ may be required.
4. *Channel Generalization*: Unseen transaction channels are handled gracefully via one-hot zero vectors without throwing exceptions, but receive default channel weighting.

---

## 10. Final Reconstruction Approach

The final system was delivered through a modular, decoupled architecture:
1. *Core Package (src/)*:
   - config.py: Centralized boundary constants, feature lists, directory paths.
   - data_loader.py: Strict schema and range validation layer.
   - feature_engineering.py: Deterministic feature mapping pipeline.
   - preprocessing.py: Scikit-learn preprocessor pipeline.
   - train.py: Training, model comparison, and artifact serialization.
   - evaluate_model.py: Comprehensive test set evaluation suite.
   - predict.py: Thread-safe, cached singleton inference engine for live scoring.
2. *Interactive Streamlit Web Dashboard (app.py)*:
   - Live 10-parameter sliders with instant reactive scoring.
   - 4 one-click risk preset profiles (Low Risk, High Risk, Borderline, Max Outlier).
   - Dynamic decision gauges and interactive audit log history.
3. *Google Colab & Standalone Distribution Package*:
   - Interactive notebook GK03_Demo.ipynb featuring zero-setup execution.
   - Standalone ZIP distribution (GK03_Colab_Submission.zip) containing pre-trained models, sample data, and source modules.
