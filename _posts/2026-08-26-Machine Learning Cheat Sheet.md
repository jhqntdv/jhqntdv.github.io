---
title: Machine Learning Cheat Sheet
date: 2026-08-31 00:00:00 +0000
categories: [Notes]
tags: [Machine Learning]
description: Machine Learning Cheat Sheet
pin: false
hidden: true
math: true
---

## Regularization (L1 vs. L2)

Regularization adds a penalty term to the loss function to prevent overfitting:

* L1 Regularization (Lasso): Adds absolute value penalty ($\lambda \sum \lvert w_i \rvert$). Shrinks coefficients strictly to zero. Can be used for feature selection (creates sparsity).
* L2 Regularization (Ridge): Adds squared penalty ($\lambda \sum w_i^2$). Shrinks coefficients close to zero but never to zero. Cannot perform feature selection (keeps all features).
* Elastic Net: Combines both L1 and L2 penalties ($\lambda_1 \sum \lvert w_i \rvert + \lambda_2 \sum w_i^2$) for feature selection with correlated predictors.

## Overfitting: Patterns, Diagnostics & Prevention

* **Definition & Core Behavior**:
  * The model fits noise, random fluctuations, and sample-specific idiosyncrasies rather than the true underlying data-generating distribution.
  * **Bias-Variance Tradeoff**: High Variance, Low Bias. The model is overly sensitive to small variations in the training set.

* **Observable Patterns & Impact on Prediction**:
  * **Generalization Gap**: Very high training accuracy (or near-zero training loss) paired with significantly worse validation/test performance.
  * **Exploding Coefficients & Complex Boundaries**: Decision boundaries become unnaturally convoluted; regression weights grow excessively large ($\lvert w \rvert \gg 0$) to fit edge-case outliers.
  * **Impact on Prediction**:
    * **High Out-of-Sample Prediction Variance**: Forecasts on unseen data become unstable and erratic.
    * **Overconfidence & Fragility**: The model makes high-confidence errors and fails under data drift or market regime shifts.

* **Diagnostic Tools & Checks**:
  * **Learning Curves (Train vs. Validation Loss)**:
    * Plot loss against epochs or training set size. Overfitting is diagnosed when training loss keeps declining while validation loss bottoms out and begins to diverge upward (the "U-curve").
  * **$k$-Fold Cross-Validation Gap**:
    * Measure score divergence:
      $$ \Delta = \text{Score}_{\text{train}} - \text{Score}_{\text{val}} $$
    * A substantial divergence indicates overfitting.
  * **Train vs. Test Residual Analysis**:
    * Compare error distributions. Test residuals that are significantly larger and display systematic structure indicate poor generalization.

* **Prevention Techniques**:
  * **1. Regularization**: Apply L1 (Lasso) to enforce sparsity or L2 (Ridge) / Elastic Net to shrink parameter magnitudes.
  * **2. Early Stopping**: Halt training dynamically when validation loss stops improving over $N$ consecutive evaluations.
  * **3. Tree Pruning & Structural Constraints**: Restrict tree complexity by setting `max_depth`, `min_samples_split`, and `min_samples_leaf`.
  * **4. Ensemble Methods (Bagging)**: Average predictions across multiple independently trained models (e.g., Random Forests) to reduce variance without increasing bias.
  * **5. Feature Selection & Dimensionality Reduction**: Remove uninformative/collinear features via PCA, Information Value (IV), or feature importance thresholds.
  * **6. Data Augmentation & Dropout**: Expand training coverage with noise injection or synthetic samples; apply Dropout in neural networks to prevent feature co-adaptation.

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
  $$ MAE = \frac{1}{n} \sum_{i=1}^n \lvert y_i - \hat{y}_i \rvert $$
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