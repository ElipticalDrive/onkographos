# NeRF Rendering (Alpha Compositing)

Also called: NeRF Integration, NeRF Rendering Equation, Standard NeRF rendering, Vanilla NeRF integration, Vanilla NeRF Volume Rendering, Neural Radiance Field Volume Rendering

## Overview
NeRF Rendering (Alpha Compositing) is the differentiable volume rendering technique that synthesizes novel views by tracing camera rays through a scene represented by implicit density and color functions, computing the final pixel color via numerical integration of these properties [S1, S2]. It differs from explicit voxel-based rendering (e.g., Plenoxels) by using continuous neural networks to evaluate density and color at arbitrary points, and from SDF-based rendering (e.g., NeuS) by directly modeling volume density $\sigma$ rather than deriving it from a signed distance field, although SDF variants exist [S4, S2]. The method solves the problem of reconstructing 3D scenes from 2D images by optimizing a neural network to minimize the photometric error between rendered and ground-truth pixels [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A trained neural network $F_\Theta$ (typically two MLPs or one multi-output MLP) that maps a 5D coordinate $(x, d)$ to volume density $\sigma$ and RGB color $c$; a set of camera rays $r(t) = o + td$ for each pixel in the target image; sampling positions $t_i$ along each ray.
*   **Outputs:** Rendered RGB image $\hat{C}(r)$ and optionally depth map $\hat{D}(r)$ for each ray.

**Data Structures:**
*   **Rays:** Defined by origin $o$ (camera center) and direction $d$ (normalized view vector).
*   **Samples:** A list of $N$ scalar distances $t_i$ along the ray, where $i=0 \dots N-1$. The 3D position is $x_i = o + t_i d$.
*   **Predictions:** For each sample $x_i$ and direction $d$, the network outputs $\sigma_i \in \mathbb{R}$ (density) and $c_i \in \mathbb{R}^3$ (color).
*   **Intermediate Values:**
    *   $\delta_i$: Distance between adjacent samples, $\delta_i = t_{i+1} - t_i$ for $i < N-1$. The last segment $\delta_{N-1}$ is typically set to a large value (e.g., $10^{10}$) to ensure the ray terminates [S5, unofficial][S6].
    *   $\alpha_i$: Opacity of sample $i$, $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$.
    *   $T_i$: Transmittance up to sample $i$, $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$. $T_1 = 1.0$.
    *   $w_i$: Weight of sample $i$, $w_i = T_i \alpha_i$.

**Numbered Steps:**
1.  **Ray Generation:** For each pixel in the target image, construct a ray $r(t) = o + td$ using the camera's intrinsic and extrinsic parameters [S1, S3].
2.  **Sampling:** Generate $N$ sample points along the ray within the near ($t_n$) and far ($t_f$) bounds. The original NeRF implementation uses **stratified sampling**: divide the interval $[t_n, t_f]$ into $N$ equal bins and draw one sample uniformly at random from each bin [S1, S8]:
    $$ t_i \sim U\left[t_n + \frac{i-1}{N}(t_f - t_n), t_n + \frac{i}{N}(t_f - t_n)\right] $$
    Sort the samples such that $t_1 < t_2 < \dots < t_N$ [S1, S8].
3.  **Network Query:** For each sample $i$, compute the 3D position $x_i = o + t_i d$. Query the neural network $F_\Theta(x_i, d)$ to obtain density $\sigma_i$ and color $c_i$ [S1, S8].
    *   *Post-processing:* Apply sigmoid to color $c_i$ to bound it to $[0, 1]$ and ReLU to density $\sigma_i$ to ensure non-negativity [S6].
4.  **Delta Calculation:** Compute the distance between adjacent samples:
    $$ \delta_i = t_{i+1} - t_i \quad \text{for } i = 1 \dots N-1 $$
    Append a large value $\delta_\infty = 10^{10}$ for the final segment to handle the end of the ray [S5, unofficial][S6].
5.  **Alpha and Transmittance Calculation:**
    *   Compute opacity for each sample:
        $$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
    *   Compute transmittance using the cumulative product of $(1 - \alpha_i)$. To prevent numerical instability, add a small epsilon ($\epsilon = 10^{-10}$) to $(1 - \alpha_i)$ before the product [S6]:
        $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) = \prod_{j=1}^{i-1} (1 - \alpha_j + \epsilon) $$
        Note: $T_1 = 1.0$ [S1, S6].
6.  **Weight Calculation:** Compute the weight for each sample:
    $$ w_i = T_i \alpha_i $$
7.  **Color Compositing:** Compute the final RGB color by summing the weighted colors:
    $$ \hat{C}(r) = \sum_{i=1}^{N} w_i c_i $$
    If a white background is assumed, add the accumulated opacity complement to the result:
    $$ \hat{C}(r) = \sum_{i=1}^{N} w_i c_i + \left(1 - \sum_{i=1}^{N} w_i\right) \mathbf{1} $$
    where $\mathbf{1}$ is a vector of ones [S5, unofficial].
8.  **Depth Compositing (Optional):** Compute the expected depth by summing the weighted sample distances:
    $$ \hat{D}(r) = \sum_{i=1}^{N} w_i t_i $$
    [S1][S5, unofficial][S6].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def nerf_render(rays_o, rays_d, t_vals, network, white_background=False, rand=True):
    """
    Render RGB and Depth for a batch of rays using NeRF alpha compositing.
    
    Args:
        rays_o: Tensor of shape (B, 3), ray origins.
        rays_d: Tensor of shape (B, 3), ray directions (normalized).
        t_vals: Tensor of shape (B, N), sample distances along rays.
        network: Function/Model that takes (x, d) and returns (sigma, rgb).
        white_background: Boolean, if True, composite against white background.
        rand: Boolean, if True, t_vals are stratified samples; if False, deterministic.
        
    Returns:
        rgb: Tensor of shape (B, 3), rendered colors.
        depth: Tensor of shape (B), rendered depths.
    """
    # 1. Query Network
    # Expand rays_o and rays_d to match t_vals shape for broadcasting
    # rays_o: (B, 1, 3), rays_d: (B, 1, 3), t_vals: (B, N)
    # x: (B, N, 3)
    x = rays_o.unsqueeze(1) + t_vals.unsqueeze(2) * rays_d.unsqueeze(1)
    
    # Query network for each sample
    # network(x, d) -> (sigma, rgb)
    # Assuming network handles batched input
    sigma, rgb = network(x, rays_d.unsqueeze(1).expand_as(x))
    
    # 2. Post-process predictions
    # Apply sigmoid to RGB and ReLU to sigma
    rgb = torch.sigmoid(rgb)
    sigma = torch.relu(sigma)
    
    # 3. Compute deltas
    # delta[i] = t[i+1] - t[i]
    deltas = t_vals[:, 1:] - t_vals[:, :-1]  # Shape (B, N-1)
    
    # Append infinity for the last segment
    if rand:
        # Shape (B, 1)
        delta_inf = torch.full((deltas.shape[0], 1), 1e10, device=deltas.device)
    else:
        # Shape (1,) or broadcastable
        delta_inf = torch.full((1,), 1e10, device=deltas.device)
        
    deltas = torch.cat([deltas, delta_inf], dim=-1)  # Shape (B, N)
    
    # 4. Compute alpha
    # alpha = 1 - exp(-sigma * delta)
    alpha = 1.0 - torch.exp(-sigma * deltas)
    
    # 5. Compute transmittance
    # T[i] = product(1 - alpha[j]) for j < i
    # Use exclusive cumprod to get T starting at 1.0
    # Add epsilon to prevent 1-alpha from being exactly 0
    epsilon = 1e-10
    one_minus_alpha = (1.0 - alpha) + epsilon
    transmittance = torch.cumprod(one_minus_alpha, dim=-1, exclusive=True)
    # transmittance shape (B, N), T[0] = 1.0
    
    # 6. Compute weights
    weights = alpha * transmittance  # Shape (B, N)
    
    # 7. Compute RGB
    # rgb_final = sum(weights * rgb)
    # weights: (B, N), rgb: (B, N, 3)
    rgb_final = torch.sum(weights.unsqueeze(-1) * rgb, dim=-2)  # Shape (B, 3)
    
    # Handle white background
    if white_background:
        # weights_sum: (B, N) -> sum over N -> (B,)
        weights_sum = torch.sum(weights, dim=-1)
        # Add complement of accumulated opacity
        rgb_final = rgb_final + (1.0 - weights_sum.unsqueeze(-1))
        
    # 8. Compute Depth
    # depth_final = sum(weights * t_vals)
    # t_vals: (B, N), weights: (B, N)
    if rand:
        depth_final = torch.sum(weights * t_vals, dim=-1)
    else:
        # Broadcast t_vals if needed (e.g. if t_vals is (1, N))
        depth_final = torch.sum(weights * t_vals, dim=-1)
        
    return rgb_final, depth_final
```

**Parameters:**
*   $N$: Number of samples per ray. Typical values range from 48 to 256 in original NeRF, often higher in training (e.g., 192) and lower in inference [S1, S3, S18].
*   $t_n, t_f$: Near and far clipping planes. $t_n$ is usually the camera center or a small offset, $t_f$ is determined by the scene's bounding volume [S1].
*   $\epsilon$: Epsilon for numerical stability in transmittance calculation. Typical value $10^{-10}$ [S6].
*   $white\_background$: Boolean flag. Default False [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Numerical Instability:** If $\sigma_i \delta_i$ is very large, $\alpha_i$ approaches 1, and $1-\alpha_i$ approaches 0. Cumulative product of values near 0 can underflow. Adding $\epsilon$ prevents this [S6].
*   **Boundary Conditions:** The last segment $\delta_{N-1}$ must be large enough to ensure the ray "terminates" (i.e., $T_N \approx 0$). Using $10^{10}$ is standard [S5, unofficial][S6].
*   **Density Non-negativity:** Density $\sigma$ must be non-negative. Applying ReLU ensures this; negative densities would cause $\alpha > 1$, breaking the probabilistic interpretation [S6].
*   **Color Bounds:** Color should be bounded to $[0, 1]$. Sigmoid is standard [S6].

## Subtopics

### [Depth-Guided Sampling](7490-depth-guided-sampling.md)
Focuses sampling on estimated surface regions using depth priors to improve quality and reduce samples.

### [Depth-Prior Guided Stratified Sampling](11292-depth-prior-guided-stratified-sampling.md)
Uses confidence intervals around depth priors for stratified sampling within bounds.

### NeRF Rendering Pipeline
Iterates over render poses, optionally downsampling resolution via `render_factor` (default 0, typical 4-8 for preview) and processing rays in chunks (default 32768) to manage memory. Saves RGB and depth as 8-bit PNGs. Computes PSNR if ground truth images are provided [S13, unofficial].

### Rendered Depth-Guided Warping
Generates warped images from source to target views using rendered depth and viewpoint differences. Samples rays on a strided grid (stride 2) for low-resolution depth, upsamples via bilinear interpolation, and warps source image. Pixel transformation: $p_{j\to i} \sim K R_{j\to i} D_j(p_j) K^{-1} p_j$ [S14, unofficial].

### NeRF-XL Multi-GPU Volume Rendering
Splits ray segments across GPUs. Each GPU $k$ maintains global transmittance $T_{1\to i}$ and color $C_{1\to N+1}$. Updates global color: $C_{1\to N+1} += T_{1\to i} \times C_{i\to i+1}$. Data gathering is only required on a single GPU (e.g., GPU 0) during inference [S15].

### NeRF Volume Rendering via Numerical Quadrature
Approximates the integral by summing contributions from sampled spheres. Uses photometric loss to back-propagate gradients to MLP weights and Tri-Mip parameters [S16].

### Raymarcher
Uses RM-LSTM to predict a single intersection point and outputs RGB directly, skipping volume rendering. RM-LSTM has 512 feature channels, runs for 10 steps, and penalizes depth outside $[0.5, 2.0]$. ~80x faster than baseline but lower PSNR (23.02 vs 23.34) [S17, unofficial].

### 2D Neural Rendering Module
Maps low-resolution feature map $IF$ to final image $I$ using upsampling and 1x1 Conv to reduce multi-view inconsistency. $Upsample(X) = Conv2D(Pixelshuffle(repeat(X, 4) + \beta_\zeta(X), 2), K)$. Uses Leaky ReLU and 1x1 kernel [S18].

### HeadNeRF Inference
Samples 64 points per ray, computes volume rendering for feature map $IF$, then passes through 2D neural rendering module. Renders >40 fps on NVIDIA 3090 [S18].

### Neus SDF Volume Rendering
Represents scene as SDF $f(x)$. Density $\phi_b(f(x)) = \frac{b e^{-b f(x)}}{(1 + e^{-b f(x)})^2}$. Weight $w(t) = \frac{\phi_s(f(p(t)))}{\int_0^{+\infty} \phi_s(f(p(u)))du}$. Surface $S = \{x | f(x)=0\}$. $b$ is a trainable hyperparameter increasing during training [S19].

### Depth Estimation from NeRF
Extracts depth by minimizing the first-order derivative of transmittance $T(t)$: $t_{surface} \approx \arg \min_t |\frac{dT(t)}{dt}|$. 3D point $p = o + t_{surface} d$ [S19].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$ per ray, where $N$ is the number of samples and $P$ is the cost of the neural network query. Total time is $O(N \cdot P \cdot H \cdot W)$ for an image of size $H \times W$.
*   **Memory Complexity:** $O(N \cdot H \cdot W)$ to store intermediate values (alpha, transmittance, weights) for all rays in a batch. Chunking is often used to reduce memory usage [S13, unofficial].
*   **Measured Performance:** Original NeRF rendering is slow (seconds per frame) due to MLP queries. Optimizations like hash embeddings (Instant-NGP) or 2D neural rendering modules (HeadNeRF) can achieve real-time speeds (>40 fps) [S17, unofficial][S18].
*   **Strengths:** High-quality novel view synthesis, differentiable, supports various auxiliary outputs (depth, normals, instance logits) [S1, S10].
*   **Limitations:** Slow rendering speed without acceleration, requires many views for training, sensitive to sampling strategy [S1][S17, unofficial].
*   **When to use:** High-quality offline rendering, training data synthesis, applications requiring differentiable rendering [S1].
*   **When not to use:** Real-time applications without acceleration techniques, scenes with extreme dynamic range or complex lighting not captured by view-dependent color [S17, unofficial].

## Background
NeRF (Neural Radiance Fields) was introduced by Mildenhall et al. in 2020 [S1]. It represents a static scene as a continuous 5D function mapping spatial location and viewing direction to density and color. The method was trained using volume rendering loss on multi-view images. Positional encoding was shown to greatly improve fine detail reconstruction [S1]. Subsequent works extended NeRF to dynamic scenes (D-NeRF), SDF-based rendering (NeuS), and multi-GPU scaling (NeRF-XL) [S4, S15].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] One Is All: Bridging the Gap between Neural Radiance Fields Architectures with Progressive Volume Distillation, https://doi.org/10.1609/aaai.v37i1.25135, fetched 2026-10-26
[S3] RHINO: regularizing the hash-based implicit neural representation, https://doi.org/10.1007/s11432-024-4490-3, fetched 2026-10-26
[S4] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26
[S5] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S6] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S7] MixRT: Mixed Neural Representations For Real-Time NeRF Rendering, https://doi.org/10.1109/3dv62453.2024.00087, fetched 2026-10-27
[S8] Points2NeRF: Generating Neural Radiance Fields from 3D point cloud, https://doi.org/10.1016/j.patrec.2024.07.002, fetched 2026-10-29
[S9] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
[S10] Instance Neural Radiance Field, https://doi.org/10.1109/ICCV51070.2023.00079, fetched 2026-10-29
[S11] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
[S12] PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation, https://doi.org/10.1145/3581783.3612590, fetched 2026-11-08
[S13] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S14] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
[S15] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
[S16] Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01811, fetched 2026-10-26
[S17] Combining Volume Rendering with Adaptive Ray Marching in Neural ..., https://adaptivevolumerenderer.github.io/, fetched 2026-10-28 (unofficial)
[S18] HeadNeRF: A Realtime NeRF-based Parametric Head Model, https://doi.org/10.1109/cvpr52688.2022.01973, fetched 2026-10-26
[S19] High-fidelity 3D Reconstruction of Plants using Neural Radiance Field, https://arxiv.org/abs/2311.04154v1, fetched 2026-10-27
