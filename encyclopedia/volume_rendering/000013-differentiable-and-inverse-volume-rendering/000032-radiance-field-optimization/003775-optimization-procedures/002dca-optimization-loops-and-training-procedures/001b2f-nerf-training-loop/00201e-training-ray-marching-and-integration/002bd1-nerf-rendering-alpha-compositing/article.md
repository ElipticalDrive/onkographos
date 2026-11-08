# NeRF Rendering (Alpha Compositing)

Also called: Instance-NeRF Rendering, NeRF Forward Pass, NeRF Rendering Equation, Standard NeRF rendering, Vanilla NeRF Volume Rendering, Vanilla NeRF integration

## Overview
NeRF Rendering (Alpha Compositing) is the differentiable volume rendering method that computes the final RGB color and depth for a camera ray by integrating the predicted volume density ($\sigma$) and emitted color ($c$) of sampled points along the ray [S1][S4, unofficial][S5]. It solves the problem of synthesizing novel views from a continuous 5D radiance field by approximating the line integral of light transport using numerical quadrature [S1, S6]. What sets it apart from its siblings is its reliance on a neural network (MLP) to query density and color at arbitrary 3D coordinates and view directions, combined with alpha compositing to handle occlusion, whereas variants like [NeRF-style Volume Rendering Integrator](5964-nerf-style-volume-rendering-integrator.md) may use explicit voxel grids or different sampling strategies [S1, S6].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A camera ray defined by origin $o$ and direction $d$; a trained neural network $F_\Theta$ mapping $(x, d) \to (c, \sigma)$; a set of sample depths $t_i$ along the ray; and rendering parameters (e.g., background color assumption).
*   **Outputs:** Rendered RGB color $\hat{C}(r)$ and expected depth $\hat{D}(r)$ for the pixel corresponding to the ray.

**Data Structures:**
*   **Sample Points:** A tensor of shape $(BATCH\_SIZE, H, W, NUM\_SAMPLES, 3)$ containing 3D coordinates $x_i = o + t_i d$.
*   **Predictions:** A tensor of shape $(BATCH\_SIZE, H, W, NUM\_SAMPLES, 4)$ containing the MLP outputs: the first 3 channels are RGB color $c_i$, and the last channel is volume density $\sigma_i$ [S5].
*   **Deltas:** A tensor of shape $(BATCH\_SIZE, H, W, NUM\_SAMPLES)$ representing the distance between adjacent samples.

**Numbered Steps:**

1.  **Ray Formulation:** For each pixel, formulate the ray $r(t) = o + td$ emitted from the camera center $o$ in direction $d$ [S7].
2.  **Sampling:** Select $N$ quadrature points $\{t_i\}_{i=1}^N$ along the ray within the traceable volume $[t_n, t_f]$ [S1, S6]. Standard NeRF uses stratified sampling: partition the interval into $N$ bins and draw one sample uniformly at random from each bin [S6]:
    $$ t_i \sim U\left[t_n + \frac{i-1}{N}(t_f - t_n), t_n + \frac{i}{N}(t_f - t_n)\right] $$
3.  **Network Query:** Evaluate the neural network $F_\Theta$ at each sample point $x_i = o + t_i d$ and view direction $d$ to obtain color $c_i$ and density $\sigma_i$ [S6].
    $$ (c_i, \sigma_i) = F_\Theta(x_i, d) $$
4.  **Activation:** Apply activation functions to the raw network outputs. Typically, apply sigmoid to RGB to bound values to $[0, 1]$ and ReLU to density to ensure non-negativity [S5]:
    $$ c_i = \text{sigmoid}(c_i^{raw}), \quad \sigma_i = \text{ReLU}(\sigma_i^{raw}) $$
5.  **Delta Calculation:** Compute the distance $\delta_i$ between adjacent samples. For $i < N$, $\delta_i = t_{i+1} - t_i$. For the last segment, append a large value (e.g., $10^{10}$) to represent the infinite tail of the ray, ensuring the final transmittance decays to zero [S4, unofficial][S5]:
    $$ \delta = [t_2-t_1, \dots, t_N-t_{N-1}, 10^{10}] $$
    If the ray direction $d$ is not unit length, normalize deltas by the ray direction norm: $\delta_i = \delta_i \cdot \|d\|$ [S4, unofficial].
6.  **Alpha Calculation:** Compute the opacity (alpha) $\alpha_i$ for each sample, representing the probability that the ray hits the object at this segment [S5]:
    $$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
7.  **Transmittance Calculation:** Compute the accumulated transmittance $T_i$, the probability that the ray travels from the camera to sample $i$ without hitting any particle. Use a cumulative product with a small epsilon ($\epsilon = 10^{-10}$) for numerical stability, starting with $T_1 = 1.0$ [S5]:
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) = \prod_{j=1}^{i-1} (1 - \alpha_j + \epsilon) $$
    Note: Some implementations compute this as `cumprod(1 - alpha, exclusive=True)` [S5].
8.  **Weight Calculation:** Compute the weight $w_i$ for each sample, which is the product of transmittance and opacity [S5]:
    $$ w_i = T_i \alpha_i $$
9.  **Color Rendering:** Compute the final RGB color as the weighted sum of sample colors [S1, S5]:
    $$ \hat{C}(r) = \sum_{i=1}^N w_i c_i $$
10. **Depth Rendering:** Compute the expected depth as the weighted sum of sample depths [S7, S8]:
    $$ \hat{D}(r) = \sum_{i=1}^N w_i t_i $$
11. **Background Handling:** If a white background is assumed (`white_back=True`), add the contribution of the unrendered background. The weight of the background is $1 - \sum w_i$ [S4, unofficial]:
    $$ \hat{C}_{final}(r) = \hat{C}(r) + (1 - \sum_{i=1}^N w_i) \cdot c_{bg} $$
    Typically $c_{bg} = (1, 1, 1)$ for white [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def nerf_render_alpha_compositing(rays_o, rays_d, t_vals, model, white_back=False):
    """
    Renders RGB and Depth using NeRF Alpha Compositing.
    
    Args:
        rays_o: Tensor of shape (B, H, W, 3), ray origins.
        rays_d: Tensor of shape (B, H, W, 3), ray directions.
        t_vals: Tensor of shape (B, H, W, N), sample depths along rays.
        model: Neural network returning (rgb, sigma) for inputs (x, d).
        white_back: Boolean, if True, composite against white background.
        
    Returns:
        rgb: Tensor of shape (B, H, W, 3), rendered colors.
        depth: Tensor of shape (B, H, W), rendered depths.
    """
    # 1. Query Network
    # Expand rays_o and rays_d to match t_vals shape for broadcasting
    # x shape: (B, H, W, N, 3)
    # d shape: (B, H, W, N, 3) - same direction for all samples on a ray
    x = rays_o.unsqueeze(3) + t_vals.unsqueeze(4) * rays_d.unsqueeze(3)
    d = rays_d.unsqueeze(3).expand_as(x)
    
    # Flatten for model input: (B*H*W*N, 6) -> (B*H*W*N, 4)
    batch_size, H, W, N = t_vals.shape
    x_flat = x.reshape(-1, 3)
    d_flat = d.reshape(-1, 3)
    
    # Get predictions: (B*H*W*N, 4)
    predictions = model(torch.cat([x_flat, d_flat], dim=-1))
    
    # Reshape back: (B, H, W, N, 4)
    predictions = predictions.reshape(batch_size, H, W, N, 4)
    
    # 2. Slice and Activate
    # rgb: (B, H, W, N, 3), sigma: (B, H, W, N)
    rgb_raw = predictions[..., :3]
    sigma_raw = predictions[..., 3]
    
    rgb = torch.sigmoid(rgb_raw)
    sigma = torch.relu(sigma_raw)
    
    # 3. Compute Deltas
    # delta between adjacent samples
    deltas = t_vals[..., 1:] - t_vals[..., :-1]  # (B, H, W, N-1)
    
    # Append large value for the last segment
    # Shape (B, H, W, 1)
    delta_inf = torch.ones(batch_size, H, W, 1, device=t_vals.device) * 1e10
    deltas = torch.cat([deltas, delta_inf], dim=-1)  # (B, H, W, N)
    
    # Normalize deltas by ray direction norm
    # norm_d shape: (B, H, W, 1)
    norm_d = torch.norm(rays_d, dim=-1, keepdim=True)
    deltas = deltas * norm_d
    
    # 4. Compute Alpha
    # alpha = 1 - exp(-sigma * delta)
    alpha = 1.0 - torch.exp(-sigma * deltas)
    
    # 5. Compute Transmittance
    # T_i = prod_{j<i} (1 - alpha_j)
    # Use cumprod with exclusive=True to start with 1.0
    # Add epsilon for stability
    epsilon = 1e-10
    transmittance = torch.cumprod(1.0 - alpha + epsilon, dim=-1, exclusive=True)
    
    # 6. Compute Weights
    weights = alpha * transmittance  # (B, H, W, N)
    
    # 7. Compute RGB
    # rgb_final = sum(weights * rgb)
    # weights shape: (B, H, W, N), rgb shape: (B, H, W, N, 3)
    rgb_final = torch.sum(weights.unsqueeze(-1) * rgb, dim=-2)
    
    # 8. Compute Depth
    # depth_final = sum(weights * t_vals)
    depth_final = torch.sum(weights * t_vals, dim=-1)
    
    # 9. Background Handling
    if white_back:
        # weights_sum shape: (B, H, W, 1)
        weights_sum = weights.sum(dim=-1, keepdim=True)
        # Add background contribution (white = 1.0)
        rgb_final = rgb_final + (1.0 - weights_sum) * 1.0
        
    return rgb_final, depth_final
```

**Parameters:**
*   $N$: Number of samples per ray. Typical values range from 48 to 256 for training, higher for inference [S2, S6].
*   $t_n, t_f$: Near and far clipping planes. Defined by the scene bounds or camera frustum [S6].
*   `white_back`: Boolean flag for background compositing. Default `False` [S4, unofficial].
*   `chunk`: Number of rays processed in parallel during rendering to manage GPU memory. Default 32768 [S9, unofficial].

**Edge Cases and Pitfalls:**
*   **Numerical Stability:** The cumulative product in transmittance calculation can underflow. Adding epsilon ($10^{-10}$) to $(1-\alpha)$ prevents this [S5].
*   **Infinite Tail:** The last delta must be large enough to ensure $T_N \approx 0$. If $t_f$ is too close to the scene, the ray may not fully attenuate, causing artifacts. A value like $10^{10}$ is standard [S4, unofficial][S5].
*   **Density Activation:** Density must be non-negative. Using ReLU or softplus is critical; raw linear output can lead to negative densities and invalid alpha values [S5].
*   **View Direction:** The view direction $d$ must be normalized before being passed to the network to ensure consistent encoding [S6].

## Subtopics

### [Depth-Guided Sampling](7490-depth-guided-sampling.md)
A ray sampling strategy that focuses scene sampling on estimated surface regions using depth priors from input views to increase sample density in relevant areas, thereby improving synthesis quality and allowing a reduction in samples per ray. It differs from standard uniform sampling by leveraging predicted depth maps to define probability density fields for surface presence, and from simple depth-clipping by incorporating occlusion-aware likelihoods and Gaussian boosting to refine sample placement around the termination expectation.

### [NeRF Volume Rendering](8914-nerf-volume-rendering.md)
This method renders pixel colors from a pre-computed radiance voxel grid using volume rendering techniques, specifically addressing the "one-to-many" warping holes found in forward warping approaches by utilizing an inpainted voxel grid. It differs from standard NeRF [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by retrieving density and color features via trilinear interpolation from a 3D voxel grid rather than evaluating a neural network at query points, and it differs from [Depth-Guided Sampling](7490-depth-guided-sampling.md) by relying on uniform or fixed sampling intervals along the ray without depth-based adaptive refinement.

### NeRF Rendering Pipeline
Iterates over a list of camera poses (`render_poses`) to generate a sequence of rendered images and depth maps [S9, unofficial]. For each pose, it optionally downsamples the resolution and focal length by a `render_factor` (default 0, no downsampling; typical values 4 or 8 for preview) to accelerate rendering [S9, unofficial]. It processes rays in chunks (default 32768) to manage memory [S9, unofficial]. The pipeline collects RGB and depth outputs, optionally computes PSNR against ground truth images if provided, and saves the results as 8-bit PNGs [S9, unofficial].

### Rendered Depth-Guided Warping
A method to generate a warped image $I_{i \to j}$ from a source view $I_i$ to an unseen target view $j$ using rendered depth $D_j$ and viewpoint difference $R_{i \to j}$, enabling consistency regularization without full-resolution rendering [S10, unofficial]. It samples rays on a strided grid (stride $s=2$) to create low-resolution patches, renders depth for these rays, and upsamples the depth map using bilinear interpolation to full resolution [S10, unofficial]. The pixel coordinate transformation is given by $p_{j \to i} \sim K R_{j \to i} D_j(p_j) K^{-1} p_j$, where $K$ is the camera intrinsic matrix [S10, unofficial].

### NeRF-XL Multi-GPU Volume Rendering
A distributed rendering algorithm for scaling NeRF across multiple GPUs [S11]. It initializes global transmittance $T_{1 \to i} = 1$ and global color $C_{1 \to N+1} = 0$ [S11]. Each GPU $k$ processes its assigned segment of the ray, gathering local transmittance and color contributions from other GPUs if necessary, and updates the global color via $C_{1 \to N+1} += T_{1 \to i} \times C_{i \to i+1}$ and transmittance via $T_{1 \to i^*} *= T_{i \to i+1}$ [S11]. During inference, data gathering is only required on a single GPU (e.g., GPU 0) to render the final image, reducing data transfer compared to training [S11].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of samples per ray and $P$ is the number of rays (pixels) [S1, S6]. The dominant cost is the neural network evaluation at $N$ points per ray.
*   **Memory Complexity:** $O(N \cdot P)$ to store intermediate tensors (positions, directions, predictions, weights) for the batch of rays [S5]. Chunking reduces peak memory usage by processing subsets of rays [S9, unofficial].
*   **Measured Performance:** On an RTX 2080 Ti, standard NeRF takes approximately 10.8 seconds per step (batch of rays) for training, significantly slower than explicit representations like Plenoxels (0.3s) or Instant NGP (0.34s) [S2].
*   **Strengths:** High-quality novel view synthesis with view-dependent effects; differentiable, enabling end-to-end training from images [S6].
*   **Limitations:** Computationally expensive due to repeated MLP evaluations; requires many samples ($N$) for accurate integration, leading to slow inference [S2, S6].
*   **When to Use:** When high-fidelity novel views are required and training time is not a constraint, or as a baseline for comparing faster methods [S6].
*   **When Not to Use:** For real-time applications or large-scale scenes where inference speed is critical; consider [NeRF-style Volume Rendering Integrator](5964-nerf-style-volume-rendering-integrator.md) or hash-based methods [S2].

## Background
NeRF (Neural Radiance Fields) was introduced by Mildenhall et al. in 2020 [S1, S6]. It represents a static scene as a continuous 5D function mapping spatial coordinates and view directions to density and color, trained using volume rendering loss on multi-view images [S6]. The method builds on classical volume rendering theory [S6] and differentiable rendering [S1]. It has become a foundational technique in 3D computer vision, inspiring numerous variants for dynamic scenes, large-scale environments, and efficient rendering [S6, S11].

## Sources
[S1] One Is All: Bridging the Gap between Neural Radiance Fields Architectures with Progressive Volume Distillation, https://doi.org/10.1609/aaai.v37i1.25135, fetched 2026-10-26
[S2] RHINO: regularizing the hash-based implicit neural representation, https://doi.org/10.1007/s11432-024-4490-3, fetched 2026-10-26
[S3] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26
[S4] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S5] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S6] Points2NeRF: Generating Neural Radiance Fields from 3D point cloud, https://doi.org/10.1016/j.patrec.2024.07.002, fetched 2026-10-29
[S7] Instance Neural Radiance Field, https://doi.org/10.1109/ICCV51070.2023.00079, fetched 2026-10-29
[S8] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
[S9] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S10] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
[S11] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
