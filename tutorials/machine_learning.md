Supervised Machine Learning in R
================
Kasper Welbers & Wouter van Atteveldt
2020-01

-   [Introduction](#introduction)
    -   [Packages used](#packages-used)
-   [Obtaining data](#obtaining-data)
-   [Training and test data](#training-and-test-data)
-   [Model 1: a decision tree](#model-1-a-decision-tree)
    -   [Validating the model](#validating-the-model)
-   [Model 2: Support vector machine](#model-2-support-vector-machine)

Introduction
============

Packages used
-------------

In this tutorial, we use the following packages: (and probably some others, feel free to add)

``` r
install.packages(c("tidyverse", "caret", "e1071", "kernlab"))
```

The main library we use is `caret`, which is a high-level library that allows you to train and test many different models using the same interface. This pacakage then calls the underlying libraries to train specific algorithms (such as `kernlab` for kernel-based SVMs).

``` r
library(caret)
```

Obtaining data
==============

For this tutorial, we will use the 'german credit data' dataset:

``` r
library(tidyverse)
d = read_csv("https://www.openml.org/data/get_csv/31/dataset_31_credit-g.arff") 
```

This dataset contains details about past credit applicants, including why they are applying, checking and saving status, home ownership, etc. The last column contains the outcome variable, namely whether this person is actually a good or bad credit risk.

To explore, you can cross-tabulate e.g. home ownership with credit risk:

``` r
table(d$housing, d$class) %>% prop.table(margin = 1)
```

So, home owners have a higher chance of being a good credit risk than people who rent or who live without paying (e.g. with family), but the difference is lower than you might expect (74% vs 60%).

Put bluntly, the goal of the machine learning algorithm is to find out which variables are good indicators of one's credit risk, and build a model to accurately predict credit risk based on a combination of these variables (called features).

Training and test data
======================

The first step in any machine learning venture is to split the data into training data and test (or validation) data, where you need to make sure that the validation set is never used in the model training or selection process.

Fort his, we use the `createDataPartition` function in the `caret` package, although for a simple case like this we could also directly have used the base R `sample` function. See the `createDataPartition` help page for details on other functionality of that function, e.g. creating multiple folds.

``` r
set.seed(99)
train = createDataPartition(y=d$class, p=0.7, list=FALSE)
train.set = d[train,]
test.set = d[-train,]
```

Model 1: a decision tree
========================

The most easily explainable model is probably the decision tree. A decision tree is a series of tests which are run in series, ending in a decision. For example, a simple tree could be: If someone is a house owner, assume good credit. If not, if they have a savings account, they're good credit, but otherwise they are a bad credit risk.

In caret, we can train a decision tree using the `rpart` method. The following code trains a model on the data, prediction species from all other variables (`class ~ .`), using the `train.set` created above.

``` r
library(caret)
tree = train(class ~ ., data=train.set, method="rpart")
```

Unlike many other algorithms, the final model of the decision tree is interpretable:

``` r
tree$finalModel
```

Each numbered line contains one node, starting from the root (all data). The first question is whether someone has `checking_status` 'no checking' (i.e. that person has no checking account). If they indeed do not have a checking account (i.e. `no_checking>=0.5`), you go to node 3, and conclude that they have a good credit risk. If they do have a checking account, you go to node 2, and then look at the duration of the requested loan, etc.

To make it easier to inspect, you can also plot this tree:

``` r
plot(tree$finalModel, uniform=TRUE, main="Classification Tree")
text(tree$finalModel, use.n=TRUE, all=F, cex=.8)
```

It might seem counterintuitive that not having a checking account is a sign of creditworthiness, but apparently that's what the data says, at least in the training data:

``` r
table(train.set$checking_status, train.set$class) %>% prop.table(margin=1)
```

So it turns out that people with `good'  credit risks eather have a full checking account, or don't use a checking account at all; but this implementation of decision trees only looks at one value at a time (in fact, all categorical variables are turned into dummies before the learning starts, hence the cryptic`&gt;=0.5\`).

Validating the model
--------------------

So, how well can this model predict the credit risk? Let's first see how it does on the training set:

``` r
train.pred = predict(tree, newdata = train.set)
acc = sum(train.pred == train.set$class) / nrow(train.set)
print(paste("Accuracy:", acc))
```

So, on the training set it gets 76% accuracy, which is not very good given that just assigning everyone 'good' rating would already get you 70% accuracy. Let's see how it does on the validation set:

``` r
pred = predict(tree, newdata = test.set)
acc = sum(pred == test.set$class) / nrow(test.set)
print(paste("Accuracy:", acc))
```

So, as expected it does even worse on the test set. To see which outcomes are misclassified, you can create a confusion matrix by tabulating the predictions and actual outcomes:

``` r
table(pred, test.set$class)
```

Finally, we can use various functions from the `caret` package to get more information on the performance. The most important one is probably `confusionMatrix`. Note that this expects the output class to be a factor, so we convert the `class` column (which tidyverse imported as a character, possibly not the best choice in this case)

``` r
confusionMatrix(pred, factor(test.set$class), mode="prec_recall")
```

The top displays the confusion matrix, showing that of the actual 'bad' credit risks 23 where correctly predicted, but 67 were wrongly predicted as good. The 'good' category does much better, with 196 correct predictions out of 210. (Note that overpredicting the most common class is a normal problem for some ML algorithms).

Below that it gives a statistical confirmation that 73% is indeed not very good, by showing that it is not significantly higher than guessing (No information).

Finally, the last block gives the common information retrieval metrics assuming `bad` as the reference class (this can be changed by setting `positive='good'` on the call). It has decent prediction (if the model predicts bad, it's correct in 62% of cases), but really bad recall: only 26% of the bad credit risks are identified.

Model 2: Support vector machine
===============================

Let's try another model, this time a support vector machine.

``` r
set.seed(1)
m = train(class ~ ., data=train.set, method="svmRadial")
pred = predict(m, newdata = test.set)
acc = sum(pred == test.set$class) / nrow(test.set)
print(paste("Accuracy:", acc))
```

The SVM, like many ML algorithms, has a number of (hyper)parameters that need to be set, including sigma (the 'reach' of examples, i.e. how many examples are used as support vectors) and C (the tradeoff between increasing the margin and misclassifications).

Although the defaults are generally reasonable, there is no real theoretical reason to choose any value of these parameters. So, the common thing to do is to do a 'grid search', i.e. an exhaustive search of a range of possible values of each parameter, and pick the one that performs best.

Of course, you cannot pick the one that performs best on the validation set (as that would optimise on your dependent variable). So, the normal approach is to use cross-validation on the training set, which means that the training set itself is split. With a 5-fold cross-validation, each model is trained 5 times on 80% of the data and tested on the remaining 20%. This is then repeated 5 times with a different split, until every case has been used in testing once.

The following code uses `caret` to run a crossvalidation (`repeatedcv`) to test different settings of `sigma` and `C`:

``` r
set.seed(1)
paramgrid = expand.grid(sigma = c(.001, .01, .1, .5), C = c(.5, .75, 1, 1.5, 2))
traincontrol <- trainControl(method = "repeatedcv", number = 5,repeats=5,verbose = FALSE)
m = train(class ~ ., data=train.set, method="svmRadial", tuneGrid=paramgrid, trControl=traincontrol, preProc = c("center","scale"))
pred = predict(m, newdata = test.set)
acc = sum(pred == test.set$class) / nrow(test.set)
print(paste("Accuracy:", acc))
```

As an exercise for the reader, is it possible to find an algorithm or setting that does improve performance?
