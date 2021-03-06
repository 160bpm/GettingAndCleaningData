---
title: "READMEGetting and Cleaning Data - Peer Assessment"
author: "Alan Yun"
date: "Sunday, February 15, 2015"
output: html_document
---
#Title: ReadMe
#Author: Alan Yun

##Overview
Purpose of this document is to decribe how the script works.


##Loading and processing the data.
Download dataset.zip file from web to current R working directory. 
Unzip file and load Train and Test Data Set 


```r
library("data.table")
library("reshape2")


wd <- getwd() ##GetWorkingDirectory
if (!file.exists("./data")) {dir.create("./data")}
wd <- paste(wd,"/data", sep="")

##download file
fileUrl <- "http://d396qusza40orc.cloudfront.net/getdata%2Fprojectfiles%2FUCI%20HAR%20Dataset.zip "
fileName <- "dataset.zip"

download.file(fileUrl, file.path(wd, fileName ),mode="wb")

##unzip file
unzip (file.path(wd, fileName), exdir = wd, overwrite=TRUE)

##get unziped file from directory
unzipd <- paste(wd,"/UCI HAR Dataset", sep="")
list.files(unzipd , recursive = TRUE)

##read data
dtTrainSubject <- data.table(read.table(file.path(unzipd, "train", "subject_train.txt")))
dtTrainX <- data.table(read.table(file.path(unzipd, "train", "X_train.txt")))
dtTrainY <- data.table(read.table(file.path(unzipd, "train", "Y_train.txt")))

dtTestSubject <- data.table(read.table(file.path(unzipd, "test", "subject_test.txt")))
dtTestX <- data.table(read.table(file.path(unzipd, "test", "X_test.txt")))
dtTestY <- data.table(read.table(file.path(unzipd, "test", "Y_test.txt")))
```

##Merge Data
Merge Test and Train data. 


```r
##merge data
##Append data
dtSubject <- rbind(dtTrainSubject, dtTestSubject)
dtSet <- rbind(dtTrainX, dtTestX)
dtLabel <- rbind(dtTrainY, dtTestY)

setnames(dtSubject, "V1", "subject")
setnames(dtLabel , "V1", "label")

##Join data
dtAll <- cbind(dtSubject, dtSet, dtLabel)
```

##Extract only the measurements on the mean and standard deviation for each measurement. 
Create subset of dataset with column for mean and standard deviation. 
Filter features using keywords, mean and std, and get only matching feature columns. 


```r
##Read features: Q2
dtFeatures <- data.table(read.table(file.path(unzipd, "features.txt")))
setnames(dtFeatures, names(dtFeatures), c("featureNum", "featureName"))

##filter only mean and std
filter <-"mean\\(\\)|std\\(\\)"
dtFeatures <- dtFeatures[grepl(filter, featureName)]

##add new veriable
dtFeatures$featureCode = dtFeatures[, paste("V", featureNum, sep="")]

##Select only matching columns
dtAll <- dtAll[, c("subject", "label", dtFeatures$featureCode), with = FALSE]
```

##Update activity names and labels. 
Merge activity names and features to replace activity and label. 


```r
##UseDescriptive activity names - Q3 and Q4
dtActivityNames <- data.table(read.table(file.path(unzipd, "activity_labels.txt")))
setnames(dtActivityNames, names(dtActivityNames), c("label", "activityName"))

dtAll <- merge(dtAll, dtActivityNames, by = "label", all.x = TRUE)
dtAll <- data.table(melt(dtAll, id=c("subject", "label", "activityName"), 
                         variable.name = "featureCode"))
dtAll <- merge(dtAll, dtFeatures[, list(featureNum, featureCode, featureName)], 
               by = "featureCode", all.x = TRUE)
```

```
## Error in merge.data.table(dtAll, dtFeatures[, list(featureNum, featureCode, : Elements listed in `by` must be valid column names in x and y
```

##Create tidy data set with the average of each variable for each activity and each subject.
Create subset of data with only necessary columns for tidy data set.  
Calculate mean of each variable for each activity and each subject and reshape table for tidy data set. 


```r
#Calculate mean by aubject, activity, and feature - Q5
dtMean <- dtAll[, mean(value), by = c("subject", "label","activityName", "featureNum","featureName")]
```

```
## Error in eval(expr, envir, enclos): 객체 'featureNum'를 찾을 수 없습니다
```

```r
setnames(dtMean , "V1", "Average")
```

```
## Error in setnames(dtMean, "V1", "Average"): Items of 'old' not found in column names: V1
```

```r
dtMean<- dtMean[order(subject, label, featureNum),]

#subset dataset to have only necessary columns
dtSub <- subset(dtMean, select = c("subject", "activityName", "featureName", "Average"))

library("reshape")
#reshape the dataset and create tidy data set
dtTidy <-cast(dtSub, subject+activityName ~ featureName)
```

```
## Using Average as value column.  Use the value argument to cast to override this choice
```

```r
dtTidy <- na.omit(dtTidy)
```

##Create output file. 
Generate output file in txt file format with out header. 


```r
#output in txt file
write.table(dtTidy, file="tidymeans.txt", quote=FALSE, row.name=FALSE)
```
