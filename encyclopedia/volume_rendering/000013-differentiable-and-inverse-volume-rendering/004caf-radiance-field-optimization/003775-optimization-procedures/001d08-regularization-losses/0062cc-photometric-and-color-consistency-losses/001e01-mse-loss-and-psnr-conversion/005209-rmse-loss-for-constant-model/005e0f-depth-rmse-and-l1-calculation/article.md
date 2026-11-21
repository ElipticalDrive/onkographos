# Depth RMSE and L1 Calculation

Also called: 2D Depth Map Evaluation, Depth Error Metrics, Depth RMSE/L1

## Overview
Computes the error between rasterized depth and ground truth depth, optionally masking by silhouette and valid depth masks depending on training state [S1, unofficial]. It provides accuracy and completeness metrics between two depth maps, supporting absolute, relative, and log-squared error variants [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Inputs: Rasterized depth map $D_{rast}$, ground truth depth map $D_{gt}$, presence silhouette mask $M_{sil}$, valid depth mask $M_{valid}$, and training state flags `mapping_iters` and `add_new_gaussians`.
- Outputs: RMSE value, L1 error value.

**Data Structures:**
- $D_{rast}$: Float tensor of shape $(H, W)$, values $> 0$ for valid depth.
- $D_{gt}$: Float tensor of shape $(H, W)$, values $> 0$ for valid depth.
- $M_{sil}$: Boolean/Float tensor of shape $(H, W)$, 1 where silhouette is present, 0 otherwise.
- $M_{valid}$: Boolean/Float tensor of shape $(H, W)$, 1 where depth is valid, 0 otherwise.

**Steps:**
1. Determine if silhouette masking is active. If `mapping_iters == 0` and `not add_new_gaussians`, apply $M_{sil}$ to the depth error calculation [S1, unofficial].
2. Create a validity mask $M_{valid}$ where $D_{rast} > 0$ and $0 < D_{gt} < 10$ [S2, unofficial]. Note: Some implementations define $M_{valid}$ simply as $(D_{gt} > 0)$ to exclude invalid depth values ($\le 0$) [S1, unofficial].
3. Apply masks to the depth maps. Multiply the difference by $M_{sil}$ (if active) and $M_{valid}$ [S1, unofficial].
4. Compute RMSE by summing the masked squared differences and dividing by the sum of the valid depth mask [S1, unofficial].
5. Compute L1 error similarly using absolute difference [S1, unofficial].

**Formulas:**
For the case where `mapping_iters==0` and `not add_new_gaussians`:
$$ rmse = \frac{\sum_{i,j} \sqrt{((D_{rast} - D_{gt}) \cdot M_{sil})^2} \cdot M_{valid}}{\sum_{i,j} M_{valid}} $$
where $D_{rast}$ is rasterized depth, $D_{gt}$ is ground truth depth, $M_{sil}$ is presence silhouette mask, and $M_{valid}$ is valid depth mask [S1, unofficial].

General RMSE:
$$ RMSE = \sqrt{\frac{1}{N}\sum (d_p - d_t)^2} $$
where $d_p$ is predicted depth, $d_t$ is target depth, and $N$ is the number of valid pixels [S2, unofficial].

L1 Error:
$$ L1 = \frac{1}{N}\sum |d_p - d_t| $$

Other variants mentioned:
- Absolute difference: $|d_p - d_t|$ [S2, unofficial].
- Relative difference: $|d_p - d_t| / d_t$ [S2, unofficial].
- Squared relative: $(d_p - d_t)^2 / d_t$ [S2, unofficial].
- Log squared difference: $(\log d_p - \log d_t)^2$ [S2, unofficial].
- LogRMSE: $\sqrt{\frac{1}{N}\sum (\log d_p - \log d_t)^2}$ [S2, unofficial].
- Accuracy threshold ratio: $r = \max(d_t/d_p, d_p/d_t)$. Compute ratios $r < 1.25$, $r < 1.25^2$, $r < 1.25^3$ [S2, unofficial].

**Parameters:**
- `mapping_iters`: Number of mapping iterations. If 0, silhouette mask is applied to depth error [S1, unofficial].
- `add_new_gaussians`: Boolean flag. If false, silhouette mask is applied to depth error [S1, unofficial].

**Pitfalls:**
- Invalid depth values ($\le 0$) are excluded by multiplying by valid_depth_mask, which is defined as $(curr\_data['depth'] > 0)$ [S1, unofficial].
- Ensure $M_{valid}$ handles the boundary condition $0 < D_{gt} < 10$ if specified by the source [S2, unofficial], or just $D_{gt} > 0$ if following the other source [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_depth_metrics(d_rast, d_gt, m_sil, mapping_iters, add_new_gaussians):
    """
    Compute RMSE and L1 error between rasterized and ground truth depth.
    
    Args:
        d_rast: Tensor (H, W), rasterized depth.
        d_gt: Tensor (H, W), ground truth depth.
        m_sil: Tensor (H, W), silhouette mask (1 if present, 0 otherwise).
        mapping_iters: int, number of mapping iterations.
        add_new_gaussians: bool, whether new Gaussians are being added.
        
    Returns:
        rmse: float, Root Mean Squared Error.
        l1: float, L1 Error.
    """
    # Step 1: Determine if silhouette masking is active
    use_sil_mask = (mapping_iters == 0) and (not add_new_gaussians)
    
    # Step 2: Create validity mask
    # Source [S2, unofficial] suggests 0 < d_gt < 10, Source [S1, unofficial] suggests d_gt > 0
    # We use the more restrictive one from [S2, unofficial] if available, otherwise [S1, unofficial]
    # Note: The exact threshold 10 might be dataset-specific. 
    # Here we follow [S1, unofficial] for simplicity as it's more common in SLAM contexts,
    # but note the discrepancy.
    m_valid = (d_gt > 0) & (d_rast > 0)
    
    # Step 3: Apply masks
    if use_sil_mask:
        # Multiply difference by silhouette mask
        diff = (d_rast - d_gt) * m_sil
    else:
        diff = d_rast - d_gt
        
    # Multiply by valid mask to exclude invalid pixels
    masked_diff = diff * m_valid
    
    # Step 4: Compute RMSE
    # Sum of squared differences
    sum_sq_diff = torch.sum(masked_diff ** 2)
    # Sum of valid mask (number of valid pixels)
    sum_valid = torch.sum(m_valid)
    
    # Avoid division by zero
    if sum_valid == 0:
        return 0.0, 0.0
        
    rmse = torch.sqrt(sum_sq_diff / sum_valid)
    
    # Step 5: Compute L1 error
    sum_abs_diff = torch.sum(torch.abs(masked_diff))
    l1 = sum_abs_diff / sum_valid
    
    return rmse.item(), l1.item()
```

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] spla-tam/SplaTAM, https://github.com/spla-tam/SplaTAM, fetched 2026-10-27 (unofficial)
[S2] hku-mars/GS-SDF, https://github.com/hku-mars/GS-SDF, fetched 2026-10-27 (unofficial)
