---
title: "Flood Detection Analysis"
author: "The Indo Guys (and a Gal)"
date: "August 31, 2018"
output: 
  ioslides_presentation: 
    smaller: yes
    transition: slower
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
library(caret)
set.seed(2)
```

## Background


Jakarta flood has become somekind of a tradition, where flood can happen yearly and can go for over 2 weeks on every occurence
Research found that there are a few reasons for the flood: 
- high precipitation
- humidity level
- tide level
- land subsidence

## Problem Statement {.flexbox .vcenter}

Can we predict if flooding is going to happen given the weather data H-1 and H-2 prior of the event itself?

## Expected Result {.flexbox .vcenter}

To confirm if there is statistical significance that we are able predict the flood before it happens, based on the weather data

## Data Sources

https://data.go.id/dataset/curah-hujan-dan-hari-hujan-menurut-bulan-dki-jakarta
https://bpbd.jakarta.go.id/waterlevel/
http://data.jakarta.go.id/group/lingkungan-hidup

## Basic data from NOAA {.flexbox .vcenter}


## Data source from NOAA


Data source sample
```{r}
banjir_data <- read.csv("data/CLEAN DATASET BANJIR.csv", stringsAsFactors = FALSE)
head(banjir_data)
```

## Data source explanation
Breakdown of the data structure

CODE | Description
-----|-----:
STN | Station number for the location.
YEARMODA | The year, month and day.
TEMP | Mean temperature for the day in degrees Fahrenheit to tenths.
DEWP | Mean dew point for the day in degrees Fahrenheit to tenths.
SLP | Mean sea level pressure for the day in millibars to tenths.


CODE | Description
-----|-----:
STP | Mean station pressure for the day in millibars to tenths.
VISIB | Mean visibility for the day in miles to tenths.     
WDSP | Mean wind speed for the day in knots to tenths.
MXSPD | Maximum sustained wind speed reported for the day in knots to tenths.
GUST | Maximum wind gust reported for the day in knots to tenths.

CODE | Description
-----|-----:
MAX | Maximum temperature reported during the day in Fahrenheit to tenths--time of max temp report varies by country and region, so this will sometimes not be the max for the calendar day.
MIN | Minimum temperature reported during the day in Fahrenheit to tenths--time of min  temp report varies by country and region, so this will sometimes not be the min for the calendar day. 
PRCP | Total precipitation (rain and/or melted snow) reported during the day in inches and hundredths.
SNDP | Snow depth in inches to tenths--last report for the day if reported more than once.
FRSHTT | Indicators for the occurrence during the day of: Fog ('F' - 1st digit). Rain or Drizzle ('R' - 2nd digit). Snow or Ice Pellets ('S' - 3rd digit). Hail ('H' - 4th digit). Thunder ('T' - 5th digit). Tornado or Funnel Cloud ('T' - 6th digit).



```{r, echo=FALSE}
banjir_data$PREDH1 <- factor(banjir_data$PREDH1, levels = c("0", "1"), labels = c("Normal", "Banjir"))
banjir_data$PREDH2 <- factor(banjir_data$PREDH2, levels = c("0", "1", "2", "3"), labels = c("Normal", "H-2", "H-1", "Banjir"))

banjir_data$MXSPD <- as.numeric(banjir_data$MXSPD)
banjir_data$PRCP <- as.numeric(banjir_data$PRCP)
```

## ETL the data
```{r}
summary(banjir_data)
```

Our source of data consist of banjir (Flood) and Normal (no flooding)
```{r}
table(banjir_data$PREDH1)
```

## Data preparation

We remove unnecessary data
```{r}
banjir_data <- subset(banjir_data, select = -c(STN, YEARMODA, PREDH2, FOG, SNOW, HAIL))
```

Also, we randomize from time series to avoid uneven spread data
```{r}

banjir_data <- banjir_data[sample(nrow(banjir_data)),]

```



```{r, echo=FALSE}
# Creating a normalize() function, which takes a vector x and for each value in that vector, subtracts the minimum value in x and divides by the range of x

normalize <- function(x){
  return ( 
    (x - min(x))/(max(x) - min(x)) 
           )
}
```


We split the data into training data (80%) and test data (20%)
```{r}
#banjir_data_n <- as.data.frame(lapply(banjir_data[,1:13], normalize))

#banjir_data_x <- merge(banjir_data_n, banjir_data$PREDH1, by="")
# n0_var <- nearZeroVar(banjir_data[,1:14])
# banjir_data <- banjir_data[,-n0_var]

banjir_data_intrain <- sample(nrow(banjir_data), nrow(banjir_data)*0.8)
banjir_data_train <- banjir_data[banjir_data_intrain, ]
banjir_data_test <- banjir_data[-banjir_data_intrain, ]

```
## The data ratio between the train data and test data

```{r}
prop.table(table(banjir_data_train$PREDH1))

prop.table(table(banjir_data_test$PREDH1))
```
Using set.seed(1), we see even spread of Normal and Banjir for both of data

## Random forest function

Here we use RepeatedCV
```{r}

rdsdata1 <- "data/randomforest1.rds"

# conditional. Remove rdsdata file to run this logic
if (!file.exists(rdsdata1)) {
   
  # Manual Search
  mtry_base <- sqrt(ncol(banjir_data_train))
  seed <- 9
  metric <- "Accuracy"
  tunegrid <- expand.grid(.mtry=c(mtry_base ^ 0, mtry_base ^ 1, mtry_base * 2, mtry_base ^ 2))
  modellist1 <- list()
  repeats <- 10
  method <- "repeatedcv"
  
  for (ntree in c(25, 50, 100, 200, 500, 1000)) {
    control <- trainControl(method=method, number=10, repeats=repeats)
  	banjir_forest_ntree <- train(PREDH1~., data=banjir_data_train, method="rf", metric=metric, tuneGrid=tunegrid, trControl=control, ntree=ntree)
  	key <- toString(ntree)
  	modellist1[[key]] <- banjir_forest_ntree
  }
  saveRDS(modellist1, file = rdsdata1)

} else {
  modellist1 <-readRDS(rdsdata1)
}


#ctrl <- trainControl(method="repeatedcv", number=5, repeats=3)
#banjir_data_forest <- train(PREDH1 ~ ., data=banjir_data_train, method="rf", trControl = ctrl)
```

```{r}
results <- resamples(modellist1)

summary(results)
```


```{r}
dotplot(results)

```

```{r}
banjir_data_forest <- modellist1$`1000`
```



## Random forest

Result of the random forest function
```{r}
banjir_data_forest
```


## Prediction based on the formula

We try to see the result of the prediction
```{r}
banjir_rf <- table(predict(banjir_data_forest, banjir_data_test[,-14]), banjir_data_test[,14])

banjir_rf
```


Accuracy is:
```{r}
(banjir_rf["Normal", "Normal"] + banjir_rf["Banjir", "Banjir"]) / sum(banjir_rf)
```

## What if we add Water Gate

We found data of water level in the Katulampa water gate

```{r}
banjir_data <- read.csv("data/CLEAN DATASET BANJIR2.csv", stringsAsFactors = FALSE)
```

```{r, echo=FALSE}
banjir_data$PREDH1 <- factor(banjir_data$PREDH1, levels = c("0", "1"), labels = c("Normal", "Banjir"))
banjir_data$PREDH2 <- factor(banjir_data$PREDH2, levels = c("0", "1", "2", "3"), labels = c("Normal", "H-2", "H-1", "Banjir"))

banjir_data$MXSPD <- as.numeric(banjir_data$MXSPD)
banjir_data$PRCP <- as.numeric(banjir_data$PRCP)
```

## ETL the data
```{r}
summary(banjir_data)
```

Our source of data consist of banjir (Flood) and Normal (no flooding)
```{r}
table(banjir_data$PREDH1)
```

## Data preparation

We remove unnecessary data
```{r}
banjir_data <- subset(banjir_data, select = -c(STN, YEARMODA, PREDH2, FOG, SNOW, HAIL))
```

Also, we randomize from time series to avoid uneven spread data
```{r}

banjir_data <- banjir_data[sample(nrow(banjir_data)),]

```



```{r, echo=FALSE}
# Creating a normalize() function, which takes a vector x and for each value in that vector, subtracts the minimum value in x and divides by the range of x

normalize <- function(x){
  return ( 
    (x - min(x))/(max(x) - min(x)) 
           )
}
```


We split the data into training data (80%) and test data (20%)
```{r}
#banjir_data_n <- as.data.frame(lapply(banjir_data[,1:13], normalize))

#banjir_data_x <- merge(banjir_data_n, banjir_data$PREDH1, by="")
# n0_var <- nearZeroVar(banjir_data[,1:14])
# banjir_data <- banjir_data[,-n0_var]

banjir_data_intrain <- sample(nrow(banjir_data), nrow(banjir_data)*0.8)
banjir_data_train <- banjir_data[banjir_data_intrain, ]
banjir_data_test <- banjir_data[-banjir_data_intrain, ]

```
## The data ratio between the train data and test data

```{r}
prop.table(table(banjir_data_train$PREDH1))

prop.table(table(banjir_data_test$PREDH1))
```
Using set.seed(1), we see even spread of Normal and Banjir for both of data

## Random forest function

Here we use RepeatedCV
```{r}

rdsdata2 <- "data/randomforest2.rds"

# conditional. Remove rdsdata file to run this logic
if (!file.exists(rdsdata2)) {
   
  # Manual Search
  mtry_base <- sqrt(ncol(banjir_data_train))
  seed <- 9
  metric <- "Accuracy"
  tunegrid <- expand.grid(.mtry=c(mtry_base ^ 0, mtry_base ^ 1, mtry_base * 2, mtry_base ^ 2))
  modellist2 <- list()
  repeats <- 10
  method <- "repeatedcv"
  
  for (ntree in c(25, 50, 100, 200, 500, 1000)) {
    control <- trainControl(method=method, number=10, repeats=repeats)
  	banjir_forest_ntree <- train(PREDH1~., data=banjir_data_train, method="rf", metric=metric, tuneGrid=tunegrid, trControl=control, ntree=ntree)
  	key <- toString(ntree)
  	modellist2[[key]] <- banjir_forest_ntree
  }
  saveRDS(modellist2, file = rdsdata2)

} else {
  modellist2 <-readRDS(rdsdata2)
}


#ctrl <- trainControl(method="repeatedcv", number=5, repeats=3)
#banjir_data_forest <- train(PREDH1 ~ ., data=banjir_data_train, method="rf", trControl = ctrl)
```

```{r}
results <- resamples(modellist2)

summary(results)
```


```{r}
dotplot(results)

```

We decided to choose 200 tree instead of 100 tree because the Kappa performance is really well

Kappa: https://stats.stackexchange.com/a/82187
```{r}
banjir_data_forest <- modellist2$`200`
```

## Random forest

Result of the random forest function
```{r}
banjir_data_forest
```


## Prediction based on the formula

We try to see the result of the prediction
```{r}
banjir_rf <- table(predict(banjir_data_forest, banjir_data_test[,-15]), banjir_data_test[,15])
table(banjir_data_test$PREDH1)
banjir_rf
```


Accuracy is:
```{r}
(banjir_rf["Normal", "Normal"] + banjir_rf["Banjir", "Banjir"]) / sum(banjir_rf)
```

```{r}

```


