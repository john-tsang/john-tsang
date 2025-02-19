---
author_profile: true
published: true
title: "Accelerate `R` with `Rcpp`: Sampling with Replacement and the Hansen-Hurwitz Estimator"
use_math: true
toc: true
toc_label: ""
toc_icon: "fas fa-folder-open"
toc_sticky: false
categories:
  - Notes
tags:
  - R
  - Rcpp
  - C++
  - Survey Sampling
  - Simulation
---
> **Result**: On average, using `Rcpp` can be about 4.5 times faster than just using `R`.

This note compares the computational time required by `R` with and without the use of `Rcpp` to complete the following tasks 1000 times each.

1. Task 1: Simple random sampling with replacement

2. Task 2: A simulation study for the Hansen-Hurwitz estimator

# `Rcpp` for Performance in `R`
[`Rcpp`](https://www.rcpp.org/) is an `R` package designed to provide an interface for `R` to use [`C++`](https://en.wikipedia.org/wiki/C%2B%2B) to accelerate computation. 

* **Advantage**: The improvement in computational time is usually tremendous, even when compared with highly optimized R base functions.

* **Disadvantage**: As its name suggests, Rcpp requires programs written in more complicated C++.

> ***My opinion***: 
> Using Rcpp may not be a good option for computationally small tasks because of C++’s complications. 
> However, it is worthwhile to **consider Rcpp for computationally intensive jobs**.

# Task 1: Simple random sampling with replacement
Consider the case of simple random sampling with replacement (equal probability of being chosen) from a sample of size 100,000. 

Generate the sampling frame: a $$N(0,1)$$ sample of size 100 000.
```R
set.seed(123465)
n.elem = 100000
frame1 = rnorm(n.elem)
```
## R Vs. Rcpp: Rcpp Faster

* The `C++` code in the source file `sample_with_replacement.cpp`: 
```c++
#include <RcppArmadillo.h>
#include <Rcpp.h>
// [[Rcpp::depends(RcppArmadillo)]]
// [[Rcpp::plugins(cpp11)]]
// [[Rcpp::export()]]
arma::vec sample_w_replacement(arma::vec x, const unsigned int sample_size) {
	const unsigned int x_size = x.size();
	arma::uvec index = arma::randi<arma::uvec>(sample_size, arma::distr_param(0,x_size-1));
	arma::vec rt = x.elem(index);
	return rt;
}
```
The function `sample_w_replacement` takes a numeric vector from `R` and the sample size of the random sample 
to be selected and returns a vector of the random sample.


* We load the library `Rcpp` and the `C++` source file in `R`.
```R
library(Rcpp)
sourceCpp("sample_with_replacement.cpp")
```


* Runtime comparison by library `microbenchmark` in `R`: Base `R` function `sample` Vs. `sample_w_replacement` with `Rcpp`
```R
library(microbenchmark)
microbenchmark(
	r = sample(frame1, n.elem, replace = TRUE),
	cpp = sample_w_replacement(frame1, n.elem),
	times = 1000L
)
```
*Output:*
```R
Unit: milliseconds
 expr    min      lq     mean  median     uq     max neval
    r 7.2487 8.14530 9.007408 8.63335 9.3552 34.6576  1000
  cpp 1.3987 1.62835 2.294706 1.92230 2.5577 23.6820  1000
```
> **Result**: The `Rcpp` implementation of simple random sampling with replacement runs faster than `sample` in `R`.

# Task 2: A Simulation Study for the Hansen-Hurwitz Estimator

This simulation study verifies the unbiasedness of the Hansen-Hurwitz estimator
by checking if the relative bias is close to 0.

## The Hansen-Hurwitz (HH) Estimator
The Hansen-Hurwitz estimator of the population mean:

$$
\hat{\bar{Y}}_{HH} = \dfrac{1}{n}\sum_{k \in U}Q_ky_k
$$

where $$Q_k$$ is the number of times that unit $$k$$ is selected in a sample
of size $$n$$, $$k = 1, 2, 3, \ldots, N$$. 

An `R` function for the HH estimator:
```R
HH.estimator = function(sample.selected) {
  return(mean(sample.selected))
}
```

## R Vs. Rcpp: Rcpp Faster
Consider the case when the sample size is 10 000 ($$n = 10000$$) and the number of replication is 1000 ($$R = 1000$$).

* An `R` function for the simulation.
```R
r.sim = function(R, n, frame1) {
	results = 0
	for (i in 1:R) {
		sample.selected = sample(frame1, n, replace = TRUE)
		results = results + HH.estimator(sample.selected)
	}
	rel.bias = (results/R - mean(frame1)) / mean(frame1) * 100
	rel.bias
}
```

* The `C++` code in the source file `sample_with_replacement.cpp`: 
```c++
#include <RcppArmadillo.h>
#include <Rcpp.h>
// [[Rcpp::depends(RcppArmadillo)]]
// [[Rcpp::plugins(cpp11)]]
// [[Rcpp::export()]]
arma::vec sample_w_replacement(arma::vec x, const unsigned int sample_size) {
	const unsigned int x_size = x.size();
	arma::uvec index = arma::randi<arma::uvec>(sample_size, arma::distr_param(0,x_size-1));
	arma::vec rt = x.elem(index);
	return rt;
}
double HH_estimator(arma::vec sample) {
	return(arma::mean(sample));
}
// [[Rcpp::export()]]
double rcpp_sim(const int R, const int n, arma::vec frame1) {
	double results = 0;
	for (int i = 0; i < R; ++i) {
		arma::vec sample_selected = sample_w_replacement(frame1, n);
		results += HH_estimator(sample_selected);
	}
	double rel_bias = (results/R - arma::mean(frame1)) / arma::mean(frame1) * 100;
	return rel_bias;
}
```
* We load the library `Rcpp` and the `C++` source file in `R`.
```R
library(Rcpp)
sourceCpp("sample_with_replacement.cpp")
```

* Runtime comparison: a sample size of $$n = 10000$$.
```R
microbenchmark(
	r = r.sim(1000, 10000, frame1),
	cpp = rcpp_sim(1000, 10000, frame1),
	times = 1000L
)
```
*Output:*
```
Unit: milliseconds
 expr      min        lq      mean    median        uq       max neval
    r 953.0094 1296.0009 1373.9247 1350.9042 1440.9796 2040.6367  1000
  cpp 212.0845  317.4542  356.2386  343.5164  386.2789  789.7178  1000
```
> **Result**: The `Rcpp` implementation runs faster.

# Further Discussion

* The function `sample` comes from base `R`, so this function is already optimized. If the `R` function we compare is a user-defined function, the improvement from `Rcpp` will be much more significant.

* In the next note, I will discuss how to use `OpenMP` to accelerate `Rcpp` computation. 