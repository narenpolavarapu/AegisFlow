# AegisFlow 🛡️

## AI-Powered Network Intrusion Detection & Attack Classification

AegisFlow is a machine-learning cybersecurity project that uses network-flow data to detect malicious traffic and classify different attack types.

The project uses the **CIC-IDS2017** cybersecurity dataset and explores the complete workflow of a practical security ML system:

**Data Inspection → Data Cleaning → Feature Preparation → Model Training → Evaluation → Error Analysis → Leakage-Aware Validation → Security Interpretation**

---

## 🎯 Project Goal

The goal of AegisFlow is not simply to achieve high accuracy.

The project focuses on understanding how machine-learning models behave in a cybersecurity environment, where:

* False negatives can represent missed attacks.
* False positives can create unnecessary security alerts.
* Class imbalance can make accuracy misleading.
* Duplicate data can cause overly optimistic evaluation.
* Different attack types can behave very differently.
* Model performance must be interpreted in a security context.

---

## 📊 Dataset

### CIC-IDS2017

Dataset: **Canadian Institute for Cybersecurity — CIC-IDS2017**

The original dataset contains network traffic representing benign activity and multiple attack categories.

For this project, all eight CSV files from the MachineLearningCSV collection were inspected.

The controlled modeling experiment focused on:

**Tuesday-WorkingHours.pcap_ISCX.csv**

This file contains:

* BENIGN
* FTP-Patator
* SSH-Patator

### Controlled dataset used

| Class       |    Samples |
| ----------- | ---------: |
| BENIGN      |     45,000 |
| FTP-Patator |      7,938 |
| SSH-Patator |      5,897 |
| **Total**   | **58,835** |

The modeling dataset contains **78 numeric network-flow features**.

> This is intentionally a controlled subset of CIC-IDS2017, not the complete dataset.

---

## 🔎 Data Inspection

Before training models, the dataset was investigated rather than blindly loaded into memory.

The project examined:

* CSV structure
* Number of columns
* Data types
* Label distributions
* Missing values
* Infinite values
* Duplicate rows
* Constant features
* Feature ranges
* Extreme values
* Attack distribution
* Ordering effects within the raw CSV files

One important discovery was that taking the first 10,000 rows of the Tuesday dataset produced only BENIGN traffic.

This demonstrated that:

> `nrows=10_000` is not necessarily a representative sample when the original dataset is ordered.

Chunk-based processing was therefore used to inspect large files without loading the entire dataset into memory.

---

## 🧹 Data Cleaning

The dataset contained missing and infinite values.

The cleaning workflow included:

1. Detecting infinite values.
2. Converting `+inf` and `-inf` to `NaN`.
3. Calculating replacement values using the training data.
4. Filling missing values using training-set medians.
5. Applying the same learned medians to the test data.

This prevents information from the test set from influencing preprocessing.

The project also identified duplicate feature vectors and investigated their impact on evaluation.

---

## 🤖 Models

Several machine-learning algorithms were evaluated during the project.

### Binary classification

The first stage classified traffic as:

* BENIGN
* ATTACK

Models investigated:

* Logistic Regression
* Decision Tree
* Random Forest
* KNN
* Support Vector Machine

The binary stage also explored:

* Confusion matrices
* False positives
* False negatives
* Precision
* Recall
* F1-score
* Probability thresholds
* ROC-AUC
* PR-AUC
* Feature coefficients
* False-negative analysis

---

# 🌐 Multiclass Classification

The project then moved beyond simple attack detection.

Instead of asking:

> "Is this traffic malicious?"

the model was asked:

> "What type of traffic is this?"

The three classes were:

```text
BENIGN
FTP-Patator
SSH-Patator
```

---

## Logistic Regression — Multiclass

On the standard stratified train/test split:

| Metric          | Result |
| --------------- | -----: |
| Accuracy        | 98.45% |
| Macro Precision | 95.74% |
| Macro Recall    | 99.05% |
| Macro F1        | 97.29% |

The confusion matrix showed that SSH-Patator generated the largest false-positive problem.

The model produced:

* 150 BENIGN → SSH-Patator false positives
* 5 SSH-Patator → BENIGN false negatives
* 2 SSH-Patator → FTP-Patator errors

This was useful because it demonstrated that aggregate accuracy alone does not explain model behavior.

---

# 🌲 Random Forest

Random Forest was then evaluated for multiclass classification.

The ordinary random train/test split produced:

**100% accuracy**

However, this result was not accepted as the final result because duplicate feature vectors existed across the training and testing sets.

Instead of ignoring this issue, the project explicitly investigated it.

---

# 🔐 Leakage-Aware Evaluation

A group-aware split was created using identical feature vectors as groups.

This ensured that identical feature groups could not appear in both training and testing sets.

### Verification

```text
Training groups: 42,700
Testing groups: 10,675
Shared groups: 0
```

Therefore, there was no exact feature-group overlap between the final training and testing sets.

---

# 🏆 Final Leakage-Aware Random Forest Result

The Random Forest was retrained using the group-aware split.

### Test set

| Class       |    Samples |
| ----------- | ---------: |
| BENIGN      |      8,977 |
| FTP-Patator |      1,289 |
| SSH-Patator |        849 |
| **Total**   | **11,115** |

### Results

| Metric          |      Result |
| --------------- | ----------: |
| Accuracy        |  **99.99%** |
| Macro Precision | **100.00%** |
| Macro Recall    |  **99.97%** |
| Macro F1        |  **99.99%** |

### Confusion Matrix

```text
                 Pred BENIGN    Pred FTP    Pred SSH

Actual BENIGN        8977           0           0
Actual FTP              1        1288           0
Actual SSH              0           0         849
```

Only **one test sample was misclassified**:

```text
FTP-Patator → BENIGN
```

No SSH-Patator attacks were missed in this evaluation.

---

# 🧠 Security Findings

### 1. Accuracy is not enough

A security model must be evaluated using:

* Precision
* Recall
* F1
* False positives
* False negatives
* Confusion matrices

A model with high accuracy can still be dangerous if it misses attacks.

---

### 2. False negatives matter

In intrusion detection:

```text
Actual Attack → Predicted BENIGN
```

is a potentially serious security failure.

The project therefore treated false-negative analysis as a first-class evaluation step.

---

### 3. Attack-type analysis matters

Aggregate recall can hide weaknesses against individual attack families.

During binary evaluation, the missed attacks were concentrated heavily in SSH-Patator and FTP-Patator traffic.

This demonstrated why multiclass analysis can provide more useful security insight than a simple BENIGN/ATTACK label.

---

### 4. Threshold selection is a security decision

For the binary Logistic Regression model:

| Threshold |  FP | FN |
| --------: | --: | -: |
|       0.3 | 267 |  7 |
|       0.5 | 215 |  9 |
|       0.7 | 113 | 14 |

Lowering the threshold reduced missed attacks but increased false alarms.

Raising the threshold reduced false alarms but increased missed attacks.

Therefore:

> The "best" threshold depends on the operational cost of false positives versus false negatives.

---

### 5. Duplicate data can produce optimistic results

The project found exact feature-vector overlap between training and testing data during the ordinary random split.

Rather than ignoring this, the project:

1. Detected the overlap.
2. Measured it.
3. Investigated label consistency.
4. Created feature groups.
5. Performed a group-aware train/test split.
6. Re-evaluated the model.

This makes the final experiment more defensible.

---

### 6. Random Forest captured nonlinear patterns

The Logistic Regression model provided a useful linear baseline.

Random Forest performed substantially better on this controlled experiment.

This suggests that the relationships between network-flow features and attack classes are not purely linear.

However, this result should not be interpreted as proof that Random Forest is universally superior for intrusion detection.

---

# ⚠️ Limitations

This project is an educational and research-oriented ML security experiment.

The results should **not** be interpreted as production IDS performance.

Important limitations include:

* Only Tuesday's CIC-IDS2017 data was used for the final controlled multiclass experiment.
* Only three classes were modeled.
* CIC-IDS2017 is benchmark traffic, not continuously changing production traffic.
* Dataset-specific patterns may make classification easier than real-world traffic.
* Feature distributions may differ significantly in another network environment.
* Attack behavior can change over time.
* The final experiment does not represent every attack category in CIC-IDS2017.
* High benchmark performance does not guarantee resistance to adversarial or evasive attacks.

The final result should therefore be described as:

> **99.99% accuracy on a leakage-aware, controlled CIC-IDS2017 Tuesday multiclass experiment**

—not as:

> "99.99% accurate real-world intrusion detection."

---

# 🛠️ Technologies

* Python
* NumPy
* Pandas
* Matplotlib
* Scikit-learn
* Jupyter Notebook
* VS Code
* CIC-IDS2017

---

# 📁 Project Structure

```text
Cybersecurity-ML/
│
├── README.md
│
├── 01_cicids2017_data_inspection.ipynb
│
└── MachineLearningCVE/
    ├── Monday-WorkingHours.pcap_ISCX.csv
    ├── Tuesday-WorkingHours.pcap_ISCX.csv
    ├── Wednesday-workingHours.pcap_ISCX.csv
    ├── Thursday-WorkingHours-Morning-WebAttacks.pcap_ISCX.csv
    ├── Thursday-WorkingHours-Afternoon-Infilteration.pcap_ISCX.csv
    ├── Friday-WorkingHours-Morning.pcap_ISCX.csv
    ├── Friday-WorkingHours-Afternoon-PortScan.pcap_ISCX.csv
    └── Friday-WorkingHours-Afternoon-DDos.pcap_ISCX.csv
```

> Large raw dataset files should generally not be committed directly to GitHub. The repository should contain the notebook, documentation, requirements, and reproducible instructions instead.

---

# 🚀 Future Work

Potential next stages for AegisFlow include:

* Full CIC-IDS2017 multiclass classification
* Cross-day validation
* Temporal validation
* Additional attack categories
* XGBoost/LightGBM comparison
* Feature selection
* Explainable AI
* Adversarial ML
* Data poisoning experiments
* Evasion attacks
* Model robustness testing
* Real-time network-flow detection
* Deployment as an inference API
* SOC-style alert generation

---

# 📌 Project Status

**Status: Completed — Phase 1**

The project successfully progressed from:

```text
Synthetic ML examples
        ↓
ML evaluation fundamentals
        ↓
Real cybersecurity dataset
        ↓
Data quality investigation
        ↓
Binary intrusion detection
        ↓
Multiclass attack classification
        ↓
Error analysis
        ↓
Leakage investigation
        ↓
Leakage-aware evaluation
```

The next phase can build on this foundation toward **AI Security / Adversarial Machine Learning**.

---

## Author

**Naren Polavarapu**

AI / Machine Learning / Cybersecurity learning project.

---

## Dataset

CIC-IDS2017 — Canadian Institute for Cybersecurity.
