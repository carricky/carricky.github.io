---
title: 'Notes on correlation'
layout: post
category: statistics
---

# Notes on correlation
## Terms
Pearson correlation coefficient, coefficient of determination

**Pearson correlation coefficient**
For a population $ \rho _{X,Y}={\frac {\operatorname {cov} (X,Y)}{\sigma _{X}\sigma _{Y}}} $

For a sample ${\displaystyle r={\frac {\sum _{i=1}^{n}(x_{i}-{\bar {x}})(y_{i}-{\bar {y}})}{{\sqrt {\sum _{i=1}^{n}(x_{i}-{\bar {x}})^{2}}}{\sqrt {\sum _{i=1}^{n}(y_{i}-{\bar {y}})^{2}}}}}}$

**coefficient of determination ($R^2$)**
> Definition: the proportion of the variance in the dependent variable that is predictable from the independent variable(s).

$ R^{2}\equiv 1-{SS_{\rm {res}} \over SS_{\rm {tot}}}.\ $

When we test the models performance, $R^2$ measures the size of the residuals from the model compared to the size of the residuals from a null model (mean value). On the training data, as the model cannot be worse than the null model, $R^2$ will always be positive. It may not be the case for the testing data as the model can do worse than the null model.

## some notes
Things above are some basics that we all know. However, recently colleagues of mine proposes that our traditional approach of evaluating our prediction model's performance is problematic. 

Previously, when we want to evaluate our model, we will first use cross-validation to generate predicted values and then calculate its Pearson correlation ($r$) between predicted vs observed values. However, when we do this, $r$ reports the degree to which the predictions are *correlated* with observations, this is not the same as $R^2$ where it represents the *accuracy* of the model. 

In the training set, when we regress observations ($\bar{y}$) on the predicted values ($\hat{y}$), the model will simply be $y=\hat{y}$ as the regression model already assures to minimize residual sum of squares (SSR) and $\hat{y}$ is already the linear combination of predictors. So there is no additional *fitting* when we evaluate the model on training data. $r$ will just be the square root of $R^2$.

However, in the testing data, when we calculate $r$, we are actually fitting a secondary model $y=a+b\hat{y}$, and this will result in a higher $R^2$ than we can get from directly apply the definition equation of $R^2$. Some test set observations have gone to the secondary model which inflates the result. So reporting this secondary $R^2$ may not give you what you think you want.

All of this is tricky and if we want to say that $R^2$ is the explained variance of the model, we have to make sure that "the model" is the one we trained, not the nontrivial secondary model.

 