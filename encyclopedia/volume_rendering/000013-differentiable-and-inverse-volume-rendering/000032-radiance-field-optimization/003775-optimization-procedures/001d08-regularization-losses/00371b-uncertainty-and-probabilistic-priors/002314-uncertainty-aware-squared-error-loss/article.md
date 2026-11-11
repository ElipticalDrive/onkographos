# Uncertainty-aware Squared Error Loss

Also called: Squared Error Loss, Uncertainty-weighted MSE

## Overview
This method minimizes the squared error between rendered and ground-truth values by incorporating a predicted uncertainty parameter $\delta$ for each sample point. It differs from standard Mean Squared Error (MSE) by dynamically weighting the error based on the model's confidence, and from [Uncertainty Prediction Loss](2961-uncertainty-prediction-loss.md) by using a specific max-based formulation that penalizes uncertainty less than the raw squared error to prevent $\delta$ from exploding. It builds on [Uncertainty and Probabilistic Priors](7917-uncertainty-and-probabilistic-priors.md).

## How it works
**Inputs:**
- Set of rays $\mathcal{R}$.
- For each ray $r \in \mathcal{R}$, $N$ sample points $c_i$.
- Ground truth color $C(r)$ and rendered color $\hat{C}(r)$.
- Predicted uncertainty $\delta_{c_i}$ for each sample point $c_i$.

**Outputs:**
- Scalar loss value $\mathcal{L}_{SE}$.

**Data Structures:**
- $\delta_{c_i}$: Scalar float representing predicted uncertainty (variance proxy) at coordinate $c_i$.

**Steps:**
1. Calculate the squared error $E(r)$ for each ray $r$:
   $$E(r) = \left\| \hat{C}(r) - C(r) \right\|_2^2$$
2. For each sample point $i$ along ray $r$, compute the term $\max(E(r) - \delta_{c_i}, 0)$. This ensures that if the predicted uncertainty $\delta_{c_i}$ exceeds the squared error, the contribution to the loss is zero.
3. Sum these terms over all sample points $i=1 \dots N$ and all rays $r \in \mathcal{R}$:
   $$\mathcal{L}_{SE} = \sum_{r \in \mathcal{R}} \sum_{i=1}^N \max(E(r) - \delta_{c_i}, 0)$$

**Regularization:**
To prevent $\delta_{c_i}$ from growing indefinitely (blowing up) to minimize the loss, a regularization term $\mathcal{L}_0$ is added:
$$\mathcal{L}_0 = \sum_{r \in \mathcal{R}} \sum_{i=1}^N \max(\delta_{c_i}, 0)$$
The total loss is a weighted sum of $\mathcal{L}_{SE}$ and $\mathcal{L}_0$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uncertainty_aware_squared_error_loss(rendered_colors, ground_truth_colors, uncertainties):
    """
    rendered_colors: Tensor of shape (num_rays, num_samples, 3)
    ground_truth_colors: Tensor of shape (num_rays, num_samples, 3)
    uncertainties: Tensor of shape (num_rays, num_samples)
    """
    # Step 1: Calculate squared error per ray
    # Note: E(r) is typically computed per ray, but the formula sums over samples.
    # Assuming rendered_colors and ground_truth_colors are aggregated per ray 
    # or the error is computed per sample. The source formula E(r) suggests ray-level error,
    # but the sum is over i=1..N. We assume E(r) is constant for all samples on ray r,
    # or the error is computed per sample. Given "E(r) is the squared error of ray r",
    # we compute ray-level MSE first.
    
    # Compute per-ray squared error
    # If inputs are per-sample, we might need to aggregate. 
    # Standard NeRF rendering produces one color per ray. 
    # However, the loss is summed over samples i. 
    # This implies E(r) is broadcast or the error is per-sample.
    # Let's assume the standard interpretation: E(r) is the error for the final rendered pixel.
    # But the sum is over N samples. This is ambiguous. 
    # Re-reading S1: "E(r) is the squared error of ray r... ci is the sample point... delta_ci is predicted uncertainty for query coordinates ci".
    # It is likely that E(r) is computed once per ray, and applied to all samples, 
    # OR the error is computed per sample. 
    # Given "max(E(r) - delta_ci, 0)", if E(r) is scalar per ray, it applies to all samples.
    
    # Let's assume E(r) is the MSE for the entire ray's rendered color vs GT color.
    # rendered_colors shape: (num_rays, 3)
    # ground_truth_colors shape: (num_rays, 3)
    
    ray_error = torch.sum((rendered_colors - ground_truth_colors) ** 2, dim=1) # Shape: (num_rays,)
    
    # Expand to match uncertainty shape (num_rays, num_samples)
    ray_error_expanded = ray_error.unsqueeze(1) # Shape: (num_rays, 1)
    
    # Step 2 & 3: Compute L_SE
    # max(E(r) - delta_ci, 0)
    diff = ray_error_expanded - uncertainties # Shape: (num_rays, num_samples)
    l_se = torch.sum(torch.clamp(diff, min=0))
    
    # Regularization L_0
    l_0 = torch.sum(torch.clamp(uncertainties, min=0))
    
    return l_se, l_0
```

## Subtopics

### Pixelwise Uncertainty Weighting
A training technique that uses rendered uncertainties to weight the regression loss, encouraging the network to memorize well-rendered regions and ignore uncertain/noisy regions [S2]. It involves ignoring pixels with RGB-D uncertainties exceeding predefined thresholds and weighing remaining pixels using $\kappa = e^{-2(\sigma_c^2 + \sigma_z^2)}$, where $\sigma_c^2$ and $\sigma_z^2$ are color and depth variances [S2]. This replaces the error term $(q - \gamma)$ with $\kappa (q - \gamma)$ in the loss equation [S2].

## Operating characteristics
- **Complexity:** $O(N \cdot R)$ where $N$ is samples per ray and $R$ is number of rays.
- **Strengths:** Robust to noise by down-weighting uncertain predictions; prevents overfitting to noisy data points.
- **Limitations:** Requires predicting uncertainty $\delta$; sensitive to threshold selection in pixelwise variants.
- **When to use:** When training data contains significant noise or artifacts.

## Background
Originated in Recursive-NeRF [S1] as an efficient dynamically growing NeRF optimization procedure. It was later adapted in U-NeRF/E-SCRNet pipelines for uncertainty-aware visual localization [S2].

## Sources
[S1] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S2] Leveraging Neural Radiance Fields for Uncertainty-Aware Visual Localization, https://doi.org/10.1109/icra57147.2024.10610126, fetched 2026-10-26
