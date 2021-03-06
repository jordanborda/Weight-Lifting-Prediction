---
title: "Weight Lifting Prediction"
date: '2022-04-1'
output: html_document
---

# Introduction

This document describes my apporoach to building a predictive model for the Weight Lifting Exercises Dataset, which can be found [here](http://groupware.les.inf.puc-rio.br/har).
The goal is to fit a predictive model to the provided data in order to predict the kind of weightlifting that was performed.
It turns out that we can fit a random forest model which predicts very well on out of sample data.

# Data retrieval and cleansing

First we retrieve the data for training and validating the model. 
We inspect the data by looking at the output of `str(pml.data)`.
In this output we notice that there are a number of variables which are set to factor, while when we look at the levels, they appear to be numeric.
Closer inspection reveals that:

* Some variables are useless (e.g. `levels(pml.data$kurtosis_yaw_belt)` -> "#DIV/0!")
* Some have actually a wide range of numeric values but also a "#DIV/0!" value.

```{r warning=FALSE, cache=TRUE}
pml.data <- read.csv("data/pml-training.csv", na.strings=c("NA",""))
#str(pml.data)
# inspect levels of factor variables to find those with only useless levels and
# the ones which should be numeric actually.
# Uncomment code below to look at factor variable levels.
#for (colName in colnames(pml.data[ ,sapply(pml.data, class) == "factor"])) {
#  print(colName)
#  print(levels(pml.data[[colName]]))
#}
# Remove useless variables from the data set.
variables <- c(
  "X", "user_name", "raw_timestamp_part_1", "raw_timestamp_part_2",
  "kurtosis_yaw_belt", "skewness_yaw_belt", "amplitude_yaw_belt", "cvtd_timestamp",
  "kurtosis_yaw_dumbbell", "skewness_yaw_dumbbell", "amplitude_yaw_dumbbell",
  "kurtosis_yaw_forearm", "skewness_yaw_forearm", "amplitude_yaw_forearm"
)
pml.data <- pml.data[ , -which(names(pml.data) %in% variables)]
# Convert factor variables which are actually numeric, to numeric variables.
variables <- c(
  "kurtosis_roll_belt", "kurtosis_picth_belt", "skewness_roll_belt",
  "skewness_roll_belt.1", "max_yaw_belt", "min_yaw_belt",
  "kurtosis_roll_arm","kurtosis_picth_arm", "kurtosis_yaw_arm",
  "skewness_roll_arm","skewness_pitch_arm", "skewness_yaw_arm",
  "kurtosis_roll_dumbbell", "kurtosis_picth_dumbbell","skewness_roll_dumbbell",
  "skewness_pitch_dumbbell","max_yaw_dumbbell", "min_yaw_dumbbell",
  "kurtosis_roll_forearm","kurtosis_picth_forearm", "skewness_roll_forearm",
  "skewness_pitch_forearm", "max_yaw_forearm", "min_yaw_forearm"
)
for (variable in variables) {
  pml.data[[variable]] <- as.numeric(as.character(pml.data[[variable]]))
}
```

There are many variables which have lots of NA values.
This makes them less suitable for prediction.
Initially, we will try to build a model using only variables that have no NA values at all.
If this doesn't work out too well we could try to include variables with a relatively low amount of NA values and apply gap filling techniques (e.g. using K-nearest neighbors).

```{r cache=TRUE}
pml.data.naCounts <- colSums(sapply(pml.data, is.na))
pml.data.complete <- pml.data[,pml.data.naCounts == 0]
```

We now have gone from `r ncol(pml.data)` variables to `r ncol(pml.data.complete)` variables.

# Training the model

Next step is to train the model.
To follow the cross-validation practices the data is split into a training and testing set.
As recommended in the course material, I choose the 60/40 split for training/testing.

```{r cache=TRUE}
library(caret)
set.seed(20052015) # Set a seed for reproducibility
# Split the training set in two parts: 60/40. The first part will be used to
# train the model, and the second part will be used for validation of the model.
inTrain <- createDataPartition(pml.data.complete$classe, p=0.6, list = FALSE)
pml.data.train <- pml.data.complete[inTrain,]
pml.data.test <- pml.data.complete[-inTrain,]
```

Before we fit the model, we check if the training set contains numeric variables which have no or close to zero variance.

```{r cache=TRUE}
nearZeroVar(pml.data.train[, sapply(pml.data.train, is.numeric)])
```

There are no numeric variables with near zero variance, so we cannot reduce the number of variables this way.
Therefore we continue by building the model, using all the remaining variables in the training data set.

```{r cache=TRUE}
library(caret)
modelFit <- train(classe ~ ., data = pml.data.train, method="rf", importance=TRUE)
```

# Verify the model

```{r cache=TRUE}
# Now test the model with the test data.
predictionResults <- confusionMatrix(pml.data.test$classe, predict(modelFit, pml.data.test))
predictionResults$table
```

The total number of predictions is: `nrow(pml.data.test)`: `r nrow(pml.data.test)`.
This equals to the the sum of all items in the prediction matrix `sum(predictionResults$table)`: `r sum(predictionResults$table)`.
Only the diagonal represent counts of correct predicted results.
As the predicted results were generated with data which was not used for testing, we can calculate the out-of-sample error rate by substracting the sum of the diagonal from the toal: `sum(predictionResults$table) - sum(diag(predictionResults$table))`: `r sum(predictionResults$table) - sum(diag(predictionResults$table))` out of `r sum(predictionResults$table)` (or `r (sum(predictionResults$table) - sum(diag(predictionResults$table))) / sum(predictionResults$table) * 100`%).
In other words, the model succeeded quite well in predicting the activities on the training data set.


# Predict "new" data

Now we have a model that seems to perform well, we use it to predict the verification data.

```{r cache=TRUE}
pml.verification <- read.csv("data/pml-testing.csv")
pml.verification <- pml.verification[, colnames(pml.verification) %in% colnames(pml.data.train)]
predict(modelFit, pml.verification)
```
![download](https://user-images.githubusercontent.com/102541316/161299431-cae18d62-3732-4529-b9c3-2193c6649772.png)

# Appendix: 

We have used `ncol pml.data.train` variables.
When we look at the variable importances in our model, we see a rather quick drop off.

```{r cache=TRUE}
variable.importances <- varImp(modelFit)
plot(variable.importances)
```

So let's see if we can train a model with a lower number of variables with similar precision.
To this end we get the row sums of the variable importances and normalize the values.

```{r cache=TRUE}
variable.importances.sums <- sort(rowSums(variable.importances$importance), decreasing = T)
variable.importances.sums <- variable.importances.sums / sum(variable.importances.sums)
```
If we take the first 16 variables we have captured about 50 percent of the overall importances:

```{r cache=TRUE}
sum(variable.importances.sums[1:16])
new.train.vars <- c("classe", names(variable.importances.sums[1:16]))
new.train.data <- pml.data.train[, colnames(pml.data.train) %in% new.train.vars]
new.test.data <- pml.data.test[, colnames(pml.data.train) %in% new.train.vars]
```

```{r cache=TRUE}
modelFit.reduced <- train(classe ~ ., data = new.train.data, method="rf", importance=TRUE)
```

With the new model, we predict again the test data:

```{r cache=TRUE}
confusionMatrix(new.test.data$classe, predict(modelFit.reduced, new.test.data))
```
We see that we get about as good results as with the initial model.
Training this model took about 30 minutes as compared to ~1.5 hours for the initial model.
