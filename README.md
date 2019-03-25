
<!-- README.md is generated from README.Rmd. Please edit that file -->

# printy

Over the years, I’ve written a lot of one-off functions for formatting
numbers in RMarkdown documents. This packages collects them in a single
location.

## Installation

You can install printy from github with:

``` r
# install.packages("devtools")
remotes::install_github("tjmahr/printy")
```

## Examples

Print a number with n digits of precision. R numbers lose precision when
converted to strings. This function converts the numbers to strings and
keeps precision. (It’s a wrapper for `sprintf()`.)

``` r
library(dplyr, warn.conflicts = FALSE)
library(printy)
test_cor <- cor(mtcars[, 1:4]) 

# Typical loss of trailing zeroes
test_cor[1:4, 3] %>% round(2) %>% as.character()
#> [1] "-0.85" "0.9"   "1"     "0.79"

test_cor[1:4, 3] %>% fmt_fix_digits(2)
#> [1] "-0.85" "0.90"  "1.00"  "0.79"
```

Don’t print a leading zero on numbers that are bounded between −1 and 1,
such as correlations or *p*-values.

``` r
fmt_leading_zero(c(-0.3, 0.4, 1))
#> [1] "-.3" ".4"  "1"

# Todo: This functions obliterates 0's
fmt_leading_zero(c(0, 0.0))
#> [1] "" ""
```

Format negative numbers with a minus sign.

``` r
fmt_minus_sign(c(1, 2, -3, -0.4, -pi))
#> [1] "1"                       "2"                      
#> [3] "&minus;3"                "&minus;0.4"             
#> [5] "&minus;3.14159265358979"
```

Putting it all together.

``` r
fmt_correlation <- function(xs, digits = 2) {
  xs %>% fmt_fix_digits(digits) %>% fmt_leading_zero() %>% fmt_minus_sign()
}

test_cor %>% 
  as.data.frame() %>% 
  tibble::rownames_to_column(".rowname") %>% 
  tibble::as_tibble() %>% 
  mutate_at(vars(-.rowname), fmt_correlation) %>% 
  rename(` ` = .rowname) %>% 
  knitr::kable(align = "lrrrr")
```

|      |  mpg |  cyl | disp |   hp |
| ---- | ---: | ---: | ---: | ---: |
| mpg  | 1.00 | −.85 | −.85 | −.78 |
| cyl  | −.85 | 1.00 |  .90 |  .83 |
| disp | −.85 |  .90 | 1.00 |  .79 |
| hp   | −.78 |  .83 |  .79 | 1.00 |

### *p*-values

Format *p*-values with *n* digits of precision, with no leading zero,
and with very small values being printed with a `<` sign.

``` r
p <- c(1, 0.1, 0.01, 0.001, 0.0001)
fmt_p_value(p, digits = 2)
#> [1] "1.00"  ".10"   ".01"   "< .01" "< .01"
fmt_p_value(p, digits = 3)
#> [1] "1.000"  ".100"   ".010"   ".001"   "< .001"
```

Print *p*-values in markdown with nice defaults.

  - Use 3 digits of precision for values less than .06
  - Otherwise, use 2 digits of precision.
  - Include *p* in markdown

<!-- end list -->

``` r
p <- c(1, 0.1, 0.06, 0.059, 0.051, 0.01, 0.001, 0.0001)
fmt_p_value_md(p)
#> [1] "*p*&nbsp;= 1.00" "*p*&nbsp;= .10"  "*p*&nbsp;= .06"  "*p*&nbsp;= .059"
#> [5] "*p*&nbsp;= .051" "*p*&nbsp;= .010" "*p*&nbsp;= .001" "*p*&nbsp;< .001"
```

These render as: *p* = 1.00, *p* = .10, *p* = .06, *p* = .059, *p* =
.051, *p* = .010, *p* = .001, *p* \< .001.

## Formatting tables from lme4 models

One thing I’ve had to do a lot is summarize mixed effects models fit
with lme4. This package provides `pretty_lme4_ranefs()` which creates a
dataframe random effect variances and covariances like those printed by
`summary()`.

For example, we can fit the model.

``` r
library(lme4)
#> Loading required package: Matrix
#> Warning: package 'Matrix' was built under R version 3.5.3
model <- lmer(Reaction ~ Days + (Days | Subject), sleepstudy)
summary(model)
#> Linear mixed model fit by REML ['lmerMod']
#> Formula: Reaction ~ Days + (Days | Subject)
#>    Data: sleepstudy
#> 
#> REML criterion at convergence: 1743.6
#> 
#> Scaled residuals: 
#>     Min      1Q  Median      3Q     Max 
#> -3.9536 -0.4634  0.0231  0.4633  5.1793 
#> 
#> Random effects:
#>  Groups   Name        Variance Std.Dev. Corr
#>  Subject  (Intercept) 611.90   24.737       
#>           Days         35.08    5.923   0.07
#>  Residual             654.94   25.592       
#> Number of obs: 180, groups:  Subject, 18
#> 
#> Fixed effects:
#>             Estimate Std. Error t value
#> (Intercept)  251.405      6.824  36.843
#> Days          10.467      1.546   6.771
#> 
#> Correlation of Fixed Effects:
#>      (Intr)
#> Days -0.138
```

`pretty_lme4_ranefs()` creates the following dataframe.

``` r
pretty_lme4_ranefs(model)
#>      Group   Parameter Variance    SD Correlations &nbsp;
#> 1  Subject (Intercept)   611.90 24.74         1.00 &nbsp;
#> 2   &nbsp;        Days    35.08  5.92          .07   1.00
#> 3 Residual      &nbsp;   654.94 25.59       &nbsp; &nbsp;
```

Which in markdown renders
as

``` r
knitr::kable(pretty_lme4_ranefs(model), align = c("l", "l", "r", "r", "r"))
```

| Group    | Parameter   | Variance |    SD | Correlations |      |
| :------- | :---------- | -------: | ----: | -----------: | :--- |
| Subject  | (Intercept) |   611.90 | 24.74 |         1.00 |      |
|          | Days        |    35.08 |  5.92 |          .07 | 1.00 |
| Residual |             |   654.94 | 25.59 |              |      |

Here’s a dumb model with a lot going on in the random effects.

``` r
model <- lmer(mpg ~ wt * hp + (drat | gear) + (hp * cyl | am), mtcars)
#> boundary (singular) fit: see ?isSingular
model
#> Linear mixed model fit by REML ['lmerMod']
#> Formula: mpg ~ wt * hp + (drat | gear) + (hp * cyl | am)
#>    Data: mtcars
#> REML criterion at convergence: 152.7432
#> Random effects:
#>  Groups   Name        Std.Dev. Corr             
#>  gear     (Intercept) 1.556809                  
#>           drat        0.166292 -1.00            
#>  am       (Intercept) 1.940271                  
#>           hp          0.004055 -0.96            
#>           cyl         0.456219 -0.98  0.93      
#>           hp:cyl      0.001508  0.95 -0.94 -0.99
#>  Residual             2.113554                  
#> Number of obs: 32, groups:  gear, 3; am, 2
#> Fixed Effects:
#> (Intercept)           wt           hp        wt:hp  
#>    48.98745     -7.80904     -0.12118      0.02737  
#> convergence code 0; 1 optimizer warnings; 0 lme4 warnings

knitr::kable(pretty_lme4_ranefs(model), 
             align = c("l", "l", "r", "r", "r", "r", "r", "r", "r"))
```

| Group    | Parameter   | Variance |   SD | Correlations |      |      |      |
| :------- | :---------- | -------: | ---: | -----------: | ---: | ---: | ---: |
| am       | (Intercept) |     3.76 | 1.94 |         1.00 |      |      |      |
|          | hp          |     0.00 | 0.00 |         −.96 | 1.00 |      |      |
|          | cyl         |     0.21 | 0.46 |         −.98 |  .93 | 1.00 |      |
|          | hp:cyl      |     0.00 | 0.00 |          .95 | −.94 | −.99 | 1.00 |
| gear     | (Intercept) |     2.42 | 1.56 |         1.00 |      |      |      |
|          | drat        |     0.03 | 0.17 |        −1.00 | 1.00 |      |      |
