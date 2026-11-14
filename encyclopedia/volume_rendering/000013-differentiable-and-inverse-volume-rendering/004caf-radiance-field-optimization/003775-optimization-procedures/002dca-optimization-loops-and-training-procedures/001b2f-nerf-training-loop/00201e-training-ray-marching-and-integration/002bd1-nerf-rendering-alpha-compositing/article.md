# NeRF Rendering (Alpha Compositing)

Also called: NeRF Integration, NeRF Rendering Equation, NeRFs, Neural Radiance Field Volume Rendering, Standard NeRF rendering, Vanilla NeRF Volume Rendering, Vanilla NeRF integration

## Overview
NeRF Rendering (Alpha Compositing) is the differentiable volume rendering technique that synthesizes novel views by tracing camera rays through a scene represented by implicit density and color functions, computing the final pixel color via numerical integration of these fields [S1, S2, S11]. It solves the problem of continuous 3D scene representation and view synthesis by approximating the volume rendering integral using stratified sampling and alpha compositing, distinguishing itself from deterministic ray tracing by leveraging probabilistic density functions to handle view-dependent effects and complex geometry [S11]. Unlike explicit mesh or point cloud representations, NeRF renders continuous volumetric data, allowing for seamless handling of translucent, transparent, and occluded surfaces [S11].

## How it works
**Inputs:**
*   Camera ray $r(t) = o + td$, defined by origin $o$ (camera center) and direction $d$ (view direction).
*   Integration bounds $t_n$ (near plane) and $t_f$ (far plane/bounding volume) [S1, S11].
*   Number of samples $N$ [S3, S11].
*   Neural network model $F_\Theta$ mapping $(x, d) \rightarrow (c, \sigma)$ [S10, S11].

**Outputs:**
*   Rendered RGB color $\hat{C}(r)$ [S1, S2].
*   Rendered Depth $\hat{D}(r)$ [S1][S6, unofficial].
*   (Optional) Accumulated opacity/alpha $\hat{\alpha}(r)$ [S6, unofficial].

**Data Structures:**
*   **Sample Points:** Array of shape $(N, 3)$ containing 3D coordinates $x_i = o + t_i d$.
*   **Ray Directions:** Array of shape $(N, 3)$ containing the normalized view direction $d$.
*   **Predictions:** Tensor of shape $(N, 4)$ (or $(N, 5)$ if including positional encoding gradients) containing predicted RGB color $c_i$ and volume density $\sigma_i$.
*   **Deltas:** Array of shape $(N-1)$ or $(N)$ representing distances between samples.

**Steps:**

1.  **Ray Generation:** For each pixel in the target image, generate a camera ray $r(t) = o + td$ using the camera's intrinsic and extrinsic parameters [S1, S3, S11].
2.  **Sampling:** Divide the interval $[t_n, t_f]$ into $N$ equally spaced bins. Draw one sample $t_i$ uniformly at random from each bin $i$ [S1, S10, S11]:
    $$ t_i \sim U\left[t_n + \frac{i-1}{N}(t_f - t_n), t_n + \frac{i}{N}(t_f - t_n)\right] $$
    Compute the 3D positions $x_i = o + t_i d$ [S4, S11].
3.  **Network Query:** Evaluate the neural network $F_\Theta$ for each sample point $x_i$ and the ray direction $d$ to obtain color $c_i$ and density $\sigma_i$ [S1, S10, S11].
    $$ [c_i; \sigma_i] = F_\Theta(x_i, d) $$
    Apply activation functions: typically sigmoid for RGB to bound values to $[0, 1]$ and ReLU for density to ensure non-negativity [S8, S15].
4.  **Delta Calculation:** Compute the distance $\delta_i$ between adjacent samples. For the last sample, append a large value (e.g., $10^{10}$) to represent the infinite tail of the ray [S6, unofficial][S8]:
    $$ \delta_i = t_{i+1} - t_i \quad \text{for } i < N-1 $$
    $$ \delta_{N-1} = 10^{10} $$
    If the ray direction is not unit length, scale deltas by the norm of the direction vector to convert to real-world distance [S6, unofficial].
5.  **Alpha Compositing:**
    *   Compute opacity $\alpha_i$ for each sample:
        $$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
    *   Compute transmittance $T_i$, the probability that the ray travels from the near bound to sample $i$ without being intercepted:
        $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
        Note: $T_1 = 1.0$. To prevent numerical instability, add a small epsilon ($\epsilon = 10^{-10}$) to the term $(1 - \alpha_i)$ before cumulative product [S8].
    *   Compute weights $w_i$:
        $$ w_i = \alpha_i T_i $$
6.  **Color Integration:** Compute the final RGB color as the weighted sum of sample colors:
    $$ \hat{C}(r) = \sum_{i=1}^{N} w_i c_i $$
7.  **Depth Integration:** Compute the expected depth as the weighted sum of sample distances:
    $$ \hat{D}(r) = \sum_{i=1}^{N} w_i t_i $$
8.  **Background Handling:** If a white background is assumed, add the complement of the accumulated opacity to the RGB result:
    $$ \hat{C}_{final}(r) = \hat{C}(r) + (1 - \sum_{i=1}^{N} w_i) \cdot 1.0 $$
    [S6, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def nerf_render(rays_o, rays_d, t_vals, model, white_background=False, rand=True):
    """
    Render RGB and Depth using NeRF alpha compositing.
    
    Args:
        rays_o: Tensor of shape (B, 3), ray origins.
        rays_d: Tensor of shape (B, 3), ray directions (normalized).
        t_vals: Tensor of shape (B, N), sample distances along rays.
        model: Callable taking (points, directions) -> (rgb, sigma).
        white_background: Boolean, whether to composite against white background.
        rand: Boolean, whether deltas include the infinite tail for stratified sampling.
        
    Returns:
        rgb: Tensor of shape (B, 3), rendered colors.
        depth: Tensor of shape (B), rendered depths.
    """
    # 1. Compute 3D points
    # points shape: (B, N, 3)
    points = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * t_vals.unsqueeze(2)
    
    # 2. Query Model
    # Reshape for batched model evaluation if necessary
    B, N = t_vals.shape
    points_flat = points.reshape(-1, 3)
    rays_d_flat = rays_d.unsqueeze(1).expand(-1, N, -1).reshape(-1, 3)
    
    # Model outputs: (B*N, 4) -> [R, G, B, Sigma]
    predictions = model(points_flat, rays_d_flat)
    
    # Reshape back to (B, N, 4)
    predictions = predictions.reshape(B, N, 4)
    
    # 3. Slice and Activate
    rgbs = torch.sigmoid(predictions[..., :3]) # Shape: (B, N, 3)
    sigmas = torch.relu(predictions[..., 3])   # Shape: (B, N)
    
    # 4. Compute Deltas
    # delta shape: (B, N-1)
    deltas = t_vals[..., 1:] - t_vals[..., :-1]
    
    if rand:
        # Append large delta for the last segment to infinity
        # delta_inf shape: (B, 1)
        delta_inf = torch.ones(B, 1, device=t_vals.device) * 1e10
        deltas = torch.cat([deltas, delta_inf], dim=-1) # Shape: (B, N)
    else:
        # For deterministic rendering, broadcast the last delta or handle appropriately
        # Standard practice in some implementations is to just use the computed deltas
        # and handle the tail via the transmittance logic or a fixed large delta
        delta_inf = torch.ones(B, 1, device=t_vals.device) * 1e10
        deltas = torch.cat([deltas, delta_inf], dim=-1)

    # 5. Compute Alpha
    # alpha shape: (B, N)
    alpha = 1.0 - torch.exp(-sigmas * deltas)
    
    # 6. Compute Transmittance
    # exp_term shape: (B, N)
    exp_term = 1.0 - alpha + 1e-10 # Add epsilon for stability
    # cumprod exclusive: T_i = product(1-alpha_j) for j < i
    # T shape: (B, N)
    transmittance = torch.cumprod(exp_term, dim=-1, exclusive=True)
    
    # 7. Compute Weights
    # weights shape: (B, N)
    weights = alpha * transmittance
    
    # 8. Compute RGB
    # rgb_final shape: (B, 3)
    rgb_final = torch.sum(weights.unsqueeze(-1) * rgbs, dim=-2)
    
    # 9. Compute Depth
    # depth_final shape: (B)
    depth_final = torch.sum(weights * t_vals, dim=-1)
    
    # 10. Background Handling
    if white_background:
        # weights_sum shape: (B, 1)
        weights_sum = weights.sum(dim=-1, keepdim=True)
        rgb_final = rgb_final + (1.0 - weights_sum) * 1.0
        
    return rgb_final, depth_final
```

**Parameters:**
*   $N$: Number of samples per ray. Typical values range from 48 to 256 for coarse/fine stages [S3, S11].
*   $t_n, t_f$: Near and far clipping planes. Determined by scene bounding volume or camera frustum [S1, S11].
*   `white_background`: Boolean flag. Default False. If True, composites against white [S6, unofficial].
*   `rand`: Boolean flag. Default True. If False, uses deterministic sampling (no jitter) [S8].

**Edge Cases and Pitfalls:**
*   **Numerical Stability:** The cumulative product of transmittance can underflow to zero, causing vanishing gradients. Adding epsilon ($10^{-10}$) to $(1-\alpha)$ before `cumprod` is critical [S8].
*   **Infinite Tail:** The last segment of the ray extends to infinity. A large finite value (e.g., $10^{10}$) is used for $\delta_{N-1}$ to approximate this, ensuring the transmittance decays correctly if density is non-zero [S6, unofficial][S8].
*   **Density Non-Negativity:** Density $\sigma$ must be non-negative. ReLU activation is standard; negative densities break the physical interpretation of alpha compositing [S8].
*   **Memory:** Evaluating $N$ points per ray for all pixels can exceed GPU memory. Chunking rays or points is required for high-resolution images [S7, S16, unofficial].

## Subtopics

### [Depth-Guided Sampling](7490-depth-guided-sampling.md)
Depth-Guided Sampling is a ray sampling strategy for neural radiance fields that focuses scene sampling on estimated surface regions using depth priors from input views to increase sample density in relevant areas, thereby improving synthesis quality and allowing a reduction in the number of samples per ray. It sets itself apart from uniform or hierarchical stratified sampling by explicitly leveraging predicted depth maps (and their uncertainties) to bias the placement of query points toward likely object surfaces, reducing redundant sampling in free space or occluded regions.

### [Depth-Prior Guided Stratified Sampling](11292-depth-prior-guided-stratified-sampling.md)
Depth-Prior Guided Stratified Sampling replaces uniform depth sampling in volume rendering by using predicted depth priors and associated confidence bounds to focus sampling on likely object surfaces.

### [Neus SDF Volume Rendering](12443-neus-sdf-volume-rendering.md)
Neus SDF Volume Rendering is a technique that renders 3D scenes represented as Signed Distance Functions (SDF) or Signed Ray Distance Functions (SRDF) by accumulating color and depth along rays using alpha compositing. It solves the problem of precisely extracting geometric surfaces from implicit representations by replacing the standard NeRF density field with an SDF-based weighting scheme. It differs from standard NeRF rendering by using a trainable S-density function to concentrate weight at the zero-level set of the SDF, ensuring the surface contributes most to the image, and adapts this theory to SRDF by treating surface normals as irrelevant to the value distribution.

### [Progressive Blur Estimation](14151-progressive-blur-estimation.md)
Progressive Blur Estimation (PBE) is a two-stage implicit function module that estimates defocus and motion blur by predicting multiple ray origins, directions, and blending weights for each pixel. It solves the problem of rendering blurry observations from a radiance field by modeling blur as a distribution of rays rather than a single ray. It sets itself apart from single-stage blur models by using a coarse stage to predict initial kernel locations and a refinement stage that incorporates ray features for fine-grained blur modeling, which is more accurate and efficient than a single-stage approach with ray features.

### [MipNerfModel](14207-mipnerfmodel.md)
The MipNerfModel is a Neural Radiance Field (NeRF) architecture that approximates integrated positional encoding by treating rays as cones or cylinders rather than infinitesimally thin lines, thereby solving aliasing artifacts caused by high-frequency scene details. It distinguishes itself from standard NeRF implementations by using multi-level sampling (coarse and fine) and integrated positional encoding to model the frequency content of the signal along the ray, which is essential for training on unbounded, outdoor scenes.

### NeRF Rendering Pipeline
Iterates over a list of camera poses to render a sequence of images. For each pose, rays are cast, and the rendering function is called. If a `render_factor` is specified, the image dimensions and focal length are downsampled by this factor to speed up preview rendering (typical values 4 or 8) [S16, unofficial]. The resulting RGB and depth maps are stored. If ground truth images are available and no downsampling is used, PSNR is computed. Results are saved as 8-bit PNGs [S16, unofficial].

### Rendered Depth-Guided Warping
A method to generate a warped image from a source view to an unseen target view using rendered depth and viewpoint difference, enabling consistency regularization without full-resolution rendering [S17, unofficial]. Rays are sampled on a strided grid (e.g., stride 2) to create low-resolution patches. Depth is rendered for these rays, then upsampled via bilinear interpolation to full resolution for inverse warping [S17, unofficial]. The pixel coordinate transformation is $p_{j\to i} \sim K R_{j\to i} D_j(p_j) K^{-1} p_j$, where $K$ is the intrinsic matrix, $R_{j\to i}$ is the rotation, and $D_j$ is the depth map [S17, unofficial].

### NeRF-XL Multi-GPU Volume Rendering
Scales NeRF rendering across multiple GPUs by partitioning the ray into segments processed by different GPUs. Global transmittance $T$ and color $C$ are maintained. Each GPU $k$ processes its segment, gathering transmittance and color from other GPUs if necessary, and updates the global accumulators: $C_{1 \to N+1} += T_{1 \to i} \times C_{i \to i+1}$ and $T_{1 \to i^*} *= T_{i \to i+1}$ [S18]. During inference, data gathering is only required on a single GPU to render the final image, reducing data transfer [S18].

### NeRF Volume Rendering via Numerical Quadrature
Approximates the volume rendering integral by summing contributions from sampled spheres inside the cone using numerical quadrature, often used in anti-aliasing methods like Tri-MipRF [S19]. The color is calculated as $\sum_i T_i (1-\exp(-\sigma_i \delta_i)) c_i$, where $\delta_i$ represents the distance between sampled spheres [S19].

### Raymarcher
A rendering procedure that uses an LSTM (RM-LSTM) to predict a single intersection point and directly outputs RGB values, skipping full volume rendering [S20, unofficial]. It extracts latent features from a CNN encoder, feeds them into the LSTM (512 channels, 10 steps), and predicts step lengths to update depth. The final depth point is queried for RGB. This method is ~80 times faster than baseline volume rendering but yields slightly lower quality (PSNR 23.02 vs 23.34) [S20, unofficial].

### 2D Neural Rendering Module
A module that maps low-resolution feature maps to final images using upsampling and 2D convolution to alleviate multi-view inconsistencies [S21]. It repeats the input feature tensor, applies a learnable MLP, adds the result, applies PixelShuffle (scale 2), and applies a 1x1 Convolution with a fixed blur kernel [S21]. This allows HeadNeRF to render in real-time (>40 fps) compared to ~5s per frame for vanilla NeRF [S21].

### Depth Estimation from NeRF
Extracts 3D point clouds by analyzing the transmittance function $T(t)$ along the ray. The surface depth $t_{surface}$ is found by minimizing the first-order derivative of $T(t)$ with respect to $t$: $t_{surface} \approx \arg \min_t |\frac{dT(t)}{dt}|$ [S22]. The 3D point is $p = o + t_{surface} d$, and its color is queried from the NeRF [S22].

### PixelNeRF Rendering Module
Takes input images and a target pose to produce an approximate rendering (RGB and features) for conditioning a diffusion U-Net [S23]. Input images are passed through a 2D U-Net to create feature images. Rays are cast, and 128 points are sampled (uniform up to distance 1, then linear in disparity). These points are reprojected into input cameras, and features are gathered. Positional encoding and statistics are appended, resulting in a tensor of shape $64 \times 64 \times 131$ (3 RGB + 128 features) [S23].

### NeRF Rendering with Stratified Sampling
Uses a hierarchical sampling strategy. Coarse sampling uses uniform distribution to determine object positions. Fine sampling uses volume density weights to improve detail representation. The loss is the sum of squared errors for both coarse and fine samples against ground truth [S24].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of samples per ray and $P$ is the number of rays (pixels). The neural network evaluation dominates the cost.
*   **Memory Complexity:** $O(N \cdot P)$ to store intermediate activations for backpropagation during training. During inference, chunking can reduce this to $O(N \cdot P_{chunk})$ [S7, S16, unofficial].
*   **Performance:** Original NeRF required significant computing power and long training times [S11]. Instant NGP reduced processing time significantly using multi-resolution hash encoding [S11]. Real-time rendering (>40 fps) is achievable with optimizations like 2D neural rendering modules (HeadNeRF) or raymarchers, though often at the cost of quality [S20, unofficial][S21].
*   **Strengths:** Handles view-dependent effects (non-Lambertian materials) naturally [S11]. Produces high-quality, photorealistic novel views with fine details [S11]. Robust to challenging scenes (metallic, translucent, transparent, occluded) where photogrammetry fails [S11].
*   **Limitations:** High computational cost for training and rendering without acceleration [S11]. Not a mesh or point cloud, making geometric extraction non-trivial (requires depth estimation or SDF conversion) [S11]. Aliasing artifacts can occur with high-frequency details if not using integrated positional encoding (MipNeRF) [S14, S24].
*   **When to Use:** Novel view synthesis, cultural heritage reconstruction, scenes with complex lighting/view-dependence.
*   **When Not to Use:** Real-time applications requiring explicit geometry (meshes) without post-processing, or when computational resources are extremely limited without acceleration techniques.

## Background
NeRF (Neural Radiance Fields) was first presented by Mildenhall et al. in 2020 [S11]. It represents a static scene as a continuous 5D function mapping spatial location and viewing direction to volume density and emitted radiance [S10, S11]. The method was trained using a differentiable volume rendering loss on multi-view images [S1, S10]. Key developments include the introduction of positional encoding for fine detail reconstruction [S1, S11], hierarchical sampling [S11], and later acceleration techniques like Instant NGP (multi-resolution hash encoding) by Müller et al. in 2022 [S11]. It is implemented in platforms like Nerfstudio and Luma AI [S11].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] One Is All: Bridging the Gap between Neural Radiance Fields Architectures with Progressive Volume Distillation, https://doi.org/10.1609/aaai.v37i1.25135, fetched 2026-10-26
[S3] RHINO: regularizing the hash-based implicit neural representation, https://doi.org/10.1007/s11432-024-4490-3, fetched 2026-10-26
[S4] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26
[S5] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
[S6] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S7] sxyu/pixel-nerf, https://github.com/sxyu/pixel-nerf, fetched 2026-10-26 (unofficial)
[S8] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S9] MixRT: Mixed Neural Representations For Real-Time NeRF Rendering, https://doi.org/10.1109/3dv62453.2024.00087, fetched 2026-10-27
[S10] Points2NeRF: Generating Neural Radiance Fields from 3D point cloud, https://doi.org/10.1016/j.patrec.2024.07.002, fetched 2026-10-29
[S11] NEURAL RADIANCE FIELDS (NERF): REVIEW AND POTENTIAL APPLICATIONS TO DIGITAL CULTURAL HERITAGE, https://doi.org/10.5194/isprs-archives-xlviii-m-2-2023-453-2023, fetched 2026-10-29
[S12] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
[S13] Instance Neural Radiance Field, https://doi.org/10.1109/ICCV51070.2023.00079, fetched 2026-10-29
[S14] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
[S15] PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation, https://doi.org/10.1145/3581783.3612590, fetched 2026-11-08
[S16] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S17] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
[S18] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
[S19] Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01811, fetched 2026-10-26
[S20] Combining Volume Rendering with Adaptive Ray Marching in Neural ..., https://adaptivevolumerenderer.github.io/, fetched 2026-10-28 (unofficial)
[S21] HeadNeRF: A Realtime NeRF-based Parametric Head Model, https://doi.org/10.1109/cvpr52688.2022.01973, fetched 2026-10-26
[S22] High-fidelity 3D Reconstruction of Plants using Neural Radiance Field, https://arxiv.org/abs/2311.04154v1, fetched 2026-10-27
[S23] ReconFusion: 3D Reconstruction with Diffusion Priors, https://doi.org/10.1109/cvpr52733.2024.02036, fetched 2026-10-26
[S24] An Index of Refraction Adaptive Neural Refractive Radiance Field for Transparent Scenes, https://doi.org/10.3390/electronics14061073, fetched 2026-10-29
