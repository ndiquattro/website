---
title: Introducing kflow
author: Nick DiQuattro
date: '2020-01-26'
slug: introducing-kflow
categories: [kubeflow]
tags: [r]
---

The ambition of kflow is to make it easier to build R based components orchestrated by Google's [Kubeflow](https://www.kubeflow.org/). Importantly, this package does *not* intend to be a full R replacement for the [python SDK](https://github.com/kubeflow/pipelines) (at least not yet!). However, I've had some good luck in wrapping the python SDK with [reticulate](https://rstudio.github.io/reticulate/), so if you need to go full R, that would be a good option.

## Installation

You can install the development version from [GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("ndiquattro/kflow")
```
## Example Usage

To illustrate how to use {kflow} we'll set up a simple component example where we predict the transmission type of a car in `mtcars` based on an input parameter. We will work with a single function that will eventually be translated to a single kubeflow component.

Note that our argument names need to follow a convention for the conversion to component to succeed. Each argument must end in a slug that identifies the argument type. The conversions for slug to kubeflow type are:

*Inputs*

* _string = String
* _int = Integer
* _bool = Bool
* _float = Float

*Outputs*

* _out = outputPath
* _metrics = Metrics
* _uimeta = UI_metadata

With all that defined, let's create the function:

```{r}
library(kflow)

tm_predict <- function(predictor_string, file_out, performance_metrics, curve_uimeta) {
  
  # Train Model
  cars_dat <- mtcars
  cars_dat$am <- factor(cars_dat$am)
  
  form <- as.formula(paste0("am ~ ", predictor_string))
  model <- glm(form, binomial, cars_dat)
  
  # Make Predictions
  cars_dat$prob_auto <- predict(model, type = "response")
  
  # Save results
  kf_write_output(cars_dat, file_out)  # This ensures the path exists then writes to a kubeflow provided path
  
  # Score and save metrics
  kf_init_metrics() %>%  # Start an empy JSON
    kf_add_metric(name = "roc", value = yardstick::roc_auc(cars_dat, am, prob_auto)$.estimate, format = "RAW") %>% 
    kf_add_metric(name = "pr-auc", value = yardstick::pr_auc(cars_dat, am, prob_auto)$.estimate, format = "RAW") %>% 
    kf_write_output(curve_uimeta)
  
  # Save ROC Curve
  roc_file <- tempfile()
  yardstick::roc_curve(test_preds_org, observed, estimated) %>%
    mutate(specificity = 1 - specificity) %>%   # convert to FPR
    filter(is.finite(.threshold)) %>%   # KF not going to like -Inf to Inf
    write.csv(roc_file, col_names = FALSE)  # Save without headers
  
  kf_init_ui_meta() %>% 
    kf_add_roc(roc_file)
}
```

```{r}
component <-
  kf_make_component(
    "tm_predict",
    "Transmission Predictor",
    "Predicts if a car has an automatic transmission based on a provided variable",
    "rocker/tidyverse:3.6.2"
  )

cat(component, sep = "\n")
```

Next let's take a look at an example of how the metrics/ui meta functions work. Essentially they are just helpers for creating JSON in a structure kubeflow expects. They can be written by `kf_write_output()` just like any other information we want to save.

You can also inspect the JSON as you go. First create the base:

```{r}
base_metrics <- kf_init_metrics()
base_metrics
```

Then add a metric:

```{r}
base_metrics %>% 
  kf_add_metric(
    name = "coolness-factor",
    value = 100,
    format = "RAW"
  )
```

You can chain as many metrics together as you'd like:

```{r}
base_metrics %>% 
  kf_add_metric(
    name = "coolness-factor",
    value = 100,
    format = "RAW"
  ) %>% 
  kf_add_metric(
    name = "badness-factor",
    value = 0,
    format = "RAW"
  )
```

When written to a `_metrics` or `_uimeta` path they will show up in the kubeflow UI!
