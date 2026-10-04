# Applied Cybersecurity Labs

A collection of hands-on projects from coursework spanning digital forensics
and incident response (DFIR), offensive security, and machine learning
applied to network security.

## Projects

| Project | Focus | Summary |
|---|---|---|
| [Machine Exploitation & Defense Evasion Investigation](./MachineDefense_Evasion_Writeup.pdf) | DFIR + Offensive Security | A two-part challenge: investigated a post-breach host (background in [Defense_Evasion_Writeup.pdf]([./Machine Defense_Evasion_Writeup.pdf](https://github.com/KaylaTee1009/applied-cybersecurity-labs/blob/79484298a07ea0216be9ea259211b9a6a0428f5d/Machine%20Defense_Evasion_Writeup.pdf))) where security logs and Defender alerts were missing, tracing the attacker's full defense-evasion chain — LSA protection tampering, Defender config changes, an AMSI bypass, Safe Mode reconfiguration, and PowerShell logging suppression — using Windows event logs, PowerShell logs, and Sysmon data. Then performed an end-to-end compromise of the Windows target itself: Nmap service enumeration, unauthenticated SMB share discovery, EVTX log parsing to recover plaintext credentials, credential verification with CrackMapExec, and an RDP session to retrieve the user flag. |
| [Network Intrusion Detection](./Network_Intrusion_Detection_Write-Up.pdf) | Security + Machine Learning | Built a two-stage intrusion detection pipeline on 3.1M rows of network traffic: a Random Forest binary classifier (Benign vs. Attack, zero false negatives) and a PyTorch DNN multi-class classifier across 6 attack types, with class-weighted loss to handle severe class imbalance. Includes a full preprocessing audit documenting a rare attack class that was inadvertently dropped during data cleaning. |

---

## Network Intrusion Detection — Detailed Write-Up

**Author:** Kayla Teekaram

### Overview
This project built a two-task intrusion detection system on a 3.1-million-row
network traffic dataset (80 features, 7 original label classes): a Random
Forest model for **binary** Benign-vs-Attack classification, and a PyTorch
deep neural network (DNN) for **multi-class** classification across specific
attack types.

### Data Preprocessing
The dataset arrived split across three CSV files (`m_1.csv`, `m_2.csv`,
`m_3.csv`). After concatenation, the following issues were identified and
cleaned up, in order:

1. **Duplicate header rows** — splitting the original file into three CSVs
   left some rows containing the literal string `"Label"` in the Label
   column; these were removed before any numeric conversion.
2. **Non-numeric Timestamp column** — contained date/time strings that
   couldn't be converted to numbers, so it was dropped entirely.
3. **Mixed-type feature columns** — all remaining feature columns were cast
   to numeric with `pd.to_numeric(errors='coerce')`; unparseable values
   became `NaN`.
4. **Sentinel value -1** — several columns used `-1` as a placeholder for
   missing data rather than `NaN`; these were replaced with `NaN` for
   consistent handling.
5. **Infinite values** — columns like flow-bytes-per-second contained ±∞ from
   division by near-zero durations; replaced with `NaN`.
6. **Row-level NaN drop** — any row still containing `NaN` after the above
   steps was dropped entirely.
7. **Zero-variance columns** — columns with standard deviation of zero were
   removed, since they carry no information and can break `StandardScaler`.

**Key finding from the preprocessing audit:** the `DDOS attack-LOIC-UDP`
class — roughly 0.05% of the raw data (~100 of ~200,000 records) —
disproportionately contained missing and infinite values and was **completely
eliminated** during the NaN/infinity cleanup. As a result, only 6 of the
original 7 classes survived into modeling, and the DNN was trained and
evaluated as a 6-class classifier rather than 7-class.

Feature scaling used `StandardScaler`, **fit only on the training split**
and then applied to the test split, to prevent data leakage. Both tasks used
an 80/20 stratified train/test split (`random_state=42`) to preserve class
proportions.

### Results

**Task 1 — Random Forest (Binary: Benign vs. Attack)**

| Class | Precision | Recall | F1-Score | Support |
|---|---|---|---|---|
| Benign (0) | 1.00 | 1.00 | 1.00 | 224,766 |
| Attack (1) | 1.00 | 1.00 | 1.00 | 142,025 |
| **Accuracy** | | | **1.00** | 366,791 |

The confusion matrix showed 224,765 true Benign and 142,025 true Attack
predictions, with only **1** Benign sample misclassified as Attack and
**0** Attack samples misclassified as Benign. False negatives — attacks
labeled as Benign — are the most operationally dangerous outcome for an IDS,
and the model produced none.

**Task 2 — DNN (Multi-Class, 6 classes after preprocessing)**

| Class | Precision | Recall | F1-Score | Support |
|---|---|---|---|---|
| Benign | 1.00 | 1.00 | 1.00 | 224,766 |
| DDOS attack-HOIC | 1.00 | 1.00 | 1.00 | 32,750 |
| DoS attacks-Hulk | 1.00 | 1.00 | 1.00 | 5,109 |
| DoS attacks-SlowHTTPTest | 0.53 | 0.92 | 0.67 | 27,978 |
| FTP-BruteForce | 0.87 | 0.41 | 0.56 | 38,671 |
| SSH-Bruteforce | 1.00 | 1.00 | 1.00 | 37,517 |
| **Accuracy** | | | **0.93** | 366,791 |

The model performed near-perfectly on high-frequency classes (Benign,
DDOS-HOIC, SSH-Bruteforce). The most notable confusion was between
**DoS-SlowHTTPTest** and **FTP-BruteForce**, which had lower recall (0.92 and
0.41 respectively) — the model frequently mistook FTP-BruteForce traffic for
other classes.

**DNN Hyperparameters**

| Hyperparameter | Value |
|---|---|
| Input dimensions | Set automatically from remaining features after preprocessing |
| Hidden layer 1 | 128 neurons |
| Hidden layer 2 | 64 neurons |
| Output layer | 6 neurons (one per surviving class) |
| Dropout rate | 0.3, applied after the first hidden layer |
| Activation | ReLU after each hidden layer |
| Loss function | CrossEntropyLoss with balanced class weights |
| Optimizer | Adam, learning rate = 0.001 |
| Batch size | 512 |
| Epochs | 50 |
| Train/test split | 80% / 20%, stratified |
| Random seed | 42 |

### Class Imbalance — DDOS attack-LOIC-UDP
`DDOS attack-LOIC-UDP` made up ~0.05% of the raw dataset. Because these
records disproportionately contained infinite flow-rate values and missing
fields, every single one was removed during the NaN/infinity cleanup step.
The notebook's preprocessing audit confirmed this explicitly — the label
encoder learned only 6 classes, with LOIC-UDP completely absent from both the
encoder and the post-cleaning label distribution.

This means the DNN's reported accuracy, precision, recall, and F1 **do not
reflect any performance on LOIC-UDP whatsoever** — a deployed version of this
model would be fully blind to that attack type. The macro-averaged F1 of 0.87
also looks better than it would if LOIC-UDP had survived, since the hardest
class to detect is missing from the evaluation entirely.

For the 6 surviving classes, `compute_class_weight('balanced')` was applied
to `CrossEntropyLoss` to give rarer classes a stronger gradient signal during
training. This improved recall on classes like DoS-SlowHTTPTest (4.45% of
data), though FTP-BruteForce still showed lower recall (0.41), suggesting it
was frequently confused with other attack types.

### AI Discussion & Verification
AI coding tools were used to help generate boilerplate code — specifically
the PyTorch `DataLoader` setup, training loop structure, and seaborn heatmap
formatting. All AI-generated code was reviewed before use, and one correction
was required.

The AI-generated DNN architecture originally applied `nn.Softmax(dim=1)` to
the output layer before passing it into `nn.CrossEntropyLoss`:

```python
# INCORRECT — softmax applied before CrossEntropyLoss
self.network = nn.Sequential(
    nn.Linear(128, 64), nn.ReLU(),
    nn.Linear(64, num_classes),
    nn.Softmax(dim=1)  # <-- wrong
)
criterion = nn.CrossEntropyLoss()
```

This is incorrect: PyTorch's `CrossEntropyLoss` already applies log-softmax
internally before computing the negative log-likelihood. Applying softmax
beforehand means the model effectively computes
`log(softmax(softmax(logits)))`, which double-applies the normalization,
compresses outputs toward uniformity, and causes gradients to vanish or
become `NaN` — the model would fail to learn anything meaningful.

The corrected version passes raw logits directly to the loss function:

```python
# CORRECT — raw logits, CrossEntropyLoss handles softmax internally
self.network = nn.Sequential(
    nn.Linear(128, 64), nn.ReLU(),
    nn.Linear(64, num_classes)  # raw logits only
)
criterion = nn.CrossEntropyLoss(weight=class_weights_tensor)
```

This was verified against the official PyTorch documentation, which states
that `CrossEntropyLoss` is equivalent to combining `LogSoftmax` and
`NLLLoss`. The fix was confirmed by observing the training loss decrease
smoothly from the first epoch with no `NaN` values in the loss curve.

---

## Skills Demonstrated
- Windows forensics and incident response (Sysmon, Windows Event Logs, PowerShell logging)
- Defense evasion technique analysis (LSA tampering, AMSI bypass, Defender manipulation)
- Network enumeration and exploitation (Nmap, SMB, CrackMapExec, RDP)
- EVTX log parsing and credential recovery
- Applied machine learning for security (scikit-learn, PyTorch)
- Large-scale data preprocessing, class imbalance handling, and rigorous model evaluation
- Critical review and correction of AI-generated code

## Stack
Python · PyTorch · scikit-learn · pandas · Nmap · Impacket/CrackMapExec ·
Sysmon · PowerShell · Windows Event Log (EVTX)
