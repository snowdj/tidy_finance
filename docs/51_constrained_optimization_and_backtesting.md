# Constrained optimization and backtesting

\index{backtesting} In this chapter, we conduct portfolio backtesting in a more realistic setting by including transaction costs and investment constraints such as no-short-selling rules. \index{short-selling}
We start with *standard* mean-variance efficient portfolios. \index{Efficient portfolio}
Then, we introduce further constraints step-by-step. 
Numerical constrained optimization is performed by the packages `quadprog` (for quadratic objective functions such as in typical mean-variance framework) and `alabama` (for more general, non-linear objectives and constraints).\index{Optimization}


```r
library(tidyverse)
library(RSQLite)
library(quadprog)
library(alabama)
library(scales)
```

## Data preparation

We start by loading the required data from our `SQLite`-database introduced in our chapter on *"Accessing & managing financial data"*. For simplicity, we restrict our investment universe to the monthly Fama-French industry portfolio returns in the following application. \index{Data!Industry Returns}


```r
tidy_finance <- dbConnect(SQLite(), "data/tidy_finance.sqlite",
  extended_types = TRUE
)

industry_returns <- tbl(tidy_finance, "industries_ff_monthly") |>
  collect()

industry_returns <- industry_returns |>
  select(-month)
```

## Recap of portfolio choice 

A common objective for portfolio optimization is to find mean-variance efficient portfolio weights, i.e., the allocation which delivers the lowest possible return variance for a given minimum level of expected returns. 
In the most extreme case, where the investor is only concerned about portfolio variance, she may choose to implement the minimum variance portfolio (MVP) weights which are given by the solution to 
$$w_\text{mvp} = \arg\min w'\Sigma w \text{ s.t. } w'\iota = 1$$
where $\Sigma$ is the $(N \times N)$ covariance matrix of the returns. The optimal weights $\omega_\text{mvp}$ can be found analytically and are $\omega_\text{mvp} = \frac{\Sigma^{-1}\iota}{\iota'\Sigma^{-1}\iota}$. In terms of code, the math is equivalent to the following. \index{Minimum variance portfolio}


```r
Sigma <- cov(industry_returns)
w_mvp <- solve(Sigma) %*% rep(1, ncol(Sigma))
w_mvp <- as.vector(w_mvp / sum(w_mvp))
```

Next, consider an investor who aims to achieve minimum variance *given a required expected portfolio return* $\bar{\mu}$ such that she chooses
$$w_\text{eff}({\bar{\mu}}) =\arg\min w'\Sigma w \text{ s.t. } w'\iota = 1 \text{ and } \omega'\mu \geq \bar{\mu}.$$
It can be shown (see Exercises) that the portfolio choice problem can equivalently be formulated for an investor with mean-variance preferences and risk aversion factor $\gamma$. The investor aims to choose portfolio weights such that
$$ w^*_\gamma = \arg\max w' \mu - \frac{\gamma}{2}w'\Sigma w\quad \text{ s.t. } w'\iota = 1.$$
The solution to the optimal portfolio choice problem is:
$$\omega^*_{\gamma}  = \frac{1}{\gamma}\left(\Sigma^{-1} - \frac{1}{\iota' \Sigma^{-1}\iota }\Sigma^{-1}\iota\iota' \Sigma^{-1} \right) \mu  + \frac{1}{\iota' \Sigma^{-1} \iota }\Sigma^{-1} \iota.$$
Empirically, this classical solution imposes many problems. 
In particular, the estimates of $\mu_t$ are noisy over short horizons, the ($N \times N$) matrix $\Sigma_t$ contains $N(N-1)/2$ distinct elements and thus, estimation error is huge. 
Seminal papers on the effect of ignoring estimation uncertainty, among others, are @Brown1976, @Jobson1980, @Jorion1986, and @Chopra1993.

Even worse, if the asset universe contains more assets than available time periods $(N > T)$, the sample covariance matrix is no longer positive definite such that the inverse $\Sigma^{-1}$ does not exist anymore. 
To address estimation issues for vast-dimensional covariance matrices, regularization techniques are a popular tool, see, e.g., @Ledoit2003, @Ledoit2004, @Ledoit2012, and @Fan2008.

Model uncertainty due to multiple feasible data generating processes in the context of portfolio choice is investigated, for instance, by @Wang2005, @Garlappi2007, and @Pflug2012. \index{Transaction costs}

On top of the estimation uncertainty, *transaction costs* are a major concern. 
Rebalancing portfolios is costly, and, therefore, the optimal choice should depend on the investor's current holdings. In the presence of transaction costs, the benefits of reallocating wealth may be smaller than the costs associated with turnover. 

This aspect has been investigated theoretically, among others, for one risky asset by @Magill1976 and @Davis1990. 
Subsequent extensions to the case with multiple assets have been proposed by @Balduzzi1999 and @Balduzzi2000. 
More recent papers on empirical approaches which explicitly account for transaction costs include @Garleanu2013, and @DeMiguel2014, and @DeMiguel2015. 

## Estimation uncertainty and transaction costs

The empirical evidence regarding the performance of a mean-variance optimization procedure in which you simply plug in some sample estimates $\hat \mu_t$ and $\hat \Sigma_t$ can be summarized rather briefly: mean-variance optimization performs poorly! The literature discusses many proposals to overcome these empirical issues. For instance, one may impose some form of regularization of $\Sigma$, rely on Bayesian priors inspired by theoretical asset pricing models [@Kan2007] or use high-frequency data to improve forecasting [@Hautsch2015]. 
One unifying framework that works easily, effectively (even for large dimensions), and is purely inspired by economic arguments is an ex-ante adjustment for transaction costs [@Hautsch2019]. 

Assume that returns are from a multivariate normal distribution with mean $\mu$ and variance-covariance matrix $\Sigma$, $N(\mu,\Sigma)$. Additionally, we assume quadratic transaction costs which penalize rebalancing such that  $$
\begin{aligned}
\nu\left(\omega_{t+1},\omega_{t^+}, \beta\right) := \frac{\beta}{2} \left(\omega_{t+1} - \omega_{t^+}\right)'\left(\omega_{t+1}- \omega_{t^+}\right),\end{aligned}$$
with cost parameter $\beta>0$ and $\omega_{t^+} := {\omega_t \circ  (1 +r_{t})}/{\iota' (\omega_t \circ (1 + r_{t}))}$. 
Intuitively, transaction costs penalize portfolio performance when the portfolio is shifted from the current holdings $\omega_{t^+}$ to a new allocation $\omega_{t+1}$. 
In this setup, transaction costs do not increase linearly such that larger rebalancing is penalized more heavily than small adjustments. 
Note that $\omega_{t^+}$ differs mechanically from $\omega_t$ due to the returns in the past period.  
Then, the optimal portfolio choice for an investor with mean variance preferences is
$$\begin{aligned}\omega_{t+1} ^* &:=  \arg\max \omega'\mu - \nu_t (\omega,\omega_{t^+}, \beta) - \frac{\gamma}{2}\omega'\Sigma\omega \text{ s.t. } \iota'\omega = 1\\
&=\arg\max
\omega'\mu^* - \frac{\gamma}{2}\omega'\Sigma^* \omega \text{ s.t.} \iota'\omega=1,\end{aligned}$$
where
$$\mu^*:=\mu+\beta \omega_{t^+} \quad  \text{and} \quad \Sigma^*:=\Sigma + \frac{\beta}{\gamma} I_N.$$
As a result, adjusting for transaction costs implies a standard mean-variance optimal portfolio choice with adjusted return parameters $\Sigma^*$ and $\mu^*$: $$\omega^*_{t+1} = \frac{1}{\gamma}\left(\Sigma^{*-1} - \frac{1}{\iota' \Sigma^{*-1}\iota }\Sigma^{*-1}\iota\iota' \Sigma^{*-1} \right) \mu^*  + \frac{1}{\iota' \Sigma^{*-1} \iota }\Sigma^{*-1} \iota.$$

An alternative formulation of the optimal portfolio can be derived as follows: 
$$\omega_{t+1} ^*=\arg\max
\omega'\left(\mu+\beta\left(\omega_{t^+} - \frac{1}{N}\iota\right)\right) - \frac{\gamma}{2}\omega'\Sigma^* \omega \text{ s.t. } \iota'\omega=1.$$
The optimal weights correspond to a mean-variance portfolio where the vector of expected returns is such that assets that currently exhibit a higher weight are considered as delivering a higher expected return. 

## Optimal portfolio choice

The function below implements the efficient portfolio weight in its general form, allowing for transaction costs (conditional on the holdings *before* reallocation). 
For $\beta=0$, the computation resembles the standard mean-variance efficient framework. `gamma` denotes the coefficient of risk aversion $\gamma$,`beta` is the transaction cost parameter $\beta$ and `w_prev` are the weights before rebalancing $\omega_{t^+}$. 


```r
compute_efficient_weight <- function(Sigma,
                                     mu,
                                     gamma = 2,
                                     beta = 0, # transaction costs
                                     w_prev = rep(
                                       1 / ncol(Sigma),
                                       ncol(Sigma)
                                     )) {
  iota <- rep(1, ncol(Sigma))
  Sigma_processed <- Sigma + beta / gamma * diag(ncol(Sigma))
  mu_processed <- mu + beta * w_prev

  Sigma_inverse <- solve(Sigma_processed)

  w_mvp <- Sigma_inverse %*% iota
  w_mvp <- as.vector(w_mvp / sum(w_mvp))
  w_opt <- w_mvp + 1 / gamma *
    (Sigma_inverse - w_mvp %*% t(iota) %*% Sigma_inverse) %*%
      mu_processed
  return(as.vector(w_opt))
}

mu <- colMeans(industry_returns)
compute_efficient_weight(Sigma, mu)
```

```
 [1]  1.428  0.270 -1.302  0.375  0.308 -0.152  0.544  0.472 -0.167 -0.776
```

The portfolio weights above indicate the efficient portfolio for an investor with risk aversion coefficient $\gamma=2$ in absence of transaction costs. Some of the positions are negative which implies short-selling, most of the positions are rather extreme. For instance, a position of $-1$ implies that the investor takes a short position worth her entire wealth to lever long positions in other assets. \index{Short-selling}
What is the effect of transaction costs or different levels of risk aversion on the optimal portfolio choice? The following few lines of code analyze the distance between the minimum variance portfolio and the portfolio implemented by the investor for different values of the transaction cost parameter $\beta$ and risk aversion $\gamma$.


```r
transaction_costs <- expand_grid(
  gamma = c(2, 4, 8, 20),
  beta = 20 * qexp((1:99) / 100)
) |>
  mutate(
    weights = map2(
      .x = gamma,
      .y = beta,
      ~ compute_efficient_weight(Sigma,
        mu,
        gamma = .x,
        beta = .y / 10000,
        w_prev = w_mvp
      )
    ),
    concentration = map_dbl(weights, ~ sum(abs(. - w_mvp)))
  )
```
The code chunk above computes the optimal weight in presence of transaction cost for different values of $\beta$ and $\gamma$ but with the same initial allocation, the theoretical optimal minimum variance portfolio. 
Starting from the initial allocation, the investor chooses her optimal allocation along the efficient frontier to reflect her own risk preferences. 
If transaction costs would be absent, the investor would simply implement the mean-variance efficient allocation. If transaction costs make it costly to rebalance, her optimal portfolio choice reflects a shift towards the efficient portfolio, whereas her current portfolio *anchors* her investment. 


```r
transaction_costs |>
  mutate(risk_aversion = as_factor(gamma)) |>
  ggplot(aes(
    x = beta,
    y = concentration,
    color = risk_aversion
  )) +
  geom_line() +
  labs(
    x = "Transaction cost parameter",
    y = "Distance from MVP",
    color = "Risk aversion",
    title = "Portfolio weights for different risk aversion and transaction cost"
  )
```

<div class="figure">
<img src="51_constrained_optimization_and_backtesting_files/figure-html/fig511-1.png" alt="The figure shows four lines that indicate that rebalancing from the initial portfolio decreases for increasing transaction costs and for higher risk aversion." width="672" />
<p class="caption">(\#fig:fig511)Portfolio weights for different risk aversion and transaction cost</p>
</div>

The figure shows that the initial portfolio is always the (sample) minimum variance portfolio and that the higher the transaction costs parameter $\beta$, the smaller is the rebalancing from the initial portfolio (which we always set to the minimum variance portfolio weights in this example). In addition, if risk aversion $\gamma$ increases, the efficient portfolio is closer to the minimum variance portfolio weights such that the investor desires less rebalancing from the initial holdings. 

## Constrained optimization

Next, we introduce constraints to the above optimization procedure. 
Very often, typical constraints such as short-selling restrictions prevent analytical solutions for optimal portfolio weights (Short-selling restrictions simply imply that negative weights are not allowed such that we require that $w_i \geq 0 \forall i$). 
However, numerical optimization allows computing the solutions to such constrained problems. For the purpose of mean-variance optimization, we rely on the `solve.QP()` function from the package `quadprog`. 

The function `solve.QP()` delivers numerical solutions to quadratic programming problems of the form 
$$\min(-\mu \omega + 1/2 \omega' \Sigma \omega) \text{ s.t. } A' \omega >= b_0.$$
The function takes one argument (`meq`) for the number of equality constraints. Therefore, the above matrix $A$ is simply a vector of ones to ensure that the weights sum up to one. In the case of short-selling constraints, the matrix $A$ is of the form 
$$A' = \begin{pmatrix}1 & 1& \ldots&1 \\1 & 0 &\ldots&0\\0 & 1 &\ldots&0\\\vdots&&\ddots&\vdots\\0&0&\ldots&1\end{pmatrix}'\qquad b_0 = \begin{pmatrix}1\\0\\\vdots\\0\end{pmatrix}.$$

Before we dive into constrained optimization, we revisit the *unconstrained* problem and replicate the analytical solutions for the minimum variance and efficient portfolio weights from above. We verify that the output is equal to the above solution. 
Note that `near()` is a safe way to compare if two vectors are pairwise equal. The alternative `==` is sensitive to small differences that may occur due to the representation of floating points on a computer while `near()` has a built in tolerance. As just discussed, we set `Amat` to a matrix with a column of ones and `bvec` to 1 to enforce the constraint that weights must sum up to one. `meq=1` means that one (out of one) constraints must be satisfied with equality.   


```r
n_industries <- ncol(industry_returns)

w_mvp_numerical <- solve.QP(
  Dmat = Sigma,
  dvec = rep(0, n_industries),
  Amat = cbind(rep(1, n_industries)),
  bvec = 1,
  meq = 1
)

all(near(w_mvp, w_mvp_numerical$solution))
```

```
[1] TRUE
```

```r
w_efficient_numerical <- solve.QP(
  Dmat = 2 * Sigma,
  dvec = mu,
  Amat = cbind(rep(1, n_industries)),
  bvec = 1,
  meq = 1
)

all(near(compute_efficient_weight(Sigma, mu), w_efficient_numerical$solution))
```

```
[1] TRUE
```

The result above shows that indeed the numerical procedure recovered the optimal weights for a scenario where we already know the analytic solution. 
For more complex optimization routines, [R's optimization task view](https://cran.r-project.org/web/views/Optimization.html) provides an overview of the wast optimization landscape in R. \index{Numerical optimization}

Next, we approach problems where no analytical solutions exist. First, we additionally impose short-sale constraints, which implies $N$ inequality constraints if the form $w_i >=0$. 


```r
w_no_short_sale <- solve.QP(
  Dmat = 2 * Sigma,
  dvec = mu,
  Amat = cbind(1, diag(n_industries)),
  bvec = c(1, rep(0, n_industries)),
  meq = 1
)
w_no_short_sale$solution
```

```
 [1]  6.34e-01 -1.91e-17  1.20e-16 -3.47e-18  6.78e-18 -6.24e-17  1.02e-01
 [8]  2.64e-01  3.38e-22 -2.22e-16
```
As expected, the resulting portfolio weights are all positive (up to numerical precision). Typically, the holdings in presence of short-sale constraints are concentrated among way fewer assets than for the unrestricted case. 
You can verify that `sum(w_no_short_sale$solution)` returns 1. In other words: `solve.QP()` provides the numerical solution to a portfolio choice problem for a mean-variance investor with risk aversion `gamma = 2` where negative holdings are forbidden. 

`solve.QP()` is fast because it benefits from a very clear problem structure with a quadratic objective and linear constraints. However, optimization often requires more flexibility. As an example, we show how to compute optimal weights, subject to the so-called [regulation T-constraint](https://en.wikipedia.org/wiki/Regulation_T), which requires that the sum of all absolute portfolio weights is smaller than 1.5, that is $\sum\limits_{i=1}^N |w_i| \leq 1.5$. 
The constraint implies an initial margin requirement of 50% and, therefore, also a non-linear constraint. \index{Regulation T}
Thus, we can no longer rely on `solve.QP()` which is defined to solve quadratic programming problems with linear constraints. 
Instead, we rely on the package `alabama`, which requires a separate definition of objective and constraint functions. 


```r
initial_weights <- rep(
  1 / n_industries,
  n_industries
)

objective <- function(w, gamma = 2) {
  -t(w) %*% (1 + mu) +
    gamma / 2 * t(w) %*% Sigma %*% w
}

inequality_constraints <- function(w, reg_t = 1.5) {
  return(reg_t - sum(abs(w)))
}

equality_constraints <- function(w) {
  return(sum(w) - 1)
}

w_reg_t <- constrOptim.nl(
  par = initial_weights,
  hin = inequality_constraints,
  fn = objective,
  heq = equality_constraints,
  control.outer = list(trace = FALSE)
)
w_reg_t$par
```

```
 [1]  4.11e-01 -2.07e-02 -9.00e-02  3.37e-02  8.03e-02 -2.25e-08  3.08e-01
 [8]  3.52e-01  6.25e-02 -1.37e-01
```

Note that the function `constrOptim.nl()` requires a starting vector of parameter values, an initial portfolio. Under the hood, `alamaba` performs numerical optimization by searching for a local minimum of the function `objective()` (subject to the equality constraints `equality_constraints()` and the inequality constraints `inequality_constraints()`). 
In order to initialize the search procedure, a starting point has to be provided. 
*If* the algorithm identifies a global minimum, the starting point should not matter.


The figure below shows the optimal allocation weights across all 10 industries for the four different strategies considered so far: minimum variance, efficient portfolio with $\gamma$ = 2, efficient portfolio with short-sale constraints, and the Regulation-T constrained portfolio.


```r
tibble(
  `No short-sale` = w_no_short_sale$solution,
  `Minimum Variance` = w_mvp,
  `Efficient portfolio` = compute_efficient_weight(Sigma, mu),
  `Regulation-T` = w_reg_t$par,
  Industry = colnames(industry_returns)
) |>
  pivot_longer(-Industry,
    names_to = "Strategy",
    values_to = "weights"
  ) |>
  ggplot(aes(
    fill = Strategy,
    y = weights,
    x = Industry
  )) +
  geom_bar(position = "dodge", stat = "identity") +
  coord_flip() +
  labs(
    y = "Allocation weight",
    title = " Optimal allocations for different investment rules"
  ) +
  scale_y_continuous(labels = percent)
```

<div class="figure">
<img src="51_constrained_optimization_and_backtesting_files/figure-html/unnamed-chunk-9-1.png" alt="The figure shows the portfolio weights for the four different strategies across the 10 different industries. The figures indicates extreme long and short positions for the efficient portfolio." width="672" />
<p class="caption">(\#fig:unnamed-chunk-9)Summary of the optimal allocation weights for the 10 industry portfolios and the 4 different allocation strategies.</p>
</div>

The results clearly indicate the effect of imposing additional constraints: The extreme holdings the investor would implement if she follows the (theoretically optimal) efficient portfolio vanish under, e.g., the regulation-t constraint.
You may wonder why an investor would deviate from what is theoretically the optimal portfolio by imposing potentially arbitrary constraints. 
The short answer is: the *efficient portfolio* is only efficient if the true parameters of the data generating process correspond to the estimated parameters $\hat\Sigma$ and $\hat\mu$. 
Estimation uncertainty may thus lead to inefficient allocations. By imposing restrictions, we implicitly shrink the set of possible weights and prevent extreme allocations which could be the result of *error-maximization* due to estimation uncertainty [@Jagannathan2003].

Before we move on, we want to propose a final allocation strategy, which reflects a somewhat more realistic structure of transaction costs instead of the quadratic specification used above. The function below computes efficient portfolio weights while adjusting for transaction costs of the form $\beta\sum\limits_{i=1}^N |(w_{i, t+1} - w_{i, t^+})|$. No closed-form solution exists, and we rely on non-linear optimization procedures.


```r
compute_efficient_weight_L1_TC <- function(mu,
                                           Sigma,
                                           gamma = 2,
                                           beta = 0,
                                           initial_weights = rep(
                                             1 / ncol(Sigma),
                                             ncol(Sigma)
                                           )) {
  objective <- function(w) {
    -t(w) %*% mu +
      gamma / 2 * t(w) %*% Sigma %*% w +
      (beta / 10000) / 2 * sum(abs(w - initial_weights))
  }

  w_optimal <- constrOptim.nl(
    par = initial_weights,
    fn = objective,
    heq = function(w) {
      sum(w) - 1
    },
    control.outer = list(trace = FALSE)
  )

  return(w_optimal$par)
}
```

## Out-of-sample backtesting

For the sake of simplicity, we committed one fundamental error in computing portfolio weights above [@Harvey2019]. We used the full sample of the data to determine the optimal allocation. To implement this strategy at the beginning of the 2000s, you will need to know how the returns will evolve until 2020. \index{Backtesting} \index{Performance evaluation}
While interesting from a methodological point of view, we cannot evaluate the performance of the portfolios in a reasonable out-of-sample fashion. We do so next in a backtesting application for three strategies. For the backtest, we recompute optimal weights just based on past available data. 
The few lines below define the general setup. We consider 120 periods from the past to update the parameter estimates before recomputing portfolio weights. Then, we update portfolio weights which is costly and affects the performance. The portfolio weights determine the portfolio return. A period later, the current portfolio weights have changed and form the foundation for transaction costs incurred in the next period. We consider three different competing strategies: the mean-variance efficient portfolio, the mean-variance efficient portfolio with ex-ante adjustment for transaction costs, and the naive portfolio, which allocates wealth equally across the different assets.


```r
window_length <- 120
periods <- nrow(industry_returns) - window_length

beta <- 50
gamma <- 2

performance_values <- matrix(NA,
  nrow = periods,
  ncol = 3
) # A matrix to collect all returns
colnames(performance_values) <- c("raw_return", "turnover", "net_return")

performance_values <- list(
  "MV (TC)" = performance_values,
  "Naive" = performance_values,
  "MV" = performance_values
)

w_prev_1 <- w_prev_2 <- w_prev_3 <- rep(
  1 / n_industries,
  n_industries
)
```

We also define two helper functions: one to adjust the weights due to returns and one for performance evaluation, where we compute realized returns net of transaction costs. 


```r
adjust_weights <- function(w, next_return) {
  w_prev <- 1 + w * next_return
  as.numeric(w_prev / sum(as.vector(w_prev)))
}

evaluate_performance <- function(w, w_previous, next_return, beta = 50) {
  raw_return <- as.matrix(next_return) %*% w
  turnover <- sum(abs(w - w_previous))
  net_return <- raw_return - beta / 10000 * turnover
  c(raw_return, turnover, net_return)
}
```

The following code chunk performs rolling-window estimation. In each period, the estimation window contains the returns available up to the current period. 
Note that we use the sample variance covariance matrix and ignore estimation of $\hat\mu$ entirely, but you might use more advanced estimators in practice. 


```r
for (p in 1:periods) {
  returns_window <- industry_returns[p:(p + window_length - 1), ]
  next_return <- industry_returns[p + window_length, ] |> as.matrix()

  Sigma <- cov(returns_window)
  mu <- 0 * colMeans(returns_window)

  # Transaction-cost adjusted portfolio
  w_1 <- compute_efficient_weight_L1_TC(
    mu = mu,
    Sigma = Sigma,
    beta = beta,
    gamma = gamma,
    initial_weights = w_prev_1
  )

  performance_values[[1]][p, ] <- evaluate_performance(w_1,
    w_prev_1,
    next_return,
    beta = beta
  )

  w_prev_1 <- adjust_weights(w_1, next_return)

  # Naive portfolio
  w_2 <- rep(1 / n_industries, n_industries)

  performance_values[[2]][p, ] <- evaluate_performance(
    w_2,
    w_prev_2,
    next_return
  )

  w_prev_2 <- adjust_weights(w_2, next_return)

  # Mean-variance efficient portfolio (w/o transaction costs)
  w_3 <- compute_efficient_weight(
    Sigma = Sigma,
    mu = mu,
    gamma = gamma
  )

  performance_values[[3]][p, ] <- evaluate_performance(
    w_3,
    w_prev_3,
    next_return
  )

  w_prev_3 <- adjust_weights(w_3, next_return)
}
```

Finally, we get to the evaluation of the portfolio strategies *net-of-transaction costs*. Note that we compute annualized returns and standard deviations. \index{Sharpe ratio}


```r
performance <- lapply(
  performance_values,
  as_tibble
) |>
  bind_rows(.id = "strategy")

performance |>
  group_by(strategy) |>
  summarize(
    Mean = 12 * mean(100 * net_return),
    SD = sqrt(12) * sd(100 * net_return),
    `Sharpe ratio` = if_else(Mean > 0,
      Mean / SD,
      NA_real_
    ),
    Turnover = 100 * mean(turnover)
  )
```

```
# A tibble: 3 × 5
  strategy   Mean    SD `Sharpe ratio` Turnover
  <chr>     <dbl> <dbl>          <dbl>    <dbl>
1 MV       -0.637  12.4         NA     214.    
2 MV (TC)  12.1    15.1          0.802   0.0311
3 Naive    12.1    15.1          0.801   0.229 
```

The results clearly speak against mean-variance optimization. Turnover is huge when the investor only considers her portfolio's expected return and variance. Effectively, the mean-variance portfolio generates a *negative* annualized return after adjusting for transaction costs. At the same time, the naive portfolio turns out to perform very well. In fact, the performance gains of the transaction-cost adjusted mean-variance portfolio are small. The out-of-sample Sharpe ratio is slightly higher than for the naive portfolio. Note the extreme effect of turnover penalization on turnover: *MV (TC)* effectively resembles a buy-and-hold strategy which only updates the portfolio once the estimated parameters $\hat\mu_t$ and $\hat\Sigma_t$indicate that the current allocation is too far away from the optimal theoretical portfolio. 

## Exercises

1. We argue that an investor with a quadratic utility function with certainty equivalent $$\max_w CE(w) = \omega'\mu - \frac{\gamma}{2} \omega'\Sigma \omega \text{ s.t. } \iota'\omega = 1$$
faces an equivalent optimization problem to a framework where portfolio weights are chosen with the aim to minimize volatility given a pre-specified level or expected returns
$$\min_w \omega'\Sigma \omega \text{ s.t. } \omega'\mu = \bar\mu \text{ and } \iota'\omega = 1. $$ Proof that there is an equivalence between the optimal portfolio weights in both cases. 
1. Consider the portfolio choice problem for transaction-cost adjusted certainty equivalent maximization with risk aversion parameter $\gamma$ 
$$\omega_{t+1} ^* :=  \arg\max_{\omega \in \mathbb{R}^N,  \iota'\omega = 1} \omega'\mu - \nu_t (\omega, \beta) - \frac{\gamma}{2}\omega'\Sigma\omega$$
where $\Sigma$ and $\mu$ are (estimators of) the variance-covariance matrix of the returns and the vector of expected returns. Assume for now that transaction costs are quadratic in rebalancing **and** proportional to stock illiquidity such that 
$$\nu_t\left(\omega, B\right) := \frac{\beta}{2} \left(\omega - \omega_{t^+}\right)'B\left(\omega - \omega_{t^+}\right)$$ where $B = \text{diag}(ill_1, \ldots, ill_N)$ is a diagonal matrix where $ill_1, \ldots, ill_N$. Derive a closed-form solution for the mean-variance efficient portfolio $\omega_{t+1} ^*$ based on the transaction cost specification above. Discuss the effect of illiquidity $ill_i$ on the individual portfolio weights relative to an investor that myopically ignores transaction costs in her decision. 
1. Use the solution from the previous exercise to update the function `compute_efficient_weight` such that you can compute optimal weights conditional on a matrix $B$ with illiquidity measures. 
1. Illustrate the evolution of the *optimal* weights from the naive portfolio to the efficient portfolio in the mean-standard deviation diagram.
1. Is it always optimal to choose the same $\beta$ in the optimization problem than the value used in evaluating the portfolio performance? In other words: Can it be optimal to choose theoretically sub-optimal portfolios based on transaction cost considerations that do not reflect the actual incurred costs? Evaluate the out-of-sample Sharpe ratio after transaction costs for a range of different values of imposed $\beta$ values.
