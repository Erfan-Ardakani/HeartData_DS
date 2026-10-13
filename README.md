# Heart Disease Data Analysis & Machine Learning

**Exploratory data analysis, preprocessing, model comparison, and hyperparameter tuning with Python**

## Overview

This project investigates a structured heart-health dataset and compares supervised machine-learning methods for predicting a binary target (`c`). It covers the main stages of a practical data science workflow, from data quality checks and exploratory analysis to classification, model evaluation, and hyperparameter optimization.

The aim is to explore how statistical analysis and computational methods can be combined to understand a real-world dataset and assess different modeling approaches. **This is an educational and exploratory project, not a medical diagnostic tool.**

## Dataset

The repository contains `Heart data.csv`. In the saved notebook output, the initial dataset has **597 observations and 14 columns**, including the binary target `c`. Variables relate to age, chest pain, blood pressure, cholesterol, heart rate, and other heart-health indicators. One duplicate record is removed during preprocessing, leaving **596 observations** for the subsequent workflow.

The original external source and license of this compiled dataset are not documented in the repository. They should be verified before redistributing or using it beyond this learning project.

## Workflow

1. **Explore the data:** inspect feature types, descriptive statistics, distributions, correlations, and relationships with the target.
2. **Check data quality:** identify missing values and duplicates; use median/mode imputation, investigate outliers, and examine infrequent categorical values.
3. **Prepare features:** one-hot encode categorical variables, create a stratified 80/20 train/test split (`random_state=42`), standardize numerical features, and apply SMOTE to the training data to address class imbalance.
4. **Compare seven classifiers:** Logistic Regression, Random Forest, K-Nearest Neighbors, Support Vector Machine, Decision Tree, XGBoost, and a Multi-Layer Perceptron neural network.
5. **Evaluate models:** compare accuracy, precision, recall, F1-score, and ROC-AUC; visualize confusion matrices and ROC curves.
6. **Tune hyperparameters:** use `GridSearchCV` with 5-fold cross-validation, evaluate tuned models, and compare baseline and tuned results.

## Results recorded in the notebook

The notebook contains saved evaluation outputs for a **120-observation test set**. The following are the recorded results for the tuned models; they have **not been independently rerun** for this README.

| Model | Test F1-score | Test ROC-AUC |
| --- | ---: | ---: |
| Neural Network (MLP) | 0.768 | 0.834 |
| XGBoost | 0.763 | 0.886 |
| Random Forest | 0.758 | **0.896** |
| Support Vector Machine | 0.750 | 0.882 |
| Logistic Regression | 0.742 | 0.878 |
| K-Nearest Neighbors | 0.713 | 0.837 |
| Decision Tree | 0.702 | 0.752 |

In these saved results, the neural network has the highest test F1-score, while Random Forest has the highest test ROC-AUC. The metrics are **exploratory rather than validated estimates of clinical performance**; see the methodological notes below.

## Repository files

```text
HeartData_DS/
├── Heart data.csv                    # Dataset used in the notebook
├── HeartDataProject.ipynb           # Main analysis and model comparison
├── HeartDataProject.html            # Exported notebook view
└── Heart_Data_ErfanArdakani.pdf     # Project report / PDF export
```

The notebook also writes `model_results.csv` and `optimized_model_results.csv` when its evaluation sections run. These generated CSV files are not included in the current repository.

## How to run

1. Clone the repository and open the project folder:

   ```bash
   git clone https://github.com/Erfan-Ardakani/HeartData_DS.git
   cd HeartData_DS
   ```

2. Install the Python libraries used in the notebook (preferably in a virtual environment):

   ```bash
   python -m pip install jupyter pandas numpy scipy matplotlib seaborn scikit-learn imbalanced-learn xgboost
   ```

3. **Update the dataset path** in `HeartDataProject.ipynb`. The notebook currently references a local Windows path. Change the data-loading statement to:

   ```python
   df = pd.read_csv("Heart data.csv")
   ```

4. Start Jupyter and execute the notebook from top to bottom:

   ```bash
   jupyter notebook HeartDataProject.ipynb
   ```

Hyperparameter grid search may be computationally intensive. Library versions are not pinned, so execution behavior and results may vary across environments.

## Methodological notes and next steps

This repository documents an exploratory learning workflow. For stronger research-grade evaluation, several improvements are needed:

- **Prevent information leakage:** some data cleaning and feature-preparation operations currently occur before the train/test split; data-derived transformations should be fitted on training data only.
- **Cross-validate resampling correctly:** SMOTE is applied before the internal cross-validation used for model tuning. A pipeline that applies SMOTE within each training fold would give a more reliable estimate.
- **Strengthen reproducibility:** add dependency versions, document the dataset provenance, and use portable file paths.
- **Improve validation:** consider repeated or nested cross-validation, calibration, uncertainty estimates, and evaluation on an external dataset.

These limitations mean the reported scores should not be interpreted as evidence of clinical reliability or generalization to new patient populations.

## Tools

Python · pandas · NumPy · SciPy · Matplotlib · Seaborn · scikit-learn · imbalanced-learn (SMOTE) · XGBoost · Jupyter Notebook

## Author

**Erfan Ardakani**  
Computer Science | Data Science & Computational Analysis  
[GitHub profile](https://github.com/Erfan-Ardakani)
