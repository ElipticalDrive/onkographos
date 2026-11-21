# Heteroscedastic Gaussian and Asinh Composite Loss

Also called: Joint Gaussian-Asinh Objective, Ltrain

## Overview
The Heteroscedastic Gaussian and Asinh Composite Loss is a training objective for neural tomography that combines a heteroscedastic Gaussian image loss with a fixed-scale asinh image loss to optimize density and temperature fields under observational noise [S1, unofficial]. It addresses the limitation of pure Gaussian losses in physical field reconstruction by incorporating a task-aware asinh term, which yields lower log-field mean absolute error (MAE) than Gaussian-only objectives [S1, unofficial]. This composite approach balances pixel-level fidelity with robustness to noise and outliers, making it suitable for inverse volume rendering where physical consistency is prioritized over raw image pixel matching [S1, unofficial].

## How it works
The method computes a composite loss $L_{\text{train}}$ by weighting a Gaussian term $L_G$ and an asinh term $L_A$. The Gaussian term uses the known observation noise standard deviation to normalize residuals, while the asinh term uses a fixed characteristic intensity scale to normalize inputs before applying the inverse hyperbolic sine transformation.

**Inputs:**
- $\theta$: Model parameters (e.g., neural network weights).
- $q$: Ray index.
- $c$: Spectral channel index.
- $\hat{I}_{\theta,q,c}$: Predicted intensity for ray $q$ and channel $c$.
- $y_{q,c}$: Ground truth observation intensity for ray $q$ and channel $c$.
- $\sigma_{q,c}$: Target-σ, the known per-pixel, per-channel observation noise standard deviation from the generative model [S1, unofficial].
- $s_c$: Channel-dependent fixed characteristic intensity scale [S1, unofficial].
- $\lambda_G$: Weight for the Gaussian loss term [S1, unofficial].
- $\lambda_A$: Weight for the asinh loss term [S1, unofficial].
- $\eta$: Noise multiplier used for calibrating $s_c$ during training [S1, unofficial].

**Outputs:**
- $L_{\text{train}}$: The scalar composite loss value to be minimized.

**Data Structures:**
- Intensity values $\hat{I}$ and $y$ are typically floating-point numbers (e.g., float32).
- $\sigma_{q,c}$ is a pre-computed tensor of noise standard deviations matching the shape of the observations.
- $s_c$ is a scalar or vector of scales, one per channel.

**Parameters:**
- $\lambda_G$: Typical values tested are $1, 10^{-1}, 10^{-3}, 10^{-5}, 0$ [S1, unofficial].
- $\lambda_A$: Fixed at $1$ in hybrid experiments; $0$ for pure Gaussian [S1, unofficial].
- $s_c$: For noise multipliers $\eta > 1$, $s_c$ is scaled by $\sqrt{\eta}$. For $\eta \in \{0, 1\}$, $s_c$ remains fixed at its baseline value. The scale remains constant within a single training run [S1, unofficial].

**Formulas:**
The composite loss is defined as:
$$ L_{\text{train}} = \lambda_G L_G + \lambda_A L_A $$

The Gaussian loss term $L_G$ is the inverse-variance-weighted squared image residual:
$$ L_G = \left( \frac{\hat{I}_{\theta,q,c} - y_{q,c}}{\sigma_{q,c}} \right)^2 $$
This is equivalent to the Gaussian Negative Log-Likelihood (NLL) up to a positive constant factor and additive terms independent of $\theta$ [S1, unofficial].

The asinh loss term $L_A$ is the absolute difference of asinh-transformed normalized intensities:
$$ L_A = \left| \text{asinh}\left(\frac{\hat{I}_{q,c}}{s_c}\right) - \text{asinh}\left(\frac{y_{q,c}}{s_c}\right) \right| $$
Note: In the "target-σ" oracle variant, $s_c$ is replaced by $\sigma_{q,c}$, but the standard method uses the fixed $s_c$ [S1, unofficial].

**Algorithm:**
1. **Initialize** model parameters $\theta$.
2. **Set** loss weights $\lambda_G$ and $\lambda_A$.
3. **Determine** the noise multiplier $\eta$ for the current training configuration.
4. **Compute** the asinh scale $s_c$:
   - If $\eta > 1$, set $s_c = s_{c,\text{base}} \times \sqrt{\eta}$.
   - If $\eta \in \{0, 1\}$, set $s_c = s_{c,\text{base}}$.
   - [G] Ensure $s_c$ is not zero to avoid division by zero in asinh normalization.
5. **For** each training batch:
   a. **Sample** rays $q$ and channels $c$.
   b. **Render** predicted intensities $\hat{I}_{\theta,q,c}$ using the current model $\theta$.
   c. **Retrieve** ground truth observations $y_{q,c}$ and noise standard deviations $\sigma_{q,c}$.
   d. **Compute** $L_G$:
      $$ L_G = \left( \frac{\hat{I}_{\theta,q,c} - y_{q,c}}{\sigma_{q,c}} \right)^2 $$
   e. **Compute** $L_A$:
      $$ L_A = \left| \text{asinh}\left(\frac{\hat{I}_{q,c}}{s_c}\right) - \text{asinh}\left(\frac{y_{q,c}}{s_c}\right) \right| $$
   f. **Compute** composite loss:
      $$ L_{\text{train}} = \lambda_G L_G + \lambda_A L_A $$
   g. **Backpropagate** $L_{\text{train}}$ to update $\theta$ via gradient descent.
6. **Repeat** until convergence.

**Edge Cases and Pitfalls:**
- **Division by Zero:** If $\sigma_{q,c} = 0$, the Gaussian loss term becomes undefined. The sources imply $\sigma_{q,c}$ is the "known per-pixel, per-channel observation noise standard deviation," which should be non-zero for noisy data. If $\sigma_{q,c}$ is zero (noise-free), the Gaussian term may need a small epsilon or the loss should default to MSE [G].
- **Sparse Views + High Noise:** The combination of sparse views (e.g., $V=20$) and high noise ($\eta=25$) leads to a failure mode in latent physical-field recovery, with density MAE increasing to 0.336 and temperature MAE to 0.380 [S1, unofficial].
- **Scale Calibration:** The asinh scale $s_c$ must be calibrated to the noise level $\eta$ for $\eta > 1$ to track the noise standard deviation. Using a fixed $s_c$ without scaling for high noise may degrade performance [S1, unofficial].

## Subtopics

### Pure Gaussian Loss
Uses only $L_G$ with $\lambda_A=0$. Performs worst among tested objectives [S1, unofficial].

### Pure Asinh Loss
Uses only $L_A$ with $\lambda_G=0$. Yields the lowest log-field MAE and is the preferred task-aware objective for reconstruction [S1, unofficial].

### Target-σ Asinh Loss
An oracle variant where $s_c$ is replaced by $\sigma_{q,c}$ in $L_A$. Used for diagnostic ablation, not standard training [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Training a single reconstruction takes approximately 3 GPU-hours on an 80 GB NVIDIA A100. A $K=10$ ensemble takes approximately 30 aggregate GPU-hours [S1, unofficial]. Rendering a 300-view set takes 10-100 minutes on a local NVIDIA RTX 4090, depending on resolution and integration step size [S1, unofficial].
- **Memory Complexity:** The multiresolution hash grid uses approximately 40M parameters (2M per hash table, 20 levels) [S1, unofficial].
- **Performance:** The pure asinh objective yields lower log-field MAE than the pure Gaussian objective. Hybrid losses land between the two regimes [S1, unofficial].
- **Scaling:** The method is robust to strong heteroscedastic observation noise ($\eta=25$) when sufficient angular coverage (300 views) is provided. However, it is unstable under sparse views and high noise [S1, unofficial].
- **Strengths:** Robust to noise; better physical field reconstruction than Gaussian-only loss; computationally efficient compared to classical grids [S1, unofficial].
- **Limitations:** Fails under sparse views + high noise; requires careful calibration of $s_c$ for different noise levels [S1, unofficial].
- **When to Use:** When physical field fidelity (density/temperature) is more important than pixel-level image fidelity, and observational noise is present [S1, unofficial].
- **When Not to Use:** When views are sparse and noise is high, or when pixel-level image fidelity is the primary goal [S1, unofficial].

## Background
The Heteroscedastic Gaussian and Asinh Composite Loss was introduced in the context of joint thermodynamic reconstruction and error localization in neural tomography [S1, unofficial]. It was developed to address the limitations of standard Gaussian losses in reconstructing physical fields (density and temperature) from noisy observational data [S1, unofficial]. The method is part of a broader effort to improve the accuracy of neural radiance field-based tomography for solar physics applications [S1, unofficial].

## Sources
[S1] Image Fidelity is Not Field Fidelity: Joint Thermodynamic Reconstruction and Error Localization in Neural Tomography, https://doi.org/10.48550/arxiv.2609.28868, fetched 2026-10-26 (unofficial)
