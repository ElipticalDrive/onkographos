# Gaussian Splatting Initialization and Parameterization

Also called: 3D Gaussian Initialization, 3D Gaussian Splatting Training Initialization, Gaussian Initialization, Gaussian Parameter Initialization, Gaussians class init, Model Setup, pre-activation parameter setup

## Overview
This method establishes the initial state of the learnable 3D Gaussian point cloud for differentiable rendering training [S3, unofficial]. It solves the problem of converting raw geometric data (typically from Structure-from-Motion pipelines like COLMAP) or random noise into a set of valid, differentiable parameters (means, covariances, opacities, and spherical harmonics) that can be optimized via gradient descent [S3, unofficial]. It is distinguished from sibling methods like [LinPrim Population Control](2295-linprim-population-control.md) by focusing exclusively on the *creation* and *parameterization* of the initial set, rather than the dynamic modification of that set during training [S3, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Source point cloud data: Typically a COLMAP `.ply` or `.bin` file containing $N$ points with coordinates $\mathbf{xyz} \in \mathbb{R}^3$ and RGB colors $\mathbf{c} \in [0, 1]^3$ [S2, S3, unofficial].
- Alternatively, a random distribution if no source data is provided [S3, unofficial].
- Configuration flags: `isotropic` (boolean, default false) [S3, unofficial], `load_path` (string) [S3, unofficial].

**Outputs:**
- A Gaussian object containing the following learnable parameters:
  - $\mathbf{\mu}$ (means): Shape $(N, 3)$ [S4, unofficial].
  - $\mathbf{s}$ (pre-activation scales): Shape $(N, 3)$ or $(N, 1)$ if isotropic [S3, unofficial].
  - $\mathbf{q}$ (pre-activation rotations): Shape $(N, 4)$, stored as quaternions $[w, x, y, z]$ [S4, unofficial].
  - $\alpha_{logit}$ (pre-activation opacity): Shape $(N,)$ [S3, unofficial].
  - $\mathbf{f}_{dc}$ (DC spherical harmonics coefficients): Shape $(N, 1, 3)$ [S4, unofficial].
  - $\mathbf{f}_{rest}$ (rest spherical harmonics coefficients): Shape $(N, D, 3)$, initialized to zero [S1, unofficial].

### Data Structures
The parameters are stored as dense tensors on the CUDA device [S4, unofficial].
- **Means**: Shape $(N, 3)$ [S4, unofficial].
- **Scaling**: Shape $(N, 3)$ [S4, unofficial].
- **Rotation**: Shape $(N, 4)$ [S4, unofficial].
- **Opacity**: Shape $(N, 1)$ or $(N,)$ [S4, unofficial].
- **Features DC**: Shape $(N, 1, 3)$ [S4, unofficial].
- **Features Rest**: Shape $(N, \text{num_coeffs}, 3)$ [S1, unofficial].

### Steps
1. **Initialize Means ($\mathbf{\mu}$)**
   Copy the XYZ coordinates from the input point cloud directly to the GPU tensor [S1, S3, unofficial]. For random initialization, means are sampled from a random distribution [S3, unofficial].

2. **Initialize Rotations ($\mathbf{q}$)**
   Initialize all quaternions to the identity rotation $[1, 0, 0, 0]$ [S2, S3, S4, unofficial]. Some implementations use random uniform quaternions generated via the Shoemake method [S1, unofficial].

3. **Initialize Scales ($\mathbf{s}$)**
   Scales are stored as pre-activation values (log-space) to allow exponential activation during rendering [S3, unofficial].
   - **Method A (KNN-based)**: Compute the mean distance to $k=50$ nearest neighbors for each point [S2, unofficial]. Clamp to a minimum of $10^{-7}$ [S3, unofficial]. Set $s_i = \log(\sqrt{d_i})$ [S3, unofficial]. Replicate across 3 dimensions or keep separate [S2, S3, unofficial].
   - **Method B (Fixed Base)**: Use a fixed base distance $d_{base} = 0.1$ [S4, unofficial]. Set $s_i = \log(\sqrt{d_{base}})$ repeated 3 times [S4, unofficial].
   - **Method C (Random)**: For random initialization, $s_i = \log((\text{rand}() + 10^{-6}) \times 0.01)$ [S3, unofficial].

4. **Initialize Opacity ($\alpha_{logit}$)**
   Opacity is stored as a logit so that $\alpha = \sigma(\alpha_{logit})$ [S1, unofficial]. Choose a target initial opacity $\alpha_{target} \in \{0.1, 0.5\}$ [S1, S4, unofficial]. Clamp to $[10^{-4}, 0.99]$ to prevent extreme logit values [S4, unofficial]. Compute $\alpha_{logit} = \ln(\alpha_{target} / (1 - \alpha_{target}))$ [S1, S4, unofficial]. Alternatively, initialize to a constant high value like $8.0$, yielding $\sigma(8.0) \approx 0.9997$ [S3, unofficial].

5. **Initialize Spherical Harmonics ($\mathbf{f}_{dc}, \mathbf{f}_{rest}$)**
   Normalize input RGB colors to the range $[-0.5, 0.5]$ [S1, unofficial]. Compute the DC coefficient (degree 0) by dividing the normalized RGB by the SH basis constant $C_0 = 0.28209479177387814$ [S1, unofficial]. Initialize all higher-order coefficients ($\mathbf{f}_{rest}$) to zero [S1, unofficial].

6. **Construct Covariance Matrices (Runtime/Forward Pass)**
   While not stored directly, the covariance is derived from $\mathbf{s}$ and $\mathbf{q}$ for rendering [S4, unofficial]. Create diagonal scaling matrix $\mathbf{S} = \text{diag}(\exp(\mathbf{s}))$ [S4, unofficial]. Normalize quaternion $\mathbf{q}$ to unit length [S4, unofficial]. Convert $\mathbf{q}$ to rotation matrix $\mathbf{R}$ (shape $N \times 3 \times 3$) [S4, unofficial]. Compute 3D covariance $\Sigma = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T$ [S4, unofficial].

### Formulas
- 3D covariance: $\Sigma = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T$, where $\mathbf{R}$ is the rotation matrix derived from quaternions and $\mathbf{S}$ is the diagonal scaling matrix [S4, unofficial].
- 2D projection covariance: $\Sigma' = \mathbf{J} \mathbf{W} \Sigma \mathbf{W}^T \mathbf{J}^T$, where $\mathbf{J}$ is the Jacobian of the perspective projection, $\mathbf{W}$ is the world-to-camera rotation matrix, and $\Sigma$ is the 3D covariance [S4, unofficial].
- Opacity activation: $\alpha = \sigma(\alpha_{logit})$ [S4, unofficial].
- Scale activation: $\mathbf{s}_{actual} = \exp(\mathbf{s})$ [S4, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def initialize_gaussians(points_xyz, points_rgb, isotropic=False):
    """
    Initialize 3D Gaussian parameters from a point cloud.
    
    Args:
        points_xyz: (N, 3) array of XYZ coordinates.
        points_rgb: (N, 3) array of RGB colors in [0, 1].
        isotropic: If True, scales are shared across x, y, z.
        
    Returns:
        A dictionary of initialized learnable parameters.
    """
    N = points_xyz.shape[0]
    device = torch.device("cuda")
    
    # 1. Means
    means = torch.tensor(points_xyz, dtype=torch.float32, device=device, requires_grad=True)
    
    # 2. Rotations (Identity)
    # Quaternion format: [w, x, y, z]
    rotations = torch.zeros((N, 4), dtype=torch.float32, device=device, requires_grad=True)
    rotations[:, 0] = 1.0  # w = 1
    
    # 3. Scales (Pre-activation)
    # Fixed base distance method [S4, unofficial]
    base_dist = 0.1
    scales_pre = torch.full((N, 3), torch.log(torch.sqrt(torch.tensor(base_dist))), dtype=torch.float32, device=device, requires_grad=True)
    
    # 4. Opacity (Pre-activation logit)
    alpha_target = 0.1
    alpha_target = torch.clamp(torch.tensor(alpha_target), min=1e-4, max=0.99)
    opacity_logit = torch.logit(alpha_target).expand(N).to(device, requires_grad=True)
    
    # 5. Spherical Harmonics
    rgb_norm = (torch.tensor(points_rgb, dtype=torch.float32, device=device) - 0.5)
    C0 = 0.28209479177387814
    features_dc = (rgb_norm / C0).unsqueeze(1)
    features_dc.requires_grad = True
    features_rest = torch.zeros((N, 0, 3), dtype=torch.float32, device=device, requires_grad=True)
    
    return {
        "means": means,
        "scales": scales_pre,
        "rotations": rotations,
        "opacity_logit": opacity_logit,
        "features_dc": features_dc,
        "features_rest": features_rest
    }

def compute_covariance(scales, rotations):
    """
    Compute 3D covariance matrix from pre-activation scales and quaternions.
    
    Args:
        scales: (N, 3) pre-activation scales.
        rotations: (N, 4) quaternions [w, x, y, z].
        
    Returns:
        cov3d: (N, 3, 3) covariance matrices.
    """
    # Exponential activation for scales
    S = torch.exp(scales)  # (N, 3)
    
    # Diagonal scaling matrix
    S_diag = torch.zeros((scales.shape[0], 3, 3), device=scales.device)
    S_diag[:, 0, 0] = S[:, 0]
    S_diag[:, 1, 1] = S[:, 1]
    S_diag[:, 2, 2] = S[:, 2]
    
    # Normalize quaternions
    q_norm = torch.nn.functional.normalize(rotations, p=2, dim=1)
    w, x, y, z = q_norm[:, 0], q_norm[:, 1], q_norm[:, 2], q_norm[:, 3]
    
    # Convert quaternion to rotation matrix
    R = torch.zeros((rotations.shape[0], 3, 3), device=rotations.device)
    R[:, 0, 0] = 1 - 2 * (y**2 + z**2)
    R[:, 0, 1] = 2 * (x*y - w*z)
    R[:, 0, 2] = 2 * (x*z + w*y)
    R[:, 1, 0] = 2 * (x*y + w*z)
    R[:, 1, 1] = 1 - 2 * (x**2 + z**2)
    R[:, 1, 2] = 2 * (y*z - w*x)
    R[:, 2, 0] = 2 * (x*z - w*y)
    R[:, 2, 1] = 2 * (y*z + w*x)
    R[:, 2, 2] = 1 - 2 * (x**2 + y**2)
    
    # Covariance: Sigma = R * S * S^T * R^T
    cov3d = R @ S_diag @ R.transpose(1, 2)
    
    return cov3d
```

### Parameters and Typical Values
- **$C_0$**: $0.28209479177387814$ [S1, unofficial].
- **Base Distance**: $0.1$ [S4, unofficial].
- **Initial Opacity Target**: $0.1$ [S1, unofficial] or $0.5$ [S4, unofficial].
- **Opacity Pre-activation Constant**: $8.0$ (yields $\approx 0.9997$) [S3, unofficial].
- **Numerical Stability Bias**: $0.3$ added to the diagonal of the 2D projected covariance matrix to prevent singularity [S4, unofficial].
- **Max Scale**: $10.0$. Applied via `torch.clamp(torch.exp(scales), max=10.0)` to prevent excessively large Gaussians [S4, unofficial].
- **Adam Learning Rates**:
  - Means: $0.00016$ [S1, unofficial].
  - Scales: $0.005$ [S1, unofficial].
  - Quaternions: $0.001$ [S1, unofficial].
  - Features DC: $0.0025$ [S1, unofficial].
  - Features Rest: $0.000125$ [S1, unofficial].
  - Opacities: $0.05$ [S1, unofficial].

### Edge Cases and Pitfalls
- **Division by Zero**: In the Jacobian computation for perspective projection, depth $z$ is clamped with $10^{-7}$ to prevent division by zero [S4, unofficial].
- **Singular Covariance**: Adding $0.3$ to the diagonal of the 2D covariance matrix ensures numerical stability [S4, unofficial].
- **Quaternion Normalization**: Quaternions must be normalized to unit length before converting to rotation matrices [S4, unofficial].
- **Extreme Logits**: Clamping the target opacity to $[10^{-4}, 0.99]$ prevents extreme logit values that could cause gradient explosion [S4, unofficial].

## Subtopics

### [Gaussian Model Initialization](2349-gaussian-model-initialization.md)
Focuses on the broader model setup, including camera parameters and scene graph initialization, rather than just the Gaussian point cloud. Choose it when configuring the full training environment alongside the splat parameters.

### [LinPrim Population Control](2295-linprim-population-control.md)
Deals with the dynamic addition and removal of Gaussians during training, which occurs after the initial parameterization. Choose it when managing adaptive density control and refinement.

## Operating characteristics
- **Time Complexity**: Initialization is $O(N)$ for copying means and features. KNN distance computation is $O(N \log N)$ or $O(N^2)$ depending on implementation [S2, unofficial].
- **Memory Complexity**: Linear in $N$ for the stored parameters [S4, unofficial].
- **Strengths**: Provides a stable starting point for optimization. Pre-activation parameterization ensures valid ranges for non-linear activations [S3, unofficial].
- **Limitations**: KNN computation can be slow for very large point clouds. Fixed base distance initialization may not capture local geometry variations as well as KNN-based methods [S3, unofficial].
- **When to Use**: Always used at the start of training [S3, unofficial]. KNN-based is preferred for dense point clouds; fixed base is simpler and faster for sparse clouds or random initialization [S2, S3, unofficial].

## Background
The method originates from implementations of 3D Gaussian Splatting training pipelines [S1, S3, unofficial]. The specific parameterization using pre-activation values (logits for opacity, logs for scale) is a common technique in differentiable rendering to handle bounded parameters [S3, unofficial]. The KNN-based scale initialization is a heuristic to ensure Gaussians are sized appropriately relative to the input point cloud density [S2, unofficial].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S3] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S4] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
