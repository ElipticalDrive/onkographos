# Depth RMSE and L1 Calculation

Also called: 2D Depth L1 Metric via Random Views, 2D Depth Map Evaluation, Depth Error Metrics, Depth RMSE/L1

## Overview
Computes the error between rasterized depth and ground truth depth, optionally masking by silhouette and valid depth masks depending on training state [S1, unofficial]. It provides accuracy and completeness metrics between two depth maps, supporting absolute, relative, and log-squared error variants [S3, unofficial]. Builds on [RMSE Loss for Constant Model](14613-rmse-loss-for-constant-model.md) by extending intensity error to spatial depth evaluation.

## How it works
**Inputs and Outputs:**
- Inputs: Rasterized depth map $D_{rast}$, ground truth depth map $D_{gt}$, presence silhouette mask $M_{sil}$, valid depth mask $M_{valid}$, and training state flags `mapping_iters` and `add_new_gaussians` [S1, unofficial].
- Outputs: RMSE value, L1 error value [S1, unofficial].

**Data Structures:**
- $D_{rast}$: Float tensor of shape $(H, W)$, values $> 0$ for valid depth [S1, unofficial].
- $D_{gt}$: Float tensor of shape $(H, W)$, values $> 0$ for valid depth [S1, unofficial].
- $M_{sil}$: Boolean/Float tensor of shape $(H, W)$, 1 where silhouette is present, 0 otherwise [S1, unofficial].
- $M_{valid}$: Boolean/Float tensor of shape $(H, W)$, 1 where depth is valid, 0 otherwise [S1, unofficial].

**Steps:**
1. Determine if silhouette masking is active based on training state [S1, unofficial]. If `mapping_iters == 0` and `not add_new_gaussians`, apply $M_{sil}$ to the depth error calculation [S1, unofficial].
2. Create a validity mask $M_{valid}$ where $D_{rast} > 0$ and $D_{gt} > 0$ [S1, unofficial]. Some implementations additionally filter $0 < D_{gt} < 10$ [S2, unofficial].
3. Apply masks to the depth maps. Multiply the difference by $M_{sil}$ (if active) and $M_{valid}$ [S1, unofficial].
4. Compute RMSE by summing the masked squared differences and dividing by the sum of the valid depth mask [S1, unofficial].
5. Compute L1 error similarly using absolute difference [S1, unofficial].

**Formulas:**
For the case where `mapping_iters==0` and `not add_new_gaussians`:
$$ rmse = \frac{\sum_{i,j} \sqrt{((D_{rast} - D_{gt}) \cdot M_{sil})^2} \cdot M_{valid}}{\sum_{i,j} M_{valid}} $$
where $D_{rast}$ is rasterized depth, $D_{gt}$ is ground truth depth, $M_{sil}$ is presence silhouette mask, and $M_{valid}$ is valid depth mask [S1, unofficial].

General RMSE:
$$ RMSE = \sqrt{\frac{1}{N}\sum (d_p - d_t)^2} $$
where $d_p$ is predicted depth, $d_t$ is target depth, and $N$ is the number of valid pixels [S3, unofficial].

L1 Error:
$$ L1 = \frac{1}{N}\sum |d_p - d_t| $$

Other variants mentioned:
- Absolute difference: $|d_p - d_t|$ [S3, unofficial].
- Relative difference: $|d_p - d_t| / d_t$ [S3, unofficial].
- Squared relative: $(d_p - d_t)^2 / d_t$ [S3, unofficial].
- Log squared difference: $(\log d_p - \log d_t)^2$ [S3, unofficial].
- LogRMSE: $\sqrt{\frac{1}{N}\sum (\log d_p - \log d_t)^2}$ [S3, unofficial].
- Accuracy threshold ratio: $r = \max(d_t/d_p, d_p/d_t)$. Compute ratios $r < 1.25$, $r < 1.25^2$, $r < 1.25^3$ [S3, unofficial].

**Parameters:**
- `mapping_iters`: Number of mapping iterations. If 0, silhouette mask is applied to depth error [S1, unofficial].
- `add_new_gaussians`: Boolean flag. If false, silhouette mask is applied to depth error [S1, unofficial].
- `H`, `W`: Fixed image resolution for rendering, typically 500 [S2, unofficial].
- `focal`, `fx`, `fy`: Focal length in pixels, typically 300 [S2, unofficial].
- `cx`, `cy`: Principal point coordinates, typically $H/2.0 - 0.5$ and $W/2.0 - 0.5$ [S2, unofficial].
- `n_imgs`: Number of random camera views to sample and average over, typically 1000 [S2, unofficial].

**Pitfalls:**
- Invalid depth values ($\le 0$) are excluded by multiplying by valid_depth_mask, which is defined as $(curr\_data['depth'] > 0)$ [S1, unofficial].
- Boundary conditions for $M_{valid}$ vary: some implementations use $D_{gt} > 0$ [S1, unofficial], while others restrict to $0 < D_{gt} < 10$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_depth_metrics(d_rast, d_gt, m_sil, mapping_iters, add_new_gaussians):
    """
    Compute RMSE and L1 error between rasterized and ground truth depth.
    
    Args:
        d_rast: 2D list or array of shape (H, W), rasterized depth.
        d_gt: 2D list or array of shape (H, W), ground truth depth.
        m_sil: 2D list or array of shape (H, W), silhouette mask (1.0 if present, 0.0 otherwise).
        mapping_iters: int, number of mapping iterations.
        add_new_gaussians: bool, whether new Gaussians are being added.
        
    Returns:
        rmse: float, Root Mean Squared Error.
        l1: float, L1 Error.
    """
    H = len(d_rast)
    W = len(d_rast[0])
    
    # Step 1: Determine if silhouette masking is active
    use_sil_mask = (mapping_iters == 0) and (not add_new_gaussians)
    
    # Step 2: Create validity mask
    # Combine conditions: depth > 0. Some implementations add 0 < d_gt < 10 [S2, unofficial].
    m_valid = [[1.0 if (d_rast[i][j] > 0 and d_gt[i][j] > 0) else 0.0 for j in range(W)] for i in range(H)]
    
    # Step 3 & 4: Compute masked squared differences and sum
    sum_sq_diff = 0.0
    sum_valid = 0.0
    sum_abs_diff = 0.0
    
    for i in range(H):
        for j in range(W):
            if m_valid[i][j] == 1.0:
                sum_valid += 1.0
                diff = d_rast[i][j] - d_gt[i][j]
                if use_sil_mask:
                    diff = diff * m_sil[i][j]
                sum_sq_diff += diff * diff
                sum_abs_diff += abs(diff)
                
    # Avoid division by zero
    if sum_valid == 0:
        return 0.0, 0.0
        
    # Step 4: Compute RMSE
    rmse = math.sqrt(sum_sq_diff / sum_valid)
    
    # Step 5: Compute L1 error
    l1 = sum_abs_diff / sum_valid
    
    return rmse, l1
```

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] spla-tam/SplaTAM, https://github.com/spla-tam/SplaTAM, fetched 2026-10-27 (unofficial)
[S2] VladimirYugay/Gaussian-SLAM, https://github.com/VladimirYugay/Gaussian-SLAM, fetched 2026-10-27 (unofficial)
[S3] hku-mars/GS-SDF, https://github.com/hku-mars/GS-SDF, fetched 2026-10-27 (unofficial)
