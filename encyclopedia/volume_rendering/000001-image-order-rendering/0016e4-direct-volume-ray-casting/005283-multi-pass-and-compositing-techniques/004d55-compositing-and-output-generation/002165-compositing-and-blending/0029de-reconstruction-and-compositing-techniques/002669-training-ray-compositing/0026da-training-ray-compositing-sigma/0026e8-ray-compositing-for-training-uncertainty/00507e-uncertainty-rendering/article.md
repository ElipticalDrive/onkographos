# Uncertainty Rendering

Also called: Uncertainty Estimation in Ray Compositing, UncertaintyRenderer

## Overview
Uncertainty Rendering estimates the epistemic uncertainty of a volume rendering result by aggregating per-sample uncertainty metrics along a ray. It solves the problem of quantifying confidence in the final pixel color by weighting local uncertainty estimates ($\beta$) by the ray compositing weights. It differs from standard direct volume rendering by outputting an additional uncertainty channel alongside RGB, derived from the variance of the neural field's predictions rather than just the mean [S2, S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `weights` (tensor of shape $[N_{rays}, N_{samples}]$), `betas` (tensor of shape $[N_{rays}, N_{samples}]$ representing per-sample uncertainty).
- Output: `uncertainty` (tensor of shape $[N_{rays}]$).

**Data Structures:**
- `weights`: Transmittance weights $w_i$ for each sample $i$ along the ray, computed as $w_i = T_i (1 - e^{-\sigma_i \delta_i})$.
- `betas`: Scalar uncertainty value $\beta_i$ associated with each sample, typically derived from the variance of the predicted density or color.

**Steps:**
1. For each ray, compute the weighted sum of per-sample uncertainties.
2. The final uncertainty is the sum of the product of weights and betas across all samples along the ray.

**Formula:**
$$
\text{uncertainty} = \sum_{i=1}^{N} w_i \cdot \beta_i
$$
Where:
- $N$ is the number of samples along the ray.
- $w_i$ is the weight of sample $i$.
- $\beta_i$ is the uncertainty of sample $i$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_uncertainty(weights: torch.Tensor, betas: torch.Tensor) -> torch.Tensor:
    """
    Compute ray-wise uncertainty by weighting per-sample uncertainties by transmittance weights.
    
    Args:
        weights: Tensor of shape (num_rays, num_samples) containing ray compositing weights.
        betas: Tensor of shape (num_rays, num_samples) containing per-sample uncertainty values.
        
    Returns:
        Tensor of shape (num_rays,) containing the aggregated uncertainty for each ray.
    """
    # Element-wise multiplication of weights and betas
    weighted_betas = weights * betas
    
    # Sum along the sample dimension (dim=-2)
    uncertainty = torch.sum(weighted_betas, dim=-2)
    
    return uncertainty
```

**Edge Cases and Pitfalls:**
- If `weights` or `betas` contain NaNs, the result will be NaN. Ensure inputs are valid [G].
- If `betas` are not normalized or scaled appropriately, the uncertainty values may not be interpretable relative to the RGB output [G].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N_{rays} \times N_{samples})$ for the element-wise multiplication and summation. This is negligible compared to the ray casting and neural network inference steps.
- **Memory Complexity:** $O(N_{rays} \times N_{samples})$ to store intermediate `weighted_betas` if not fused, or $O(N_{rays})$ if computed in-place.
- **Strengths:** Provides a per-pixel confidence metric without significant additional computational overhead. Useful for adaptive sampling or filtering noisy reconstructions.
- **Limitations:** The quality of uncertainty depends entirely on the quality of the per-sample `betas`. If `betas` are poorly estimated, the uncertainty metric is misleading.
- **When to use:** When visualizing confidence in neural radiance fields or volume renderings, especially for downstream tasks like adaptive sampling or outlier detection.
- **When not to use:** When only RGB output is required and computational budget is extremely tight (though the cost is minimal).

## Background
Uncertainty rendering in the context of ray compositing is primarily associated with the SDFStudio framework, which extends NeRF-based methods to include uncertainty estimation [S2, unofficial]. The method leverages the probabilistic nature of some neural field formulations or post-hoc variance estimation to provide $\beta$ values, which are then composited similarly to RGB [S2, unofficial]. It is used in surface reconstruction pipelines to identify regions of low confidence [S2, unofficial].

## Sources
[S1] nerfstudio-project/nerfstudio, https://github.com/nerfstudio-project/nerfstudio, fetched 2026-10-26 (unofficial)
[S2] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
