
<!-- README.md is generated from README.Rmd. Please edit that file -->

# shinytune

<!-- badges: start -->

<!-- badges: end -->

The goal of shinytune is to make it easy to explore `tune` objects,
similar to `shinystan`.

To do this I need to:

  - \[ \] Figure out exactly what `tune` is producing
  - \[ \] Think about some sensible
    summaries/visualisations/explorations that could be applied to that
  - \[ \] (Optionally) Compare to `shinystan` for reference (I don’t
    want to anchor too strongly to it, though)

## Exploring `tune`

Firstly, I need to figure out what `tune` object actually contains.

Let’s create one following the Getting Started guide on the [tune
website](https://tidymodels.github.io/tune/articles/getting_started.html).

``` r
library(tidymodels)
#> ── Attaching packages ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── tidymodels 0.0.3 ──
#> ✓ broom     0.5.2     ✓ purrr     0.3.3
#> ✓ dials     0.0.4     ✓ recipes   0.1.9
#> ✓ dplyr     0.8.4     ✓ rsample   0.0.5
#> ✓ ggplot2   3.2.1     ✓ tibble    2.1.3
#> ✓ infer     0.5.1     ✓ yardstick 0.0.5
#> ✓ parsnip   0.0.5
#> ── Conflicts ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── tidymodels_conflicts() ──
#> x purrr::discard()    masks scales::discard()
#> x dplyr::filter()     masks stats::filter()
#> x dplyr::lag()        masks stats::lag()
#> x ggplot2::margin()   masks dials::margin()
#> x recipes::step()     masks stats::step()
#> x recipes::yj_trans() masks scales::yj_trans()
library(tune)
```

### Tune objects

Let’s just create the whole set of outputs for the Getting Started
vignette.

Firstly we’ll set up the data:

``` r
library(AmesHousing)

ames <- make_ames()

set.seed(4595)
data_split <- initial_split(ames, strata = "Sale_Price")
ames_train <- training(data_split)
ames_test  <- testing(data_split)
```

Then the baseline
recipe

``` r
ames_rec <- recipe(Sale_Price ~ Longitude + Latitude, data = ames_train) %>% 
  step_log(Sale_Price, base = 10) %>% 
  step_ns(Longitude, deg_free = tune("long df")) %>% 
  step_ns(Latitude,  deg_free = tune("lat df"))

ames_rec
#> Data Recipe
#> 
#> Inputs:
#> 
#>       role #variables
#>    outcome          1
#>  predictor          2
#> 
#> Operations:
#> 
#> Log transformation on Sale_Price
#> Natural Splines on Longitude
#> Natural Splines on Latitude
```

Then update the parameters to use a better function with a wider range:

``` r
ames_param <- ames_rec %>% 
  parameters() %>% 
  update(
    `long df` = spline_degree(), 
    `lat df` = spline_degree()
  )

ames_param
#> Collection of 2 parameters for tuning
#> 
#>       id parameter type object class
#>  long df       deg_free    nparam[+]
#>   lat df       deg_free    nparam[+]
```

Then we’ll set up the (grid) search space of values for these
parameters:

``` r
spline_grid <- grid_max_entropy(ames_param, size = 10)
spline_grid
#> # A tibble: 10 x 2
#>    `long df` `lat df`
#>        <int>    <int>
#>  1         3        6
#>  2        10       10
#>  3         8        7
#>  4         3       10
#>  5         7       10
#>  6         4        3
#>  7         4        8
#>  8         7        4
#>  9         5        6
#> 10        10        5
```

Then our (linear) model:

``` r
lm_mod <- linear_reg() %>% 
    set_engine("lm")

lm_mod
#> Linear Regression Model Specification (regression)
#> 
#> Computational engine: lm
```

Then we’ll set up the cross validation scheme to search over:

``` r
set.seed(2453)
cv_splits <- vfold_cv(ames_train, v = 10, strata = "Sale_Price")
```

Then finally we’ll do the tuning using `tune_grid()`:

``` r
ames_res <- tune_grid(
    ames_rec,
    model = lm_mod,
    resamples = cv_splits,
    grid = spline_grid
)
ames_res
#> #  10-fold cross-validation using stratification 
#> # A tibble: 10 x 4
#>    splits           id     .metrics          .notes          
#>  * <list>           <chr>  <list>            <list>          
#>  1 <split [2K/221]> Fold01 <tibble [20 × 5]> <tibble [0 × 1]>
#>  2 <split [2K/220]> Fold02 <tibble [20 × 5]> <tibble [0 × 1]>
#>  3 <split [2K/220]> Fold03 <tibble [20 × 5]> <tibble [0 × 1]>
#>  4 <split [2K/220]> Fold04 <tibble [20 × 5]> <tibble [0 × 1]>
#>  5 <split [2K/220]> Fold05 <tibble [20 × 5]> <tibble [0 × 1]>
#>  6 <split [2K/220]> Fold06 <tibble [20 × 5]> <tibble [0 × 1]>
#>  7 <split [2K/220]> Fold07 <tibble [20 × 5]> <tibble [0 × 1]>
#>  8 <split [2K/220]> Fold08 <tibble [20 × 5]> <tibble [0 × 1]>
#>  9 <split [2K/220]> Fold09 <tibble [20 × 5]> <tibble [0 × 1]>
#> 10 <split [2K/218]> Fold10 <tibble [20 × 5]> <tibble [0 × 1]>
```
