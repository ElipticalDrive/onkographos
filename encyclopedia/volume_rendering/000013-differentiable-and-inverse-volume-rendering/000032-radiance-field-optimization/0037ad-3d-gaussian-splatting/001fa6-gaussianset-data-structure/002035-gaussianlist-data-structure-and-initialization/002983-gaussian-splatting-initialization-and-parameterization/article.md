# Gaussian Splatting Initialization and Parameterization

Also called: 3D Gaussian Initialization, 3D Gaussian Splatting Training Initialization, Gaussian Initialization, Gaussian Parameter Initialization, Gaussians class init, Initialization Details, Model Setup, pre-activation parameter setup

## Overview
This method establishes the initial state of the learnable 3D Gaussian parameters (position, scale, rotation, opacity, and color features) required to begin differentiable volume rendering optimization. It solves the problem of providing a valid, non-degenerate starting point for the optimizer, ensuring that initial opacities are in a trainable range and scales are geometrically meaningful relative to the input data density. It sets itself apart from [Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md) by handling full geometric and attribute seeding rather than just color parameterization, and from [Angle-pose Cuboid Uniform Initialization (ACUI)](4503-angle-pose-cuboid-uniform-initialization-acui.md) by relying on sparse reconstruction data rather than uniform volumetric sampling [S1, 3163-spherical-harmonics-initialization.md, 4503-angle-pose-cuboid-uniform-initialization-acui.md, unofficial].

## How it works
**Inputs:**
- Input point cloud data: Typically a COLMAP `.ply` or `.bin` file containing $N$ points with 3D coordinates $\mathbf{xyz} \in \mathbb{R}^3$ and RGB colors $\mathbf{c} \in [0, 1]^3$.
- Optional depth maps: For surface-aligned initialization [S2, unofficial].
- Configuration flags: `isotropic` (boolean), `load_path` (string).

**Outputs:**
- Learnable parameters stored as tensors (or `nn.Parameter` objects) on the GPU:
  - $\mathbf{\mu}$ (means): Shape $(N, 3)$.
  - $\mathbf{s}$ (scales): Shape $(N, 3)$ or $(N, 1)$ if isotropic. Stored as pre-activation values.
  - $\mathbf{q}$ (rotations): Shape $(N, 4)$, quaternion format $[w, x, y, z]$.
  - $\alpha$ (opacity): Shape $(N, 1)$. Stored as logit (pre-activation).
  - $\mathbf{f}_{dc}$ (DC SH features): Shape $(N, 1, 3)$.
  - $\mathbf{f}_{rest}$ (Rest SH features): Shape $(N, D, 3)$, initialized to zeros.

**Data Structures:**
- **Gaussian Parameters:** Stored as contiguous tensors. For standard implementations, scales and opacities are stored as pre-activation values to ensure valid ranges after sigmoid/softplus activation during rendering [S3, S4, unofficial].
- **Optimizer State:** Adam optimizer buffers allocated with capacity $buf\_capacity = num\_active \times 4$. Separate buffers for parameter, exponential moving average ($exp\_avg$), and exponential moving average squared ($exp\_avg\_sq$) for each of the 6 groups: means, scales, quats, featuresDc, featuresRest, opacities [S1, unofficial].

**Steps:**

1. **Initialize Means ($\mathbf{\mu}$):**
   - Copy input XYZ coordinates directly to GPU memory.
   - Center the point cloud by subtracting the mean: $\mathbf{\mu} = \mathbf{xyz} - \text{mean}(\mathbf{xyz})$ [S3, unofficial].

2. **Initialize Rotations ($\mathbf{q}$):**
   - *Standard:* Initialize all quaternions to identity $[1, 0, 0, 0]$ [S3, S4, unofficial].
   - *Surface-Aligned (Optional):* If depth maps are available, estimate normal maps from spatial gradients. Compute the quaternion that aligns the local z-axis with the estimated surface normal [S2, unofficial].
   - *Random (Optional):* Generate random unit quaternions using the method from [Sho94]:
     $$
     \begin{aligned}
     u_1, u_2, u_3 &\sim \text{Uniform}[0, 1] \\
     q_0 &= \sqrt{1-u_1} \sin(2\pi u_2) \\
     q_1 &= \sqrt{1-u_1} \cos(2\pi u_2) \\
     q_2 &= \sqrt{u_1} \sin(2\pi u_3) \\
     q_3 &= \sqrt{u_1} \cos(2\pi u_3)
     \end{aligned}
     $$
     [S1, unofficial].

3. **Initialize Scales ($\mathbf{s}$):**
   - Compute the distance to the $k$-nearest neighbors (KNN) for each point. Common $k$ values are 50 [S3, unofficial] or derived from orientation-aware KNN [S2, unofficial].
   - Let $d_i$ be the mean distance to neighbors for point $i$.
   - Compute pre-activation scale: $s_i = \log(\sqrt{d_i})$ or $s_i = \log(d_i \times s_{init})$ where $s_{init} = 1.5$ [S2, S3, unofficial].
   - *Fixed Base Distance Variant:* Some implementations use a fixed base distance $D_{base} = 0.1$. Compute $dist2 = \max(0.1, \min(1e-7, \text{random\_noise}))$. Then $s_i = \log(\sqrt{dist2})$ repeated for x, y, z dimensions [S4, unofficial].
   - *Random Variant:* For sparse-large-variance init, $s_i = \log((\text{rand}() + 1e-6) \times 0.01)$ [S3, S5, unofficial].
   - If isotropic, replicate the single scale value across x, y, z dimensions [S3, unofficial].

4. **Initialize Opacity ($\alpha$):**
   - Initialize as logits (pre-activation).
   - *Standard:* Set initial probability to 0.1 or 0.5. Compute logit: $\alpha_i = \log(p / (1-p))$. For $p=0.1$, $\alpha_i = \log(0.1/0.9) \approx -2.197$ [S1, unofficial]. For $p=0.5$, $\alpha_i = 0$ [S4, unofficial].
   - *High Opacity Variant:* Set pre-activation value to 8.0, yielding $\sigma(8.0) \approx 0.9997$ [S3, unofficial].
   - *Clamping:* Clamp initial logit values to a safe range, e.g., corresponding to probabilities in $[1e-4, 0.99]$, to prevent extreme gradients [S4, unofficial].

5. **Initialize Color Features ($\mathbf{f}_{dc}, \mathbf{f}_{rest}$):**
   - Convert input RGB colors $\mathbf{c}$ to range $[-0.5, 0.5]$.
   - Compute DC coefficient: $\mathbf{f}_{dc} = \mathbf{c} / C_0$, where $C_0 = 0.28209479177387814$ [S1, S3, unofficial].
   - Initialize $\mathbf{f}_{rest}$ to zeros [S1, S3, unofficial].
   - *Metallic/Roughness (Optional):* If using PBR, initialize albedo from point colors, metallic to 0.5, and roughness to 0.1 [S2, unofficial].

6. **Construct Covariance Matrices (for Rendering):**
   - Create diagonal scaling matrix $\mathbf{S} = \text{diag}(\exp(\mathbf{s}))$. Clamp $\exp(\mathbf{s})$ to max 10.0 to prevent excessively large Gaussians [S4, unofficial].
   - Convert quaternion $\mathbf{q}$ to rotation matrix $\mathbf{R}$ (shape $N \times 3 \times 3$). Normalize $\mathbf{q}$ first [S4, unofficial].
   - Compute 3D covariance: $\Sigma = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T$ [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def initialize_gaussians(points_xyz, points_rgb, num_neighbors=50, isotropic=False, init_type="standard"):
    """
    Initialize 3D Gaussian parameters from a point cloud.
    
    Args:
        points_xyz: (N, 3) numpy array of XYZ coordinates.
        points_rgb: (N, 3) numpy array of RGB colors in [0, 1].
        num_neighbors: int, number of KNN neighbors for scale estimation.
        isotropic: bool, if True, scales are shared across x, y, z.
        init_type: str, "standard", "surface", "random".
        
    Returns:
        Dict of initialized torch tensors on CUDA.
    """
    N = points_xyz.shape[0]
    device = torch.device("cuda")
    
    # 1. Means
    means = torch.tensor(points_xyz, dtype=torch.float32, device=device)
    means = means - means.mean(dim=0, keepdim=True)
    
    # 2. Rotations
    if init_type == "surface":
        # Placeholder for normal-based initialization
        # Requires depth maps and gradient estimation
        quats = torch.zeros(N, 4, device=device)
        # ... compute quaternion aligning z-axis with normal ...
        quats[:, 0] = 1.0 # Identity fallback if normals fail
    elif init_type == "random":
        # Sho94 random quaternion generation
        u1 = torch.rand(N, device=device)
        u2 = torch.rand(N, device=device)
        u3 = torch.rand(N, device=device)
        quats = torch.zeros(N, 4, device=device)
        quats[:, 0] = torch.sqrt(1 - u1) * torch.sin(2 * np.pi * u2)
        quats[:, 1] = torch.sqrt(1 - u1) * torch.cos(2 * np.pi * u2)
        quats[:, 2] = torch.sqrt(u1) * torch.sin(2 * np.pi * u3)
        quats[:, 3] = torch.sqrt(u1) * torch.cos(2 * np.pi * u3)
    else: # standard
        quats = torch.zeros(N, 4, device=device)
        quats[:, 0] = 1.0 # Identity [w, x, y, z]
        
    # 3. Scales
    # Compute KNN distances (simplified: assume dists are precomputed or use dummy)
    # In practice, use a KD-tree library. Here we simulate dists.
    # dists: (N,)
    dists = torch.ones(N, device=device) * 0.1 # Fallback base distance
    
    # If using KNN, replace dists with actual mean KNN distance
    # dists = compute_knn_mean_distance(means, k=num_neighbors)
    
    if init_type == "random":
        # Sparse-large-variance
        scales_pre = torch.log((torch.rand(N, 1, device=device) + 1e-6) * 0.01)
    else:
        # Standard: log(sqrt(dist)) or log(dist * 1.5)
        # Using base distance 0.1 variant if dists are not meaningful
        base_dist = 0.1
        dist2 = torch.clamp(dists, min=1e-7)
        # Some implementations use log(sqrt(dist2))
        scales_pre = torch.log(torch.sqrt(dist2)).unsqueeze(1)
        
        # If isotropic, keep as (N, 1). If anisotropic, repeat to (N, 3)
        if not isotropic:
            scales_pre = scales_pre.repeat(1, 3)
            
    # 4. Opacity (Logits)
    # Standard: logit(0.1)
    p_init = 0.1
    opacities = torch.log(torch.tensor(p_init, device=device) / (1 - p_init))
    opacities = opacities.unsqueeze(1) # (N, 1)
    
    # Clamp opacities to prevent extreme logits
    # Corresponding to p in [1e-4, 0.99]
    min_logit = torch.log(torch.tensor(1e-4, device=device) / (1 - 1e-4))
    max_logit = torch.log(torch.tensor(0.99, device=device) / (1 - 0.99))
    opacities = torch.clamp(opacities, min=min_logit, max=max_logit)
    
    # 5. Color Features (SH Degree 0)
    C0 = 0.28209479177387814
    colors_norm = torch.tensor(points_rgb, dtype=torch.float32, device=device) / 255.0 # Assuming 0-255 input
    # Or if input is already 0-1: colors_norm = torch.tensor(points_rgb, dtype=torch.float32, device=device)
    # Convert to [-0.5, 0.5]
    colors_norm = colors_norm - 0.5
    features_dc = (colors_norm / C0).unsqueeze(1) # (N, 1, 3)
    features_rest = torch.zeros(N, 15, 3, device=device) # Assuming SH degree 3 (15 coeffs)
    
    return {
        "_xyz": means,
        "_scaling": scales_pre,
        "_rotation": quats,
        "_opacity": opacities,
        "_features_dc": features_dc,
        "_features_rest": features_rest
    }

def compute_covariance(scaling_pre, rotation):
    """
    Compute 3D covariance matrix from pre-activation scales and quaternions.
    """
    # Clamp scaling to prevent excessively large Gaussians
    scaling = torch.exp(scaling_pre)
    scaling = torch.clamp(scaling, max=10.0)
    
    # Normalize quaternions
    rotation = torch.nn.functional.normalize(rotation, dim=1)
    
    # Convert quaternion to rotation matrix
    # q = [w, x, y, z]
    w, x, y, z = rotation[:, 0:1], rotation[:, 1:2], rotation[:, 2:3], rotation[:, 3:4]
    
    R = torch.zeros(rotation.shape[0], 3, 3, device=rotation.device)
    R[:, 0, 0] = 1 - 2 * (y**2 + z**2)
    R[:, 0, 1] = 2 * (x*y - w*z)
    R[:, 0, 2] = 2 * (x*z + w*y)
    R[:, 1, 0] = 2 * (x*y + w*z)
    R[:, 1, 1] = 1 - 2 * (x**2 + z**2)
    R[:, 1, 2] = 2 * (y*z - w*x)
    R[:, 2, 0] = 2 * (x*z - w*y)
    R[:, 2, 1] = 2 * (y*z + w*x)
    R[:, 2, 2] = 1 - 2 * (x**2 + y**2)
    
    # Diagonal scaling matrix
    S = torch.zeros(rotation.shape[0], 3, 3, device=rotation.device)
    S[:, 0, 0] = scaling[:, 0]
    S[:, 1, 1] = scaling[:, 1]
    S[:, 2, 2] = scaling[:, 2]
    
    # Covariance Sigma = R S S^T R^T
    cov3d = R @ S @ S.transpose(1, 2) @ R.transpose(1, 2)
    
    return cov3d
```

## Subtopics

### [Gaussian Model Initialization](2349-gaussian-model-initialization.md)
Populates learnable parameters (position, features, scaling, rotation, and opacity) from input data, typically a Structure-from-Motion (SfM) point cloud. It sets itself apart from sibling methods like [Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md) by handling full geometric and attribute seeding rather than just color parameterization, and from [Angle-pose Cuboid Uniform Initialization (ACUI)](4503-angle-pose-cuboid-uniform-initialization-acui.md) by relying on sparse reconstruction data rather than uniform volumetric sampling.

### Random Large-Variance Initialization
Relaxes the SfM-only initialization assumption with sparse-large-variance random init. Scales are initialized using the log of random values scaled by 0.01: $\text{pre\_act\_scales} = \log((\text{rand}() + 1e-6) \times 0.01)$ [S3, S5, unofficial].

### Neural Point Cloud Initialization from COLMAP
A method to initialize the neural point cloud and their features for Point-NeRF using an external point cloud reconstruction (specifically COLMAP). It solves the problem of how to provide initial positions, variances, and feature vectors for the neural points [S6]. The variance parameter $\gamma_i$ for each point $p_i$ is initialized to 0.3. Feature vectors $f_i$ are acquired by finding the view where the camera is closest to $p_i$ (after ruling out occluded views) and unprojecting $p_i$ onto the feature maps from that view [S6].

### Temporal Point Cloud Initialization via Chamfer Distance Minimization
A technique to reduce training time for dynamic video scenes by initializing the point cloud of the current frame using the learned model from the previous frame, leveraging temporal correlation between nearby frames [S7]. It minimizes the Chamfer distance $CD(S_1, S_2)$ between the previous frame's point cloud $S_1$ and the current frame's initial cloud $S_2$ via gradient descent. After refining positions, the top-$k$ closest points in $S_1$ are used to average their spherical harmonics parameters as the initial radiance field parameter for $S_2$. This strategy achieves a 4x improvement in training runtime [S7].

## Operating characteristics
- **Time Complexity:** Initialization is $O(N \log N)$ due to KNN computation if used, or $O(N)$ if using a fixed base distance. KNN is typically the bottleneck.
- **Memory Complexity:** $O(N)$ for storing the initial parameters. The Adam optimizer state adds $6 \times 4 \times N$ floats (parameter, exp_avg, exp_avg_sq for 6 groups) [S1, unofficial].
- **Performance:** Initialization quality significantly impacts convergence speed. Using SfM points with KNN-based scales provides a geometrically aware start, reducing the number of iterations needed to cover the scene compared to random initialization [S1, S3, unofficial].
- **Strengths:** Provides a valid, non-degenerate starting point. Pre-activation parameterization ensures stable gradients. KNN-based scaling adapts to point cloud density.
- **Limitations:** Relies on the quality of the input SfM reconstruction. Poor SfM (sparse or noisy points) leads to poor initialization. KNN computation can be slow for very large $N$ [S1, S3, unofficial].
- **When to use:** Standard for static scene reconstruction from multi-view images. Use surface-aligned initialization when depth maps are available and surface accuracy is critical [S2, unofficial]. Use temporal initialization for dynamic video scenes [S7].

## Background
The method originates from the original 3D Gaussian Splatting paper [Kerbl et al., SIGGRAPH 2023], which introduced the use of 3D Gaussians as a primitive for explicit radiance field representation. The specific initialization details (KNN scales, logit opacities, SH features) were refined in subsequent implementations and variants to improve training stability and convergence [S1, S3, S4, unofficial]. The concept of using pre-activation values for scales and opacities is a common optimization technique in neural rendering to handle bounded parameters [S3, S4, unofficial].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S3] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S4] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S5] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] Point-NeRF: Point-based Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00536, fetched 2026-10-26
[S7] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
