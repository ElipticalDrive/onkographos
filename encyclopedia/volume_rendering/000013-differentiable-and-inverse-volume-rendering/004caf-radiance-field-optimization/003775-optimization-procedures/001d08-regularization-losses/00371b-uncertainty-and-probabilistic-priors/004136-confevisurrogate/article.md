# ConfEviSurrogate

Also called: Conformal CNN Surrogate with Normal Inverse-Gamma Head

## Overview
ConfEviSurrogate is a CNN-based surrogate model for scientific volume data that provides calibrated uncertainty bounds, addressing the lack of reliable confidence intervals in standard deep learning surrogates [S1, unofficial]. It replaces the standard output layer with a four-parameter Normal Inverse-Gamma head to separate aleatoric and epistemic uncertainty in a single forward pass [S1, unofficial]. A conformal prediction calibration step provides finite-sample coverage guarantees for prediction intervals, a property absent from other surrogate methods in this section [S1, unofficial]. It distinguishes itself from sibling methods like [Uncertainty Prediction Loss](2961-uncertainty-prediction-loss.md) and [Uncertainty-aware Squared Error Loss](2964-uncertainty-aware-squared-error-loss.md) by integrating a Normal Inverse-Gamma distribution head with conformal prediction to ensure finite-sample coverage guarantees [S1, unofficial].

## How it works
**Inputs:** A 3D spatial domain representing the simulation state [S1, unofficial].
**Outputs:** A predicted volume field and a voxel-wise uncertainty estimate (mean and variance parameters) [S1, unofficial].

**Data Structures:**
The model replaces the standard single-output layer of a CNN with a four-parameter Normal Inverse-Gamma head [S1, unofficial]. The parameters are: $\mu$ (predicted mean), $\lambda$ (precision scaling factor for the mean), $\alpha$ (shape parameter of the inverse-gamma distribution), and $\beta$ (scale parameter of the inverse-gamma distribution) [S1, unofficial]. These parameters define a predictive distribution $p(y|x) = \mathcal{N}(\mu, \sigma^2) \times \text{Inv-Gamma}(\sigma^2; \alpha, \beta)$, where $\sigma^2$ is the variance [G]. This formulation allows the network to predict both the expected value and the associated uncertainty [S1, unofficial].

**Steps:**
1. **Forward Pass:** The CNN encoder processes the input volume. The decoder outputs the four NIG parameters ($\mu, \lambda, \alpha, \beta$) for each voxel [S1, unofficial].
2. **Uncertainty Decomposition:** The model separates uncertainty into aleatoric uncertainty (captured by the variance $\sigma^2$ inherent in the data distribution, modeled by the Inverse-Gamma component) and epistemic uncertainty (captured by the uncertainty in the mean prediction $\mu$, modeled by the Normal component conditioned on $\sigma^2$) [S1, unofficial].
3. **Conformal Calibration:** A held-out calibration set is used to adjust the prediction intervals [S1, unofficial]. For a given confidence level $1-\epsilon$, non-conformity scores are computed on the calibration set. The sources do not specify the exact quantile formula used to expand the intervals [S1, unofficial].
4. **Prediction:** For a new input, the model outputs the mean $\mu$ and the calibrated prediction interval, where the interval width is derived from the conformal score and the predicted variance [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def confevi_surrogate_forward(input_volume, model, calibration_residuals, confidence_level):
    """
    Performs a forward pass and returns calibrated prediction intervals.
    
    Args:
        input_volume: 3D array of shape (H, W, D) or (B, H, W, D).
        model: CNN model with NIG head returning mu, lambda, alpha, beta.
        calibration_residuals: 1D array of absolute residuals |y_true - y_pred| from a held-out calibration set.
        confidence_level: Float, e.g., 0.95 for 95% confidence.
        
    Returns:
        mean_pred: Predicted mean volume.
        lower_bound: Lower bound of the prediction interval.
        upper_bound: Upper bound of the prediction interval.
    """
    # 1. Forward pass to get NIG parameters
    mu, lam, alpha, beta = model(input_volume)
    
    # 2. Compute predictive variance
    # Marginal variance of NIG(mu, lambda, alpha, beta) is beta/(alpha-1) * (1 + 1/lambda) for alpha > 1 [G]
    if alpha > 1:
        predictive_variance = (beta / (alpha - 1)) * (1 + 1.0 / lam)
    else:
        predictive_variance = float('inf')
        
    predictive_std = math.sqrt(predictive_variance)
    
    # 3. Apply conformal calibration
    # The conformal margin is the empirical quantile of the calibration residuals [G]
    # [S1, unofficial] The sources do not specify the exact quantile formula for conformal calibration.
    n_cal = len(calibration_residuals)
    quantile_index = int(math.ceil((n_cal + 1) * confidence_level))
    sorted_residuals = sorted(calibration_residuals)
    calibration_margin = sorted_residuals[quantile_index - 1]
    
    # 4. Compute bounds
    lower_bound = mu - calibration_margin
    upper_bound = mu + calibration_margin
    
    return mu, lower_bound, upper_bound
```

**Parameters:**
- **Calibration Quantile:** Determined by the desired confidence level $1-\epsilon$ and the size of the calibration set [G].
- **NIG Parameters:** $\mu, \lambda, \alpha, \beta$ are learned during training [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Data Scarcity:** Conformal calibration requires a held-out calibration set that may be unavailable for data-scarce simulations [S1, unofficial].
- **Full-Volume Prediction:** The CNN architecture generates full-volume predictions rather than point queries, which increases memory usage and computational cost for large volumes [S1, unofficial].
- **Alpha Parameter:** If $\alpha \le 1$, the marginal variance is undefined. The implementation must handle this case, typically by clamping $\alpha$ to a value greater than 1 or using a large constant for variance [G].

## Subtopics

### Bayesian Inference Surrogate Framework
This variant trains a surrogate model and then applies Bayesian inference to obtain the posterior distribution over input parameters conditioned on a user-specified spatial feature [S1, unofficial]. It uses a Fisher Information Matrix prior to accelerate prior probability computation by three orders of magnitude compared to deep ensembles [S1, unofficial]. Posterior characterization requires approximately 1 second of burn-in plus 100–200 sampling steps [S1, unofficial]. It is chosen when characterizing input parameter distributions for specific output features is required rather than predicting outputs directly [S1, unofficial]. However, it has been validated only on 2D time-varying fields (up to $1024 \times 256 \times 200$) with at most 4 parameters; extension to volumetric data and higher-dimensional parameter spaces remains an open challenge [S1, unofficial].

## Operating characteristics
**Time Complexity:** Uncertainty estimation takes 0.03–0.36 seconds per inference, significantly faster than Monte Carlo Dropout (0.83–25.17 seconds) and deep ensembles (1.60–3.00 seconds) [S1, unofficial].
**Memory Complexity:** Not given in the sources.
**Performance:** On three datasets (MPAS-Ocean at $192 \times 96 \times 96$, Nyx at $256^3$, CloverLeaf3D at $64^3$), ConfEviSurrogate achieved the highest voxel-level correlation between predicted uncertainty and true error, outperforming both Monte Carlo Dropout and deep ensembles [S1, unofficial].
**Strengths:** Provides calibrated uncertainty bounds with finite-sample coverage guarantees; separates aleatoric and epistemic uncertainty; fast inference compared to ensemble methods [S1, unofficial].
**Limitations:** Requires a held-out calibration set; full-volume prediction increases memory usage; not validated on high-dimensional parameter spaces or complex volumetric data in the Bayesian variant [S1, unofficial].
**When to Use:** When calibrated uncertainty is critical and computational efficiency is important, and sufficient data is available for calibration [S1, unofficial].
**When Not to Use:** When data is scarce (no calibration set) or when the problem involves high-dimensional parameter spaces with limited validation [S1, unofficial].

## Background
ConfEviSurrogate was introduced to address the lack of calibrated uncertainty bounds in CNN surrogates for scientific volume data [S1, unofficial]. It builds on the principles of uncertainty prediction and probabilistic priors, extending them with conformal prediction techniques [S1, unofficial]. The method was validated on ocean and astrophysics simulation datasets, demonstrating its effectiveness in providing reliable uncertainty estimates for large-scale scientific simulations [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
