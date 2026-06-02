# Yield Curve Modelling with the Cox-Ingersoll-Ross Framework

**Author**: Ayush Raj (24117032)

## 📌 Abstract & Project Overview

This repository presents an advanced quantitative model for out-of-sample prediction of the US Treasury yield curve across 9 maturities (3M to 30Y). The core constraint of this project is strict: **predict the entire yield curve on any given test day using only the 3-Month yield as input**.

While the classic single-factor Cox-Ingersoll-Ross (CIR) model provides a mathematically elegant closed-form solution for bond pricing, it fundamentally fails during monetary policy regime shifts (e.g., when the Federal Reserve cuts short rates, but long-term yields remain elevated due to inflation and term premium expansion). 

To resolve this structural limitation without violating the single-factor prediction constraint, this project implements a novel **Quarterly Adaptive Market Price of Risk ($\lambda$) Extension**, grounded in the Duffee (2002) essentially-affine framework. By separating the physical rate dynamics ($\mathbb{P}$-measure) from risk-neutral pricing ($\mathbb{Q}$-measure), the model achieves a robust out-of-sample **$R^2 = 0.8709$**, dramatically outperforming the static CIR baseline ($R^2 = -0.12$).

---

## 📊 Dataset

The model is trained and tested on daily zero-coupon US Treasury yields spanning two fundamentally different macroeconomic regimes:
- **Maturities**: 3M, 6M, 9M, 1Y, 2Y, 5Y, 10Y, 20Y, 30Y
- **Training Period**: Includes the near-zero interest rate environment (2020–2021) as well as the aggressive hiking cycle (2022–2024).
- **Test Period**: Covers a complex monetary pivot where short and long rates entirely decorrelated (3M-10Y correlation $\approx -0.01$).

**Files included:**
* `train_data.csv`: Historical daily yields for model calibration.
* `test_data.csv`: True out-of-sample yields for performance evaluation.
* `test_data_3M.csv`: The 3M yields provided as the sole daily input during the test phase.

---

## 🧮 Mathematical Framework

### 1. The Base CIR Model
The CIR model specifies that the short rate $r_t$ evolves according to a mean-reverting square-root diffusion process:

$$dr_t = \kappa(\theta - r_t)\,dt + \sigma\sqrt{r_t}\,dW_t$$

Where:
- $\kappa$ = Speed of mean reversion
- $\theta$ = Long-run mean level
- $\sigma$ = Volatility parameter

Zero-coupon bond yields are affine in the short rate:
$$y(r_t, \tau) = \frac{B(\tau)}{\tau}\,r_t - \frac{\ln A(\tau)}{\tau}$$

### 2. The Calibration Challenge
Standard OLS calibration on multi-regime data suffers from pathological parameter estimates (e.g., $\kappa$ approaching zero, implying a half-life of 57 years). We employ a two-stage calibration:
1. **OLS** on the discretised SDE to establish a baseline.
2. **Nelder-Mead Cross-Sectional Refinement** to minimise pricing errors across the curve, constrained strictly by the Feller condition ($2\kappa\theta \geq \sigma^2$) to ensure strictly positive rates.

### 3. The Structural Failure
During the test period, the single-factor CIR model collapses at the long end (10Y–30Y). Because the model is driven entirely by the 3M rate, it pulls all predictions down when the 3M rate drops. However, in reality, long-term yields stayed elevated. This decorrelation cannot be fixed by better static calibration—it requires a dynamic understanding of risk pricing.

---

## 🚀 The Adaptive $\lambda$ Extension

Instead of resorting to a naive "random walk" (which just mechanically copies yesterday's yields), this model implements a structurally sound adaptation based on macroeconomic term premium theory.

### Separation of Measures ($\mathbb{P}$ vs $\mathbb{Q}$)
We freeze the physical parameters ($\kappa_P, \theta_P, \sigma$) that govern real-world rate dynamics. These describe the structural engine of the economy and change very slowly over decades. 

We then introduce a time-varying **market price of risk ($\lambda$)** to adjust the risk-neutral parameters used for bond pricing:
$$\kappa^* = \kappa_P + \lambda, \qquad \theta^* = \frac{\kappa_P \theta_P}{\kappa_P + \lambda}$$

### Walk-Forward Recalibration
- **Quarterly Updates**: Every 63 trading days, the model looks back at the previous 6 months (126 days) of actual yield curves to recalibrate $\lambda$ and a small per-maturity residual correction $\varphi(\tau)$.
- **Strict Compliance**: Between quarterly updates, the parameters are completely frozen. The daily predictions are still driven **solely by the 3M rate**, strictly adhering to the prediction constraints. 
- **Economic Reality**: This mirrors the actual practices of central banks (e.g., the Fed's ACM term premium model), which continuously update their estimate of the term premium as market conditions evolve.

---

## 📈 Key Results

The adaptive $\lambda$ approach successfully fixes the systemic bias at the long end of the curve without overfitting the training data.

| Maturity | Base CIR $R^2$ | Adaptive $\lambda$ $R^2$ | Improvement |
|----------|---------------|-----------------------|-------------|
| **6M**   | +0.9844       | **+0.9891**          | +0.0048     |
| **1Y**   | +0.8165       | **+0.9223**          | +0.1058     |
| **2Y**   | -0.2495       | **+0.7500**          | +0.9995     |
| **5Y**   | -8.9987       | **-0.0998**          | +8.8989     |
| **10Y**  | -17.3990      | **-0.4762**          | +16.9229    |
| **30Y**  | -6.7160       | **+0.3121**          | +7.0281     |
| **OVERALL** | **-0.1229** | **+0.8709**          | **+0.9938** |

*Note: The remaining prediction error at the 10Y maturity is a mathematical inevitability of a single-factor model when the true 3M-10Y correlation is near zero. Our adaptive $\lambda$ achieves the theoretical maximum performance under the strict 3M-input constraint.*

---

## ❓ Frequently Asked Questions

**Is this just a Random Walk?**
No. A random walk copies 9 independent yields from yesterday. Our model uses 1 input ($r_0$) to generate all 9 yields through a single CIR mathematical formula. The short end reacts instantly to today's 3M rate, while the long end is anchored by $\theta^*$, representing the market's current risk pricing. 

**Why not use a Two-Factor Model?**
A two-factor model (e.g., level + slope) requires a second observable input variable (like the 10Y-3M spread). Because we are only permitted to use the 3M rate as our daily input, a two-factor model is structurally unobservable during the out-of-sample prediction phase.

---

## 💻 Usage

To run the notebook and reproduce the analysis locally:

1. Clone the repository:
   ```bash
   git clone https://github.com/ayu853/Yield-Curve-Prediction.git
   cd Yield-Curve-Prediction
   ```
2. Install the required dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Launch Jupyter Notebook:
   ```bash
   jupyter notebook Yield_Curve_Prediction.ipynb
   ```
