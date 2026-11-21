# Uncertainty-aware Squared Error Loss

Also called: Heteroscedastic Gaussian Loss, Squared Error Loss, Uncertainty-Aware Loss Functions, Uncertainty-weighted MSE

## Overview
Uncertainty-aware Squared Error Loss methods model photometric observations using probabilistic distributions (typically Gaussian) where the variance represents aleatoric uncertainty, allowing the optimization to downweight unreliable or noisy observations [S2]. Unlike standard Mean Squared Error (MSE) which treats all pixels equally, these losses dynamically adjust the contribution of each pixel based on the network's predicted confidence, effectively preserving sharp boundaries while smoothing valid surfaces [S2]. This approach builds on [Uncertainty and Probabilistic Priors](7917-uncertainty-and-probabilistic-priors.md) by explicitly predicting variance parameters alongside scene properties to regularize the data fidelity term [2961-uncertainty-prediction-loss.md][10417-confevisurrogate.md].

## How it works
The method extends the standard differentiable rendering pipeline by adding an uncertainty prediction head. The core mechanism involves two primary formulations found in the literature: a max-based squared error penalty and a negative log-likelihood (NLL) Gaussian loss.

### Inputs and Outputs
*   **Inputs:**
    *   Ray $r$ and its associated sample points $c_i$.
    *   Rendered color/intensity $I_p$ (or $\hat{C}(r)$) for the ray.
    *   Ground truth color/intensity $I_{gt}$ (or $C(r)$).
    *   Predicted uncertainty $\beta_i$ (or $\delta_{c_i}$) for each sample point or pixel.
*   **Outputs:**
    *   Scalar loss value $L$ to be minimized during gradient descent.

### Data Structures
The network outputs an additional channel for uncertainty. For a query coordinate $c_i$, the network predicts a scalar $\beta_i \in \mathbb{R}^+$ (variance proxy) or $\delta_{c_i} \in \mathbb{R}^+$.
*   **Layout:** Typically a single float per sample point/pixel.
*   **Initialization:** Not given in the sources. [G] Common practice initializes uncertainty to small positive values or zeros to prevent early collapse, but specific initialization schemes vary by implementation (e.g., Recursive-NeRF uses alpha linear initialization for related stability).

### Formulas and Steps

#### Variant 1: Max-Based Squared Error (Recursive-NeRF)
This formulation penalizes the uncertainty prediction $\delta$ if it underestimates the squared error $E(r)$ [S1].

1.  Calculate the squared error for ray $r$:
    $$ E(r) = \| \hat{C}(r) - C(r) \|_2^2 $$
    where $\hat{C}(r)$ is the rendered color and $C(r)$ is the ground truth color [S1].

2.  Compute the loss $\mathcal{L}_{SE}$:
    $$ \mathcal{L}_{SE} = \sum_{r \in \mathcal{R}} \sum_{i=1}^N \max(E(r) - \delta_{c_i}, 0) $$
    where $\mathcal{R}$ is the set of rays, $N$ is the number of samples per ray, and $\delta_{c_i}$ is the predicted uncertainty for sample point $c_i$ [S1].
    *   **Logic:** If the squared error $E(r)$ is greater than the predicted uncertainty $\delta_{c_i}$, the term is positive, penalizing the model. If $\delta_{c_i}$ is large (high uncertainty), the penalty is reduced. This punishes $\delta_i$ less than the squared error [S1].

3.  Apply regularization to prevent $\delta$ from blowing up:
    $$ \mathcal{L}_0 = \sum_{r \in \mathcal{R}} \sum_{i=1}^N \max(\delta_{c_i}, 0) $$
    This encourages $\delta_i$ to be close to zero when the error is low [S1].

4.  Total Loss:
    $$ \mathcal{L}_{total} = \mathcal{L}_{SE} + \lambda \mathcal{L}_0 $$
    where $\lambda$ is a weighting factor. Not given in the sources.

#### Variant 2: Negative Log-Likelihood Gaussian Loss (Satellite NeRF)
This formulation derives the loss from the negative log-likelihood of a Gaussian distribution with mean $\mu = I_{gt}$ and variance $\beta_i^2$ [S2].

1.  For each pixel $i$ in the image:
    *   Let $I_{p1}^i$ be the NeRF-rendered pixel value.
    *   Let $I_{gt}^i$ be the ground truth pixel value.
    *   Let $\beta_i$ be the predicted per-ray uncertainty (standard deviation proxy).

2.  Compute the photometric loss $L_{rgb1}$:
    $$ L_{rgb1} = \frac{1}{N} \sum_{i=1}^{N} \left( \frac{(I_{p1}^i - I_{gt}^i)^2}{2\beta_i^2} + \log \beta_i \right) $$
    where $N$ is the total number of pixels [S2].
    *   **Logic:** The term $\frac{1}{2\beta_i^2}$ acts as a dynamic weight. High uncertainty ($\beta_i$) reduces the weight of the squared error term. The $\log \beta_i$ term prevents $\beta_i$ from collapsing to zero [S2].

3.  (Optional) For physical imaging model consistency, compute $L_{rgb2}$ using synthetic physical images $I_{p2}^i$:
    $$ L_{rgb2} = \frac{1}{N} \sum_{i=1}^{N} \frac{(I_{p2}^i - I_{gt}^i)^2}{2\beta_i^2} $$
    This uses transient uncertainty $\beta_i$ [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uncertainty_aware_squared_error_loss(rendered_color, ground_truth_color, predicted_uncertainty, mode='nll'):
    """
    Compute uncertainty-aware loss.
    
    Args:
        rendered_color: Tensor of shape (N,) or (H, W, C), rendered pixel values.
        ground_truth_color: Tensor of same shape, ground truth pixel values.
        predicted_uncertainty: Tensor of same spatial shape, predicted uncertainty (beta or delta).
        mode: 'max_se' for Recursive-NeRF style, 'nll' for Gaussian NLL style.
        
    Returns:
        Scalar loss tensor.
    """
    # Ensure uncertainty is positive to avoid division by zero or log(0)
    # [G] Small epsilon added for numerical stability, standard in ML
    eps = 1e-6
    uncertainty = torch.clamp(predicted_uncertainty, min=eps)
    
    if mode == 'nll':
        # Variant 2: Negative Log-Likelihood
        # L = 1/N * sum( (error^2 / (2 * beta^2)) + log(beta) )
        squared_error = (rendered_color - ground_truth_color) ** 2
        # Note: In S2, beta is used directly in denominator as beta^2, 
        # implying beta is the standard deviation or scale parameter.
        # Formula: (I_p - I_gt)^2 / (2 * beta^2)
        term1 = squared_error / (2.0 * uncertainty ** 2)
        term2 = torch.log(uncertainty)
        loss = torch.mean(term1 + term2)
        
    elif mode == 'max_se':
        # Variant 1: Max-Based Squared Error
        # L_SE = sum( max(E(r) - delta, 0) )
        # Note: S1 sums over rays and samples. Here we assume flat tensors.
        squared_error = torch.norm(rendered_color - ground_truth_color, p=2, dim=-1) ** 2
        # delta is predicted_uncertainty
        delta = predicted_uncertainty
        
        # max(E(r) - delta, 0)
        se_term = torch.clamp(squared_error - delta, min=0.0)
        loss_se = torch.sum(se_term)
        
        # Regularization L_0 = sum(max(delta, 0))
        # [G] S1 uses max(delta, 0), but delta should be positive. 
        # We clamp to ensure non-negative regularization.
        reg_term = torch.clamp(delta, min=0.0)
        loss_reg = torch.sum(reg_term)
        
        # [G] Lambda is not specified in S1. Default to 1.0 or configurable.
        lambda_reg = 1.0
        loss = loss_se + lambda_reg * loss_reg
        
    else:
        raise ValueError("Unknown mode")
        
    return loss
```

### Parameters
*   **$\lambda$ (Regularization weight):** Not given in the sources. [G] Typically a small value (e.g., $10^{-3}$ to $10^{-1}$) to balance the uncertainty regularization.
*   **Thresholds (for Pixelwise Weighting):** Predefined thresholds for RGB-D uncertainties to ignore pixels. Not given in the sources.

### Edge Cases and Pitfalls
*   **Uncertainty Collapse:** If the uncertainty $\beta_i$ or $\delta_{c_i}$ collapses to zero, the loss becomes unstable (division by zero or infinite gradient). The $\log \beta_i$ term in NLL or the $\max(\delta, 0)$ regularization helps mitigate this [S2][S1].
*   **Boundary Artifacts:** Without proper uncertainty modeling, sharp boundaries may be smoothed out. The uncertainty-aware loss preserves sharp boundaries by assigning higher uncertainty to ambiguous regions [S2].
*   **Initialization:** Poor initialization of uncertainty can lead to slow convergence. Recursive-NeRF uses alpha linear initialization to improve stability [S1].

## Subtopics

### Pixelwise Uncertainty Weighting
A training technique that uses rendered uncertainties to weight the regression loss, encouraging the network to memorize well-rendered regions and ignore uncertain/noisy regions [S3].
*   **Mechanism:**
    1.  Ignore pixels in the loss calculation if their RGB-D uncertainties exceed predefined thresholds [S3].
    2.  Weigh the remaining pixels using a function $\kappa = e^{-2(\sigma_c^2 + \sigma_z^2)}$, where $\sigma_c^2$ and $\sigma_z^2$ are color and depth variances from U-NeRF [S3].
    3.  Replace the error term $(q - \gamma)$ with $\kappa (q - \gamma)$ in the E-SCRNet loss equation [S3].
*   **When to choose:** Use when dealing with noisy RGB-D data or when specific pixels are known to be unreliable (e.g., reflections, occlusions) and need to be downweighted rather than ignored entirely [S3].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels or samples, similar to standard MSE, with minor overhead for computing uncertainty and log/exponential functions.
*   **Memory Complexity:** $O(N)$ to store the predicted uncertainty maps alongside color/density fields.
*   **Performance:** Improves robustness under complex lighting conditions and geometric distortions by dynamically weighing observations [S2]. It helps overcome blurred boundaries and enhances reconstruction accuracy of terrain structures and object edges [S2].
*   **Strengths:** Handles aleatoric uncertainty effectively; preserves sharp boundaries; robust to noise.
*   **Limitations:** Requires an additional network output (uncertainty); sensitive to initialization of uncertainty parameters; may require careful tuning of regularization weights.
*   **When to use:** When the input data contains significant noise, illumination inconsistencies, or when robust reconstruction of sharp features is critical.
*   **When not to use:** When computational resources are extremely limited and the overhead of an extra output head is prohibitive, or when the data is clean and well-calibrated.

## Background
The concept of modeling photometric observations with Gaussian distributions where variance represents aleatoric uncertainty was introduced in the context of neural radiance fields to tackle illumination inconsistency and geometric distortion [S2]. Recursive-NeRF proposed a max-based squared error loss to prevent uncertainty from blowing up while penalizing underestimation of error [S1]. Pixelwise uncertainty weighting was later applied in U-NeRF and E-SCRNet to improve visual localization by leveraging RGB-D uncertainties [S3]. These methods are part of a broader trend in differentiable rendering to incorporate probabilistic priors for more robust scene reconstruction [2961-uncertainty-prediction-loss.md][10417-confevisurrogate.md].

## Sources
[S1] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S2] Shading and Geometric Constraint Neural Radiance Field for DSM Reconstruction from Multi-View Satellite Images, https://doi.org/10.3390/rs18071091, fetched 2026-11-05
[S3] Leveraging Neural Radiance Fields for Uncertainty-Aware Visual Localization, https://doi.org/10.1109/icra57147.2024.10610126, fetched 2026-10-26
