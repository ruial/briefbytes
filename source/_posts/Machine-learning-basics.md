---
title: Machine learning basics
date: 2020-02-08 15:38:06
tags: [data, R]
---

In this post I will briefly describe how to achieve a top 20% score for the Titanic survival classification problem and the house prices regression problem in Kaggle. The caret library in R makes it easy to test different machine learning algorithms.

## Machine learning

Machine learning algorithms are able to learn from past data how to make predictions on new data. The most common types of algorithms are supervised or unsupervised.

Supervised algorithms include classification (predict a class) and regression (predict continuous variable) problems. Training data contains labels and common metrics to evaluate models include accuracy, precision and recall for classification, or root mean square error (RMSE) and R-squared (R2) for regression.

Unsupervised algorithms usually consist on dimensionality reduction (PCA, t-sne), clustering (k-means, DBSCAN) and anomaly detection. These do not require training data, and there are different techniques to evaluate each algorithm.

## Classification

The Titanic problem is a binary classification problem that consists on predicting the passengers who survived the disaster. Let's take a look at the data.

```
> glimpse(titanic_train)
Observations: 891
Variables: 12
$ PassengerId <int> 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 2...
$ Survived    <int> 0, 1, 1, 1, 0, 0, 0, 0, 1, 1, 1, 1, 0, 0, 0, 1, 0, 1, 0, 1, 0, 1, 1, 1, 0, 1, 0, 0, 1, 0, 0, 1, ...
$ Pclass      <int> 3, 1, 3, 1, 3, 3, 1, 3, 3, 2, 3, 1, 3, 3, 3, 2, 3, 2, 3, 3, 2, 2, 3, 1, 3, 3, 3, 1, 3, 3, 1, 1, ...
$ Name        <fct> "Braund, Mr. Owen Harris", "Cumings, Mrs. John Bradley (Florence Briggs Thayer)", "Heikkinen, Mi...
$ Sex         <fct> male, female, female, female, male, male, male, male, female, female, female, female, male, male...
$ Age         <dbl> 22, 38, 26, 35, 35, NA, 54, 2, 27, 14, 4, 58, 20, 39, 14, 55, 2, NA, 31, NA, 35, 34, 15, 28, 8, ...
$ SibSp       <int> 1, 1, 0, 1, 0, 0, 0, 3, 0, 1, 1, 0, 0, 1, 0, 0, 4, 0, 1, 0, 0, 0, 0, 0, 3, 1, 0, 3, 0, 0, 0, 1, ...
$ Parch       <int> 0, 0, 0, 0, 0, 0, 0, 1, 2, 0, 1, 0, 0, 5, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 5, 0, 2, 0, 0, 0, 0, ...
$ Ticket      <fct> A/5 21171, PC 17599, STON/O2. 3101282, 113803, 373450, 330877, 17463, 349909, 347742, 237736, PP...
$ Fare        <dbl> 7.2500, 71.2833, 7.9250, 53.1000, 8.0500, 8.4583, 51.8625, 21.0750, 11.1333, 30.0708, 16.7000, 2...
$ Cabin       <fct> , C85, , C123, , , E46, , , , G6, C103, , , , , , , , , , D56, , A6, , , , C23 C25 C27, , , , B7...
$ Embarked    <fct> S, C, S, S, S, Q, S, S, S, C, S, S, S, S, S, S, Q, S, S, C, S, S, Q, S, S, S, C, S, Q, S, C, C, ...
```

Feature engineering is usually more important than the machine learning algorithms applied. Some features like Cabin contain too many missing values and should be dropped. Other features can be added, like the Title (Mr, Ms) from the name, or the Fsize (family size) taking into account the SibSp (siblings/spouses), Parch (parents/children) and Ticket (number of duplicate tickets). Any missing values in the observations must be imputed.

As a baseline, since most people died, a naive algorithm could predict that all people died and would achieve 62% accuracy on the train set. A smarter baseline model would predict that all men died and women survived, which would result in 79% accuracy. A machine learning algorithm should recognize patterns in the data and have better accuracy. Cross-validation (CV) is useful to prevent models from overfitting (achieving high train accuracy but low test accuracy).

I've tried logistic regression, random forest, gradient boosting, however the algorithm that achieved best results in Kaggle was a simple tree classifier with 83% CV accuracy and 80% accuracy in the test set. The smart baseline would result in 77% test accuracy, so a small improvement was made.

{% asset_img "tree.png" "Decision tree" %}

With this decision tree, the Title, Pclass, Fsize, Sex and Embarked are the features used to decide the fate of a passenger. A simple visualization of the train data shows hints on why this tree was made.

{% asset_img "titanic-data.png" "Titanic data" %}

The tree is very interpretable. It captures that on Pclass 3, big families die, as well as Misses who embarked on S. For Pclass 1 and 2 it predicts that children and women live.

## Regression

For this problem the predicted variable, Price, is skewed to the right, so a log transformation is helpful to make the distribution more normal and improve the predictions. The train dataset contains 1460 observations and 81 variables, most of which are useless.

A lot of feature engineering is required to have good results. A few variables have missing values but can be imputed with related variables. Right skewed variables should be transformed using a log function. Encoding a few categorical values as ordinals and removing outliers also improves results. A correlation matrix can be helpful to select the most important features for models that are sensitive to multicollinearity, like linear regression.

{% asset_img "correlation.png" "Correlation matrix" %}

Lasso regression achieved 0.11183 CV RMSE and 0.11911 test RMSE. Elasticnet was close behind, while ridge/linear regression and gradient boosting were worst but still an improvement over a mean baseline.

To improve results I used k-means clustering to create 2 clusters and train a lasso model for each cluster. With this approach, the cluster for each test observation had to be predicted. Performance improved to 0.11897 RMSE.

{% asset_img "clusters.png" "Clusters" %}

With a weighted combination of all models, a final 0.11765 RMSE was reached. To further improve results more feature engineering and hyperparameter tuning would be required.

## Conclusion

The goal of reaching the top 20% on these beginner Kaggle problems was achieved. Full code with detailed comments is available on [github](https://github.com/ruial/kaggle-problems).
