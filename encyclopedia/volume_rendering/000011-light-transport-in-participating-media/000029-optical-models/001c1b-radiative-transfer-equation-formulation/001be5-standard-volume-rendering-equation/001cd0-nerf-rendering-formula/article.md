# NeRF Rendering Formula

Also called: Discrete Volumetric Rendering Equation, NeRF-style Volume Rendering, Radiance Field Rendering Formula, Volume Rendering (NeRF-style)

## Overview
The NeRF Rendering Formula is a discrete approximation of the volume rendering equation used to composite raw neural network predictions (color and density/opacity) into final pixel colors, depths, and accumulation maps [S3][S4, S5, unofficial]. It differs from the continuous Standard volume rendering equation by operating on a fixed set of discrete sample points along a ray, assuming single-path light transport without explicit path tracing [S3]. It is the core reconstruction function for NeRFs and 3D Gaussian Splats, enabling differentiable rendering of implicit or explicit radiance fields [S2, S5, unofficial].

## How it works
**Inputs:**
*   `rays_o`: Ray origins, shape $(N, 3)$ or $(H, W, 3)$ [S4, S5, unofficial].
*   `rays_d`: Ray directions, shape $(N, 3)$ or $(H, W, 3)$ [S4, S5, unofficial].
*   `z_vals`: Sample depths along the ray, shape $(N, K)$ where $K$ is the number of samples per ray [S4, unofficial].
*   `implicit_fn`: A function (e.g., MLP or Gaussian field) that takes 3D positions and returns raw density $\sigma_{raw}$ and color features $f$ [S1][S4, unofficial].
*   `white_bkgd`: Boolean flag to composite onto a white background [S4, unofficial].
*   `raw_noise_std`: Standard deviation for noise added to density during training [S4, unofficial].

**Outputs:**
*   `rgb_map`: Composite RGB color, shape $(N, 3)$ or $(H, W, 3)$ [S4, unofficial].
*   `depth_map`: Expected depth, shape $(N,)$ or $(H, W)$ [S4, unofficial].
*   `acc_map`: Accumulated opacity, shape $(N,)$ or $(H, W)$ [S4, unofficial].

**Data Structures:**
*   `raw`: Tensor of shape $(N, K, 4)$ containing concatenated color features and density predictions from `implicit_fn`.
*   `alpha`: Tensor of shape $(N, K)$ containing computed opacity values.
*   `transmittance`: Tensor of shape $(N, K)$ containing cumulative transmittance.
*   `weights`: Tensor of shape $(N, K)$ containing the final rendering weights.

**Steps:**

1.  **Sample and Predict:**
    For each ray, sample $K$ 3D positions $x_i$ along the ray using `z_vals` and ray parameters. Pass $x_i$ through `implicit_fn` to get raw density $\sigma_{raw}$ and color features $f$ [S1][S4, unofficial].
    $$ \text{raw} = \text{implicit\_fn}(x) \quad \text{shape: } (N, K, 4) $$
    The first 3 channels are color features, the 4th is raw density [S4, unofficial].

2.  **Compute Raw Color:**
    Apply sigmoid activation to the color features to bound them to $[0, 1]$ [S4, unofficial].
    $$ c_i = \sigma(\text{raw}_{i, :, :3}) $$

3.  **Compute Raw Density with Noise (Training Only):**
    If `raw_noise_std` > 0, add Gaussian noise to the raw density to prevent overfitting to sparse samples [S4, unofficial].
    $$ \sigma_{noisy} = \text{relu}(\text{raw}_{i, :, 3} + \mathcal{N}(0, \text{raw\_noise\_std})) $$
    If `raw_noise_std` is 0 or during inference, $\sigma_{noisy} = \text{relu}(\text{raw}_{i, :, 3})$ [S4, unofficial].

4.  **Compute Segment Distances:**
    Calculate the distance between adjacent samples. Handle the last segment by appending a large value (infinity) to ensure the last sample's contribution is handled correctly if it extends to infinity [S4, unofficial].
    $$ \text{dists}_i = z_{i, j+1} - z_{i, j} \quad \text{for } j = 0 \dots K-2 $$
    $$ \text{dists}_i = \text{concat}(\text{dists}_i, 10^{10}) $$
    Convert to real-world distance by multiplying by the norm of the ray direction (if directions are not unit length) [S4, unofficial].
    $$ \Delta t_i = \text{dists}_i \cdot \| \text{rays\_d}_i \| $$

5.  **Compute Opacity ($\alpha$):**
    Convert density and segment length to opacity using the exponential attenuation model [S1][S4, unofficial].
    $$ \alpha_i = 1 - \exp(-\sigma_{noisy, i} \cdot \Delta t_i) $$
    Values are constrained between 0 and 1 [S3].

6.  **Compute Transmittance ($T$):**
    Compute cumulative transmittance using exclusive cumulative product. This represents the probability that light travels from the camera to sample $i$ without being absorbed or scattered [S4, S5, unofficial].
    $$ T_i = \prod_{j=0}^{i-1} (1 - \alpha_j) $$
    Implementation detail: Use `cumprod` on $(1 - \alpha)$ with a small epsilon ($10^{-10}$) added for numerical stability, shifted to be exclusive [S4, unofficial].
    $$ \text{ones} = \text{ones\_like}(\alpha[:, :1]) $$
    $$ T = \text{cumprod}(\text{concat}(\text{ones}, 1 - \alpha + 10^{-10}), \text{axis}=-1)[:, :-1] $$

7.  **Compute Weights ($w$):**
    The weight for each sample is the product of transmittance and the opacity of the current sample [S1][S4, unofficial].
    $$ w_i = T_i \cdot \alpha_i $$
    Alternatively expressed as $w = T \cdot (1 - \exp(-\sigma \cdot \Delta t))$ [S1].

8.  **Aggregate Outputs:**
    Compute the final pixel color, depth, and accumulation map using the weights [S4, unofficial].
    $$ \text{rgb\_map} = \sum_{i=0}^{K-1} w_i \cdot c_i $$
    $$ \text{depth\_map} = \sum_{i=0}^{K-1} w_i \cdot z_i $$
    $$ \text{acc\_map} = \sum_{i=0}^{K-1} w_i $$

9.  **Background Composite (Optional):**
    If `white_bkgd` is True, composite the result onto a white background using the accumulated opacity [S4, unofficial].
    $$ \text{rgb\_map} = \text{rgb\_map} + (1 - \text{acc\_map}) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def nerf_rendering_formula(z_vals, rays_d, implicit_fn, white_bkgd=False, raw_noise_std=0.0):
    """
    z_vals: (N, K) depths along rays
    rays_d: (N, 3) ray directions
    implicit_fn: callable returning (color_features, density) for 3D points
    """
    # 1. Sample positions and predict
    # Assuming rays_o is available or z_vals are relative to origin
    # For simplicity, assume implicit_fn takes (N, K, 3) positions
    # x = rays_o[:, None, :] + z_vals[:, :, None] * rays_d[:, None, :] 
    # Note: rays_o is not in inputs above, but required for 3D pos. 
    # We assume implicit_fn handles the ray bundle or x is passed externally.
    # Here we assume x is computed externally or implicit_fn takes ray bundle.
    # Based on S1: implicit_output = implicit_fn(cur_ray_bundle)
    
    # Let's assume we have positions 'x' of shape (N, K, 3)
    # raw: (N, K, 4)
    raw = implicit_fn(x) 
    
    # 2. Compute Color
    rgb = torch.sigmoid(raw[..., :3]) # (N, K, 3)
    
    # 3. Compute Density with Noise
    raw_density = raw[..., 3]
    if raw_noise_std > 0:
        noise = torch.randn_like(raw_density) * raw_noise_std
        raw_density = raw_density + noise
    sigma = F.relu(raw_density) # (N, K)
    
    # 4. Compute Distances
    dists = z_vals[:, 1:] - z_vals[:, :-1] # (N, K-1)
    dists = torch.cat([dists, torch.ones_like(dists[:, :1]) * 1e10], dim=-1) # (N, K)
    
    # Convert to real-world distance
    dists = dists * torch.norm(rays_d, dim=-1, keepdim=True) # (N, K)
    
    # 5. Compute Alpha
    alpha = 1.0 - torch.exp(-sigma * dists) # (N, K)
    
    # 6. Compute Transmittance
    # Exclusive cumprod
    ones = torch.ones_like(alpha[:, :1])
    transmittance = torch.cumprod(torch.cat([ones, 1.0 - alpha + 1e-10], dim=-1), dim=-1)[:, :-1] # (N, K)
    
    # 7. Compute Weights
    weights = alpha * transmittance # (N, K)
    
    # 8. Aggregate
    rgb_map = torch.sum(weights[..., None] * rgb, dim=-2) # (N, 3)
    depth_map = torch.sum(weights * z_vals, dim=-1) # (N,)
    acc_map = torch.sum(weights, dim=-1) # (N,)
    
    # 9. Background Composite
    if white_bkgd:
        rgb_map = rgb_map + (1.0 - acc_map[..., None])
        
    return rgb_map, depth_map, acc_map
```

**Edge Cases and Pitfalls:**
*   **Numerical Stability:** When $\alpha_i \approx 1$, transmittance becomes 0. Adding epsilon ($10^{-10}$) to $(1-\alpha)$ prevents `cumprod` from becoming exactly 0 too early due to floating-point errors, ensuring stable gradients [S4, unofficial].
*   **Opaque Surfaces:** Once an opaque surface is encountered ($\alpha_i \approx 1$), $T_i$ for subsequent samples becomes 0, effectively switching off contributions from behind the surface [S5, unofficial].
*   **Density Bounds:** Raw density is passed through `relu` to ensure non-negative values, as negative density is physically invalid and breaks the exponential attenuation model [S4, unofficial].
*   **Infinite Distance:** The last segment distance is set to a large value ($10^{10}$) to approximate the infinite tail of the ray, ensuring that if the last sample has non-zero density, it contributes to the final color [S4, unofficial].

## Subtopics

### Concentrate-and-Scatter Method
A post-processing rendering technique for neural radiance fields that applies depth-of-field blur [S6]. It computes a signed defocus map based on concentrated depth, aperture parameters, and focus distance. Radiance is scattered from in-focus pixels to out-of-focus neighbors using a pixel-wise algorithm that traverses neighboring pixels and calculates scattering weights based on the defocus radius [S6].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of rays/pixels and $K$ is the number of samples per ray. The dominant cost is the evaluation of `implicit_fn` for all sample points [S4, unofficial].
*   **Memory Complexity:** $O(N \cdot K)$ to store raw predictions, intermediate tensors (alpha, transmittance, weights), and gradients during backpropagation [S4, unofficial].
*   **Performance:** Highly parallelizable on GPUs. Performance depends heavily on the complexity of `implicit_fn` (e.g., MLP depth/width) and the number of samples $K$ [S4, unofficial].
*   **Strengths:** Differentiable, allowing end-to-end training of the scene representation from images. Simple to implement and integrate with standard deep learning frameworks [S3][S4, unofficial].
*   **Limitations:** Assumes single-path light transport, ignoring complex effects like caustics or multiple scattering [S3]. Requires dense sampling for high-quality results, which can be computationally expensive [S4, unofficial].
*   **When to Use:** Rendering neural radiance fields (NeRFs, 3DGS) where differentiable training from multi-view images is required [S3][S5, unofficial].
*   **When Not to Use:** When physically accurate global illumination effects (multiple bounces) are critical, or when real-time performance is required with very high sample counts without optimization (e.g., hierarchical sampling) [S3].

## Background
The NeRF Rendering Formula is derived from the discrete form of the Volume Rendering Equation (VRE) [S3]. It was popularized by the NeRF (Neural Radiance Fields) paper [S3][S5, unofficial] and is also used in 3D Gaussian Splats [S3][S5, unofficial]. The method approximates the continuous VRE by assuming single-path summation along the ray trajectory, avoiding the need for path tracing [S3]. The learned parameters (color and opacity) are not strictly physically literal but serve as an approximation learnable from offline samples, effectively approximating surface rendering for opaque objects using a volume-based formulation [S3].

## Sources
[S1] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S2] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S3] 14 Rendering Volume and Subsurface Scattering - Foundations of Visual ..., https://learnvisualcomputing.github.io/rendering-rte.html, fetched 2026-10-27
[S4] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S5] Volume Rendering, https://www.thomasantony.com/notes/202510052138-volume-rendering/, fetched 2026-10-30 (unofficial)
[S6] DoF-NeRF: Depth-of-Field Meets Neural Radiance Fields, https://doi.org/10.1145/3503161.3548088, fetched 2026-10-29
