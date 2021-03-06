Non-Standard Evaluation and Function Composition in R
=====================================================

In this article we will discuss composing standard-evaluation interfaces (SE) and composing non-standard-evaluation interfaces (NSE) in [`R`](https://www.r-project.org).

In [`R`](https://www.r-project.org) the package [`tidyeval`/`rlang`](https://CRAN.R-project.org/package=rlang) is a tool for building domain specific languages intended to allow easier composition of NSE interfaces.

To use it you must know some of its structure and notation. Here are some details paraphrased from the major `tidyeval`/`rlang` client, the package dplyr: [`vignette('programming', package = 'dplyr')`](https://cran.r-project.org/web/packages/dplyr/vignettes/programming.html)).

-   "`:=`" is needed to make left-hand-side re-mapping possible (adding yet another "more than one assignment type operator running around" notation issue).
-   "`!!`" substitution requires parenthesis to safely bind (so the notation is actually "`(!! )`", not "`!!`").
-   Left-hand-sides of expressions are names or strings, while right-hand-sides are `quosures`/expressions.

Example
-------

Let's apply `tidyeval`/`rlang` notation to the task of building re-usable generic in [`R`](https://www.r-project.org).

``` r
# setup
suppressPackageStartupMessages(library("dplyr"))
packageVersion("dplyr")
```

    ## [1] '0.7.0'

[`vignette('programming', package = 'dplyr')`](https://cran.r-project.org/web/packages/dplyr/vignettes/programming.html) includes the following example:

``` r
my_mutate <- function(df, expr) {
  expr <- enquo(expr)
  mean_name <- paste0("mean_", quo_name(expr))
  sum_name <- paste0("sum_", quo_name(expr))

  mutate(df, 
    !!mean_name := mean(!!expr), 
    !!sum_name := sum(!!expr)
  )
}
```

We can try this:

``` r
d <- data.frame(a=1)
my_mutate(d, a)
```

    ##   a mean_a sum_a
    ## 1 1      1     1

### SE Example

For this example we can figure out how to use `tidyeval`/`rlang` notation to build a standard interface version of a function that adds one to a column and lands the value in an arbitrary column:

``` r
tidy_add_one_se <- function(df, res_var_name, input_var_name) {
  input_var <- as.name(input_var_name)
  res_var <- res_var_name
  mutate(df,
         !!res_var := (!!input_var) + 1)
}

tidy_add_one_se(d, 'res', 'a')
```

    ##   a res
    ## 1 1   2

And we can re-wrap `tidy_add_one_se` as into a "add one to self" function as we show here:

``` r
tidy_increment_se <- function(df, var_name) {
  tidy_add_one_se(df, var_name, var_name)
}

tidy_increment_se(d, 'a')
```

    ##   a
    ## 1 2

### NSE Example

We can also use the `tidyeval`/`rlang` notation more as it is intended: to wrap or compose a non-standard interface in another non-standard interface.

``` r
tidy_add_one_nse <- function(df, res_var, input_var) {
  input_var <- enquo(input_var)
  res_var <- quo_name(enquo(res_var))
  mutate(df,
         !!res_var := (!!input_var) + 1)
}

tidy_add_one_nse(d, res, a)
```

    ##   a res
    ## 1 1   2

And we even wrap this again as a new "add one to self" function:

``` r
tidy_increment_nse <- function(df, var) {
  var <- enquo(var)
  tidy_add_one_nse(df, !!var, !!var)
}

tidy_increment_nse(d, a)
```

    ##   a
    ## 1 2

(The above `enquo()` then "`!!`" pattern is pretty much necissary, as the simpler idea of just passing `var` through doesn't work.)

#### An Issue

We could try use `base::substitute()` instead of `quo_name(enquo())` in the non-standard-evaluation wrapper. At first this appears to work, but it runs into trouble when we try to compose non-standard-evaluation functions with each other.

``` r
tidy_add_one_nse_subs <- function(df, res_var, input_var) {
  input_var <- enquo(input_var)
  res_var <- substitute(res_var)
  mutate(df,
         !!res_var := (!!input_var) + 1)
}

tidy_add_one_nse_subs(d, res, a)
```

    ##   a res
    ## 1 1   2

However this seemingly similar variation is not re-composable in the same manner.

``` r
tidy_increment_nse_subs <- function(df, var) {
  var <- enquo(var)
  tidy_add_one_nse_subs(df, !!var, !!var)
}

tidy_increment_nse_subs(d, a)
```

    ## Error: LHS must be a name or string

Likely there is some way to get this to work, but my point is:

-   The obvious way didn't work.
-   Some NSE functions can't be re-used in standard NSE composition. You may not know which ones those are ahead of time. Presumably functions from major packages are so-vetted, but you may not be able to trust "one off compositions" to be safe to re-compose.

wrapr::let
----------

It is easy to specify the function we want with [`wrapr`](https://CRAN.R-project.org/package=wrapr) as follows (both using standard evaluation, and using non-standard evaluation):

### SE version

``` r
library("wrapr")

wrapr_add_one_se <- function(df, res_var_name, input_var_name) {
  wrapr::let(
    c(RESVAR= res_var_name,
      INPUTVAR= input_var_name),
    df %>%
      mutate(RESVAR = INPUTVAR + 1)
  )
}

wrapr_add_one_se(d, 'res', 'a')
```

    ##   a res
    ## 1 1   2

Standard composition:

``` r
wrapr_increment_se <- function(df, var_name) {
  wrapr_add_one_se(df, var_name, var_name)
}

wrapr_increment_se(d, 'a')
```

    ##   a
    ## 1 2

### NSE version

Non-standard evaluation interface:

``` r
wrapr_add_one_nse <- function(df, res_var, input_var) {
  wrapr::let(
    c(RESVAR= substitute(res_var),
      INPUTVAR= substitute(input_var)),
    df %>%
      mutate(RESVAR = INPUTVAR + 1)
  )
}

wrapr_add_one_nse(d, res, a)
```

    ##   a res
    ## 1 1   2

`wrapr::let()`'s NSE composition pattern seems to work even when applied to itself:

``` r
wrapr_increment_nse <- function(df, var) {
  wrapr::let(
    c(VAR= substitute(var)),
    wrapr_add_one_nse(df, VAR, VAR)
  )
}

wrapr_increment_nse(d, a)
```

    ##   a
    ## 1 2

### Abstract Syntax Tree Version

Or, if you are uncomfortable with macros being implemented through string-substitution one can use `wrapr::let()` in "language mode" (where it works directly on abstract syntax trees).

#### SE re-do

``` r
wrapr_add_one_se <- function(df, res_var_name, input_var_name) {
  wrapr::let(
    c(RESVAR= res_var_name,
      INPUTVAR= input_var_name),
    df %>%
      mutate(RESVAR = INPUTVAR + 1),
    subsMethod= 'langsubs'
  )
}

wrapr_add_one_se(d, 'res', 'a')
```

    ##   a res
    ## 1 1   2

``` r
wrapr_increment_se <- function(df, var_name) {
  wrapr_add_one_se(df, var_name, var_name)
}

wrapr_increment_se(d, 'a')
```

    ##   a
    ## 1 2

### NSE re-do

``` r
wrapr_add_one_nse <- function(df, res_var, input_var) {
  wrapr::let(
    c(RESVAR= substitute(res_var),
      INPUTVAR= substitute(input_var)),
    df %>%
      mutate(RESVAR = INPUTVAR + 1),
    subsMethod= 'langsubs'
  )
}

wrapr_add_one_nse(d, res, a)
```

    ##   a res
    ## 1 1   2

``` r
wrapr_increment_nse <- function(df, var) {
  wrapr::let(
    c(VAR= substitute(var)),
    wrapr_add_one_nse(df, VAR, VAR),
    subsMethod= 'langsubs'
  )
}

wrapr_increment_nse(d, a)
```

    ##   a
    ## 1 2

Conclusion
----------

`tidyeval`/`rlang` provides general tools to compose or daisy-chain non-standard-evaluation functions (i.e., write new non-standard-evaluation functions in terms of others. This tries to abrogate the issue that it can be hard to compose non-standard function interfaces (i.e., one can not [parameterize them or program over them](https://www.youtube.com/watch?v=iKLGxzzm9Hk) without a tool such as `tidyeval`/`rlang`). In contrast `wrapr::let()` concentrates on standard evaluation, providing a tool that allows one to re-wrap non-standard-evaluation interfaces as standard evaluation interfaces.

A lot of the `tidyeval`/`rlang` design is centered on treating variable names as lexical closures that capture an environment they should be evaluated in. This does make them more like general `R` functions (which also have this behavior).

However, creating so many long-term bindings is actually counter to some common data analyst practice.

The `my_mutate(df, expr)` example itself from `vignette('programming', package = 'dplyr')` even shows the pattern I am referring to: the analyst transiently pairs a chosen concrete data set to abstract variable names. One argument is the data and the other is the expression to be applied to that data (and only that data, with clean code not capturing values from environments).

Many methods are written expecting to be re-run on different data (for example `predict()`). This has the huge advantage that it documents your intent to change out what data is being applied (such as running a procedure twice, once on training data and once on future application data).

This is a principle we also strongly apply in our [join controller](http://www.win-vector.com/blog/2017/06/use-a-join-controller-to-document-your-work/) which has no issue sharing variables out as an external spreadsheet, because it thinks of variable names (here meaning columns names) as fundamentally being strings (not as `quosures` temporally working "under cover" in string representations).
