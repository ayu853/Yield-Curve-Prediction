# Yield Curve Modelling with the Cox-Ingersoll-Ross Framework

**Author**: Ayush Raj (24117032)

## Project Overview

This project models and predicts the US Treasury yield curve out-of-sample using the Cox-Ingersoll-Ross (CIR) framework. Given the challenge of predicting the entire yield curve (6M to 30Y) using only the 3-Month rate as a daily input, this project implements a novel **Adaptive Market Price of Risk ($\lambda$) extension** to capture term premium dynamics during monetary policy regime shifts. 

The standard single-factor CIR model fundamentally fails to capture the decorrelation between short and long rates during stress periods and monetary pivots. By leveraging the Duffee (2002) essentially-affine framework, we separate physical rate dynamics from risk-neutral pricing. The physical parameters are calibrated and frozen, while the risk premium $\lambda$ adapts quarterly in a walk-forward manner. This principled economic approach achieves a robust out-of-sample $R^2 = 0.87$, significantly outperforming static models.

## Dataset

The model is trained and tested on zero-coupon Treasury yields across 9 maturities:
- **3M, 6M, 9M, 1Y, 2Y, 5Y, 10Y, 20Y, 30Y**

The dataset spans multiple distinct rate regimes, moving from near-zero rate periods (e.g., 2020-2021) into the high-rate environment of 2023-2024, providing a highly rigorous testing ground for the predictive capabilities of the model.
* `train_data.csv`: Daily yields for training.
* `test_data.csv`: Daily yields for out-of-sample evaluation.
* `test_data_3M.csv`: Daily 3M yields serving as the sole input constraint for testing predictions.

## Methodology

1. **Two-Stage Calibration**: 
   - **Stage 1**: OLS regression on the discretised SDE to estimate initial parameters.
   - **Stage 2**: Nelder-Mead cross-sectional refinement to minimise pricing errors across all maturities, subject to a strict Feller condition penalty.
2. **Base CIR Evaluation**: Demonstrates the systematic failures of single-factor models when the short-rate and long-rate decorrelate (e.g., policy pivots where 3M drops but 10Y stays elevated).
3. **Adaptive Market Price of Risk ($\lambda$)**:
   - The core innovation of this repository. Instead of copying recent yields mechanically (a random walk), the model separates the physical probability measure ($\mathbb{P}$) from the pricing measure ($\mathbb{Q}$).
   - Structural parameters ($\kappa_P, \theta_P, \sigma$) that govern interest rate dynamics are frozen from training.
   - Only the market price of risk $\lambda$ (how much extra return investors demand) is recalibrated quarterly using a 126-day rolling window, preventing look-ahead bias and adhering to standard macroeconomic practices (like the Federal Reserve's ACM model).

## Results & Critical Analysis

- **Base CIR**: Collapses entirely at the long end (10Y-30Y) with deeply negative $R^2$ scores, pulling all yields artificially toward the short rate.
- **Adaptive $\lambda$ Extension**: Succeeds in accurately forecasting the curve, correcting the cross-sectional term premium bias.
- **Final Performance**: The Adaptive $\lambda$ extension dramatically improves predictive accuracy from a Base CIR $R^2 = -0.12$ to an overall $R^2 = 0.8709$ on the test set.

## Running the Notebook

All logic, model classes, time-series evaluations, and theoretical analyses are contained within `Yield_Curve_Prediction.ipynb`.

1. Install requirements:
   ```bash
   pip install -r requirements.txt
   ```
2. Open the notebook:
   ```bash
   jupyter notebook Yield_Curve_Prediction.ipynb
   ```
