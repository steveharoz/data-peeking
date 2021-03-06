Data Peeking Is Worse than You Thought
================
by Steve Haroz

An R version of the simulation of data peeking (optional stopping) by Sam Schwarzkopf.

Explanation and Matlab version at <https://neuroneurotic.net/2016/08/25/realistic-data-peeking-isnt-as-bad-as-you-thought-its-worse/>

``` r
# install.packages('MASS') # for mvrnorm
library(ggplot2)
library(dplyr)
library(parallel)
```

``` r
# significant level
significantP = 0.05
# multiple rho values
rhos = seq(0,.9, .1)
```

Define the main simulation function
-----------------------------------

``` r
SimulOptStopCorr = function (rho, significantP = 0.05) {
  
  # Number of simulations
  Ni = 5000
  # Maximum sample size
  mN = 150
  # Starting sample size
  sN = 3
  
  # Non-significance levels
  LowP = 0.1
  MedP = 0.3
  HigP = 0.5
  
  # results data.frame
  row_count = Ni*4
  Pvals = data.frame(
    rho      = rep(rho, row_count),
    category = rep(NA, row_count), 
    n        = rep(NA, row_count), 
    r        = rep(NA, row_count), 
    p        = rep(NA, row_count)
  )
  
  # Simulate Ni times
  for (i in 1:Ni) {
    # Criteria fulfilled?
    CriterionReached = c(0,0,0,0)
    
    # Full data set
    X = MASS::mvrnorm(mN, c(0,0), matrix(c(1,rho,rho,1), 2))
    
    # Starting with small sample
    n = sN; 
    
    while (mean(CriterionReached) < 1) {
      # Analyse current sample
      currentAnalysis = cor.test(X[1:n,1], X[1:n,2])
      r = currentAnalysis$estimate
      p = currentAnalysis$p.value
      
      # Significant p only
      if ((p < significantP || n == mN) && CriterionReached[1]==0) {
        CriterionReached[1] = 1
        Pvals[(i-1)*4+1,] = list(rho, 'Significant only', n, r, p) 
      }
      # Significant p or low p 
      if ((p < significantP || p > LowP || n == mN) && CriterionReached[2]==0) {
        CriterionReached[2] = 1
        Pvals[(i-1)*4+2,] = list(rho, 'Significant or p > 0.1', n, r, p) 
      }
      # Significant p or medium p 
      if ((p < significantP || p > MedP || n == mN) && CriterionReached[3]==0) {
        CriterionReached[3] = 1
        Pvals[(i-1)*4+3,] = list(rho, 'Significant or p > 0.3', n, r, p) 
      }
      # Significant p or high p 
      if ((p < significantP || p > HigP || n == mN) && CriterionReached[4]==0) {
        CriterionReached[4] = 1
        Pvals[(i-1)*4+4,] = list(rho, 'Significant or p > 0.5', n, r, p) 
      }
      
      # Increase sample size
      n = n + 1
      if (n > mN)
        # Don't allow it to surpass maximum
        n = mN
    }
  }
  
  # output a status update
  message('Finished rho = ', rho)
  flush.console()
  
  return(Pvals)
}
```

Run the simulation
------------------

``` r
# Calculate the number of cores
coreCount = max(1, detectCores() - 1)
# Initiate cluster
cluster = makeCluster(coreCount)

# run the simulation for each rho on a different core
system.time(Pvals <- parLapply(cluster, rhos, SimulOptStopCorr))
# don't need parallelism any more
stopCluster(cluster)

# combine sim results into one big data frame
Pvals = bind_rows(Pvals)
```

    ##    user  system elapsed 
    ##    0.12    0.00  265.10

Plot the positive hit rate
--------------------------

``` r
# determine the proportion significant for each condition
proportionData = Pvals %>%
  group_by(category, rho) %>%
  summarise(proportionSignificant = mean(p < significantP))

# simple graph theme with no gridlines
mytheme = 
  theme_classic(15) +
  theme(
    axis.line.x = element_line(colour='black', size=1, linetype='solid'),
    axis.line.y = element_line(colour='black', size=1, linetype='solid'))

# false positive plot
ggplot(proportionData) +
  aes(x=rho, y=proportionSignificant, color=category, group=category) +
  geom_hline(yintercept=significantP, linetype='dashed', alpha=0.5) +
  geom_line(size=1.5) +
  geom_point(shape=21, fill='white', size=3, stroke=1.5) +
  # make it fancy
  geom_text( hjust=0, nudge_x = .03,
    data=filter(proportionData, rho == max(rho)), 
    aes(label=category)) +
  scale_x_continuous(limits=c(0, 1.2), breaks=seq(min(rhos), max(rhos), 0.1)) +
  scale_color_brewer(palette = 'Set1', guide='none') +
  labs(
    x = expression(paste('True effect size (', rho, ')')),
    y = paste0('Proportion (p < ', significantP, ')'),
    color = NULL
  ) +
  mytheme
```

![](data_peeking_files/figure-markdown_github/hit_rate_plot-1.png)

False positive proportion (rho = 0)
-----------------------------------

``` r
proportionData %>% 
  filter(rho == 0) %>%
  tidyr::spread(category, proportionSignificant) %>% 
  knitr::kable()
```

|  rho|  Significant only|  Significant or p &gt; 0.1|  Significant or p &gt; 0.3|  Significant or p &gt; 0.5|
|----:|-----------------:|--------------------------:|--------------------------:|--------------------------:|
|    0|            0.4038|                     0.0556|                      0.112|                     0.1544|

D' Analysis
-----------

``` r
d_prime = function(pH, pFA) {
  # Returns d-prime for given hit rate and false alarms:  z(pH)-z(pFA)
  # Second argument returns the response bias:    (z(pH)+z(pFA))/2
  # Third argument returns the criterion:     -z(pFA)
  
  pH = pmin(pmax(0.0001, pH), 0.9999)

  return(qnorm(pH) - qnorm(pFA))
}

# get dprime for the "proportion significant" data
dprimeData = proportionData %>% 
  group_by(category) %>%
  mutate(proportionFalsePositive = min(proportionSignificant)) %>%
  mutate(sensitivity = d_prime(proportionSignificant, proportionFalsePositive)) %>%
  # drop extreme values
  filter(sensitivity < 3)

# plot it
ggplot(dprimeData) +
  aes(x=rho, y=sensitivity, color=category, group=category) +
  geom_hline(yintercept=0, linetype='dashed', alpha=0.5) +
  geom_line(size=1.5) +
  geom_point(shape=21, fill='white', size=3, stroke=1.5) +
  # make it fancy
  geom_text( hjust=0, nudge_x = .03,
    data=dprimeData %>% group_by(category) %>% filter(rho == max(rho)),
    aes(label=category)) +
  scale_x_continuous(limits=c(0, 1.2), breaks=seq(min(rhos), max(rhos), 0.1)) +
  scale_color_brewer(palette = 'Set1', guide='none') +
  labs(
    x = expression(paste('True effect size (', rho, ')')),
    y = paste0('Proportion (p < ', significantP, ')'),
    color = NULL
  ) +
  mytheme
```

![](data_peeking_files/figure-markdown_github/d_prime-1.png)
