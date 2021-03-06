### <p align="center"> Monotonic Optimal Binning (MOB) </p>
### <p align="center">  for Consumer Credit Risk Scorecard Development </p>

The **MOB** (Monotonic Optimal Binning) package is a collection of R functions that would generate the monotonic binning and perform the WoE (Weight of Evidence) transformation used in consumer credit scorecard developments. Being a piecewise constant transformation in the context of logistic regressions, the WoE has also been employed in other use cases, such as consumer credit loss estimation, prepayment, and even fraud detection models.  

In addition to monotonic binning and WoE transformation, Information Value and KS statistic of each independent variables are also calculated to evaluate the variable predictiveness. 

In order to be useful in a production environment, the package also provides functionalities that would allow users to do batch processing of binning and transformations with many independent variables concurrently in a multi-core computing engine. 


#### Package Dependencies
R version 3.5, base, stats, parallel, Hmisc, gbm

#### Installation
Download the [mob.R](https://raw.githubusercontent.com/statcompute/MonotonicBinning/master/code/mob.R) file, save it in your computer, and then run following R scripts.
( Thanks to the suggestion provided by [Vlad Perervenko](https://github.com/VladPerervenko) ). 
```r
install.packages("import")

import::from("mob.R", qtl_bin) 

str(qtl_bin)
# function (data, y, x)
```


#### Features
* #### Monotonic Binning & Variable Predictiveness

The function **manual_bin()** is the building block of all monotonic binning algorithms included in the package and will generate the binning outcome based upon a list of cutting points for each numeric attribute. If there is any missing value in the attribute, two different treatments should be considered, depending on the ratio between good and bad accounts. First of all, with only good or only bad accounts in missing values, all cases should be merged into a category with the similar bad rate. Secondly, with both good and bad acounts in missing values, all cases should be treated as a standalone category. In the example shown below, there are 213 cases with missing values in the attribute named "tot_derog" with 70 bad and the rest good accounts. Therefore, all 213 cases are assigned into a separate category with the 32.86% bad rate and the value of WoE = 0.6416. 

```r
manual_bin(df, "bad", "tot_derog", c(1, 2))
#   bin                           rule freq   dist mv_cnt bad_freq bad_rate     woe     iv      ks
#    00                      is.na($X)  213 0.0365    213       70   0.3286  0.6416 0.0178  2.7716
#    01                        $X <= 1 3741 0.6409      0      560   0.1497 -0.3811 0.0828 18.9469
#    02               $X > 1 & $X <= 2  478 0.0819      0      121   0.2531  0.2740 0.0066 16.5222
#    03                         $X > 2 1405 0.2407      0      445   0.3167  0.5871 0.0970  0.0000

```

Currently, there are four monotonic binning algorithms supported in the **MOB** package. 

1. The function **qtl_bin()** implements the iterative discretization based on quantiles across the whole data sample. In each step, a convergence criterion of the Spearman Correlation Coefficient equal to 1 or -1 would be evaluated. 
```r
qtl_bin(df, bad, tot_derog)
# $df
#  bin                           rule freq   dist mv_cnt bad_freq bad_rate     woe     iv      ks
#   00                      is.na($X)  213 0.0365    213       70   0.3286  0.6416 0.0178  2.7716
#   01                        $X <= 1 3741 0.6409      0      560   0.1497 -0.3811 0.0828 18.9469
#   02               $X > 1 & $X <= 2  478 0.0819      0      121   0.2531  0.2740 0.0066 16.5222
#   03               $X > 2 & $X <= 4  587 0.1006      0      176   0.2998  0.5078 0.0298 10.6623
#   04                         $X > 4  818 0.1401      0      269   0.3289  0.6426 0.0685  0.0000
# $cuts
# [1] 1 2 4
```

2. The function **bad_bin()** is a two-step process and implements a revised version of the iterative discretization. The discretization is determined by the data sample with Y = 1 and the discretization outcome is evaluated across the whole data sample based upon the abovementioned convergence criterion. Compared with **qtl_bin()**, **bad_bin()** would tend to generate a more coarse binning outcome and therefore lower statistics of KS or IV. 
```r
bad_bin(df, bad, tot_derog)
# $df
#  bin                           rule freq   dist mv_cnt bad_freq bad_rate     woe     iv      ks
#   00                      is.na($X)  213 0.0365    213       70   0.3286  0.6416 0.0178  2.7716
#   01                        $X <= 2 4219 0.7228      0      681   0.1614 -0.2918 0.0563 16.5222
#   02               $X > 2 & $X <= 4  587 0.1006      0      176   0.2998  0.5078 0.0298 10.6623
#   03                         $X > 4  818 0.1401      0      269   0.3289  0.6426 0.0685  0.0000
# $cuts
# [1] 2 4
```

3. The function **iso_bin()** is driven by the isotonic regression and therefore doesn't require the iteration. As a result, the process is much faster and a monotonic outcome is guaranteed. In addition, given a more granular binning outcome, **iso_bin()** is able to deliver results with higher KS and IV. 
```r
iso_bin(df, bad, tot_derog)
# $df
#  bin                           rule freq   dist mv_cnt bad_freq bad_rate     woe     iv      ks
#   00                      is.na($X)  213 0.0365    213       70   0.3286  0.6416 0.0178  2.7716
#   01                        $X <= 0 2850 0.4883      0      367   0.1288 -0.5559 0.1268 20.0442
#   02               $X > 0 & $X <= 1  891 0.1526      0      193   0.2166  0.0704 0.0008 18.9469
#   03               $X > 1 & $X <= 2  478 0.0819      0      121   0.2531  0.2740 0.0066 16.5222
#   04               $X > 2 & $X <= 3  332 0.0569      0       86   0.2590  0.3050 0.0058 14.6321
#   05              $X > 3 & $X <= 23 1064 0.1823      0      353   0.3318  0.6557 0.0931  0.4370
#   06                        $X > 23    9 0.0015      0        6   0.6667  2.0491 0.0090  0.0000
# $cuts
# [1]  0  1  2  3 23
```

4. The function **gbm_bin()** is driven by the generalized boosted modeling with the monotone restriction. Similar to **iso_bin()**, it also would generate more granular binning outcomes with higher KS and IV statistics, albeit a longer computing time due to the boosting process. 
```r
gbm_bin(df, bad, tot_derog)
# $df
#  bin                           rule freq   dist mv_cnt bad_freq bad_rate     woe     iv      ks
#   00                      is.na($X)  213 0.0365    213       70   0.3286  0.6416 0.0178  2.7716
#   01                        $X <= 0 2850 0.4883      0      367   0.1288 -0.5559 0.1268 20.0442
#   02               $X > 0 & $X <= 1  891 0.1526      0      193   0.2166  0.0704 0.0008 18.9469
#   03               $X > 1 & $X <= 2  478 0.0819      0      121   0.2531  0.2740 0.0066 16.5222
#   04               $X > 2 & $X <= 3  332 0.0569      0       86   0.2590  0.3050 0.0058 14.6321
#   05               $X > 3 & $X <= 9  848 0.1453      0      282   0.3325  0.6593 0.0750  3.2492
#   06              $X > 9 & $X <= 14  166 0.0284      0       56   0.3373  0.6808 0.0157  0.9371
#   07                        $X > 14   59 0.0101      0       21   0.3559  0.7629 0.0071  0.0000
# $cuts
# [1]  0  1  2  3  9 14
```

As previously demonstrated, above 4 functions can apply the binning algorithm to 1 independent variable at a time. However, in a real-world model development environment, we often need to apply the binning algorithm to hundreds or even thousands of independent variables. As a result, we also want a wrapper around each binning algorithm that allows us to run through many variables at once and, more importantly, that distributes the computational task across multiple CPUs in a contemporary computing server or workstation.

The function **batch_bin()** takes two input parameters, "data" for a data frame with the last column as the dependent variable and "method" for an integer indicating the binning algorithm to be called for. For instance, **batch_bin(df, 1)** will apply the **qtl_bin()** function to all (numeric) independent varaibles in a data frame "df" and generates a summary output as below.
```r
#|var            |  nbin|  unique|  miss|  min|   median|      max|       ks|      iv|
#|:--------------|-----:|-------:|-----:|----:|--------:|--------:|--------:|-------:|
#|tot_derog      |     5|      29|   213|    0|      0.0|       32|  18.9469|  0.2055|
#|tot_tr         |     5|      67|   213|    0|     16.0|       77|  15.7052|  0.1302|
#|age_oldest_tr  |    10|     460|   216|    1|    137.0|      588|  19.9821|  0.2539|
#|tot_open_tr    |     3|      26|  1416|    0|      5.0|       26|   6.7157|  0.0240|
#|tot_rev_tr     |     3|      21|   636|    0|      3.0|       24|   9.0104|  0.0717|
#|tot_rev_debt   |     3|    3880|   477|    0|   3009.5|    96260|   8.5102|  0.0627|
#|tot_rev_line   |     9|    3617|   477|    0|  10573.0|   205395|  26.4924|  0.4077|
#|rev_util       |     2|     101|     0|    0|     30.0|      100|  15.1570|  0.0930|
#|bureau_score   |    12|     315|   315|  443|    692.5|      848|  34.8028|  0.7785|
#|ltv            |     7|     145|     1|    0|    100.0|      176|  15.6254|  0.1538|
#|tot_income     |     4|    1639|     5|    0|   3400.0|  8147167|   9.1526|  0.0500|
```

If we are specifically interested in the detailed binning outcome of the attribute "tot_income", then **batch_bin(df, 1)$BinLst[["tot_income"]]** would deliver the result as below, which is the identical output from **qtl_bin(df, bad, tot_income)**.
```r
# $df
#  bin                           rule freq   dist mv_cnt bad_freq bad_rate     woe     iv     ks
#   00                      is.na($X)    5 0.0009      5        1   0.2000 -0.0303 0.0000 0.0026
#   01                     $X <= 2570 1947 0.3336      0      486   0.2496  0.2553 0.0234 9.1526
#   02         $X > 2570 & $X <= 4510 1995 0.3418      0      406   0.2035 -0.0086 0.0000 8.8608
#   03                      $X > 4510 1890 0.3238      0      303   0.1603 -0.2999 0.0266 0.0000
# $cuts
# [1] 2570 4510
```


* #### Binning Deployment & WoE Transformation

After the generation of monotononic binning, it is by all means important to apply these binning outcomes to new datasets that should be used in the model development or deployment. Currently, there are two functions to support the deployment of binning outcomes, including **cal_woe(data, xname, spec)** and **batch_woe(data, slst)**. 

1. The function **cal_woe()** takes three input parameters. The `data` parameter is a data frame that we would deploy binning outcomes. The `xname` parameter is the name of a numeric variable to which the WoE transformation should be applied. The `spec` parameter is the binning specification table generated by each binning function, e.g. **qtl_bin(...)$df** or **gbm_bin(...)$df**. Alternatively, users can also manually create such a specification table based on the identical meta data. 
There are two components in the output of **cal_woe()**, including a data frame with the PSI (Population Stability Index) summary and another data frame that is the copy of the input data frame with an additional column for the WoE transformation. 

```r
ltv_bin <- qtl_bin(df, bad, ltv)

ltv_woe <- cal_woe(df[sample(seq(nrow(df)), 1000, replace = T), ], "ltv", ltv_bin$df)

str(ltv_woe, max.level = 1)
# List of 2
#  $ df :'data.frame':	1000 obs. of  13 variables:
#  $ psi:'data.frame':	7 obs. of  8 variables:
#  - attr(*, "class")= chr "psi"

ltv_woe
#  bin                           rule   dist     woe cal_freq cal_dist cal_woe    psi
#   01                       $X <= 84 0.1638 -0.7690      177    0.177 -0.7690 0.0010
#   02             $X > 84 & $X <= 93 0.1645 -0.3951      143    0.143 -0.3951 0.0030
#   03             $X > 93 & $X <= 99 0.1501  0.0518      154    0.154  0.0518 0.0001
#   04            $X > 99 & $X <= 103 0.1407  0.0787      125    0.125  0.0787 0.0019
#   05           $X > 103 & $X <= 109 0.1324  0.1492      149    0.149  0.1492 0.0020
#   06           $X > 109 & $X <= 117 0.1237  0.3263      133    0.133  0.3263 0.0007
#   07           $X > 117 | is.na($X) 0.1249  0.5041      119    0.119  0.5041 0.0003

head(ltv_woe$df, 1)
# ... bureau_score ltv tot_income bad woe.ltv
# ...          667  83       2500   1  -0.769
```

2. The function **batch_woe()** basically is the wrapper around **cal_woe()** with the purpose to allow users to apply WoE transformations to many independent variables simultaneously. The `data` parameter is the data frame that we would deploy binning outcomes and the `slst` parameter is the list of multiple binning specification tables that is either the direct output from the function **batch_bin** or created manually by combining outputs from multiple binning functions. 
Simlarly, there are also two components in the output of **batch_woe()**, a list of PSI tables for transformed variables and a data frame with a row index and all transformed variables. The default printout is a PSI summation of all input variables to be transformed. As shown below, all PSI values are below 0.1 and therefore none is concerning. 
  
```r
binout <- batch_bin(df, 1)

woeout <- batch_woe(df[sample(seq(nrow(df)), 2000, replace = T), ], binout$BinLst)

woeout 
#     tot_derog tot_tr age_oldest_tr tot_open_tr tot_rev_tr tot_rev_debt ...
# psi    0.0027 0.0044        0.0144      0.0011      3e-04       0.0013 ...

str(woeout, max.level = 1)
# List of 2
#  $ psi:List of 11
#  $ df :'data.frame':	2000 obs. of  12 variables:
#  - attr(*, "class")= chr "psiSummary"

head(woeout$df, 1)
#  idx_ woe.tot_derog woe.tot_tr woe.age_oldest_tr woe.tot_open_tr woe.tot_rev_tr ...
#     1       -0.3811    -0.0215           -0.5356         -0.0722        -0.1012 ...
```
