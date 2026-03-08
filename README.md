Here is a simpler `README.md` you can copy‑paste and adapt to your current (messy) notebooks.

***

# WDI Country Development & Health‑Risk Project

This project uses World Development Indicators (WDI) data for about 190 countries to:

- identify **development clusters** (developed vs. less developed countries);  
- build a **health‑risk index** that highlights countries with the highest health risks;  
- explain this index using regularized regression and other ML tools.

> Note: the Jupyter notebooks are exploratory and contain EDA and experiments; the core logic and numbers are still reproducible.

***

## Data

- Source: World Development Indicators (World Bank). [wdi.worldbank](http://wdi.worldbank.org)
- Period: roughly 2003–2023 (about 20 years).  
- Unit of analysis: country; time‑series are aggregated to country‑level features.

**Main indicators used:**

- Economic: GDP per capita, GNI per capita.  
- Health: infant mortality, maternal mortality risk, life expectancy at birth. [wdi.worldbank](https://wdi.worldbank.org/table/2.18)
- Access & infrastructure: access to electricity, access to basic water, internet use.  
- Social: school enrollment, urban population share.

From these, I construct:

- **5‑year summaries** (recent levels).  
- **20‑year summaries** (levels + growth rates).

***

## Part 1 – Development Clustering

### 1.1 HAC + Random Forest

- I apply **hierarchical agglomerative clustering (HAC)** on 5‑year summary features and obtain 4 “development level” clusters.  
- Then I train a **RandomForestClassifier** to predict these cluster labels from the original indicators.  
- I use the random forest’s **feature importances** to see which variables drive the separation between developed and less developed countries. [ewadirect](https://www.ewadirect.com/proceedings/tns/article/view/16425)

Key drivers:

- Infant mortality (strongest signal).  
- GDP / GNI per capita.  
- Access to basic infrastructure (electricity, basic water).

A small **surrogate decision tree** is fitted on the RF predictions to emulate the forest with simple rules and to make the cluster logic easier to interpret.

### 1.2 20‑Year Country Index (18 → 3 clusters)

- I extend the feature set to include **20‑year trends** (growth rates for GDP, mortality, life expectancy, access variables, etc.).  
- HAC on this richer feature set produces **18 clusters**, which I then manually merge into a **3‑cluster “country index”**:
  - High‑development countries.  
  - Strongly developing / converging countries.  
  - Low‑development countries.

Random forest, **PCA**, and another surrogate tree are used again to:

- confirm which features matter most;  
- reduce dimensionality;  
- understand the structure of the 3‑cluster index.

Across these steps, **infant mortality** repeatedly appears as the dominant discriminator of development.

***

## Part 2 – Health‑Risk Index

### 2.1 Building the index

Goal: construct a **health‑risk score** and a **health‑risk rank** for each country.

Core inputs:

- Infant mortality.  
- Maternal mortality risk.  
- Life expectancy.  
- Selected 20‑year changes in these indicators (with caps on extreme life‑expectancy growth).

Steps:

1. Standardize the health indicators.  
2. Run **PCA** and use the **first principal component (PCA1)** to derive weights.  
3. Combine the indicators into a **weighted sum** (health‑risk score), where:
   - Higher infant/maternal mortality and lower life expectancy → higher risk.  
   - Growth variables get smaller weights and are sometimes capped, so “catch‑up” countries are not over‑rewarded. [pmc.ncbi.nlm.nih](https://pmc.ncbi.nlm.nih.gov/articles/PMC5590867/)

The continuous score is then converted into a **rank** (e.g. 1 = lowest risk, 5 = highest risk) using quantile cuts and simple level‑based rules (such as minimum life‑expectancy thresholds for the lowest‑risk group).

### 2.2 Validating the health‑risk ranking

I cross‑tab the health‑risk rank with the 3‑cluster country index from Part 1.

- Among 61 countries in the **lowest‑development** cluster, **57 fall into the highest health‑risk rank**.  
- No high‑development countries appear in the highest health‑risk rank.

This shows that the health‑risk index aligns well with the development typology: low‑development countries are systematically high‑risk, and high‑development countries are not falsely flagged.

***

## Part 3 – Predicting Health Risk (Ridge Regression)

To understand which broader factors are associated with the health‑risk score, I fit a **ridge regression** model.

- Target: continuous **health‑risk score**.  
- Predictors: a wide set of standardized WDI indicators, including:
  - The 3‑cluster development index (hac18_3).  
  - Income variables, access variables, and other socio‑economic features (excluding direct components of the score where needed).

I use cross‑validation over a grid of alphas (RidgeCV) and additional checks for coefficient stability.

**Performance (cross‑validated):**

- \(R^2 \approx 0.67\).  
- RMSE ≈ 0.35 on the standardized outcome.

**Main findings:**

- The **development‑level cluster** (hac18_3) is the strongest single predictor of the health‑risk score.  
- Internet use and access to electricity (overall and urban) are also important.  
- Given the small sample (~190 countries) and high collinearity among predictors, more complex models (e.g. boosted trees) would likely overfit and are not necessary.

***

## How to use this repository

Right now, the project consists mainly of exploratory Jupyter notebooks. A typical workflow is:

1. Open the clustering notebook(s) to see how the **development clusters** and **3‑cluster country index** were created.  
2. Open the health‑index notebook to see:
   - how the **health‑risk score and rank** are constructed;  
   - how they align with the development clusters.  
3. Open the ridge‑regression notebook to:
   - inspect cross‑validated performance;  
   - review coefficient patterns and main drivers.

Over time, these notebooks can be refactored into cleaner scripts and a more formal pipeline, but even in their current state they reproduce the main results: a development typology and a health‑risk ranking for WDI countries, plus an interpretable model that links health risk to underlying development conditions.

---
