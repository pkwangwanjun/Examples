ROC\_AUC
================

``` r
library(wrapr)
library(WVPlots)
```

    ## Warning: replacing previous import 'vctrs::data_frame' by 'tibble::data_frame'
    ## when loading 'dplyr'

``` r
extreme_example <- function(n, sensitivity, specificity) {
  d <- data.frame(
    y = sample(
      c(TRUE, FALSE), 
      size = n, 
      replace = TRUE),
    score = 0.0
  )
  d$score[d$y] <- sample(
    c(0.25, 0.75), 
    size = sum(d$y), 
    replace = TRUE,
    prob = c(1 - sensitivity, sensitivity))
  d$score[!d$y] <- sample(
    c(0.25, 0.75), 
    size = sum(!d$y), 
    replace = TRUE,
    prob = c(specificity, 1 - specificity))
  d
}
```

Examples where ROC plots are not contained in each other.

``` r
ROCPlot(
  extreme_example(1000, 0.5, 1),
  xvar = 'score', 
  truthVar = 'y', truthTarget = TRUE, 
  title = 'example with great specificity')
```

![](ROC_AUC_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

``` r
ROCPlot(
  extreme_example(1000, 1, 0.5), 
  xvar = 'score', 
  truthVar = 'y', truthTarget = TRUE, 
  title = 'example with great sensitivity')
```

![](ROC_AUC_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

Beta ROC plots.

``` r
beta_example <- function(n, shape1_pos, shape2_pos, shape1_neg, shape2_neg) {
  d <- data.frame(
    y = sample(
      c(TRUE, FALSE), 
      size = n, 
      replace = TRUE),
    score = 0.0
  )
  d$score[d$y] <- rbeta(sum(d$y), shape1 = shape1_pos, shape2 = shape2_pos)
  d$score[!d$y] <- rbeta(sum(!d$y), shape1 = shape1_neg, shape2 = shape2_neg)
  d
}
```

``` r
d <- beta_example(
  10000,
  shape1_pos = 6, 
  shape2_pos = 6,
  shape1_neg = 1, 
  shape2_neg = 2)
  
DoubleDensityPlot(
  d,
  xvar = 'score',
  truthVar = 'y',
  truth_target = TRUE,
  title = "Asymmetric example where scores are beta-distributed")
```

![](ROC_AUC_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

``` r
ROCPlot(
  d, 
  xvar = 'score', 
  truthVar = 'y', truthTarget = TRUE, 
  title = "Asymmetric example where scores are beta-distributed")
```

![](ROC_AUC_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

``` r
d1 <- beta_example(
  10000,
  shape1_pos = 6, 
  shape2_pos = 6,
  shape1_neg = 1, 
  shape2_neg = 2)

DoubleDensityPlot(
  d1,
  xvar = 'score',
  truthVar = 'y',
  truth_target = TRUE,
  title = "Example where scores are beta-distributed (d1)")
```

![](ROC_AUC_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

``` r
d2 <- beta_example(
  10000,
  shape1_pos = 2, 
  shape2_pos = 1,
  shape1_neg = 6, 
  shape2_neg = 6)

DoubleDensityPlot(
  d2,
  xvar = 'score',
  truthVar = 'y',
  truth_target = TRUE,
  title = "Example where scores are beta-distributed (d2)")
```

![](ROC_AUC_files/figure-gfm/unnamed-chunk-8-2.png)<!-- -->

``` r
ROCPlotPair2(
  nm1 = 'd1',
  frame1 = d1,
  xvar1 = 'score',
  truthVar1 = 'y',
  truthTarget1 = TRUE,
  nm2 = 'd2',
  frame2 = d2,
  xvar2 = 'score',
  truthVar2 = 'y',
  truthTarget2 = TRUE,
  title = 'comparison')
```

![](ROC_AUC_files/figure-gfm/unnamed-chunk-8-3.png)<!-- -->

``` r
d <- beta_example(
  10000,
  shape1_pos = 2, 
  shape2_pos = 1,
  shape1_neg = 1, 
  shape2_neg = 2)
  
DoubleDensityPlot(
  d,
  xvar = 'score',
  truthVar = 'y',
  truth_target = TRUE,
  title = "Symmetric example where scores are beta-distributed")
```

![](ROC_AUC_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

``` r
ROCPlot(
  d, 
  xvar = 'score', 
  truthVar = 'y', truthTarget = TRUE, 
  title = "Symmetric example where scores are beta-distributed")
```

![](ROC_AUC_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->