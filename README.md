# Bias-Aware Employee Attrition Prediction System

## Executive Summary
This project delivers an end-to-end HR Analytics solution designed to predict employee flight risk, diagnose root causes, and audit algorithmic bias. Unlike standard black-box prediction models, this system prioritizes **Explainability (SHAP)** and **Fairness (Disparate Impact Analysis)** to ensure ethical decision-making in human resources.

The solution consists of a Machine Learning pipeline (Random Forest) and an interactive Tableau dashboard, the "Retention Command Center," which allows HR leaders to move from raw data to targeted intervention strategies.

## The Retention Command Center (Dashboard)

The Tableau dashboard provides three distinct views tailored to specific organizational stakeholders:

### 1. Executive Pulse (Strategy View)
* **Target Audience:** CHRO / Executives
* **Key Features:** Visualizes workforce stability trends and identifies high-level risk pockets.
* **Primary Insight:** Displays the "Mid-Tenure Crisis," showing that attrition risk peaks between years 3 and 5, indicating a retention ceiling for experienced staff.

### 2. HR Action List (Operational View)
* **Target Audience:** HR Managers / Business Partners
* **Key Features:** A triage tool that filters high-risk employees by their primary driver (e.g., Low Satisfaction, Training Expiry).
* **Function:** Generates specific intervention scripts (e.g., "Schedule Career Progression Meeting") based on the model's output, moving beyond simple risk scoring to actionable retention steps.

### 3. Fairness & Governance (Audit View)
* **Target Audience:** Legal / Risk Officers
* **Key Features:** A real-time audit system monitoring the model for gender and racial bias.
* **Function:** Implements a "Traffic Light" warning system. If the Disparate Impact Ratio falls below 0.80 for any protected group, the dashboard flags those predictions for mandatory human review.

## Key Business Insights

1.  **The "Mid-Tenure" Cliff:** Analysis reveals that attrition risk does not peak at hiring but spikes significantly between Years 3 and 5. This suggests a systemic failure in career progression for mid-level employees.
2.  **Satisfaction vs. Training:** While training intervals were analyzed, "Satisfaction Score" proved to be the dominant predictor. Employees with scores of 1 or 2 are consistently flagged as high risk, regardless of recent training history.
3.  **Role-Specific Burnout:** The "Production Technician" role exhibits the highest risk density, isolating the problem to specific departmental conditions rather than company-wide culture.
4.  **Algorithmic Bias Detection:** The initial model iteration revealed a Disparate Impact Ratio of 0.70 for female employees (Selection Rate: 19% vs. 13% for males). The system explicitly flags this bias to prevent automated discrimination.

## Technical Architecture

### 1. Data Processing & Engineering
* **Data Cleaning:** Processed raw HR records containing inconsistent date formatting and negative date values (e.g., training dates logged after survey dates).
* **Feature Engineering:**
    * **Time-to-Event Metrics:** Calculated `Days_Since_Training` to measure skill stagnation.
    * **Title Bucketing:** Consolidated high-cardinality job titles into top 20 categories plus an "Other" bucket to improve model generalization.
    * **Interaction Terms:** Derived interactions between Tenure and Satisfaction to identify long-serving but disengaged employees.

### 2. Machine Learning Pipeline
* **Model:** Random Forest Classifier.
* **Configuration:** Utilized `class_weight='balanced'` to address class imbalance (attrition is a minority event).
* **Optimization Strategy:** Optimized for **Recall** to minimize False Negatives, prioritizing the identification of at-risk employees over precision.

### 3. Explainability & Governance
* **Global Explainability:** Utilized SHAP (SHapley Additive exPlanations) to rank feature importance, confirming Tenure and Role as top drivers.
* **Local Explainability:** Generated individual risk factors for every employee (e.g., "High Risk due to Low Satisfaction + Mid-Tenure").
* **Fairness Audit:** Calculated the Disparate Impact Ratio according to the EEOC "Four-Fifths Rule."

## Repository Structure

```text
├── data/
│   ├── Messy_HR_Dataset_Detailed.csv      # Raw input data
│   └── HR_Attrition_Dashboard_Data.csv    # Processed data for Tableau
├── notebooks/
│   └── bias_attrition_analysis.ipynb      # Python analysis, modeling, and audit
├── dashboard-report/
│   ├── report                             # detailed report
│   └── dashboard_screenshots/             # Dashboard visualization assets
├── README.md                              # Project documentation
└── requirements.txt                       # Python dependencies
