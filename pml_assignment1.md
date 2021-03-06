# Practical Machine Learning - Assignment 1
Michael Farrell  
January 30, 2016  


#  Human Activity Recognition of Weight Lifting Exercises

## Overview
Using activity trackers and other wearable technology devices that are capable of gathering large amounts of data, relatively inexpensively, has provided a data source for machine learning that can inform us on health improvement and behavioral patterns. The next logical step is to use the data to quantify the quality of exercise.

The quality of executing an activity, the âhow (well)â, has only received little attention so far, even though it potentially provides useful information for a large variety of applications. In this project, the goal is to use data from accelerometers on the belt, forearm, arm, and dumbell of a group of participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways. In this project we use the data collected to predict if the exercise is performed in a correct manner. 

## Data
The data for this project come from the paper "Qualitative Activity Recognition of Weight Lifting Exercises"" [see the paper](http://groupware.les.inf.puc-rio.br/work.jsf?p1=11201). In the paper the authors define the quality of execution and investigate three aspects that pertain to qualitative activity recognition: 

* specifying correct execution
* detecting execution mistakes
* providing feedback on the to the user

The dataset is the record of a set of activity monitor for the six participants as they perform one set of 10 repetitions of the Unilateral Dumbbell Biceps Curl in five different fashions
* Class A - exactly according to the specification
* Class B - throwing the elbows to the front
* Class C - lifting the dumbbell only halfway
* Class D - lowering the dumbbell only halfway
* Class E - throwing the hips to the front (Class E).

All together 39,242 samples with 159 variables are available in the original dataset, but we will use a reduced subset of 19,622 samples.

## Model Development 
Having the data in hand, my process for creating a predictive model became:
1. Feature Selection and Preprocessing
2. Model Function Selection
3. Performance Evaluation

## Feature Selection
The original dataset has 160 variables and some data issues so  will eliminate

1. In the process of loading the data variables with data that is not meaningful is converted to NA. Then the non-numeric grouping and timestamp variables in the first seven columns are eliminated. 

2. Features with null values are removed and then features near zero variance are deleted as well. This features have little information to offer the model and will reduce performance if included.

3. Where there are features a high correlation with other variables, one is eliminated. Highly correlated variables can sometimes reduce the performance of a model, and will be excluded.


```r
# Read in the data performing NA replacment
pmlTrain <- read.csv("./datasets/pml-training.csv", na.strings=c("#DIV/0!"," ", "", "NA", "NAs","NULL"))
# Remove the first 7 variables which contain grouping and timestamp info
featrList <- pmlTrain[,-c(1:7)]
# Remove variables with null values
featrList = featrList[, colSums(is.na(featrList)) == 0]
# Remove features with near zero variance
zvar <- nearZeroVar(featrList[sapply(featrList, is.numeric)], saveMetrics=TRUE)
featrList = featrList[, zvar[, 'nzv'] == 0] 
# Remove features with a high correlation to other features
corMat <- cor(na.omit(featrList[sapply(featrList, is.numeric)]))
corMatDOF <- expand.grid(row = 1:52, col = 1:52)
corMatDOF$correlation <- as.vector(corMat) 
highCor <- findCorrelation(corMat, cutoff = .7, verbose = TRUE)
featrList <- featrList[, -highCor] 
# Now write them out to fix the dataset for our model
write.csv(pmlTrain[,names(featrList)],'pml_red_training.csv', row.names = FALSE)
```

After all of this preoprocessing we were left with this group of features that will be used in the predictive models.

```r
# We are left with these 31 features
names(featrList)
```

```
##  [1] "gyros_belt_x"         "gyros_belt_y"         "gyros_belt_z"        
##  [4] "magnet_belt_x"        "magnet_belt_y"        "roll_arm"            
##  [7] "pitch_arm"            "yaw_arm"              "total_accel_arm"     
## [10] "gyros_arm_y"          "gyros_arm_z"          "magnet_arm_x"        
## [13] "magnet_arm_z"         "roll_dumbbell"        "pitch_dumbbell"      
## [16] "yaw_dumbbell"         "total_accel_dumbbell" "gyros_dumbbell_y"    
## [19] "magnet_dumbbell_z"    "roll_forearm"         "pitch_forearm"       
## [22] "yaw_forearm"          "total_accel_forearm"  "gyros_forearm_x"     
## [25] "gyros_forearm_y"      "accel_forearm_x"      "accel_forearm_z"     
## [28] "magnet_forearm_x"     "magnet_forearm_y"     "magnet_forearm_z"    
## [31] "classe"
```

## Model Training
The data is reloaded into a training and test dataset using a 60%/40% split.  


```r
pmlRed <- read.csv('pml_red_training.csv')
in_train <- createDataPartition(pmlRed$classe, p=0.6, list=FALSE )
training <- pmlRed[in_train,]
testing <- pmlRed[-in_train,]
```

The first model is a Decision Tree model. Decision Trees are simpler models, their attraction lies in the fact that their results are easy to view, understand and to explain. Here cross validation is used by setting up a caret trainControl parameter object which will then perform a 5 fold cross validation during model training.

The results are good with an accuracy 


```r
set.seed(33)
rpFit1 <- rpart(classe ~ .,data=training, method="class", parms=list(split="information"),
                control=rpart.control(usesurrogate = 0, maxsurrogate = 0))
rpPred1 <- predict(rpFit1, testing, type="class")
rpConf <- confusionMatrix(testing$classe, rpPred1)
rpConf$table
```

```
##           Reference
## Prediction    A    B    C    D    E
##          A 1905   72   87  168    0
##          B  420  701  266  131    0
##          C  112   95 1049  112    0
##          D  186  113  154  830    3
##          E  131  361  252  176  522
```

```r
rpConf$overall["Accuracy"]
```

```
##  Accuracy 
## 0.6381596
```
The results are good with an accuracy of 0.6381596 and a confidence interval from 0.6274133 to 0.6488031.

The second model is a Random Forest model. A single decision tree provides a simple model of the world, but for this case it is too simple to be specific. Random Forest builds hundreds of decision trees and with the ensemble is reduces instability, creates a more robust model and delivers greater accuracy.


```r
set.seed(33)
rfCtrl <- trainControl(method="cv", 5)
rfFit1 <- train(classe ~ ., data=training, method="rf", trControl=rfCtrl, ntree=250)
rfFit1$finalModel
```

```
## 
## Call:
##  randomForest(x = x, y = y, ntree = 250, mtry = param$mtry) 
##                Type of random forest: classification
##                      Number of trees: 250
## No. of variables tried at each split: 2
## 
##         OOB estimate of  error rate: 1.58%
## Confusion matrix:
##      A    B    C    D    E class.error
## A 3343    3    1    1    0 0.001493429
## B   38 2223   15    2    1 0.024572181
## C    0   23 2015   16    0 0.018987342
## D    0    0   63 1862    5 0.035233161
## E    0    5    6    7 2147 0.008314088
```

```r
rfPred1 <- predict(rfFit1, testing)
rfConf <- confusionMatrix(testing$classe, rfPred1)
rfConf$table
```

```
##           Reference
## Prediction    A    B    C    D    E
##          A 2224    4    3    1    0
##          B   22 1480   13    0    3
##          C    1   15 1349    3    0
##          D    0    0   33 1251    2
##          E    0    0    4    6 1432
```

```r
rfConf$overall["Accuracy"]
```

```
##  Accuracy 
## 0.9859801
```

```r
# Calculating Out of Bag error
oob_err_rate = function(reference, prediction) {
  sum(prediction != reference)/length(reference)
}
oobErr <- oob_err_rate(testing$classe, rfPred1)
```

The results are better with an accuracy of 0.9859801 and a confidence interval from 0.9831266 to 0.9884636.
 
The out-of-bag (OOB) estimate of the error is calculated using the observations that were not in the "bag" - the "bag" being the subset of training data.

This "unbiased" estimate of error suggests that when the resulting model is applied to new observations, the answer will be in the range of this OOB error.

OOB estimate of error rate: 0.0140199

one of the problems with a random forest, compared with a single decision tree, is that it becomes more difficult to readily understand the discovered knowledge since there are now hundreds of trees to understand.

One way to get an idea of the knowledge discovered is to consider the importance of the variables. This variable importance graph which shows the variables by decreasing importance and using the mean decrease in the Gini coefficient, a measure of how each node contributes to the homogeneity of the nodes and leaves in the resulting random forest.


```r
varImpPlot(rfFit1$finalModel, sort=TRUE, type=2, pch=21, 
           main="Feature Importance")
```

![](pml_assignment1_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

## Performance Evaluation

The Random Forest has the better error rate and so will be used in the evaluation of the final test set. The final test set, pml-testing.csv, and the data is preprocessed in the manner of the original training set.  


```r
# Load the final test dataset
pmlTest <- read.csv("./datasets/pml-testing.csv", na.strings=c("#DIV/0!"," ", "", "NA", "NAs", "NULL"))
# Make the list of 
testNames <- names(featrList)
# To set the test datasets final column as "problem_id"
testNames[31] <- names(pmlTest)[160]
# Write the thinner test file back out
write.csv(pmlTest[,testNames],'pml_red_test.csv', row.names = FALSE)
# This is final test set.
pmlFinal <- read.csv('pml_red_test.csv')
pml_write_files = function(x){
    n = length(x)
    for(i in 1:n){
        filename = paste0("problem_id_",i,".txt")
        write.table(x[i],file=filename,quote=FALSE,row.names=FALSE,col.names=FALSE)
    }
}
# Load the final test set and predict answer
answers <- predict(rfFit1, newdata=pmlFinal)
```






