# 3D Gaussian Initialization

Also called: Initial Parameter Setup

## Overview
3D Gaussian Initialization establishes the initial state of the Gaussian primitive set before optimization begins, mapping input geometric data to Gaussian centers, covariances, colors, and opacities [S1, S2]. It differs from generic model initialization by deriving scale heuristics from camera parameters and initial depth, and setting opacity based on the number of input views to ensure gradient propagation [S1]. While [Gaussian Model Initialization](2349-gaussian-model-initialization.md) covers general parameter structures, this method focuses on geometric and photometric seeding strategies for sparse or novel view synthesis contexts [S1, S2].

## How it works
**Inputs:**
- Input geometric primitives: A set of points $\{p_i\}$ with associated colors $c_i$ and initial depth values $D^{\text{init}}_i$ [S1].
- Camera parameters: Vertical focal length $f$ and image height $H$ [S1].
- View count $N$: The number of input views used for reconstruction [S1].

**Outputs:**
- A Gaussian set defined by tensors for position ($\mathbf{xyz}$), rotation quaternions ($\mathbf{rot}$), scale vectors ($\mathbf{scale}$), opacity scalar ($\alpha$), and spherical harmonic coefficients ($\mathbf{features}$) [S1].

**Data Structures:**
Each Gaussian $i$ is represented by:
- $\mathbf{xyz}_i \in \mathbb{R}^3$: Center position [G].
- $\mathbf{rot}_i \in \mathbb{R}^4$: Unit quaternion $(w, x, y, z)$ [G].
- $\mathbf{scale}_i \in \mathbb{R}^3$: Scale factors along principal axes [G].
- $\alpha_i \in \mathbb{R}$: Opacity raw parameter [S1].
- $\mathbf{features}_{dc, i} \in \mathbb{R}^3$: Zeroth-order spherical harmonic coefficients (RGB color) [S1].
- $\mathbf{features}_{rest, i}$: Higher-order SH coefficients, initialized to zero [S1].

**Steps:**

1.  **Position Mapping:**
    Assign the input point coordinates directly to the Gaussian centers [G].
    $$ \mathbf{xyz}_i = p_i $$

2.  **Rotation Initialization:**
    Initialize all rotation quaternions to the identity quaternion $(1, 0, 0, 0)$, implying an initial isotropic orientation [S1].
    $$ \mathbf{rot}_i = (1, 0, 0, 0) $$

3.  **Scale Estimation:**
    Compute the initial scale using an isometric (spherical) assumption. The radius $r$ is derived from the initial depth $D^{\text{init}}$ to ensure the Gaussian covers the corresponding pixel footprint properly [S1].
    $$ r = \frac{f \cdot D^{\text{init}}_i}{H} $$
    The scale vector is set to isotropic values:
    $$ \mathbf{scale}_i = (r, r, r) $$

4.  **Color Initialization:**
    Set the zeroth-order spherical harmonic coefficient to the input color. Higher-order coefficients are zero-initialized [S1].
    $$ \mathbf{features}_{dc, i} = c_i $$
    $$ \mathbf{features}_{rest, i} = \mathbf{0} $$

5.  **Opacity Initialization:**
    Set the initial opacity based on the number of input views $N$ to ensure gradient propagation and better alignment [S1]. The raw parameter value $\alpha_{\text{raw}}$ maps to probability $P$ via the sigmoid function. The sources specify target probabilities:
    - If $N=2$, target $P=0.6$.
    - If $N=3$, target $P=0.5$.
    - If $N=4$, target $P=0.35$.
    
    To implement this, compute the raw logit:
    $$ \alpha_{\text{raw}, i} = \text{sigmoid}^{-1}(P_N) = \ln\left(\frac{P_N}{1 - P_N}\right) $$
    [G] Standard sigmoid inverse is used. For $P=0.1$ (mentioned in parent principles as a generic low value), $\alpha_{\text{raw}} \approx -2.197$. The specific values for $N=2,3,4$ are:
    - $N=2: \alpha_{\text{raw}} \approx 0.405$
    - $N=3: \alpha_{\text{raw}} \approx 0.0$
    - $N=4: \alpha_{\text{raw}} \approx -0.847$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def initialize_gaussians(points, colors, depths, focal_length, image_height, num_views):
    """
    Initialize 3D Gaussians from point cloud data.
    
    Args:
        points: Tensor of shape (N, 3), input point coordinates.
        colors: Tensor of shape (N, 3), input RGB colors.
        depths: Tensor of shape (N,), initial depth values D_init.
        focal_length: Float, vertical focal length f.
        image_height: Int, image height H.
        num_views: Int, number of input views N.
        
    Returns:
        Dict containing initialized Gaussian parameters.
    """
    N = points.shape[0]
    
    # 1. Position
    xyz = points.clone()
    
    # 2. Rotation (Identity quaternion)
    rot = torch.zeros(N, 4, device=points.device)
    rot[:, 0] = 1.0
    
    # 3. Scale (Isometric)
    radius = (focal_length * depths) / image_height
    radius = torch.clamp(radius, min=1e-6)
    scale = torch.stack([radius, radius, radius], dim=1)
    
    # 4. Color (SH DC)
    features_dc = colors.clone()
    features_rest = torch.zeros(N, 15, device=points.device)
    
    # 5. Opacity
    if num_views == 2:
        target_prob = 0.6
    elif num_views == 3:
        target_prob = 0.5
    elif num_views == 4:
        target_prob = 0.35
    else:
        target_prob = 0.1
        
    target_prob = torch.clamp(torch.tensor(target_prob), min=1e-6, max=1-1e-6)
    opacity_raw = torch.log(target_prob / (1 - target_prob))
    opacity = torch.full((N,), opacity_raw, device=points.device)
    
    return {
        "xyz": xyz,
        "rot": rot,
        "scale": scale,
        "features_dc": features_dc,
        "features_rest": features_rest,
        "opacity": opacity
    }
```

**Edge Cases and Pitfalls:**
- **Depth Accuracy:** The scale calculation relies on $D^{\text{init}}$, and high-quality monocular depth complements the approach [S1].
- **View Count Sensitivity:** Initial opacity is set based on the number of views $N$ to ensure gradient propagation and better alignment [S1].
- **Isotropic Assumption:** Initializing as spheres $(r, r, r)$ treats Gaussians as isotropic initially, with rotation and scale optimized freely after the first 8,000 iterations of regularized optimization [S1].

## Subtopics

### Gaussian Initialization Strategies
This variant compares uniform foreground placement versus FBP (Filtered Back Projection)-image-guided placement [S2]. Uniform initialization filters air regions using a threshold and places Gaussians uniformly across the remaining foreground volume with identical covariance and intensity [S2]. FBP-image-guided initialization leverages prior information from the FBP reconstruction to place Gaussians [S2]. The FBP-guided approach consistently outperforms uniform initialization by a significant margin and achieves comparable quality to a dedicated two-stage fitting process while avoiding the extra computational overhead of the fitting stage [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Provides a physically meaningful starting point (scale relative to depth) which stabilizes training, especially in sparse view scenarios [S1]. The opacity is set based on the number of views to ensure gradient propagation and better alignment [S1].
- **Limitations:** Relies on accurate initial depth or point cloud data [S1]. Uniform initialization yields significantly lower quality than guided methods [S2].
- **When to use:** Sparse novel view synthesis or CT reconstruction where initial depth or FBP priors are available [S1, S2].
- **When not:** Dense view synthesis with abundant initial depth data where generic initialization suffices [G].

## Background
The method originates from adaptations of 3D Gaussian Splatting (3DGS) for sparse view synthesis and coherent reconstruction [S1]. Key implementations include CoherentGS, which explicitly details the isometric scale derivation and view-dependent opacity settings [S1], and 3DGR-CT, which explores FBP-guided initialization for CT reconstruction [S2]. The approach builds on the general 3DGS framework but introduces specific heuristics for initialization to handle the ill-posed nature of sparse input data [S1, S2].

## Sources
[S1] CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians, https://doi.org/10.1007/978-3-031-73404-5_2, fetched 2026-10-27
[S2] 3DGR-CT: Sparse-view CT reconstruction with a 3D Gaussian representation, https://doi.org/10.1016/j.media.2025.103585, fetched 2026-10-26
