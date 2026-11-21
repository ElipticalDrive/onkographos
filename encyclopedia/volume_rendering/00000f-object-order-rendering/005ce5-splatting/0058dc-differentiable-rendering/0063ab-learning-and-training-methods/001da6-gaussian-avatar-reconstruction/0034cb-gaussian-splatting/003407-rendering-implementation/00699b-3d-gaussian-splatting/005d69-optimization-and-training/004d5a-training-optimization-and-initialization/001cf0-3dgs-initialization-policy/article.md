# 3DGS Initialization Policy

Also called: Initialization Policy, OTS Initialization

## Overview
The 3DGS Initialization Policy defines the protocol for seeding the initial set of $N$ 3D Gaussian primitives before optimization begins, ensuring a stable starting point for gradient-based training. It distinguishes itself from dynamic densification strategies (like those in [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md)) by establishing the initial geometric and opacity distribution statically, often using a power-law scaling for opacity to match specific rendering assumptions.

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry (e.g., SfM point cloud or empty space), target number of Gaussians $N$.
*   **Output:** A set of $N$ Gaussians with initialized position $\mathbf{p}_i$, rotation $q_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and appearance parameters.

**Data Structures:**
Each Gaussian $i$ is stored with:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position.
*   $q_i \in \mathbb{R}^4$: Unit quaternion for rotation.
*   $\mathbf{s}_i \in \mathbb{R}^3$: Scale (often initialized isotropically or based on local density).
*   $\alpha_i \in [0, 1]$: Opacity.
*   $h_i$: Spherical harmonic coefficients for color.

**Steps:**
1.  **Position Initialization:**
    *   For synthetic scenes (e.g., Nerf-Synthetic), initialize $N$ Gaussians with random positions uniformly distributed within a unit cube centered at the scene origin [S1].
    *   For real-world scenes, positions are typically derived from the Structure-from-Motion (SfM) point cloud [S1].
2.  **Rotation and Scale Initialization:**
    *   Rotations $q_i$ are typically initialized to identity (random or uniform if specified by the variant) [S1].
    *   Scales $\mathbf{s}_i$ are initialized based on the local point density or a fixed small value [S1].
3.  **Opacity Initialization (OTS Weight):**
    *   All Gaussians are initialized to a uniform opacity value $\alpha_0$.
    *   This value is derived using a power function of the total number of Gaussians $N$ [S1]:
        $$ \alpha_0 = 2 \cdot N^{\gamma} $$
    *   The exponent $\gamma$ depends on the rendering method:
        *   $\gamma = 0.35$ for standard 3DGS [S1].
        *   $\gamma = 0.55$ for OTS-based (Order-Independent Transparency) methods [S1].
    *   Note: The result is typically clamped or interpreted as the initial weight for the opacity parameter [S1].
4.  **Appearance Initialization:**
    *   Spherical harmonic coefficients are initialized to zero (or a default color) [S1].

**Parameters:**
*   $N$: Total number of Gaussians. Typical values include 4k, 12k, 36k, 100k, 500k, 1M, 5M depending on scene complexity and experiment [S1].
*   $\gamma$: Exponent for opacity scaling (0.35 for 3DGS, 0.55 for OTS) [S1].

**Edge Cases and Pitfalls:**
*   **Incorrect Geometry Initialization:** Starting with random positions in real-world scenes without SfM guidance can lead to artifacts like "second ears" or fuzzy surfaces if not corrected early [S4].
*   **Opacity Scaling:** Using the wrong exponent $\gamma$ can lead to suboptimal convergence or incorrect alpha blending behavior [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_gaussians(N, scene_type="synthetic", use_ots=False):
    """
    Initialize N 3D Gaussians.
    
    Args:
        N (int): Total number of Gaussians.
        scene_type (str): 'synthetic' or 'real_world'.
        use_ots (bool): If True, use OTS opacity scaling (gamma=0.55), else 3DGS (gamma=0.35).
        
    Returns:
        dict: Dictionary containing initialized Gaussian parameters.
    """
    # 1. Initialize Positions
    if scene_type == "synthetic":
        # Random positions in unit cube around origin
        positions = np.random.uniform(-1, 1, size=(N, 3))
    else:
        # Use SfM point cloud (not given in sources, placeholder for external data)
        # positions = get_sfm_points(N) 
        positions = np.zeros((N, 3)) # Placeholder

    # 2. Initialize Rotations (Identity)
    rotations = np.tile([1.0, 0.0, 0.0, 0.0], (N, 1)) # w, x, y, z

    # 3. Initialize Scales (Placeholder: typically based on local density)
    scales = np.ones((N, 3)) * 0.1 # Placeholder value

    # 4. Initialize Opacity
    if use_ots:
        gamma = 0.55
    else:
        gamma = 0.35
    
    # Calculate initial opacity weight
    # Note: The source says "2 * N^0.35" and "2 * N^0.55". 
    # This is likely a scaling factor for the opacity parameter, not the raw alpha value 
    # which must be in [0,1]. The exact interpretation of "OTS weight" is not fully detailed 
    # beyond the formula [S1]. We assume this value is used to set the initial alpha.
    opacity_weight = 2.0 * (N ** gamma)
    
    # Clamp or normalize if necessary (implementation detail [G])
    # Typically, opacity is initialized to a small value or the weight itself if it represents 
    # the initial alpha. Given the formula, for N=100k, 2*100000^0.35 ≈ 2*21.5 ≈ 43, 
    # which is > 1. Thus, this is likely a scaling factor for the *logit* or a specific 
    # parameterization. Without explicit formula for alpha from weight, we store the weight [G].
    opacities = np.full(N, opacity_weight) 

    # 5. Initialize Appearance (SH Coefficients)
    # Assuming 3 bands (9 coeffs per channel, 3 channels = 27 coeffs)
    sh_coeffs = np.zeros((N, 27)) 

    return {
        "positions": positions,
        "rotations": rotations,
        "scales": scales,
        "opacities": opacities,
        "sh_coeffs": sh_coeffs
    }
```

## Subtopics

### [DART (Distance-Aware Replication on Tangent-planes)](12512-dart-distance-aware-replication-on-tangent-planes.md)
DART is a pre-processing initialization method that enriches a sparse SfM point cloud by generating additional samples on local tangent planes. It addresses sparse seeding in scene peripheries and low-texture regions by assigning disk radii that grow with the point’s distance from a robust scene center. Unlike dynamic densification, DART acts once before optimization using only initial SfM geometry.

### [Gaussian Allocation Policy](16605-gaussian-allocation-policy.md)
The Gaussian Allocation Policy (GAP) is a reinforcement learning method that determines how to distribute a fixed budget of 3D Gaussians among learned anchor tokens to maximize rendering fidelity. It differs from heuristic initialization by treating allocation as a discrete optimization problem solved via Group Sequence Policy Optimization (GSPO).

### Geometric Correction via Neural Gaussians
A strategy to maximize overlap between cells and emitted neural Gaussians by dynamically aligning cell geometry with Gaussian properties [S2]. It involves: 1. Defining the cell's quaternion as a weighted average of the quaternions of its emitted Gaussians [S2]. 2. Periodically updating cell centers using $\mathbf{x}_n = \frac{\sum_{k=1}^K \boldsymbol{\mu}_{n,k}}{K}$, where $\mathbf{x}_n$ is the cell center, $K$ is the number of Gaussians in the cell, and $\boldsymbol{\mu}_{n,k}$ is the mean of the k-th Gaussian in cell n [S2]. 3. Learning 3D covariances and positions of first-layer cells in accordance with updates to second-layer Gaussians [S2].

### Gaussian Pruning (Monocular)
Prunes randomly initialized 3D Gaussians to prevent incorrect initial geometry for other views, enhancing camera tracking performance in monocular SLAM [S3, unofficial]. On TUM RGB-D (Mono), ATE RMSE drops from 46.6 cm (w/o pruning) to 3.96 cm (Ours) on average across f1/desk, f2/xyz, f3/office [S3, unofficial].

### Inversion Initialization
An initialization strategy that starts optimization from an inversion result to prevent geometric artifacts [S4]. Incorrect geometry initialization at the start of finetuning causes artifacts like a second ear, an unrealistic forehead, and a fuzzy surface [S4].

## Operating characteristics
*   **Time Complexity:** Initialization is $O(N)$ for position and opacity assignment [S1].
*   **Memory Complexity:** $O(N)$ to store the initial Gaussian parameters [S1].
*   **Performance:** The choice of initialization affects convergence speed and final quality. Proper opacity scaling ($\gamma$) is critical for OTS methods [S1].
*   **Strengths:** Provides a consistent starting point; OTS-specific scaling improves rendering fidelity for order-independent methods [S1].
*   **Limitations:** Random initialization in synthetic scenes may not capture scene geometry well without subsequent densification [S1]. Incorrect initialization can lead to persistent artifacts [S4].

## Background
The 3DGS Initialization Policy was introduced in the context of 3D Gaussian Splatting optimization. The specific opacity scaling formula ($2 \cdot N^{0.35}$ for 3DGS and $2 \cdot N^{0.55}$ for OTS) was detailed in "Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?" [S1]. The policy aims to align the initial primitive distribution with the assumptions of the rendering pipeline.

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S2] Duplex-GS: Proxy-Guided Weighted Blending for Real-Time Order-Independent Gaussian Splatting, https://doi.org/10.1109/tcsvt.2026.3666188, fetched 2026-10-30
[S3] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S4] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
