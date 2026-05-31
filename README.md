# From Data to Defense: Forecasting System Threats using Machine Learning

A complete end-to-end machine learning pipeline for predicting the likelihood of a Windows system being infected by malware, built as a Kaggle competition submission.

---

## Problem Statement

Given telemetry data collected by antivirus software — covering hardware specs, software versions, OS configurations, and regional information — the goal is to predict whether a system will be infected by malware (`target = 1`) or not (`target = 0`).

---

## Dataset

The dataset originates from Kaggle's **System Threat Forecaster** competition and contains two files:

| File | Description |
|------|-------------|
| `train.csv` | Labelled training data with hardware, software, and security features |
| `test.csv` | Unlabelled test data for generating predictions |

**Key characteristics of the training data:**
- 76 raw features across numerical, categorical, binary, and ID types
- Mix of `int64`, `float64`, and `object` dtypes
- Some missing values (up to ~98% for `SMode`)
- 165 duplicate rows
- A **balanced target variable** — roughly equal malware detected vs. not detected
- Multiple high cardinality features

---

## Project Structure

```
notebook.ipynb          # Main analysis and modelling notebook
submission.csv          # Final predictions on the test set
```

---

## Workflow

### 1. Data Exploration
- Loaded and inspected both train and test sets
- Identified and removed 165 duplicate rows
- Separated features into: **19 numerical**, **27 categorical**, **16 binary**, and **15 ID** columns
- Visualised missing values, target distribution, and feature cardinality
- Dropped `MachineID` (pure row identifier with no predictive value)

### 2. Exploratory Data Analysis

**Univariate Analysis**
- Histograms and boxplots for continuous features (`SystemVolumeCapacityMB`, `TotalPhysicalRAMMB`, `PrimaryDiskCapacityMB`, display specs)
- Count plots for all categorical and binary features
- Top-10 version plots for high-cardinality version strings (`EngineVersion`, `AppVersion`, `SignatureVersion`, etc.)

**Key findings:**
- Most devices are consumer-grade with 4–8 GB RAM, 256–512 GB storage, and 15.5-inch 1366×768 HD displays
- x64 Windows 10 Home on HDDs dominates the dataset
- The majority of systems are genuine, licensed, and have TPM + firewall enabled
- Gamers show a notably higher malware detection rate (~55%) vs. non-gamers (~49%)

**Bivariate Analysis**
- Ranked Proportion Plots to compare malware detection rates across categories
- Pearson, Spearman, and Cramér's V correlation heatmaps to detect redundant features

**Redundant features identified and dropped:**
`OSBuildLab`, `OSBuildNumberOnly`, `SKUEditionName`, `OSSkuFriendlyName`, `OSInstallLanguageID`, `Processor`, `OSVersion`, and other highly correlated or constant columns

### 3. Feature Engineering

| Technique | Details |
|-----------|---------|
| **Feature Splitting** | Decomposed version strings (`EngineVersion`, `AppVersion`, `SignatureVersion`, `NumericOSVersion`) into Major/Minor/Build/Revision components |
| **Datetime Extraction** | Extracted year, month, day, hour, minute from `DateAS` and `DateOS` |
| **Feature Binning** | Grouped `MDC2FormFactor`, `ChassisType`, `PowerPlatformRole`, `OSBranch`, `OSEdition`, `OSInstallType`, `AutoUpdateOptionsName`, `LicenseActivationChannel`, `FlightRing` into cleaner, lower-cardinality categories |
| **Feature Creation** | Engineered: `Ram_per_core`, `Aspect_Ratio`, `Primary_Disk_Allocated`, `Pixel_Density`, `Free_Disk_Space`, `Days_since_OS_Installation` |

Post-engineering correlation analysis removed further redundant features including `SignatureVersion_Minor/Major/Revision`, `NumericOS_*`, `ProductName`, and `OsPlatformSubRelease`.

### 4. Data Preprocessing

A `ColumnTransformer` pipeline was constructed with separate strategies per feature type:

| Column Type | Imputation | Encoding/Scaling |
|-------------|------------|-----------------|
| Numerical | Mean | MinMaxScaler |
| Binary | Most frequent | OrdinalEncoder |
| ID | Most frequent | None |
| Categorical | Most frequent | OneHotEncoder (ignore unknown) |

An 80/20 train-validation split was used (`random_state=42`).

### 5. Model Building

Eight baseline classifiers were evaluated with 5-fold cross-validation:

- Logistic Regression
- Decision Tree
- Random Forest
- AdaBoost
- Bagging Classifier
- **XGBoost** ✓
- **LightGBM** ✓
- **Random Forest** ✓

The top three models by test accuracy were selected for further tuning.

### 6. Feature Selection & Hyperparameter Tuning

RFECV (Recursive Feature Elimination with Cross-Validation) was applied to each of the three finalist models to identify the most predictive feature subsets.

Grid search (`GridSearchCV`) was used for XGBoost and LightGBM; randomised search (`RandomizedSearchCV`) for Random Forest.

**Best hyperparameters:**

| Model | Best Parameters |
|-------|----------------|
| XGBoost | `max_depth=5`, `learning_rate=0.1`, `n_estimators=200`, `reg_lambda=10` |
| LightGBM | `max_depth=7`, `learning_rate=0.1`, `num_leaves=31`, `min_child_samples=30`, `reg_lambda=10` |
| Random Forest | `max_depth=8`, `n_estimators=50`, `min_samples_leaf=10` |

### 7. Final Results

| Model | Train Accuracy | Test Accuracy |
|-------|---------------|--------------|
| XGBoost (tuned) | ~66.5% | ~62.8% |
| LightGBM (tuned) | ~65.4% | ~62.9% |
| Random Forest (tuned) | ~63.09 | ~61.04 |

**LightGBM** was selected as the final model for submission due to its best test accuracy and relatively small train/test gap.

---

## Dependencies

```
pandas
numpy
scikit-learn
lightgbm
xgboost
seaborn
matplotlib
scipy
```

Install all dependencies with:

```bash
pip install pandas numpy scikit-learn lightgbm xgboost seaborn matplotlib scipy
```

---

## Usage

This notebook is designed to run on **Kaggle** with the competition dataset available at `/kaggle/input/System-Threat-Forecaster/`. To run locally, update the data paths and ensure the dependencies above are installed.

```bash
jupyter notebook notebook.ipynb
```

The final submission is saved as `submission.csv` with columns `id` and `target`.

---

## Key Insights

- Gamer devices and desktop/notebook form factors show higher malware detection rates, likely due to riskier download behaviour and greater internet exposure
- Newer Windows versions (Windows 10, rs4 branch) show higher detection rates — possibly reflecting improved detection capabilities rather than increased vulnerability
- x64 architecture has the highest detection rate, consistent with its dominant market share
- Security features (firewalls, RTP) correlate with higher detection rates, suggesting they surface threats rather than preventing them
- Software version and update recency are among the most informative features for prediction
