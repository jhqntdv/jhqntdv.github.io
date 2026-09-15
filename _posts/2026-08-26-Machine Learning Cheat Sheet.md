---
title: Machine Learning Cheat Sheet
date: 2026-08-31 00:00:00 +0000
categories: [Machine Learning]
tags: [Machine Learning]
description: Machine Learning Cheat Sheet
pin: false
math: true
---

## Regularization (L1 vs. L2)

Regularization adds a penalty term to the loss function to prevent overfitting:

* L1 Regularization (Lasso): Adds absolute value penalty ($\lambda \sum |w_i|$). Shrinks coefficients strictly to zero. Can be used for feature selection (creates sparsity).
* L2 Regularization (Ridge): Adds squared penalty ($\lambda \sum w_i^2$). Shrinks coefficients close to zero but never to zero. Cannot perform feature selection (keeps all features).
* Elastic Net: Combines both L1 and L2 penalties ($\lambda_1 \sum |w_i| + \lambda_2 \sum w_i^2$) for feature selection with correlated predictors.

## Model Evaluation Metrics

### 1. Classification Metrics & Confusion Matrix

* **Confusion Matrix**:

  | | Predicted Positive ($\hat{Y}=1$) | Predicted Negative ($\hat{Y}=0$) |
  | :--- | :--- | :--- |
  | **Actual Positive ($Y=1$)** | **TP** (True Positive) | **FN** (False Negative / Type II Error) |
  | **Actual Negative ($Y=0$)** | **FP** (False Positive / Type I Error) | **TN** (True Negative) |

* **Precision**: Proportion of predicted positives that are truly positive (avoids false positives, e.g., spam detection).
  $$ Precision = \frac{TP}{TP + FP} $$

* **Recall (Sensitivity / TPR)**: Proportion of actual positives that are correctly identified (avoids false negatives, e.g., disease detection).
  $$ Recall = \frac{TP}{TP + FN} $$

* **F1-Score**: Harmonic mean of Precision and Recall, balancing both especially on imbalanced datasets.
  $$ F1 = 2 \times \frac{Precision \times Recall}{Precision + Recall} $$

* **ROC Curve**: Plots True Positive Rate (TPR) vs. False Positive Rate (FPR) across different decision thresholds.
  $$ \text{TPR (Recall)} = \frac{TP}{TP + FN}, \quad \text{FPR} = \frac{FP}{FP + TN} $$

* **AUC**: Area under the ROC curve, measuring overall discrimination capability ($AUC = 1$: perfect classifier, $AUC = 0.5$: random guess).

* **PR-AUC vs ROC-AUC**:

  | Feature | ROC-AUC | PR-AUC |
  | :--- | :--- | :--- |
  | Baseline | 0.5 random guess | Proportional to positive class proportion |
  | Imbalance sensitivity | Low | High |
  | Focus | Both classes equally | Positive class only |
  | Interpretation | Prob (1's > 0's); Ranking | Average precision across recall values |
  | Best for | General use | Imbalanced datasets|

### 2. Regression: MSE vs. MAE
* **Mean Squared Error (MSE)**: Penalizes large errors heavily (due to squaring).
  $$ MSE = \frac{1}{n} \sum_{i=1}^n (y_i - \hat{y}_i)^2 $$
  * *Choose MSE when*: Large errors are especially undesirable/costly, and the dataset is relatively clean with few extreme outliers.
* **Mean Absolute Error (MAE)**: Penalizes errors linearly and is robust to extreme values.
  $$ MAE = \frac{1}{n} \sum_{i=1}^n |y_i - \hat{y}_i| $$
  * *Choose MAE when*: The dataset has significant outliers or heavy-tailed noise, and you want predictions to be robust against extreme observations.


## Ensemble Learning

Ensemble learning combines multiple base models (weak learners) to produce a single stronger predictive model. The fundamental difference lies in their objective: **Bagging** trains independent models in parallel on random data subsets to reduce **variance** (prevent overfitting), while **Boosting** trains models sequentially to iteratively correct the errors of previous models to reduce **bias** (prevent underfitting).

* **Bagging (Bootstrap Aggregating)**: Trains multiple models in parallel on random bootstrap samples (with replacement) and aggregates predictions via averaging or voting.
  * Classic Application: Random Forest
* **Boosting**: Trains models sequentially where each subsequent model focuses on the residual errors of the previous ones.
  * Classic Application: Gradient Boosting

## Feature Selection
Three types of feature selection:
- Filter methods: looking at statistics
- Wrapper methods: programmatically evaluating feature subsets (forward, backward methods, etc.). Looks at RMSE, Accuracy, Cross-Validation score.
- Embedded methods: feature selection is part of the model training process (L1, L2, Tree-based models)

Weight of Evidence (WoE) and Information Value (IV)
* Binning is required for WoE and IV calculation.
* WoE and IV are part of filter methods
  * WoE - measures the strength of a feature for "separation"
    $$ WoE_i = \ln \left(\frac{G_i}{B_i}\right) $$
    where $G_i$ is the % of non-events (in terms of total non-events) in the $i$-th bin and $B_i$ is the % of events (in terms of total events) in the $i$-th bin.
  * IV - measures the predictive power of a feature
    $$ IV = \sum_{i} (G_i - B_i) \times WoE_i $$

## Imbalanced Data using SMOTE
SMOTE (Synthetic Minority Oversampling Technique) finds "k-nearest neighbors" for the minority class.
* The sampling-strategy decides how many more minority class samples to generate. The higher the ratio, the more sensitive the model is to the minority class.

* Confusion Matrix for Imbalanced Data before SMOTE

  | | Predicted 1 | Predicted 0 |
  | :--- | :--- | :--- |
  | **Actual 1** | **TP (Near 0)**<br>Model fails to capture the rare class. | **FN (High)**<br>Most 1s are missed. |
  | **Actual 0** | **FP (Low)**<br>Few false alarms because the model rarely predicts 1. | **TN (High)**<br>Model overwhelmingly predicts majority 0. |

* Confusion Matrix for Imbalanced Data after SMOTE

  | | Predicted 1 | Predicted 0 |
  | :--- | :--- | :--- |
  | **Actual 1** | **TP (Increases)**<br>More minority cases are captured (Higher Recall). | **FN (Decreases)**<br>Fewer missed detections; miss rate drops. |
  | **Actual 0** | **FP (Increases)**<br>More false alarms as the model becomes more aggressive in predicting 1. | **TN (Decreases)**<br>Some class 0 samples are now misclassified as 1. |

* **SMOTE** Recall (TP Rate) $\uparrow$, but at the cost of Precision $\downarrow$.

## General Rule

### 1. Classification Decision Flow

Step 1: Is the dataset balanced?
* Balanced Data: Use ROC-AUC or Accuracy.
* Highly Imbalanced Data: Use PR-AUC as it focuses on predicting the minority class.

Step 2: Which error has a higher business cost?
* High False Negative (FN) Cost: Missing an event is critical.
  * Examples: Cancer detection, failing to identify a fraudulent transaction, missing a loan default.
  * Optimization Target: Recall or F2-Score.
* High False Positive (FP) Cost: False alarms are highly disruptive.
  * Examples: Spam filtering, incorrectly flagging a low-risk client for audit.
  * Optimization Target: Precision or F0.5-Score.
* Balanced Costs:
  * Optimization Target: F1-Score (Harmonic mean of Precision and Recall).

Step 3: Threshold Adjustment
* Train using PR-AUC, then evaluate the financial impact of FPs and FNs (Cost-Benefit Matrix) to manually set an optimal threshold that maximizes expected profit.