The Perils of Prevalence Adjustment
================

## Introduction

I would like to show how the *point-wise shift model homotopy as
<code>P</code>* is not unbiased and not the same as the *unbiased shift
model homotopy defined as <code>U</code>*. We define a [probability
model
homotopy](https://win-vector.com/2020/10/10/upcoming-series-probability-model-homotopy/)
as a set of related models indexed by a variable
[here](https://win-vector.com/2020/10/10/upcoming-series-probability-model-homotopy/).
In the context of this note it is enough to think of a model homotopy as
a correction scheme we intend to apply when a model is applied to new
data.

In our introductory note we mentioned two related link shift
corrections.

  - The <code>P</code>-correction, which is essentially a point-wise
    application of Bayes’ Law.
  - The <code>U</code>-correction, which is designed to get prevalence
    right.

Our point is, these are not the same correction. Thus the
<code>P</code>-correction is not always unbiased.

### Example Setup

Let’s demonstrate this in [<code>R</code>](https://www.r-project.org).

First we attach our packages.

``` r
library(wrapr)
library(numbers)
```

Now let’s build our simulated model performance data.

``` r
bal_size <- 8
n_imb_rep <- 80
d <- data.frame(
  prediction = c(
    rep((bal_size - 1)/bal_size, bal_size),
    rep(c(1/2, 1/2, 1/2, 1/4, 1/4, 1/4, 1/4, 1/4, 1/4), n_imb_rep),
    rep(1/bal_size, bal_size)),
  truth = c(
    rep(TRUE, (bal_size - 1)), FALSE,
    rep(c(TRUE, TRUE, FALSE, TRUE, rep(FALSE, 5)), n_imb_rep),
    TRUE, rep(FALSE, (bal_size - 1))
  ))
d$orig_row_id <- seq_len(nrow(d))

knitr::kable(head(d))
```

| prediction | truth | orig\_row\_id |
| ---------: | :---- | ------------: |
|      0.875 | TRUE  |             1 |
|      0.875 | TRUE  |             2 |
|      0.875 | TRUE  |             3 |
|      0.875 | TRUE  |             4 |
|      0.875 | TRUE  |             5 |
|      0.875 | TRUE  |             6 |

The column `prediction` is behaving like an unbiased probability model
predicting `truth`. In particular `prediction` and `truth` have the same
expected value on this data set.

``` r
colMeans(subset(d, select= -orig_row_id)) %.>%
  knitr::kable(.)
```

|            |         x |
| :--------- | --------: |
| prediction | 0.3369565 |
| truth      | 0.3369565 |

``` r
prevalence <- mean(d$truth)

prevalence
```

    ## [1] 0.3369565

``` r
epsilon <- 1.0e-9
stopifnot(abs(prevalence  -  mean(d$prediction)) < epsilon)
prevalence  -  mean(d$prediction)
```

    ## [1] 0

## Bayes’ Correction (<code>P</code>)

### Derivation

Now the idea is the <code>P</code> correction is as follows. By Bayes’
Law we have:

``` 
  P[outcome==TRUE | evidence] = 
     P[outcome==TRUE] P[evidence | outcome==TRUE] / P[evidence]
     
  (1 - P[outcome==TRUE | evidence]) = 
     (1 - P[outcome==TRUE]) P[evidence | outcome==FALSE] / P[evidence]
```

So in terms of odds-ratios we have:

``` 
 P[outcome==TRUE | evidence] / (1 - P[outcome==TRUE | evidence]) = 
    (P[outcome==TRUE] / (1 - P[outcome==TRUE])) *
    (P[evidence | outcome==TRUE] / P[evidence | outcome==FALSE])
```

Taking logs of both-sides allows us to re-write this in terms of
`logit`.

``` r
logit <- function(x) {
  log( x / (1 - x) )
}
```

Which gives us

``` 
  logit(P[outcome==TRUE | evidence]) = 
     logit(P[outcome==TRUE]) + log(P[evidence | outcome==TRUE] / P[evidence | outcome==FALSE])
```

The idea is:

  - `log(P[evidence | outcome==TRUE] / P[evidence | outcome==FALSE])` is
    independent of the outcome prevalence of the data set it was
    estimated on.
  - `logit(P[outcome==TRUE])` is a function of a prevalence.

So it is tempting to take only the `log(P[evidence | outcome==TRUE] /
P[evidence | outcome==FALSE])` term from our model, and training data
and plug in an estimate of the prevalence of the population we intend to
work with. For example we showed in [“A Gruesome Example of Bayes’
Law”](https://win-vector.com/2020/09/11/a-gruesome-example-of-bayes-law/)
how to take an conditional evidence odds-ratio built on a population
with a prevalence near 50%, and apply it to a population with low
disease prevalence. Without this correction the predictions would be
very far off\!

This is what we formally call the <code>P</code> [model
homotopy](https://win-vector.com/2020/10/10/upcoming-series-probability-model-homotopy/).
The homotopy notation lets us give the method a name other than
“correction” if we so need. The potential downside of discussing the
<code>P</code> model homotopy only as a “correction” is running afoul of
the following sophistry: “that wrong corrections are not corrections, so
there is no such thing as wrong corrections.” So what you are trying to
study slips out of your hands.

### A Concrete Example

Back to a concrete example.

Suppose we intend to apply this model, built on a population with 0.34
prevalence on a new population with 50% prevalence. Let’s suppose our
50% prevalence is just a deterministic even re-sampling of our data (a
very easy case\!.

``` r
# get how many times to replicate each row group
count_table <- aggregate(
  count ~ truth, 
  data = transform(d, count = 1), 
  FUN = length)

multiple <- Reduce(LCM, count_table$count)
multiple
```

    ## [1] 15128

``` r
count_table$n_reps <- multiple / count_table$count

knitr::kable(count_table)
```

| truth | count | n\_reps |
| :---- | ----: | ------: |
| FALSE |   488 |      31 |
| TRUE  |   248 |      61 |

``` r
# replicate each row group by the target number of times
rep_table <- count_table %.>%
 lapply(
  seq_len(nrow(.)),
  function(i) {
    data.frame(
      truth = .$truth[[i]],
      row_rep = seq_len(.$n_reps[[i]]))
  }) %.>%
  do.call(rbind, .)

d_2 <- merge(d, rep_table, by = 'truth') %.>%
  .[order(.$orig_row_id, .$row_rep), ]
rownames(d_2) <- NULL

knitr::kable(head(d_2))
```

| truth | prediction | orig\_row\_id | row\_rep |
| :---- | ---------: | ------------: | -------: |
| TRUE  |      0.875 |             1 |        1 |
| TRUE  |      0.875 |             1 |        2 |
| TRUE  |      0.875 |             1 |        3 |
| TRUE  |      0.875 |             1 |        4 |
| TRUE  |      0.875 |             1 |        5 |
| TRUE  |      0.875 |             1 |        6 |

``` r
d_2$orig_row_id <- NULL
d_2$row_rep <- NULL
```

``` r
prevalence_2 <- mean(d_2$truth)

prevalence_2
```

    ## [1] 0.5

Now notice our original uncorrected model does not match the prevalence
on this new population. It is off and behaving in a biased manner.

``` r
stopifnot(abs(prevalence_2  - mean(d_2$prediction)) > 1e-2)
mean(d_2$prediction)
```

    ## [1] 0.3594494

The Bayes/<code>P</code> correction is given by shifting the model in
logit/link-space by the following factor.

``` r
delta <- -logit(prevalence) + logit(prevalence_2)

delta
```

    ## [1] 0.6768867

``` r
sigmoid <- function(x) {
  1 / (1 + exp(-x))
}

d_2$p_adjusted_prediction <- sigmoid(
  logit(d_2$prediction) + delta)
```

### The Result

Here is a summary of what we have. The `prediction` column is the
original model’s predictions, the `truth` column is now the expected
value of the truth indicator on the new re-sampled data set and the
`p_adjusted_prediction` is our new Bayes adjusted prediction.

``` r
aggregate(. ~ prediction, data = d_2, FUN = mean) %.>%
  knitr::kable(.)
```

| prediction |     truth | p\_adjusted\_prediction |
| ---------: | --------: | ----------------------: |
|      0.125 | 0.2194245 |               0.2194245 |
|      0.250 | 0.2824074 |               0.3961039 |
|      0.500 | 0.7973856 |               0.6630435 |
|      0.875 | 0.9323144 |               0.9323144 |

And we have our problem. Both the original and adjusted prediction
remain biased.

``` r
colMeans(d_2) %.>%
  knitr::kable(.)
```

|                         |         x |
| :---------------------- | --------: |
| truth                   | 0.5000000 |
| prediction              | 0.3594494 |
| p\_adjusted\_prediction | 0.5105872 |

``` r
stopifnot(abs(prevalence_2  - mean(d_2$p_adjusted_prediction)) > 1e-2)
mean(d_2$p_adjusted_prediction)
```

    ## [1] 0.5105872

Neither model estimate equals the actual new data prevalence of 0.5,
though the corrected value is in this case closer.

### Why the adjusted model is still biased

If the original model was unbiased (which it was), and the adjustment is
correct (which it is), how can the result be biased (which it is)?

The answer is: the adjustment is unbiased for *perfect* models, which
are not the kind we always see in practice. A perfect model has
absolutely no structure in the residuals. That is a perfect model has
`P[outcome==TRUE | prediction = p] = p` for *all* <code>p</code>. This
wasn’t the case for our example model. Here are the expected values of
truth on the original un-weighted data set conditioned on the predicted
probability.

``` r
d %.>%
  subset(., select = -orig_row_id) %.>%
  aggregate(
    . ~ prediction, 
    data = ., 
    FUN = mean) %.>%
  knitr::kable(.)
```

| prediction |     truth |
| ---------: | --------: |
|      0.125 | 0.1250000 |
|      0.250 | 0.1666667 |
|      0.500 | 0.6666667 |
|      0.875 | 0.8750000 |

Notice that prediction groups that were perfect in the original
(`0.125`, and `0.875`) do match the truth expectation in the re-sampled
data set. In this case the re-sampling is the exact same adjustment as
the shift, so things that matched before adjustment match after
adjustment.

For prediction groups that were not perfect in the original (`0.250` and
`0.500`) the re-scaling of data moves the answer at a rate determined by
the actual prevalence seen and correction moves at a rate determined by
the prediction. As these differ, the two estimates move different
amounts. Cancellations such as between the `0.250` prediction group
over-predicting in the original data frame, and the `0.500` group
under-predicting in the original data frame are not necessarily
preserved during the transform.

If Bayes’ correction gets the expected value right is something we have
to check, not a property of the transform in all cases. The obvious way
to establish unbiasedness would be to insist the mode obey the *very*
strong additional per-prediction range balance conditions we mentioned
before. These conditions are typically not all met in production, unless
one has taken extra care to [calibrate the
model](https://en.wikipedia.org/wiki/Platt_scaling) (perhaps through
isotonic regression, but even this will fail if the model has
order-reversals; binning the predictions and using a nested model *can*
fix the issue in general).

So roughly: the <code>P</code>-correction is biased because it is
fine-detail correct. The <code>P</code> correction tries to respect the
predicted rates of each prediction group in its adjustment, and does not
impose a global compromise between the predicts to repair the global
average. However, the <code>U</code>-correction does work for a global
compromise, so it gets averages right by compromising the predictions in
each group.

## The unbiased <code>U</code> correction.

Let’s explore a correction whose only goal is to get the global average
right. We hope it does more, but this is all it is really designed to
do. What we do is apply a shift in link space similar to the last
section, but we search for a shift that gets expected values right.

``` r
f <- function(d) {
  mean(d_2$truth) - mean(sigmoid(logit(d_2$prediction) + d))
}

delta_2 <- uniroot(f, c(-2, 2), tol = .Machine$double.eps)

delta_2
```

    ## $root
    ## [1] 0.630759
    ## 
    ## $f.root
    ## [1] 0
    ## 
    ## $iter
    ## [1] 6
    ## 
    ## $init.it
    ## [1] NA
    ## 
    ## $estim.prec
    ## [1] 1.245352e-08

This lands a new set of adjusted predictions.

``` r
d_2$u_adjusted_prediction <- sigmoid(
  logit(d_2$prediction) + delta_2$root)

aggregate(. ~ prediction, data = d_2, FUN = mean) %.>%
  knitr::kable(.)
```

| prediction |     truth | p\_adjusted\_prediction | u\_adjusted\_prediction |
| ---------: | --------: | ----------------------: | ----------------------: |
|      0.125 | 0.2194245 |               0.2194245 |               0.2116261 |
|      0.250 | 0.2824074 |               0.3961039 |               0.3851245 |
|      0.500 | 0.7973856 |               0.6630435 |               0.6526615 |
|      0.875 | 0.9323144 |               0.9323144 |               0.9293449 |

And these predictions are, as we hoped mean `0.5`.

``` r
stopifnot(abs(prevalence_2  -  mean(d_2$u_adjusted_prediction)) < epsilon)
mean(d_2$u_adjusted_prediction)
```

    ## [1] 0.5

## Platt Scaling

We can also try [Platt
scaling](https://en.wikipedia.org/wiki/Platt_scaling) and a Platt-shift
(no scaling, just a new intercept term).

### Standard Platt Scaling

As we expect from [knowledge of logistic
regression](https://win-vector.com/2011/09/14/the-simpler-derivation-of-logistic-regression/),
we see Platt scaling is unbiased in that it gets the global average
right.

``` r
Platt_scaler <- glm(
  truth ~ logit(prediction), 
  data = d_2, 
  family = binomial())

d_2$Platt_scaled_prediction <- predict(
  Platt_scaler,
  newdata = d_2,
  type = 'response')

aggregate(. ~ prediction, data = d_2, FUN = mean) %.>%
  knitr::kable(.)
```

| prediction |     truth | p\_adjusted\_prediction | u\_adjusted\_prediction | Platt\_scaled\_prediction |
| ---------: | --------: | ----------------------: | ----------------------: | ------------------------: |
|      0.125 | 0.2194245 |               0.2194245 |               0.2116261 |                 0.0689418 |
|      0.250 | 0.2824074 |               0.3961039 |               0.3851245 |                 0.2896237 |
|      0.500 | 0.7973856 |               0.6630435 |               0.6526615 |                 0.7882819 |
|      0.875 | 0.9323144 |               0.9323144 |               0.9293449 |                 0.9946869 |

``` r
stopifnot(abs(prevalence_2  -  mean(d_2$Platt_scaled_prediction)) < epsilon)
mean(d_2$Platt_scaled_prediction)
```

    ## [1] 0.5

The Platt scaling is unbiased, but it achieves this by trading off
systematic over and under prediction among the scoring groups. We didn’t
give the method enough degrees of freedom to make substantial
corrections. A good note on re-calibrating classifiers can be found
[here](https://win-vector.com/2019/07/16/an-ad-hoc-method-for-calibrating-uncalibrated-models-2/).

### Platt Shifting

Platt shifting work similarly, with one fewer degree of freedom.

``` r
Platt_shifter <- glm(
  truth ~ 1, 
  offset = logit(prediction),
  data = d_2, 
  family = binomial())

d_2$Platt_shifted_prediction <- predict(
  Platt_shifter,
  newdata = d_2,
  type = 'response')

d_2 %.>%
  subset(., select = c(prediction,  truth, u_adjusted_prediction, Platt_shifted_prediction)) %.>%
  aggregate(. ~ prediction, data = ., FUN = mean) %.>%
  knitr::kable(.)
```

| prediction |     truth | u\_adjusted\_prediction | Platt\_shifted\_prediction |
| ---------: | --------: | ----------------------: | -------------------------: |
|      0.125 | 0.2194245 |               0.2116261 |                  0.2116261 |
|      0.250 | 0.2824074 |               0.3851245 |                  0.3851245 |
|      0.500 | 0.7973856 |               0.6526615 |                  0.6526615 |
|      0.875 | 0.9323144 |               0.9293449 |                  0.9293449 |

``` r
stopifnot(abs(prevalence_2  -  mean(d_2$Platt_shifted_prediction)) < epsilon)
mean(d_2$Platt_shifted_prediction)
```

    ## [1] 0.5

We see Platt-shifting is exactly the <code>U</code> correction.

## Conclusion

The point of this note was: Bayesian corrections are not necessarily
unbiased. This should not be too surprising as insisting on
un-biasedness is part of the frequentest foundation, not the Bayesian.
What we have observed is: rhetoric chains of the form “this is the right
correction”, “a perfect correction would have this additional property”,
and “therefore this correction would also have this property” can fall
apart.

This brings me to what I love about this blog. I can publicly work
simple tutorial examples in a way there really is no room for in the
formal literature.
