# DSSIM Loss

Also called: SSIM, Combined DSSIM and MSE Loss, DSSIM-2D + MSE, DVAO Loss Function, Structural Dissimilarity Index Loss

## Overview
DSSIM Loss is a composite regularization loss used in volumetric ambient occlusion (VAO) and 3D Gaussian Splatting optimization to balance structural similarity and pixel-wise accuracy [S1][S2, unofficial]. It addresses the limitation of pure Mean Squared Error (MSE) by penalizing structural discrepancies between rendered predictions and ground truth targets, which is critical for preserving high-frequency details in volumetric data [S1]. It sets itself apart from its parent [SSIM Metric Calculation](3650-ssim-metric-calculation.md) by incorporating a weighted MSE term and adapting the SSIM computation to 3D volumes via 2D slice averaging or full 3D neighborhoods [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Prediction volume $p$ and target volume $t$, both of shape $W \times H \times D$.
*   **Outputs:** A scalar loss value $L(p,t)$.

**Data Structures:**
*   Volumes are treated as dense arrays of voxels.
*   Local neighborhoods are defined either as 2D sliding windows (for DSSIM-2D) or 3D cubic blocks (for DSSIM-3D) [S1].

**Parameters:**
*   $c_1 = 0.01$: Stability constant for mean calculation [S1].
*   $c_2 = 0.03$: Stability constant for variance/covariance calculation [S1].
*   $\alpha = 5$: Hyperparameter balancing DSSIM and MSE terms [S1]. Training is not highly sensitive to this value [S1].
*   $\lambda$: Hyperparameter balancing MSE and SSIM in the 3DGS variant [S2, unofficial].

**Formulas:**
The total loss is a combination of Structural Dissimilarity (DSSIM) and Mean Squared Error (MSE) [S1][S2, unofficial]:

$$ L(p,t) = \text{DSSIM}(p,t) + \alpha \cdot \text{MSE}(p,t) $$

Where:
$$ \text{MSE}(p,t) = \frac{1}{N} \sum_{i=1}^{N} (p_i - t_i)^2 $$
$$ \text{DSSIM}(p,t) = 1 - \frac{(2\mu_p \mu_t + c_1)(2\sigma_{pt} + c_2)}{(\mu_p^2 + \mu_t^2 + c_1)(\sigma_p^2 + \sigma_t^2 + c_2)} $$

Here, $N = W \cdot H \cdot D$ is the total number of voxels. $\mu_x$, $\sigma_x^2$, and $\sigma_{xy}$ represent the mean, variance, and covariance within a local neighborhood for volumes $x$ and $y$ [S1].

**Steps:**
1.  **Compute Local Statistics:** For each voxel, calculate the local mean ($\mu$), variance ($\sigma^2$), and covariance ($\sigma_{pt}$) of the prediction $p$ and target $t$ within the defined neighborhood.
    *   For **DSSIM-2D**: The neighborhood is a 2D window applied to each slice along the z-dimension. The final DSSIM is the average of the DSSIM values computed over all slices [S1].
    *   For **DSSIM-3D**: The neighborhood is a 3D block surrounding each voxel, assessing structural similarity along the z-dimension directly [S1].
2.  **Compute DSSIM:** Apply the DSSIM formula using the local statistics and constants $c_1, c_2$ [S1].
3.  **Compute MSE:** Calculate the mean squared error across all $N$ voxels [S1].
4.  **Combine:** Weight the MSE by $\alpha$ and add it to the DSSIM term [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dssim_loss(p, t, alpha=5.0, c1=0.01, c2=0.03, window_size=3, use_3d=False):
    """
    Compute DSSIM Loss for volumetric data.
    
    Args:
        p: Prediction volume (W, H, D)
        t: Target volume (W, H, D)
        alpha: Weight for MSE term
        c1, c2: Stability constants
        window_size: Size of the local neighborhood (assumed square/cubic)
        use_3d: If True, use 3D neighborhoods; if False, use 2D slice averaging
    
    Returns:
        Scalar loss value
    """
    # 1. Compute MSE
    mse = torch.mean((p - t) ** 2)
    
    # 2. Compute DSSIM
    if use_3d:
        # 3D Neighborhood: Compute local stats in 3D blocks
        # Note: In practice, this involves sliding a 3D window over the volume
        # For implementation, use 3D convolution or pooling for mean/variance
        mu_p = conv3d_mean(p, window_size)
        mu_t = conv3d_mean(t, window_size)
        sigma_p2 = conv3d_var(p, window_size)
        sigma_t2 = conv3d_var(t, window_size)
        sigma_pt = conv3d_cov(p, t, window_size)
        
        # Global DSSIM from local stats
        numerator = (2 * mu_p * mu_t + c1) * (2 * sigma_pt + c2)
        denominator = (mu_p**2 + mu_t**2 + c1) * (sigma_p2 + sigma_t2 + c2)
        ssim_map = (numerator / denominator)
        dssim = 1.0 - torch.mean(ssim_map)
    else:
        # 2D Neighborhood: Average over z-slices
        dssim_sum = 0.0
        count = 0
        for z in range(p.shape[2]):
            p_slice = p[:, :, z]
            t_slice = t[:, :, z]
            
            # 2D Local Stats
            mu_p_s = conv2d_mean(p_slice, window_size)
            mu_t_s = conv2d_mean(t_slice, window_size)
            sigma_p2_s = conv2d_var(p_slice, window_size)
            sigma_t2_s = conv2d_var(t_slice, window_size)
            sigma_pt_s = conv2d_cov(p_slice, t_slice, window_size)
            
            numerator = (2 * mu_p_s * mu_t_s + c1) * (2 * sigma_pt_s + c2)
            denominator = (mu_p_s**2 + mu_t_s**2 + c1) * (sigma_p2_s + sigma_t2_s + c2)
            ssim_map = numerator / denominator
            dssim_sum += torch.mean(ssim_map)
            count += 1
        
        dssim = 1.0 - (dssim_sum / count)
    
    # 3. Combine
    loss = dssim + alpha * mse
    return loss
```

**Edge Cases and Pitfalls:**
*   **Precision:** Mixed precision training is recommended to manage memory, as noted in the source implementation [S1].
*   **Boundaries:** Boundary effects in sliding window calculations must be handled (e.g., padding or ignoring edge voxels) to ensure accurate local statistics [S1].
*   **Degenerate Input:** If variance is zero (constant regions), the denominator in DSSIM approaches $c_1 c_2$, preventing division by zero due to the stability constants [S1].

## Subtopics

### 3DGS Loss Function
A variant used in PyGS (Pyramidal 3D Gaussian Splatting) that applies the same weighted combination of MSE and SSIM but in the context of 2D rendered images from 3D Gaussians [S2, unofficial]. It uses the formula $L = \lambda \cdot \text{MSE}(C, \hat{C}) + (1 - \lambda) \cdot \text{SSIM}(C, \hat{C})$, where $C$ is the ground-truth color and $\hat{C}$ is the rendered color [S2, unofficial]. Unlike the volumetric DSSIM, this operates on 2D pixel spaces. It is coupled with specific density control strategies: splitting and cloning Gaussians are retained within original levels/clusters, and levels are routinely reassigned based on scales to preserve multi-scale characteristics [S2, unofficial]. The hyperparameter $\lambda$ balances the terms, though its specific value is not explicitly stated in the source [S2, unofficial].

## Operating characteristics
*   **Time and Memory:** Training using this loss function takes approximately 20 hours on a single RTX 2080 Ti GPU, requiring 10.5 GB of GPU memory [S1].
*   **Performance:** The method produces accurate volumetric ambient occlusion with sufficient detail while maintaining brightness consistency with ground truth [S1].
*   **2D vs 3D SSIM:** Evaluation showed no large difference between DSSIM-2D and DSSIM-3D (Test SSIM 0.879 vs 0.874) [S1]. The 2D variant performed similarly because the large receptive field of the prior convolutional layers forces the network to learn coherent 3D structure regardless of the loss function's explicit 3D penalty [S1].
*   **Strengths:** Balances structural fidelity with pixel accuracy; robust to hyperparameter tuning ($\alpha$) [S1].
*   **Limitations:** Computationally heavier than pure MSE due to local neighborhood calculations [S1].

## Background
The DSSIM Loss for volumetric data was proposed in the context of Deep Volumetric Ambient Occlusion [S1]. It was developed to improve upon standard MSE losses which often result in blurry outputs, by incorporating the Structural Similarity Index Measure (SSIM) adapted for 3D volumes [S1]. The approach was implemented in PyTorch with mixed precision training [S1].

## Sources
[S1] Deep Volumetric Ambient Occlusion, https://arxiv.org/abs/2008.08345v2, fetched 2026-10-26
[S2] PyGS: Large-scale Scene Representation with Pyramidal 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2405.16829, fetched 2026-10-26 (unofficial)
