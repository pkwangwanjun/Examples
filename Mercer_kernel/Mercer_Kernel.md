Mercer\_Kernel
================

John Mount December 2020

I’ve written before on Kernel methods (in the context of support vector
machines).

-   <https://win-vector.com/2011/10/07/kernel-methods-and-support-vector-machines-de-mystified/>
-   <https://win-vector.com/2015/02/14/how-sure-are-you-that-large-margin-implies-low-vc-dimension/>

Here we will show an example of non-kernel functions.

Both `abs(x dot y)` and `relu(x dot y)` (`=max(0, x dot y)`) are not
positive semi-definite Mercer kernels when `x, y` are 2-dimensional
vectors or larger. This can be subtle as for scalars `abs(x * y)` =
`abs(x) * abs(y)` is exactly the kind of decomposition used to establish
kernel properties.

By Mercer’s theorem anything that generates only positive semi-definite
Gram matrices can be realized as the inner product of vectors mapped
into a, possibly infinite dimensional, vector space.

``` r
n <- 4  # dimension of Gram matrix
d <- 2  # dimension of vectors

# abs(x.y) and relu(x.y) are not Mercer Kernels
#
# Gram matrix
# True for 3x3 case by Sylvester's criterion
# https://math.stackexchange.com/questions/1355088/is-the-absolute-value-of-a-p-d-s-matrix-p-d-s
# https://math.stackexchange.com/questions/1355088/is-the-absolute-value-of-a-p-d-s-matrix-p-d-s#comment2758302_1355088
# and true for scalar arguments as abs(x*y) = abs(x)*abs(y)

dot_f <- function(x, y) {
  sum(x*y)
}

relu_f <- function(x, y) {
  max(0, sum(x*y))
}

abs_f <- function(x, y) { 
  abs(sum(x*y))
}

# build the Gram matrix for a given kernel
Gram_matrix <- function(n, k_f, x) {
  idxs = expand.grid(
    i = seq_len(n), 
    j = seq_len(n))
  matrix(vapply(
    seq_len(nrow(idxs)), 
    function(i) k_f(x[idxs[i, 1], ], x[idxs[i, 2], ]), 
    numeric(1)),
    nrow = n,
    ncol = n)
}
```

``` r
fn <- function(n, d) {
  # x <- matrix(rnorm(n = n*d), nrow = n, ncol = d)
  x <- matrix(
    sample(c(-1, 0, 1), 
           size = n*d, replace = TRUE), 
    nrow = n, 
    ncol = d)
  mat_relu <- Gram_matrix(n = n, k_f = relu_f, x = x)
  eigen_relu = min(eigen(mat_relu)$values)
  mat_abs <- Gram_matrix(n = n, k_f = abs_f, x = x)
  eigen_abs = min(eigen(mat_abs)$values)
  list(v = max(eigen_relu, eigen_abs), 
       x = x, 
       eigen_relu = eigen_relu, 
       eigen_abs = eigen_abs)
}

example <- lapply(
  1:10000,
  function(i) fn(n = n, d = d))

idx_relu = which.min(vapply(example, function(v) v$eigen_relu, numeric(1)))
print(example[[idx_relu]])

idx_abs = which.min(vapply(example, function(v) v$eigen_abs, numeric(1)))
print(example[[idx_abs]])

idx <- which.min(vapply(example, function(v) v$v, numeric(1)))
example <- example[[idx]]
print(example)
x <- example$x

t(x)
```

``` r
x <- matrix(c(-1, -1, -1, 0, -1, 1, 0, -1), nrow = n, ncol = d)

t(x)
```

    ##      [,1] [,2] [,3] [,4]
    ## [1,]   -1   -1   -1    0
    ## [2,]   -1    1    0   -1

The dot-product is a Kernel, in fact the prototypical one. In fact
`g^t G(a) g`, (`G(a)` being the Gram matrix with
`G(a)[i, j] = a(i) . a(j)`) is equal to `||[a_1, ... a_n] g||^2`.

``` r
mat_dot <- Gram_matrix(n = n, k_f = dot_f, x = x)
mat_dot
```

    ##      [,1] [,2] [,3] [,4]
    ## [1,]    2    0    1    1
    ## [2,]    0    2    1   -1
    ## [3,]    1    1    1    0
    ## [4,]    1   -1    0    1

``` r
eigen_dot <- eigen(mat_dot)
if(min(eigen_dot$values)<0) {
  stop("expected eigen_dot to have only non-negative eigenvalues")
}
eigen_dot
```

    ## eigen() decomposition
    ## $values
    ## [1] 3.000000e+00 3.000000e+00 2.220446e-15 8.881784e-16
    ## 
    ## $vectors
    ##            [,1]      [,2]       [,3]       [,4]
    ## [1,]  0.0000000 0.8164966  0.5773503  0.0000000
    ## [2,] -0.8164966 0.0000000  0.0000000 -0.5773503
    ## [3,] -0.4082483 0.4082483 -0.5773503  0.5773503
    ## [4,]  0.4082483 0.4082483 -0.5773503 -0.5773503

``` r
mat_abs <- Gram_matrix(n = n, k_f = abs_f, x = x)
mat_abs
```

    ##      [,1] [,2] [,3] [,4]
    ## [1,]    2    0    1    1
    ## [2,]    0    2    1    1
    ## [3,]    1    1    1    0
    ## [4,]    1    1    0    1

``` r
eigen_abs <- eigen(mat_abs)
eigen_abs
```

    ## eigen() decomposition
    ## $values
    ## [1]  3.5615528  2.0000000  1.0000000 -0.5615528
    ## 
    ## $vectors
    ##           [,1]          [,2]          [,3]       [,4]
    ## [1,] 0.5573454 -7.071068e-01  0.000000e+00  0.4351621
    ## [2,] 0.5573454  7.071068e-01 -1.110223e-16  0.4351621
    ## [3,] 0.4351621 -2.220446e-16 -7.071068e-01 -0.5573454
    ## [4,] 0.4351621 -1.110223e-16  7.071068e-01 -0.5573454

``` r
example_abs <- eigen_abs$vectors[, n]
example_abs
```

    ## [1]  0.4351621  0.4351621 -0.5573454 -0.5573454

``` r
v_abs <- as.numeric(t(example_abs) %*% mat_abs %*% example_abs)
if(v_abs>=0) {
  stop("expected v_abs negative")
}
v_abs
```

    ## [1] -0.5615528

``` r
mat_relu <- Gram_matrix(n = n, k_f = relu_f, x = x)
mat_relu
```

    ##      [,1] [,2] [,3] [,4]
    ## [1,]    2    0    1    1
    ## [2,]    0    2    1    0
    ## [3,]    1    1    1    0
    ## [4,]    1    0    0    1

``` r
eigen_relu <- eigen(mat_relu)
eigen_relu
```

    ## eigen() decomposition
    ## $values
    ## [1]  3.1935271  2.2949629  0.7050371 -0.1935271
    ## 
    ## $vectors
    ##            [,1]       [,2]       [,3]       [,4]
    ## [1,] -0.6845604  0.4744647 -0.2264430  0.5049593
    ## [2,] -0.4230816 -0.7677000  0.3663925  0.3120820
    ## [3,] -0.5049593 -0.2264430 -0.4744647 -0.6845604
    ## [4,] -0.3120820  0.3663925  0.7677000 -0.4230816

``` r
example_relu <- eigen_relu$vectors[, n]
example_relu
```

    ## [1]  0.5049593  0.3120820 -0.6845604 -0.4230816

``` r
v_relu <- as.numeric(t(example_relu) %*% mat_relu %*% example_relu)
if(v_relu>=0) {
  stop("expected v_relu negative")
}
v_relu
```

    ## [1] -0.1935271