Crossfit Data
================

``` r
library(kerasformula)
library(polyreg)
seed <- 12345
```

Below find publically-available data from the Crossfit annual open, an amateur athletics competition consisting of five workouts each year. "Rx", as in "perscription", denotes the heaviest weights and most complex movements. In this analysis, we restrict the predictors to height, weight, age, region, and performance in the first of the five rounds. We also restrict analysis who did all five rounds "Rx" and reported that other data. The analysis is repeated for men and women and 2017 and 2018. In each case, `kerasformula` and `xvalPoly` are compared in terms of mean absolute error. (Note `kms` will standardize the outcome by default and so the test stats are on the same scale).

Men 2018 Competition
====================

``` r
competitions <- c("Men_Rx_2018", "Men_Rx_2017", "Women_Rx_2018", "Women_Rx_2017")
MAE_results <- matrix(nrow=3, ncol=4, 
                      dimnames=list(c("kms", "xvalPoly (best)", "xvalPoly (worst)"), competitions))

for(i in 1:4){
  
  Rx <- read.csv(paste0(competitions[i], ".csv"))
  colnames(Rx) <- gsub("[[:punct:]]", "", colnames(Rx))    # forgetmenot
  colnames(Rx) <- tolower(colnames(Rx))
  colnames(Rx) <- gsub("x18", "open", colnames(Rx))
  colnames(Rx) <- gsub("x17", "open", colnames(Rx))

  Rx_tmp <- dplyr::select(Rx, heightm, weightkg, age, open1percentile, overallpercentile)
  Rx_complete <- Rx_tmp[complete.cases(Rx_tmp), ]
  
  P <- ncol(model.matrix(overallpercentile ~ ., Rx_complete))

  Rx_kms_out <- kms(overallpercentile ~ ., Rx_complete, 
                  layers = list(units = c(P, P, NA), 
                                activation = c("relu", "relu", "linear"), 
                                dropout = c(0.4, 0.3, NA), 
                                use_bias = TRUE, 
                                kernel_initializer = NULL, 
                                kernel_regularizer = "regularizer_l1_l2", 
                                bias_regularizer = "regularizer_l1_l2", 
                                activity_regularizer = "regularizer_l1_l2"),
                  seed=seed, Nepochs=5, 
                  validation_split = 0, pTraining = 0.9, verbose=0)
  
  Rx_complete_01 <- as.data.frame(lapply(Rx_complete, kerasformula:::zero_one))
  xval.out <- xvalPoly(Rx_complete_01, maxDeg = 3, maxInteractDeg = 2)
  Rx_kms_out$MAE_predictions
  MAE_results[1,i] <- Rx_kms_out$MAE_predictions
  MAE_results[2,i] <- min(xval.out)
  MAE_results[3,i] <- max(xval.out)
  
}
```

The results
===========

Out-of-sample mean absolute error (MAE) for `kms` vs. `xvalPoly` (for the latter, the lowest MAE for the models corresponding to the three polynomial degrees is selected).

``` r
MAE_results
```

                     Men_Rx_2018 Men_Rx_2017 Women_Rx_2018 Women_Rx_2017
    kms                0.1158143   0.1369244     0.1213506     0.1580093
    xvalPoly (best)    0.1084508   0.1325944     0.1115333     0.1465626
    xvalPoly (worst)   0.1091021   0.1336209     0.1127202     0.1485722