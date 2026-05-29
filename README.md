# Recipes and Ratings

## Introduction

This project analyzes recipes and user ratings from Food.com. The dataset consists of two tables:

* **RAW_recipes.csv** containing recipe information such as cooking time, ingredients, tags, and preparation steps.
* **interactions.csv** containing user ratings and reviews for recipes.

The recipes dataset contains **83,782 recipes**, while the interactions dataset contains **731,927 user interactions**. After cleaning and merging the datasets, I study how recipe characteristics relate to user ratings.

The primary question explored in this project is:

> **Do shorter recipes tend to receive higher average ratings than longer recipes?**

This question is interesting because home cooks often balance convenience and quality when choosing recipes. If shorter recipes are rated just as highly as longer recipes, users may prefer recipes that require less preparation time.

The main variables used throughout the analysis are:

* `minutes`: total preparation and cooking time.
* `avg_rating`: average non-zero user rating for a recipe.
* `n_steps`: number of instruction steps.
* `n_ingredients`: number of ingredients.
* `tags`: recipe categories and descriptive labels.
* `submitted`: recipe submission date.

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

Several preprocessing steps were performed before analysis:

1. Replaced ratings equal to 0 with missing values (`NaN`) because Food.com ratings use a 1–5 scale and 0 typically indicates no rating.
2. Computed the average rating (`avg_rating`) for each recipe using non-missing ratings.
3. Merged average ratings back into the recipes dataset using a left join.
4. Created a `missing_rating` indicator column.
5. Converted `submitted` into a datetime variable.
6. Removed recipes with cooking times less than or equal to 0 minutes.
7. Removed recipes requiring 500 minutes or more to reduce the influence of extreme outliers.

After cleaning, the resulting dataset retained most recipes while providing meaningful rating information for analysis.

### Univariate Analysis

The distribution of cooking time is heavily right-skewed. Most recipes require less than one hour to prepare, while a relatively small number require several hours.

The distribution of average ratings is strongly concentrated near 5 stars. Most recipes receive high ratings, making prediction challenging because the target variable exhibits relatively little variation.

### Bivariate Analysis

A scatterplot of cooking time versus average rating shows no strong linear relationship. Ratings remain concentrated near 5 stars across nearly all cooking times.

To summarize the relationship more clearly, recipes were grouped into four cooking-time categories:

| Time Group | Mean Rating |
| ---------- | ----------- |
| 0–30 min   | 4.645       |
| 31–60 min  | 4.607       |
| 61–120 min | 4.627       |
| 120+ min   | 4.591       |

Recipes requiring more than 120 minutes tend to have slightly lower average ratings than recipes requiring 30 minutes or less, though the difference is small.

---

## Assessment of Missingness

### NMAR Analysis

The missingness of `avg_rating` may be **NMAR (Not Missing At Random)**.

A recipe only receives an average rating when users choose to leave ratings. The decision to rate a recipe may depend on the user's unobserved experience with the recipe. Users who strongly liked or disliked a recipe may be more likely to submit ratings, while others may not rate the recipe at all.

Additional information such as page views, recipe saves, or cooking activity would help distinguish NMAR from MAR.

### Missingness Dependency

Two permutation tests were performed to determine whether the missingness of `avg_rating` depends on other variables.

#### Dependency on Cooking Time (`minutes`)

* **Null Hypothesis:** Missingness of `avg_rating` does not depend on cooking time.
* **Alternative Hypothesis:** Missingness of `avg_rating` depends on cooking time.

Using the absolute difference in mean cooking time as the test statistic, the permutation test produced a p-value approximately equal to 0.

Therefore, there is strong evidence that rating missingness depends on cooking time.

#### Dependency on Recipe Name Length (`name_length`)

* **Null Hypothesis:** Missingness of `avg_rating` does not depend on recipe name length.
* **Alternative Hypothesis:** Missingness of `avg_rating` depends on recipe name length.

The permutation test produced a p-value of approximately 0.574.

Therefore, there is insufficient evidence that rating missingness depends on recipe name length.

Together, these tests suggest that missingness depends on some observed variables (such as cooking time) but not others.

---

## Hypothesis Testing

To directly investigate the project question, recipes were divided into:

* **Short recipes:** 30 minutes or less.
* **Long recipes:** 120 minutes or more.

### Hypotheses

**Null Hypothesis**

The mean average rating of short recipes is equal to the mean average rating of long recipes.

**Alternative Hypothesis**

Long recipes have lower average ratings than short recipes.

### Test Statistic

Mean rating of short recipes minus mean rating of long recipes.

### Results

Observed difference:

* Short recipes mean rating: 4.645
* Long recipes mean rating: 4.593
* Difference: 0.052

A permutation test with 1,000 simulations produced a p-value approximately equal to 0.

Therefore, the null hypothesis was rejected. There is strong statistical evidence that long recipes receive slightly lower ratings than short recipes, although the practical difference is relatively small.

---

## Framing a Prediction Problem

The prediction task is:

> **Predict a recipe's average rating (`avg_rating`) using only information available when the recipe is posted.**

This is a **regression problem** because the response variable is numerical.

The evaluation metric is:

* **RMSE (Root Mean Squared Error)**

RMSE is appropriate because prediction errors are measured directly in rating-star units and are easily interpretable.

Features used are limited to information available at posting time, including:

* Cooking time
* Number of ingredients
* Number of steps
* Submission date
* Recipe tags

Information generated after posting, such as ratings, reviews, and number of interactions, is excluded.

---

## Baseline Model

The baseline model uses:

* `minutes`
* `n_steps`
* `n_ingredients`

A Linear Regression model was trained using a pipeline with median imputation and feature standardization.

### Results

| Metric | Value |
| ------ | ----- |
| RMSE   | 0.633 |
| R²     | 0.001 |

The baseline model performs only slightly better than predicting the overall mean rating.

---

## Final Model

The final model uses additional engineered features:

* `log_minutes`
* `year`
* `month`
* Tag indicators (`dessert`, `easy`, `healthy`, `vegetarian`, etc.)
* `n_steps`
* `n_ingredients`

A Ridge Regression model was selected because it remains interpretable while helping reduce overfitting when correlated predictors are included.

### Results

| Model                      | RMSE  | R²    |
| -------------------------- | ----- | ----- |
| Baseline Linear Regression | 0.633 | 0.001 |
| Final Ridge Regression     | 0.632 | 0.007 |

The final model achieves a small improvement over the baseline model. The limited improvement is expected because recipe ratings are highly concentrated near 5 stars and therefore difficult to predict accurately.

---

## Fairness Analysis

To evaluate fairness, model performance was compared between:

* **Group X:** recipes taking 30 minutes or less.
* **Group Y:** recipes taking more than 30 minutes.

### Hypotheses

**Null Hypothesis**

The model has equal RMSE for both groups.

**Alternative Hypothesis**

The model has different RMSE values across groups.

### Test Statistic

Absolute difference in RMSE between groups.

### Results

* RMSE (short recipes): 0.604
* RMSE (longer recipes): 0.654
* Difference: 0.050

A permutation test produced a p-value of approximately 0.001.

Because the p-value is below 0.05, the null hypothesis is rejected. The model performs significantly better on shorter recipes than on longer recipes, indicating that prediction accuracy differs across these groups.
