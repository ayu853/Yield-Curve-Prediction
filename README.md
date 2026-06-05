# Yield Curve Modelling with the Cox-Ingersoll-Ross Framework

**Author**: Ayush Raj | **Roll No**: 24117032

---

## Abstract

This repository presents a quantitative framework for out-of-sample prediction of the US Treasury zero-coupon yield curve using the Cox-Ingersoll-Ross (1985) short-rate model. The core prediction constraint is strict: **on any given test day, the model is only permitted to observe the 3-Month (3M) yield and must reconstruct all remaining maturities (6M through 30Y) in closed form**.

The primary evaluation metric is the out-of-sample R² on the **6M–2Y maturity range** — the segment where a single-factor short-rate model retains genuine predictive content. The benchmark target is R² > 0.85 on this range.

The project proceeds from a base CIR implementation (which fails the benchmark at R² = 0.778) to a Static Market Price of Risk extension grounded in the Duffee (2002) essentially-affine framework, which achieves **R² = 0.9311** on the primary 6M–2Y range — well above the benchmark — using no test-period information whatsoever.

---

## Dataset

The model is trained and evaluated on daily US Treasury zero-coupon yields across two structurally distinct macroeconomic regimes:

- **Maturities**: 3M, 6M, 9M, 1Y, 2Y, 5Y, 10Y, 20Y, 30Y
- **Training Period**: January 2016 – April 2024 (covers the near-zero rate era and the aggressive 2022–2023 tightening cycle)
- **Test Period**: April 2024 – April 2026 (covers the monetary policy pivot: inversion-to-normalisation transition)

**Files included:**

| File | Description |
|------|-------------|
| `train_data.csv` | Daily yields across all 9 maturities — used for calibration |
| `test_data.csv` | Held-out actuals — used only to compute out-of-sample R² |
| `test_data_3M.csv` | 3M yield only — the sole permitted daily input during prediction |

---

## Mathematical Framework

### 1. The CIR Short-Rate Model

The Cox-Ingersoll-Ross (1985) model specifies that the instantaneous short rate $r_t$ evolves under the physical measure $\mathbb{P}$ as:

$$dr_t = \kappa(\theta - r_t)\,dt + \sigma\sqrt{r_t}\,dW_t$$

where $\kappa$ is the mean-reversion speed, $\theta$ is the long-run equilibrium rate, and $\sigma$ is the volatility parameter. The $\sqrt{r_t}$ diffusion term ensures non-negativity provided the Feller condition holds: $2\kappa\theta \geq \sigma^2$.

The zero-coupon yield at maturity $\tau$ has a closed-form affine solution:

$$y(r_t, \tau) = \frac{B(\tau)}{\tau}\,r_t - \frac{\ln A(\tau)}{\tau}$$

where $B(\tau)$ and $A(\tau)$ are functions of $(\kappa, \theta, \sigma)$ and $\gamma = \sqrt{\kappa^2 + 2\sigma^2}$. This affine structure — where yields are linear in $r_t$ — is the foundation of the model's predictive power at short maturities.

### 2. Why Short Maturities Are Predictable but Long Maturities Are Not

The loading on $r_t$ in the yield formula is $B(\tau)/\tau$, which varies with maturity:

| Maturity | $B(\tau)/\tau$ | Interpretation |
|----------|---------------|----------------|
| 6M | ~0.99 | Yields move almost one-for-one with the 3M rate |
| 1Y | ~0.97 | Strong co-movement |
| 2Y | ~0.90 | Good co-movement |
| 10Y | ~0.55 | Weak — long-run mean dominates |
| 30Y | ~0.32 | Very weak — almost entirely driven by $\theta$ |

This is why the evaluation benchmark is set on the 6M–2Y range: it is precisely the segment where the affine structure of CIR retains genuine predictive leverage from the 3M input.

### 3. Calibration

Parameters are estimated in two stages, both using only training data:

1. **OLS on the discretised SDE** — a Pearson-GLS regression that recovers $(\kappa, \theta, \sigma)$ from the time-series of the 3M rate.
2. **Nelder-Mead cross-sectional refinement** — minimises mean squared pricing error across all maturities and every third training day. The Feller condition is enforced as a hard constraint.

**Calibration pathology**: The unconstrained OLS gives $\kappa = 0.012$ (half-life ~57 years) and $\theta = 10.55\%$ — economically implausible values caused by regime averaging. The training data contains two conflicting regimes (near-zero rates 2016–2021 and high rates 2022–2024), and OLS averages across both, destroying the mean-reversion signal.

### 4. The Static Market Price of Risk Extension

The base CIR model conflates two conceptually distinct objects: real-world rate dynamics ($\mathbb{P}$-measure) and market risk pricing ($\mathbb{Q}$-measure). The extension separates them.

Under the $\mathbb{Q}$-measure, the risk premium $\lambda$ transforms the parameters:

$$\kappa^* = \kappa_P + \lambda, \qquad \theta^* = \frac{\kappa_P\,\theta_P}{\kappa_P + \lambda}, \qquad \sigma^* = \sigma$$

The physical parameters $(\kappa_P = 0.3,\ \theta_P = 3.5\%,\ \sigma = 5\%)$ are fixed from bounded cross-sectional calibration on training data. A single static $\lambda = 0.1446$ is then estimated from the training set, yielding risk-neutral parameters $\kappa^* = 0.445$ and $\theta^* = 2.36\%$.

This transforms $\theta_P = 3.5\%$ (the real-world expected long-run rate) to $\theta^* = 2.36\%$ (the risk-neutral pricing level), capturing the term premium that investors demand for holding long-duration bonds. No test-period data is used at any step.

---

## Results

### Primary Evaluation Range: 6M–2Y (R² > 0.85 target)

| Maturity | Base CIR $R^2$ | Static $\lambda$ $R^2$ | Improvement |
|----------|---------------|----------------------|-------------|
| 6M | +0.984 | **+0.995** | +0.011 |
| 9M | +0.928 | **+0.978** | +0.050 |
| 1Y | +0.817 | **+0.939** | +0.122 |
| 2Y | −0.250 | **+0.620** | +0.870 |
| **6M–2Y overall** | **+0.778** | **+0.931** | **+0.153** |

The static $\lambda$ extension passes the 0.85 benchmark (R² = 0.9311). The base CIR fails (R² = 0.778), primarily due to the 2Y maturity collapsing to R² = −0.250 during the monetary policy pivot.

### Full Curve Reference (6M–30Y)

Predictions are generated for all maturities (6M–30Y) as required. The 5Y–30Y range is reported for academic completeness and to demonstrate the structural limits of a single-factor model:

| Maturity | Base CIR $R^2$ | Static $\lambda$ $R^2$ |
|----------|---------------|----------------------|
| 5Y | −8.999 | −6.946 |
| 10Y | −17.399 | −28.930 |
| 20Y | −11.277 | −21.948 |
| 30Y | −6.716 | −18.545 |

The long-end failure is not a calibration error — it is a mathematical consequence of the 3M-10Y test-period correlation being effectively zero ($\rho = -0.01$). When short and long rates decouple during a monetary policy pivot, no single-factor model driven by the 3M rate alone can predict long-term yields.

---

## Why This Is Not a Random Walk

A model that simply copies yesterday's yields (a random walk) would require 9 independent copy operations — one per maturity. This model uses 1 input ($r_0$, today's 3M rate) to generate all yields through a single closed-form equation.

Key distinctions:

1. **Structural constraint across maturities**: The CIR formula enforces a specific mathematical relationship. A 100 bps move in $r_0$ shifts the 6M prediction by ~99 bps but the 30Y prediction by only ~32 bps — a constraint no random walk satisfies.
2. **Responds to today's rate, not yesterday's curve**: If the 3M rate drops 50 bps on a given day, all yield predictions update instantly through the formula. A random walk shows zero response to today's movement.
3. **Economic content in $\lambda$**: The risk premium $\lambda = 0.1446$ is not a residual correction — it is the market price of interest rate risk, transforming $\theta_P = 3.5\%$ to $\theta^* = 2.36\%$ through the Duffee (2002) essentially-affine framework. No test observations are used.

---

## Frequently Asked Questions

**Why is the primary evaluation range 6M–2Y and not the full curve?**

The instructor confirmed that the out-of-sample R² benchmark applies to the 6M–2Y range. This is also mathematically motivated: the 3M-10Y correlation during the test period is $\approx -0.01$ — effectively zero. Any single-factor model driven by the 3M rate carries no predictive information about 10Y movements in this regime. Predictions beyond 2Y are generated and reported, but they demonstrate the single-factor ceiling rather than model quality.

**Why not a Two-Factor Model?**

A two-factor model (level + slope) requires a second observable state variable. Because the prediction constraint permits only the 3M rate as daily input, a second factor is unobservable during testing. This is precisely the constraint that makes the problem non-trivial.

**Why not update $\lambda$ using test-period yields?**

Updating $\lambda$ using observed test yields would violate the spirit of the prediction constraint and introduce look-ahead bias. The static $\lambda$ is estimated entirely from training data and frozen for all test-period predictions — ensuring the evaluation is genuinely out-of-sample.

**Why does the base CIR fail so badly at the long end?**

With $\kappa = 0.012$ (from unconstrained OLS), the loading $B(\tau)/\tau \approx 1$ for all maturities — the model treats every maturity as a near-perfect copy of $r_0$. During 2024–2026, the 3M rate fell from 4.9% to 2.2% while the 10Y rate barely moved (3.73% to 3.63%). A model that predicts all yields move with $r_0$ will catastrophically mis-predict the long end during this decoupling.

---

## References

- Cox, J., Ingersoll, J. & Ross, S. (1985). *A theory of the term structure of interest rates.* Econometrica, 53(2), 385–407.
- Duffee, G. (2002). *Term premia and interest rate forecasts in affine models.* Journal of Finance, 57(1), 405–443.
- Adrian, T., Crump, R. & Moench, E. (2013). *Pricing the term structure with linear regressions.* Journal of Financial Economics, 110(1), 110–138.
- Brigo, D. & Mercurio, F. (2006). *Interest Rate Models — Theory and Practice*, 2nd ed., Springer.
- Campbell, J. & Thompson, S. (2008). *Predicting excess stock returns out of sample.* Review of Financial Studies, 21(4), 1509–1531.

---

## Usage

```bash
git clone https://github.com/ayu853/Yield-Curve-Prediction.git
cd Yield-Curve-Prediction
pip install -r requirements.txt
jupyter notebook Yield_Curve_Prediction.ipynb
```
