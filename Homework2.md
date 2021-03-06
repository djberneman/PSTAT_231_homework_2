---
title: "PSTAT 231 Homework 2"
author: "Dylan Berneman"
output: 
  html_document:
    toc: true
    toc_float: true
    latex_engine: 'pdflatex'
    keep_md: true
---



## Linear Regression

For this lab, we will be working with a data set from the UCI (University of California, Irvine) Machine Learning repository ([see website here](http://archive.ics.uci.edu/ml/datasets/Abalone)). The full data set consists of $4,177$ observations of abalone in Tasmania. (Fun fact: [Tasmania](https://en.wikipedia.org/wiki/Tasmania "Tasmania") supplies about $25\%$ of the yearly world abalone harvest.)

![*Fig 1. Inside of an abalone shell.*](abalone_pic.jpg){width="152"}

The age of an abalone is typically determined by cutting the shell open and counting the number of rings with a microscope. The purpose of this data set is to determine whether abalone age (**number of rings + 1.5**) can be accurately predicted using other, easier-to-obtain information about the abalone.

The full abalone data set is located in the `\data` subdirectory. Read it into *R* using `read_csv()`. Take a moment to read through the codebook (`abalone_codebook.txt`) and familiarize yourself with the variable definitions.

Make sure you load the `tidyverse` and `tidymodels`!


### Question 1

Your goal is to predict abalone age, which is calculated as the number of rings plus 1.5. Notice there currently is no `age` variable in the data set. Add `age` to the data set.

Assess and describe the distribution of `age`.

```r
head(abalone)
```

```
## # A tibble: 6 x 9
##   type  longest_shell diameter height whole_weight shucked_weight viscera_weight
##   <chr>         <dbl>    <dbl>  <dbl>        <dbl>          <dbl>          <dbl>
## 1 M             0.455    0.365  0.095        0.514         0.224          0.101 
## 2 M             0.35     0.265  0.09         0.226         0.0995         0.0485
## 3 F             0.53     0.42   0.135        0.677         0.256          0.142 
## 4 M             0.44     0.365  0.125        0.516         0.216          0.114 
## 5 I             0.33     0.255  0.08         0.205         0.0895         0.0395
## 6 I             0.425    0.3    0.095        0.352         0.141          0.0775
## # ... with 2 more variables: shell_weight <dbl>, rings <dbl>
```

```r
abalone['age'] = abalone['rings'] + 1.5
```

### Question 2

Split the abalone data into a training set and a testing set. Use stratified sampling. You should decide on appropriate percentages for splitting the data.


```r
set.seed(91362)
abalone_split <- initial_split(abalone, prop = 0.80, strata = 'age')
abalone_train <- training(abalone_split)
abalone_test <- testing(abalone_split)
```

### Question 3

Using the **training** data, create a recipe predicting the outcome variable, `age`, with all other predictor variables. Note that you should not include `rings` to predict `age`. Explain why you shouldn't use `rings` to predict `age`.

Steps for your recipe:

1.  dummy code any categorical predictors

2.  create interactions between
    -   `type` and `shucked_weight`,
    -   `longest_shell` and `diameter`,
    -   `shucked_weight` and `shell_weight`
3.  center all predictors, and

4.  scale all predictors.

You'll need to investigate the `tidymodels` documentation to find the appropriate step functions to use.

```r
simple_abalone_recipe <- recipe(age ~ type + longest_shell + diameter + height + whole_weight 
                                + shucked_weight + viscera_weight + shell_weight, data = abalone_train)
abalone_recipe <- recipe(age ~ type + longest_shell + diameter + height + whole_weight + shucked_weight
                         + viscera_weight + shell_weight, data = abalone_train) %>% 
                         step_dummy(all_nominal_predictors()) %>% 
                         step_interact(terms = ~ shucked_weight:shell_weight) %>% 
                         step_interact(terms = ~ starts_with('type'):shucked_weight) %>%
                         step_interact(terms = ~longest_shell:diameter)
abalone_recipe = abalone_recipe %>% step_normalize(all_predictors())
```
Answer:
You shouldn't use `rings` to predict `age` because `age` is already a function of `rings`.

### Question 4

Create and store a linear regression object using the `"lm"` engine.

```r
lm_model <- linear_reg() %>% set_engine("lm")
```

### Question 5

Now:

1.  set up an empty workflow,

2.  add the model you created in Question 4, and

3.  add the recipe that you created in Question 3.

```r
lm_wflow <- workflow()
lm_wflow <- lm_wflow %>% add_model(lm_model)
lm_wflow <- lm_wflow %>% add_recipe(abalone_recipe)
```

### Question 6

Use your `fit()` object to predict the age of a hypothetical female abalone with longest_shell = 0.50, diameter = 0.10, height = 0.30, whole_weight = 4, shucked_weight = 1, viscera_weight = 2, shell_weight = 1.

```r
lm_fit <- fit(lm_wflow, abalone_train)
fem_abalone <- data.frame(type = 'F', longest_shell = 0.50, diameter = 0.10, height = 0.30, 
                          whole_weight = 4, shucked_weight = 1, viscera_weight = 2, shell_weight = 1)
predict(lm_fit, fem_abalone)
```

```
## # A tibble: 1 x 1
##   .pred
##   <dbl>
## 1  23.8
```

### Question 7

Now you want to assess your model's performance. To do this, use the `yardstick` package:


1.  Create a metric set that includes *R^2^*, RMSE (root mean squared error), and MAE (mean absolute error).
2.  Use `predict()` and `bind_cols()` to create a tibble of your model's predicted values from the **training data** along with the actual observed ages (these are needed to assess your model's performance).
3.  Finally, apply your metric set to the tibble, report the results, and interpret the *R^2^* value.

```r
abalone_metrics <- metric_set(rmse, rsq, mae)
abalone_train_res <- predict(lm_fit, new_data = abalone_train %>% select(-age))
abalone_train_res <- bind_cols(abalone_train_res, abalone_train %>% select(age))
head(abalone_train_res)
```

```
## # A tibble: 6 x 2
##   .pred   age
##   <dbl> <dbl>
## 1  9.46   8.5
## 2  7.99   8.5
## 3  9.36   9.5
## 4  9.62   8.5
## 5 10.3    8.5
## 6 10.0    9.5
```

```r
abalone_metrics(abalone_train_res, truth = age, estimate = .pred)
```

```
## # A tibble: 3 x 3
##   .metric .estimator .estimate
##   <chr>   <chr>          <dbl>
## 1 rmse    standard       2.13 
## 2 rsq     standard       0.561
## 3 mae     standard       1.53
```

### Required for 231 Students

In lecture, we presented the general bias-variance tradeoff, which takes the form:

$$
E[(y_0 - \hat{f}(x_0))^2]=Var(\hat{f}(x_0))+[Bias(\hat{f}(x_0))]^2+Var(\epsilon)
$$

where the underlying model $Y=f(X)+\epsilon$ satisfies the following:

- $\epsilon$ is a zero-mean random noise term and $X$ is non-random (all randomness in $Y$ comes from $\epsilon$);
- $(x_0, y_0)$ represents a test observation, independent of the training set, drawn from the same model;
- $\hat{f}(.)$ is the estimate of $f$ obtained from the training set.

### Question 8

Which term(s) in the bias-variance tradeoff above represent the reproducible error? Which term(s) represent the irreducible error?

Answer:
The reproducible error terms are $Var(\hat{f}(x_0))$ and $[Bias(\hat{f}(x_0))]^2$.
The irreducible error is $Var(\epsilon)$.

### Question 9

Using the bias-variance tradeoff above, demonstrate that the expected test error is always at least as large as the irreducible error.

$$
E[(y_0 - \hat{f}(x_0))^2]=Var(\hat{f}(x_0))+[Bias(\hat{f}(x_0))]^2+Var(\epsilon)\\
=\mathrm{E}\left[\left(\hat{f}\left(\mathbf{x}_{0}\right)-\mathrm{E} \hat{f}\left(\mathbf{x}_{0}\right)\right)^{2}\right]+\left[\mathrm{E}\left[\hat{f}\left(\mathbf{x}_{0}\right)\right]-f\left(\mathbf{x}_{0}\right)\right]^{2}+\operatorname{Var}(\varepsilon)\\ = 0+0 +Var(\epsilon)\\ where: \ \ \ \ \ \ E[(y_0 - \hat{f}(x_0))^2]\ = expected\ test\ error\\ {Var}(\varepsilon) = irreducible\ error 
$$

### Question 10

Prove the bias-variance tradeoff.

Hints:

- use the definition of $Bias(\hat{f}(x_0))=E[\hat{f}(x_0)]-f(x_0)$;
- reorganize terms in the expected test error by adding and subtracting $E[\hat{f}(x_0)]$


$$
E[(y_0 - \hat{f}(x_0))^2]=Var(\hat{f}(x_0))+[Bias(\hat{f}(x_0))]^2+Var(\epsilon)\\
=\mathrm{E}\left[\left(\hat{f}\left(\mathbf{x}_{0}\right)-\mathrm{E}(\hat{f}\left(\mathbf{x}_{0}\right))\right)^{2}\right]+\left[\mathrm{E}\left[\hat{f}\left(\mathbf{x}_{0}\right)\right]-f\left(\mathbf{x}_{0}\right)\right]^{2}+\operatorname{Var}(\varepsilon)\\ = 0 +Var(\epsilon)\\ \Rightarrow\ Var(\hat{f}(x_0))+[Bias(\hat{f}(x_0))]^2 =\ 0\\\Rightarrow\ Var(\hat{f}(x_0))=-[Bias(\hat{f}(x_0))]^2 \\therefore,\\ [Bias(\hat{f}(x_0))]^2\ \Uparrow\ when\ Var(\hat{f}(x_0))\ {\Downarrow}\\ Var(\hat{f}(x_0))\ \Uparrow\ when\ [Bias(\hat{f}(x_0))]^2\ \Downarrow
$$
