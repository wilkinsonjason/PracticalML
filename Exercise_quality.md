Synopsis
--------

The purpose of this report is to predict the how well exercises were
performed using data collected in relation to personal activity. More
information as to the source of the data can be found here:
<http://groupware.les.inf.puc-rio.br/har> . A machine learning algorith
built through cross validaiton and the random forest method seems most
ideal for this.

    library(caret)

    ## Loading required package: lattice
    ## Loading required package: ggplot2

    set.seed(1)

    filename <- "http://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
    filename2 <- "http://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"
    temp <- tempfile()
    download.file(filename, temp)
    traindf <- read.csv(temp)

    temp2 <- tempfile()
    download.file(filename2, temp2)
    validation <- read.csv(temp2)

Data Cleaning
-------------

First, the data in the training set required a little cleaning, removing
NA columns and sparse variables that would have little impact on the
model, and also timestamp and name variables.

    #NA column removal

    NAcols <- rep(FALSE, ncol(traindf))

    for (i in 1:ncol(traindf)) {
      if( sum(is.na(traindf[,i])) > 500) {
        NAcols[i] <- TRUE
      }
    }
    traindf2 <- traindf[,!NAcols] 

    #near zero variable removal
    nsv <- nearZeroVar(traindf2, saveMetrics = TRUE)
    zerocols <- rep(FALSE, ncol(traindf2))

    for (i in 1:ncol(traindf2)){
      if(nsv[i,4]){
        zerocols[i] <- TRUE
      }
    }
    traindf3 <- traindf2[,!zerocols]

    #remove timestamp, name etc
    traindf4 <- traindf3[,-c(1:5)]

Cross Validation
----------------

    inTrain <- createDataPartition(traindf4$classe, p = .75, list = FALSE)
    training <- traindf4[inTrain,]
    testing <- traindf4[-inTrain,]
    dimtrain <- dim(training)
    dimtest <- dim(testing)

The data supplied has already been divided once into a test set with 20
observations and training set with 19622 observations. Henceforth, the
original test set will be used only as a validation set in the
submission phase and the training set will be divided into a training
set with 14718 observations and a test set with 4904 for proper
crossvalidation and to test the validity and accuracy of our model.

    fitcontrol <- trainControl(method = "cv", number = 3)
    modfit <- train(classe~., data = training, method = "rf", prox = TRUE, trControl = fitcontrol, importance = TRUE)

    ## Loading required package: randomForest
    ## randomForest 4.6-10
    ## Type rfNews() to see new features/changes/bug fixes.

    modfit

    ## Random Forest 
    ## 
    ## 14718 samples
    ##    53 predictors
    ##     5 classes: 'A', 'B', 'C', 'D', 'E' 
    ## 
    ## No pre-processing
    ## Resampling: Cross-Validated (3 fold) 
    ## 
    ## Summary of sample sizes: 9811, 9812, 9813 
    ## 
    ## Resampling results across tuning parameters:
    ## 
    ##   mtry  Accuracy  Kappa  Accuracy SD  Kappa SD
    ##   2     1         1      0.001        0.002   
    ##   30    1         1      4e-04        5e-04   
    ##   50    1         1      0.003        0.003   
    ## 
    ## Accuracy was used to select the optimal model using  the largest value.
    ## The final value used for the model was mtry = 27.

    modfit$finalModel

    ## 
    ## Call:
    ##  randomForest(x = x, y = y, mtry = param$mtry, importance = TRUE,      proximity = TRUE) 
    ##                Type of random forest: classification
    ##                      Number of trees: 500
    ## No. of variables tried at each split: 27
    ## 
    ##         OOB estimate of  error rate: 0.2%
    ## Confusion matrix:
    ##      A    B    C    D    E class.error
    ## A 4183    1    0    0    1   0.0004779
    ## B    6 2839    2    1    0   0.0031601
    ## C    0    3 2563    1    0   0.0015582
    ## D    0    0    9 2402    1   0.0041459
    ## E    0    2    0    2 2702   0.0014782

Using accuracy to select the optimal model, 27 predictors were included,
via the Random Forest ("rf") method, which again uses cross validation
internally though the "cv" method. Random forests are an ensemble
learning method for classification (and regression) that operate by
constructing a multitude of decision trees at training time and
outputting the class that is the mode of the classes output by
individual trees.

    varImpPlot(modfit$finalModel, main = "Predictors Sorted by Importance", pch = 16, cex = .8, col="purple", type = 1, n.var = 27)

![plot of chunk
unnamed-chunk-5](./Exercise_quality_files/figure-markdown_strict/unnamed-chunk-5.png)

Conclusions and Accuracy Prediction
-----------------------------------

The model was then applied to the heldout testing set to get a
prediction for accuracy:

    predic <- predict(modfit, newdata = testing)
    confMat <- confusionMatrix(predic, testing$classe)
    confMat$table

    ##           Reference
    ## Prediction    A    B    C    D    E
    ##          A 1395    1    0    0    0
    ##          B    0  947    1    0    0
    ##          C    0    1  854    2    0
    ##          D    0    0    0  801    2
    ##          E    0    0    0    1  899

    accuracy <- (sum((predic==testing$classe))/dim(testing)[1])*100

    oob <- 100 - accuracy

**The anticipated accuracy is 99.8369 % and out of sample error rate is
0.1631 %.**

As can be seen from the confusion matrices for the training, and testing
sets, the random forest method does an excellent job of anticipating out
of sample error, as they are very similar.
