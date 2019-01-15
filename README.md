# CovarianceMatrices.jl

[![Build Status](https://travis-ci.org/gragusa/CovarianceMatrices.jl.svg?branch=master)](https://travis-ci.org/gragusa/CovarianceMatrices.jl)
[![Coverage Status](https://coveralls.io/repos/gragusa/CovarianceMatrices.jl/badge.svg?branch=master&service=github)](https://coveralls.io/github/gragusa/CovarianceMatrices.jl?branch=master)

Heteroskedasticity and Autocorrelation Consistent Covariance Matrix Estimation for Julia.

## Introduction

This package provides types and methods useful to obtain consistent estimates of the long run covariance matrix of a random process.

Three classes of estimators are considered:

1. **HAC** 
    - heteroskedasticity and autocorrelation consistent (Andrews, 1996; Newey and West, 1994)
2. **HC**  
    - hetheroskedasticity consistent (White, 1982)
3. **CRVE** 
    - cluster robust (Arellano, 1986; Bell, 2002)

The typical applications of these estimators are __(a)__ to estimate the long run covariance matrix of a stochastic process, and/or __(b)__ conduct robust inference about parameters of generalized linear models (GLM).


## Installation

The package is registered on [METADATA](http::/github.com/JuliaLang/METADATA.jl), so to install
```julia
pkg> add CovarianceMatrices
```

## Quick and dirty

Given a `GLM` object, we want to construct the variance covariance matrix of the coefficients.

This can be done by using `vcov`. 

```julia
vcov(::DataFrameRegressionModel, ::T) where T<:RobustVariance
```
Standard errors can be obtained by 
```julia
stderror(::DataFrameRegressionModel, ::T) where T<:RobustVariance
```

`RobustVariance` is an abstract type. Subtypes of `RobustVariance` are `HAC`, `HC` and `CRHC`. These in turns are subtypes with concrete types give n by

- `HAC`
    - `TruncatedKernel`
    - `BartlettKernel`
    - `ParzenKernel`
    - `TukeyHanningKernel`
    - `QuadraticSpectralKernel `

- `HC`
    - `HC1`
    - `HC2`
    - `HC3`
    - `HC4`
    - `HC4m`
    - `HC5`

- `CRHC`
    - `CRHC1`
    - `CRHC2`
    - `CRHC3`


### HAC (Heteroskedasticity and Autocorrelation Consistent)

```julia
vcov(::DataFrameRegressionModel, ::T) where T<:HAC
```

Consider the following artificial data (a regression with autoregressive error component):
```julia
using CovarianceMatrices
using DataFrames
using Random
Random.seed!(1)
n = 500
x = randn(n,5)
u = Array{Float64}(undef, 2*n)
u[1] = rand()
for j in 2:2*n
    u[j] = 0.78*u[j-1] + randn()
end
df = DataFrame()
df[:y] = y
for j in enumerate([:x1, :x2, :x3, :x4, :x5])
    df[j[2]] = x[:,j[1]]
end
```
Using the data in `df`, the coefficient of the regression can be estimated using `GLM`

```julia
lm1 = lm(@formula(y~x1+x2+x3+x4+x5), df)
```

Given the autocorrelation in the residuals, to obtain a consistent covariance of estimator of the coefficient we have to use a `HAC` estimator which in turns requires choosing a kernel function. 


```julia
QuadraticSpectralKernel(::Real; prewhiten = false)
QuadraticSpectralKernel(::Andrews; prewhiten = false)
QuadraticSpectralKernel(::NeweyWest; prewhiten = false)
```

So, for instance, calling

```julia
vcov(lm1, ParzenKernel(1.34, prewhiten = true))
```

will return the estimate of the covariance matrix of the coefficient of `lm1` using a the Parzen Kernel, a bandwidth equal to 1.34. The estimating equation will be also prewhiten. 


To get an estimate that uses the Quadratic Spectral Kernel and data-dependent bandwidth selected _à la_ Andrews (and no prewhitening)
```julia
vcov(lm1, QuadraticSpectralKernel(Andrews))
```
The following uses instead Newey-West method of selecting the optimal bandwidth:
```julia
vcov(lm1, QuadraticSpectralKernel(NeweyWest))
```


### HC (Heteroskedastic consistent)

As in the HAC case, `vcov` and `stderr` are the main functions. They know get as argument the type of robust variance being sought
```julia
vcov(::DataFrameRegressionModel, ::HC)
```
Where `HC` is an abstract type with the following concrete types:

- `HC0`
    - This is Hal White (1982) robust variance estimator
- `HC1`
    - This is equal to `H0` multiplyed it by n/(n-p), where n is the sample size and p is the number of parameters in the model.
- `HC2`
    - A modification of HC0 that involves dividing the squared residual by 1-h, where h is the leverage for the case (Horn, Horn and Duncan, 1975)
- `HC3`
    - A modification of HC0 that approximates a jackknife estimator. Squared residuals are divided by the square of 1-h (Davidson and Mackinnon, 1993).
- `HC4`
    - A modification of HC0 that divides the squared residuals by 1-h to a power that varies according to h, n, and p, with an upper limit of 4 (Cribari-Neto, 2004).
- `HC4m`
    - Similar to HC4 but with smaller bias (Cribari-Neto and Da Silva, 2011)
- `HC5`
    - A modification of HC0 that divides the squared residuals by 1-h to a power that varies according to h, n, and p, with an upper limit of 4 (Cribari-Neto, 2004). (Cribari-Neto, Souza and Vasconcellos, 2007)

Example:

```julia
obs = 50
df = DataFrame(x = randn(obs))
df[:y] = df[:x] + sqrt.(df[:x].^2).*randn(obs)
lm = fit(LinearModel,@formula(y~x), df)
vcov(ols)
vcov(ols, HC0())
vcov(ols, HC1())
vcov(ols, HC2())
vcov(ols, HC3())
vcov(ols, HC4m())
vcov(ols, HC5())
```


### CRHC (Cluster robust heteroskedasticty consistent)
The API of this class of variance estimators is subject to change, so please use with care. The difficulty is that `CRHC` type needs to have access to the variable along which dimension the clustering mast take place. For the moment, the following approach works --- as long as no missing values are present in the original dataframe.

Example:

```julia
using RDatasets
df = dataset("plm", "Grunfeld")
lm = glm(@formula(Inv~Value+Capital), df, Normal(), IdentityLink())
```

The `Firm` clustered variance and standard errors can be obtained: 

```julia
vcov(lm, CRHC1(convert(Array{Float64}, df[:Firm])))
stderror(lm, CRHC1(convert(Array{Float64}, df[:Firm])))
```