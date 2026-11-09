# ConfEviSurrogate

Also called: Conformal CNN Surrogate with Normal Inverse-Gamma Head

## Overview
ConfEviSurrogate is a CNN-based surrogate model for scientific simulations that provides statistically calibrated uncertainty bounds, addressing the lack of valid confidence intervals in standard deep learning surrogates [S1, unofficial]. It distinguishes itself from sibling methods like [Uncertainty Prediction Loss](2961-uncertainty-prediction-loss.md) by combining a Normal Inverse-Gamma (NIG) head for uncertainty decomposition with conformal prediction for finite-sample coverage guarantees, ensuring that predicted uncertainty correlates with actual reconstruction error [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A spatial feature or input parameter vector defining the simulation state.
*   **Output:** A full-volume prediction of the simulation field and a four-parameter uncertainty estimate (mean, variance, aleatoric, and epistemic components) for every voxel [S1, unofficial].

**Data Structures:**
The model replaces the standard single-value output layer of a CNN with a four-parameter head. The parameters correspond to the Normal Inverse-Gamma distribution $\mathcal{NIG}(\mu, \lambda, \alpha, \beta)$:
1.  $\mu$: The predicted mean of the field value.
2.  $\lambda$: The precision scaling factor (related to the number of observations).
3.  $\alpha$: The shape parameter of the inverse-gamma distribution (related to aleatoric uncertainty/noise).
4.  $\beta$: The scale parameter of the inverse-gamma distribution (related to epistemic uncertainty/model confidence) [S1, unofficial].

**Algorithm Steps:**
1.  **Forward Pass:** The CNN encoder processes the input to generate feature maps. The decoder produces the full-volume prediction. The final layer outputs the four NIG parameters $(\mu, \lambda, \alpha, \beta)$ for each voxel simultaneously [S1, unofficial].
2.  **Uncertainty Decomposition:** The model separates uncertainty into aleatoric (data noise) and epistemic (model ignorance) components using the NIG parameters. The predictive variance for a voxel is derived from the NIG distribution properties [S1, unofficial].
3.  **Conformal Calibration:** To ensure finite-sample coverage, a conformal prediction step is applied. This requires a held-out calibration set. Non-conformity scores (e.g., absolute residuals) are computed on this set. The prediction interval for a new input is constructed by taking the predicted mean $\pm$ the $(1-\alpha)(1+1/N)$-th quantile of the calibration scores, where $N$ is the size of the calibration set [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def confevi_surrogate_forward(input_data, model_weights):
    """
    Performs a forward pass of the ConfEviSurrogate model.
    Returns full-volume predictions and NIG parameters.
    """
    # 1. Feature Extraction and Decoding
    features = encoder(input_data, model_weights)
    
    # 2. Output Layer: Four-parameter NIG Head
    # Instead of a single scalar output, we output 4 channels
    mu = decoder_mean(features)      # Predicted mean field
    lambda_param = decoder_lambda(features) # Precision scaling
    alpha = decoder_alpha(features)  # Shape parameter (aleatoric)
    beta = decoder_beta(features)    # Scale parameter (epistemic)
    
    return mu, lambda_param, alpha, beta

def conformal_calibrate(calibration_set, model_weights):
    """
    Computes the quantile offset for conformal prediction intervals.
    calibration_set: List of (input, true_value) pairs.
    """
    residuals = []
    for x, y_true in calibration_set:
        mu_pred, _, _, _ = confevi_surrogate_forward(x, model_weights)
        # Non-conformity score: absolute difference
        score = abs(y_true - mu_pred)
        residuals.append(score)
    
    residuals.sort()
    n = len(residuals)
    # Quantile level for (1-alpha) coverage
    # Note: alpha here is the significance level, not the NIG parameter
    quantile_index = int(math.ceil((1 - significance_level) * (n + 1))) - 1
    quantile_offset = residuals[quantile_index]
    
    return quantile_offset

def get_prediction_interval(mu_pred, quantile_offset, confidence_level=0.95):
    """
    Constructs the prediction interval using conformal calibration.
    """
    lower_bound = mu_pred - quantile_offset
    upper_bound = mu_pred + quantile_offset
    return lower_bound, upper_bound
```

**Parameters:**
*   **Significance Level ($\alpha$):** Typically 0.05 or 0.1 for 95% or 90% coverage.
*   **Calibration Set Size:** Must be sufficiently large to estimate quantiles reliably. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Data Scarcity:** Conformal calibration requires a held-out calibration set. If the simulation is expensive and data is scarce, obtaining a valid calibration set may be impossible [S1, unofficial].
*   **Full-Volume vs. Point Queries:** The CNN architecture generates predictions for the entire volume at once. This differs from point-query methods and may lead to higher memory usage and different error propagation characteristics [S1, unofficial].
*   **Precision:** The separation of aleatoric and epistemic uncertainty relies on the stability of the NIG parameter estimation. Degenerate inputs (e.g., constant fields) may cause numerical issues in the inverse-gamma parameterization [G].

## Subtopics

### Bayesian Inference Surrogate Framework
This variant shifts the goal from predicting outputs to characterizing which input parameters produce a given output feature. It trains a surrogate and then applies Bayesian inference to obtain the posterior distribution over parameters conditioned on a user-specified spatial feature [S1, unofficial]. It uses a Fisher Information Matrix prior to accelerate prior probability computation by three orders of magnitude compared to deep ensembles. Posterior characterization requires approximately 1 second of burn-in plus 100–200 sampling steps. It has been validated only on 2D time-varying fields (up to $1024 \times 256 \times 200$) with at most 4 parameters; extension to volumetric data and higher-dimensional parameter spaces remains an open challenge [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Inference Time:** Uncertainty estimation takes 0.03–0.36 seconds per prediction. This is significantly faster than Monte Carlo Dropout (0.83–25.17 s) and deep ensembles (1.60–3.00 s) [S1, unofficial].
*   **Memory:** The model outputs a full volume, requiring memory proportional to the grid resolution. The four-parameter head increases output channel count by 4x compared to a standard single-output CNN [S1, unofficial].

**Measured Performance:**
*   On three datasets (MPAS-Ocean at $192 \times 96 \times 96$, Nyx at $256^3$, CloverLeaf3D at $64^3$), ConfEviSurrogate achieved the highest voxel-level correlation between predicted uncertainty and true error, outperforming both Monte Carlo Dropout and deep ensembles [S1, unofficial].
*   On a dataset with $512^3$ resolution, PSNR dropped to 30.93 dB, which is below the VDL-Surrogate’s 33.66 dB [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Provides statistically valid confidence intervals (finite-sample coverage). Fast inference compared to ensemble-based uncertainty methods. Good correlation between predicted uncertainty and actual error.
*   **Limitations:** Requires a held-out calibration set. Full-volume generation may be memory-intensive. Performance may degrade on very high-resolution grids compared to specialized surrogates like VDL-Surrogate [S1, unofficial].

## Background
ConfEviSurrogate was introduced to address the lack of calibrated uncertainty bounds in CNN surrogates for scientific simulations [S1, unofficial]. It builds on the principles of [Uncertainty and Probabilistic Priors](7917-uncertainty-and-probabilistic-priors.md) by integrating conformal prediction with a probabilistic output head. The method is part of the broader effort to make deep learning surrogates reliable for scientific discovery, where understanding model confidence is as important as prediction accuracy [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
