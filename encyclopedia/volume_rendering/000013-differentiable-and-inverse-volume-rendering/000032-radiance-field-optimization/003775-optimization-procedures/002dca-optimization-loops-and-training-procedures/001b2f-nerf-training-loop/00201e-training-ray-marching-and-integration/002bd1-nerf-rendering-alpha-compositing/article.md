# NeRF Rendering (Alpha Compositing)

Also called: NeRF Forward Pass, NeRF Rendering Equation, Vanilla NeRF Volume Rendering, Volume Rendering Integration

## Overview
NeRF Rendering (Alpha Compositing) is the differentiable integration step that computes the final RGB color and depth for a pixel by synthesizing predictions from a neural radiance field along a camera ray [S1][S2, unofficial][S3, S4]. It solves the problem of mapping discrete, noisy neural network outputs (density and color) into a coherent, continuous image by accounting for occlusion and transparency via alpha compositing [S2, unofficial][S4]. What sets it apart from its siblings is its specific reliance on the stratified sampling intervals and the exponential transmittance decay defined by the original NeRF formulation, distinguishing it from explicit voxel-based integrators like [NeRF-style Volume Rendering Integrator](5964-nerf-style-volume-rendering-integrator.md) or hash-based query methods [NeRF Network Query with Hashed Embeddings](3107-nerf-network-query-with-hashed-embeddings.md).

## How it works
**Inputs:**
1.  `rays`: A tensor of shape `(B, H, W, 3)` containing ray origins $o$ and directions $d$.
2.  `t_vals`: A tensor of shape `(N,)` or `(B, H, W, N)` containing the sample distances along the ray. These are typically generated via stratified sampling over the interval $[t_n, t_f]$ [S4].
3.  `model`: A neural network $F_\Theta$ that takes a 5D coordinate $(x, d)$ and outputs raw density and color [S1, S4].
4.  `white_back`: Boolean flag indicating if the background should be treated as white (1.0) or black (0.0) [S2, unofficial].

**Outputs:**
1.  `rgb_final`: Rendered RGB image of shape `(B, H, W, 3)`.
2.  `depth_final`: Rendered depth map of shape `(B, H, W)`.

**Data Structures:**
*   **Predictions Tensor**: The output of the network reshaped to `(B, H, W, N, 4)`. The last dimension contains `[r, g, b, sigma]` [S3].
*   **Deltas Tensor**: Distances between adjacent samples. Shape `(B, H, W, N)`.

**Steps:**

1.  **Network Query**: Evaluate the NeRF network for all sampled points. For each ray, sample points $x_i = o + t_i d$. The network outputs raw values $v_i = [r_i, g_i, b_i, \sigma_i^{raw}]$ [S3, S4].
2.  **Activation**: Apply activation functions to the raw outputs [S3]:
    *   Color: $c_i = \text{sigmoid}(v_i[:3])$. This bounds color to $[0, 1]$.
    *   Density: $\sigma_i = \text{ReLU}(v_i[3])$. This ensures non-negative density [S3].
3.  **Delta Calculation**: Compute the distance between adjacent samples [S2, unofficial][S3]:
    $$ \delta_i = t_{i+1} - t_i $$
    For the last sample, append a large value (e.g., $10^{10}$) to ensure the ray terminates properly, representing the infinite distance to the end of the ray [S2, unofficial][S3].
    $$ \delta_{last} = 10^{10} $$
4.  **Alpha Calculation**: Compute the opacity (alpha) for each sample interval [S3]:
    $$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
5.  **Transmittance Calculation**: Compute the probability that a ray passes through all previous samples without hitting anything [S2, unofficial][S3]. To prevent numerical instability ($\ln(0)$), add a small epsilon ($\epsilon = 10^{-10}$) to the term $(1-\alpha)$ before cumulative product [S3]:
    $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j + \epsilon) $$
    Note: $T_1 = 1.0$ (exclusive product starting from index 1) [S3].
6.  **Weight Calculation**: Compute the weight for each sample, representing the probability that the visible surface lies exactly at sample $i$ [S3]:
    $$ w_i = \alpha_i T_i $$
7.  **Color Integration**: Compute the final RGB color as a weighted sum [S2, unofficial][S3]:
    $$ \hat{C}(r) = \sum_{i=1}^{N} w_i c_i $$
8.  **Depth Integration**: Compute the final depth as a weighted sum of the sample distances [S2, unofficial][S3]:
    $$ \hat{D}(r) = \sum_{i=1}^{N} w_i t_i $$
9.  **Background Handling**: If `white_back` is true, add the contribution of the background (assumed white) to the result [S2, unofficial]:
    $$ \text{weights\_sum} = \sum_{i=1}^{N} w_i $$
    $$ \text{rgb\_final} = \text{rgb\_final} + (1 - \text{weights\_sum}) \cdot 1.0 $$
    If `white_back` is false, the background is black, so no addition is needed (or add 0) [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def nerf_render_alpha_compositing(model, rays_o, rays_d, t_vals, white_back=False, rand=True):
    """
    Renders RGB and Depth using NeRF alpha compositing.
    
    Args:
        model: NeRF MLP model.
        rays_o: Ray origins, shape (B, H, W, 3).
        rays_d: Ray directions, shape (B, H, W, 3).
        t_vals: Sample distances, shape (N,) or (B, H, W, N).
        white_back: Boolean, if True, assumes white background.
        rand: Boolean, if True, t_vals are per-pixel random stratified samples.
    
    Returns:
        rgb: Rendered RGB image, shape (B, H, W, 3).
        depth: Rendered depth map, shape (B, H, W).
    """
    # 1. Prepare sample points and directions
    # Expand t_vals to match ray dimensions if necessary
    if t_vals.dim() == 1:
        # t_vals is (N,), need to broadcast to (B, H, W, N)
        # rays_o: (B, H, W, 3), rays_d: (B, H, W, 3)
        # We need to create a grid of points x_i = o + t_i * d
        # Reshape for broadcasting: rays_o (B, H, W, 1, 3), t_vals (1, 1, 1, N, 1)
        rays_o_exp = rays_o.unsqueeze(3) # (B, H, W, 1, 3)
        rays_d_exp = rays_d.unsqueeze(3) # (B, H, W, 1, 3)
        t_vals_exp = t_vals.unsqueeze(0).unsqueeze(0).unsqueeze(0).unsqueeze(4) # (1, 1, 1, N, 1)
        
        # Sample points x_i
        pts = rays_o_exp + t_vals_exp * rays_d_exp # (B, H, W, N, 3)
        
        # Directions for the network are the ray directions, repeated for each sample
        dirs = rays_d.unsqueeze(3).expand_as(pts) # (B, H, W, N, 3)
    else:
        # t_vals is already (B, H, W, N)
        pts = rays_o.unsqueeze(3) + t_vals.unsqueeze(4) * rays_d.unsqueeze(3)
        dirs = rays_d.unsqueeze(3).expand_as(pts)

    # 2. Query Network
    # Concatenate positions and directions: (B, H, W, N, 6)
    inputs = torch.cat([pts, dirs], dim=-1)
    
    # Flatten for batched model evaluation if model expects 2D input (N_samples, 6)
    batch_size, H, W, N, _ = inputs.shape
    inputs_flat = inputs.reshape(-1, 6)
    
    # Get raw predictions: (N_samples, 4) -> [r, g, b, sigma_raw]
    raw_preds = model(inputs_flat)
    
    # Reshape back to (B, H, W, N, 4)
    predictions = raw_preds.reshape(batch_size, H, W, N, 4)

    # 3. Activate Outputs
    # RGB: Sigmoid
    rgb = torch.sigmoid(predictions[..., :3]) # (B, H, W, N, 3)
    # Density: ReLU
    sigma_raw = predictions[..., 3] # (B, H, W, N)
    sigma = torch.relu(sigma_raw) # (B, H, W, N)

    # 4. Compute Deltas
    # Delta between adjacent samples
    # t_vals shape depends on input. If (N,), diff is (N-1,). If (B,H,W,N), diff is (B,H,W,N-1).
    if t_vals.dim() == 1:
        deltas = t_vals[1:] - t_vals[:-1] # (N-1,)
        # Append large value for the last segment
        delta_inf = torch.tensor([1e10], device=sigma.device)
        deltas = torch.cat([deltas, delta_inf], dim=0) # (N,)
        # Broadcast deltas to (B, H, W, N)
        deltas = deltas.unsqueeze(0).unsqueeze(0).unsqueeze(0).expand_as(sigma)
    else:
        # t_vals is (B, H, W, N)
        deltas = t_vals[..., 1:] - t_vals[..., :-1] # (B, H, W, N-1)
        # Append large value
        # Shape of last dim append: (B, H, W, 1)
        delta_inf = torch.ones_like(deltas[..., :1]) * 1e10
        deltas = torch.cat([deltas, delta_inf], dim=-1) # (B, H, W, N)

    # 5. Compute Alpha
    # alpha = 1 - exp(-sigma * delta)
    alpha = 1.0 - torch.exp(-sigma * deltas) # (B, H, W, N)

    # 6. Compute Transmittance
    # T_i = prod_{j=1}^{i-1} (1 - alpha_j + eps)
    # Use exclusive cumulative product
    eps = 1e-10
    # (1 - alpha + eps)
    term = 1.0 - alpha + eps
    # Cumulative product along the last dimension (samples)
    # exclusive=True means T_1 = 1.0
    transmittance = torch.cumprod(term, dim=-1, exclusive=True) # (B, H, W, N)

    # 7. Compute Weights
    weights = alpha * transmittance # (B, H, W, N)

    # 8. Compute RGB
    # rgb_final = sum(weights * rgb)
    # weights: (B, H, W, N), rgb: (B, H, W, N, 3)
    rgb_final = torch.sum(weights.unsqueeze(-1) * rgb, dim=-2) # (B, H, W, 3)

    # 9. Compute Depth
    # depth_final = sum(weights * t_vals)
    # If t_vals was (N,), we need to broadcast it to (B, H, W, N) for the sum
    if t_vals.dim() == 1:
        # t_vals is (N,), expand to (B, H, W, N)
        t_vals_exp = t_vals.unsqueeze(0).unsqueeze(0).unsqueeze(0).expand_as(weights)
    else:
        t_vals_exp = t_vals
        
    depth_final = torch.sum(weights * t_vals_exp, dim=-1) # (B, H, W)

    # 10. Background Handling
    if white_back:
        weights_sum = torch.sum(weights, dim=-1, keepdim=True) # (B, H, W, 1)
        rgb_final = rgb_final + (1.0 - weights_sum) # Add white background contribution

    return rgb_final, depth_final
```

**Parameters:**
*   `white_back`: Default `False`. If `True`, adds $(1 - \sum w_i)$ to the RGB result [S2, unofficial].
*   `rand`: Default `True`. Determines if `t_vals` are per-pixel random stratified samples or fixed grid points. Affects how `deltas` and `depth` are computed/broadcasted [S3].
*   `chunk`: Not part of the core math, but in implementation, rays are often processed in chunks (e.g., 32768) to manage GPU memory [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Numerical Stability**: When $\sigma \delta$ is large, $\exp(-\sigma \delta)$ underflows to 0, making $\alpha=1$. This is correct. However, when computing transmittance, if $\alpha$ is very close to 1, $(1-\alpha)$ is close to 0. The cumulative product can underflow to 0. Adding $\epsilon=10^{-10}$ prevents $\ln(0)$ errors during gradient computation if log-loss is used, though standard MSE loss is more robust [S3].
*   **Boundary Conditions**: The last delta must be large enough to ensure the ray "exits" the scene. $10^{10}$ is a common heuristic [S2, unofficial][S3].
*   **Density Activation**: Density must be non-negative. ReLU is standard [S3]. Sigmoid for color is standard [S3].

## Subtopics

### [Depth-Guided Sampling](7490-depth-guided-sampling.md)
A sampling strategy that focuses scene sampling on estimated surface regions using depth priors from input views to increase sample density in relevant areas, improving synthesis quality and allowing reduction of samples per ray [7490-depth-guided-sampling.md].

### [NeRF Volume Rendering](8914-nerf-volume-rendering.md)
Rendering pixel colors from the warped radiance voxel grid using volume rendering techniques, involving ray intersection, trilinear interpolation, and alpha compositing [8914-nerf-volume-rendering.md].

### NeRF Rendering Pipeline
A procedural wrapper for the rendering step. It iterates over a list of camera poses (`render_poses`), optionally downsampling the image resolution by a `render_factor` (typically 4 or 8 for previews) to speed up computation, and processes rays in chunks (default 32768) to manage memory. It handles saving outputs as 8-bit PNGs and computing PSNR against ground truth images if available [S5, 5964-nerf-style-volume-rendering-integrator.md, unofficial].

### Rendered Depth-Guided Warping
A method to generate a warped image $I_{i \to j}$ from a source view $I_i$ to an unseen target view $j$ using rendered depth $D_j$ and viewpoint difference $R_{i \to j}$. It samples rays on a strided grid (e.g., stride $s=2$) to create low-resolution patches, renders depth, upsamples it via bilinear interpolation, and uses the pixel coordinate transformation $p_{j \to i} \sim K R_{j \to i} D_j(p_j) K^{-1} p_j$ to warp the source image. This enables consistency regularization without full-resolution rendering [S6, 7421-active-learning-strategy-for-nerf-distillation.md, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot R)$, where $N$ is the number of samples per ray and $R$ is the number of rays (pixels). The neural network evaluation dominates the cost.
*   **Memory Complexity**: $O(N \cdot R \cdot C)$, where $C$ is the channel size of the network output. Storing intermediate tensors (predictions, deltas, transmittance) for all rays and samples requires significant VRAM, often necessitating chunked processing [S5, unofficial].
*   **Measured Performance**: On an RTX 2080 Ti, a PyTorch implementation can render a single step in ~0.12s for a batch of 1024 rays, compared to 0.177s for the original JAX implementation [S2, unofficial].
*   **Strengths**: Differentiable, allowing end-to-end training with image supervision. Simple to implement.
*   **Limitations**: Computationally expensive due to the large number of samples required for noise-free rendering. Sensitive to the choice of $t_n$ and $t_f$ (near/far planes).
*   **When to use**: Standard NeRF training and novel view synthesis.
*   **When not to use**: Real-time applications without acceleration structures (like [Chunked Network Evaluation](3177-chunked-network-evaluation.md) or explicit fields).

## Background
NeRF (Neural Radiance Fields) was introduced by Mildenhall et al. in 2020 [S1, S4]. It represents a static scene as a continuous 5D function mapping spatial coordinates and viewing directions to volume density and emitted color. The rendering equation used is derived from classical volume rendering [S4], specifically the quadrature rule described by Max [S4]. The method optimizes a multi-layer perceptron (MLP) using the mean squared error between rendered and ground truth images [S4]. It has become a foundational technique in inverse graphics and 3D reconstruction [S1, S4].

## Sources
[S1] One Is All: Bridging the Gap between Neural Radiance Fields Architectures with Progressive Volume Distillation, https://doi.org/10.1609/aaai.v37i1.25135, fetched 2026-10-26
[S2] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S3] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S4] Points2NeRF: Generating Neural Radiance Fields from 3D point cloud, https://doi.org/10.1016/j.patrec.2024.07.002, fetched 2026-10-29
[S5] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S6] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
