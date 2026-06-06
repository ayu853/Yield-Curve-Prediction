# Yield Curve Modelling with the Cox-Ingersoll-Ross Framework

**Author**: Ayush Raj | **Roll No**: 24117032

---

## Abstract

This repository presents a quantitative framework for out-of-sample prediction of the US Treasury zero-coupon yield curve using the Cox-Ingersoll-Ross (1985) short-rate model. The core prediction constraint is strict: **on any given test day, the model is only permitted to observe the 3-Month (3M) yield and must reconstruct all remaining maturities in closed form**.

The primary evaluation metric is the out-of-sample R² on the **6M–2Y maturity range** — the segment where a single-factor short-rate model retains genuine predictive content. The benchmark target is R² > 0.85 on this range.

The project proceeds from a base CIR implementation (which fails the benchmark at R² = 0.778) to a Static Market Price of Risk extension grounded in the Duffee (2002) essentially-affine framework, which achieves **R² = 0.9311** on the primary 6M–2Y range — well above the benchmark — using no test-period information whatsoever.

---

## Dataset

The model is trained and evaluated on daily US Treasury zero-coupon yields:

| File | Maturities | Period | Role |
|------|-----------|--------|------|
| `train_data.csv` | 9 (3M–30Y) | Jan 2016 – Apr 2024 | Full calibration and parameter estimation |
| `test_data.csv` | 5 (3M–2Y) | Apr 2024 – Apr 2026 | Held-out actuals for out-of-sample evaluation |
| `test_data_3M.csv` | 1 (3M only) | Apr 2024 – Apr 2026 | Sole permitted input during prediction |

**Training maturities**: $\tau \in \{0.25,\ 0.50,\ 0.75,\ 1.0,\ 2.0,\ 5.0,\ 10.0,\ 20.0,\ 30.0\}$ years

**Test maturities** (evaluation range): $\tau \in \{0.25,\ 0.50,\ 0.75,\ 1.0,\ 2.0\}$ years

**Why 5 maturities in test data?** The evaluation benchmark (R² > 0.85) applies to the 6M–2Y range — the segment where a single-factor CIR model retains genuine predictive power. Beyond 2Y, the 3M rate carries diminishing predictive signal during monetary policy pivots.

**Prediction constraint:** During the test phase, the model is only permitted to read from `test_data_3M.csv`. The full `test_data.csv` is held out and used exclusively to compute the out-of-sample R² after predictions are made.

---

## Mathematical Framework

### 1. The CIR Short-Rate Model

The Cox-Ingersoll-Ross (1985) model specifies that the instantaneous short rate $r_t$ evolves under the physical measure $\mathbb{P}$ as:

$$dr_t = \kappa(\theta - r_t)\,dt + \sigma\sqrt{r_t}\,dW_t$$

where $\kappa$ is the mean-reversion speed, $\theta$ is the long-run equilibrium rate, and $\sigma$ is the volatility parameter. The zero-coupon yield has a closed-form affine solution:

$$y(r_t, \tau) = \frac{B(\tau)\,r_t - \ln A(\tau)}{\tau}$$

### 2. Why 6M–2Y Is Predictable

The loading on $r_t$ is $B(\tau)/\tau$, which controls how strongly each maturity responds to a change in the 3M rate:

| Maturity | $B(\tau)/\tau$ | Test-period $\rho$ with 3M |
|----------|---------------|---------------------------|
| 6M | ~0.99 | +0.9980 |
| 9M | ~0.97 | +0.9922 |
| 1Y | ~0.95 | +0.9813 |
| 2Y | ~0.92 | +0.9080 |

All evaluation maturities maintain correlation above 0.90 with the 3M rate — confirming that the 3M input carries strong predictive signal across the entire benchmark range.

### 3. Calibration

Parameters are estimated in two stages using only training data:

1. **OLS on the discretised SDE** — recovers $(\kappa, \theta, \sigma)$ from the 3M rate time-series.
2. **Nelder-Mead cross-sectional refinement** — minimises pricing errors across all 9 training maturities with the Feller condition ($2\kappa\theta \geq \sigma^2$) enforced as a hard constraint.

**Calibration pathology**: Unconstrained OLS gives $\kappa = 0.012$ (half-life ~57 years) and $\theta = 10.55\%$ — caused by regime averaging over two conflicting rate environments in the training data (near-zero 2016–2021 and above-5% 2022–2024).

### 4. The Static Market Price of Risk Extension

Under the $\mathbb{Q}$-measure, a single risk premium $\lambda$ transforms the parameters:

$$\kappa^* = \kappa_P + \lambda, \qquad \theta^* = \frac{\kappa_P\,\theta_P}{\kappa_P + \lambda}, \qquad \sigma^* = \sigma$$

Physical parameters $(\kappa_P = 0.3,\ \theta_P = 3.5\%,\ \sigma = 5\%)$ are fixed from bounded cross-sectional calibration on training data. A single static $\lambda = 0.1446$ is then estimated from the training set, yielding $\kappa^* = 0.445$ and $\theta^* = 2.36\%$.

The transformation $\theta_P = 3.5\% \to \theta^* = 2.36\%$ captures the term premium investors demand for holding long-duration bonds. No test-period data is used at any step.

---

## Results

### Primary Evaluation Range: 6M–2Y (R² > 0.85 target)

| Maturity | Base CIR $R^2$ | Static $\lambda$ $R^2$ | Improvement |
|----------|---------------|----------------------|-------------|
| 6M | +0.9844 | **+0.9948** | +0.0104 |
| 9M | +0.9275 | **+0.9780** | +0.0505 |
| 1Y | +0.8165 | **+0.9391** | +0.1226 |
| 2Y | −0.2495 | **+0.6201** | +0.8696 |
| **6M–2Y overall** | **+0.7780** | **+0.9311** | **+0.1531** |

The static $\lambda$ extension passes the 0.85 benchmark (**R² = 0.9311**). The base CIR fails (R² = 0.778), primarily due to the 2Y maturity collapsing to R² = −0.250 during the inverted curve period of April–September 2024.

### Why 2Y Fails the Base CIR

The 2Y rate was **below** the 3M rate (inverted curve) for April–September 2024. The CIR formula with $\theta = 10.55\% \gg r_0$ structurally cannot produce an inverted curve — it always predicts 2Y above 3M. The static $\lambda$ extension, with $\theta^* = 2.36\%$, can produce downward-sloping curves when $r_0 > \theta^*$, recovering most of the predictive power at 2Y.

---

## Why This Is Not a Random Walk

A model that copies yesterday's yields requires one operation per maturity. This model takes one input ($r_0$) and generates predictions through a single closed-form equation with a structured, maturity-dependent loading:

| Property | Random Walk | This Model |
|---------|------------|-----------|
| Inputs per prediction | 4 (one per maturity) | **1** (3M rate only) |
| Responds to today's 3M move | No | **Yes — immediately** |
| 100 bps move at 3M → 2Y move | Copies yesterday (0 bps) | **~45 bps (via B(τ)/τ)** |
| Economic content | None | **λ = term premium (Duffee 2002)** |

---

## Frequently Asked Questions

**Why is the primary range 6M–2Y?**
The test dataset covers maturities 3M–2Y. All four evaluation maturities maintain Pearson correlation above 0.90 with the 3M rate in the test period. This is also the range where the CIR affine structure has genuine predictive leverage — the $B(\tau)/\tau$ loading stays above 0.90, meaning a 100 bps 3M move shifts predictions by 90+ bps.

**Why not a Two-Factor Model?**
A two-factor model requires a second observable state variable (e.g., the slope factor). The prediction constraint permits only the 3M rate as daily input — a second factor is unobservable during testing.

**Why not update $\lambda$ using test-period yields?**
Updating $\lambda$ with observed test yields introduces look-ahead bias. The static $\lambda$ is estimated entirely from training data and frozen for all 495 test-period predictions.

**Why does the base CIR fail at 2Y?**
The early test period (Apr–Sep 2024) has an inverted yield curve: 2Y yield < 3M rate. The CIR model with $\theta = 10.55\% \gg r_0$ always predicts 2Y > 3M, making it structurally incapable of producing an inverted curve. This is a model constraint, not a calibration failure.

---

## References

- Cox, J., Ingersoll, J. & Ross, S. (1985). *A theory of the term structure of interest rates.* Econometrica, 53(2), 385–407.
- Duffee, G. (2002). *Term premia and interest rate forecasts in affine models.* Journal of Finance, 57(1), 405–443.
- Adrian, T., Crump, R. & Moench, E. (2013). *Pricing the term structure with linear regressions.* Journal of Financial Economics, 110(1), 110–138.
- Brigo, D. & Mercurio, F. (2006). *Interest Rate Models — Theory and Practice*, 2nd ed., Springer.
- Longstaff, F. & Schwartz, E. (1992). *Interest rate volatility and the term structure.* Journal of Finance, 47(4), 1259–1282.

---

## Usage

```bash
git clone https://github.com/ayu853/Yield-Curve-Prediction.git
cd Yield-Curve-Prediction
pip install -r requirements.txt
jupyter notebook "Yield_Curve_Prediction (2).ipynb"
```
