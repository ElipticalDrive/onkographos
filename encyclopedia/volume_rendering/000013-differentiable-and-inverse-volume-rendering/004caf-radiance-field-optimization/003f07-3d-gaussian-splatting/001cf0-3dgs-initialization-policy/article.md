# 3DGS Initialization Policy

Also called: Initialization Policy, OTS Initialization

## Overview
The 3DGS Initialization Policy defines the deterministic starting configuration for the set of 3D Gaussian primitives in a scene, solving the problem of arbitrary initial placement by providing a reproducible baseline for optimization [S1]. It sets apart from heuristic or random initialization strategies by using a fixed power-law scaling for the initial opacity/OTS weight and a uniform random distribution within a bounding volume [S1].

## How it works
The method initializes a set of $N$ Gaussians within a defined spatial volume [S1]. The core of the policy is the uniform assignment of opacity (or OTS weight) derived from the total number of Gaussians, ensuring that the initial integrated opacity of the scene is consistent regardless of $N$ [S1].

**Inputs and Outputs:**
*   **Input:** Total number of Gaussians $N$; scene bounding box (typically a unit cube centered at the origin for synthetic scenes); rendering method type (`3dgs` or `ots`) [S1].
*   **Output:** A set of $N$ Gaussian primitives, each with position $\mathbf{p}$, covariance $\Sigma$, opacity $\alpha$, and other attributes (color, rotation) [S1].

**Data Structures:**
Each Gaussian primitive stores:
*   $\mathbf{p} \in \mathbb{R}^3$: Position (randomly sampled) [S1].
*   $\Sigma \in \mathbb{R}^{3 \times 3}$: Covariance matrix (initialized to identity or small random perturbation, though specific covariance initialization is not detailed in the primary source, standard 3DGS practice implies isotropic or small random scales) [G].
*   $\alpha \in [0, 1]$: Opacity (uniformly calculated) [S1].
*   Other attributes (SH coefficients, rotation quaternion) are initialized to neutral values (e.g., zero rotation, uniform color) or random values depending on the specific implementation, but the policy specifically mandates uniform opacity [S1].

**Steps:**
1.  **Define Bounding Volume:** For standard synthetic scenes (e.g., Nerf-Synthetic), define a unit cube centered at the scene origin [S1].
2.  **Sample Positions:** Generate $N$ random positions $\mathbf{p}_i$ uniformly distributed within the bounding volume for $i = 1 \dots N$ [S1].
3.  **Calculate Uniform Value:** Compute the initial opacity/OTS weight $\alpha$ for all Gaussians using a power function of $N$ [S1].
4.  **Initialize Attributes:** Assign the calculated $\alpha$ to all $N$ Gaussians [S1].

**Formula:**
The opacity $\alpha$ is given by:
$$
\alpha = 2 N^{\beta}
$$
where:
*   $N$ is the total number of Gaussians [S1].
*   $\beta = 0.35$ for standard 3DGS [S1].
*   $\beta = 0.55$ for OTS-based methods [S1].

**Parameters:**
*   $N$: Total number of Gaussians. Typical values cited are 4k, 12k, 36k, 100k, 500k, 1m, and 5m [S1].
*   Bounding Box: Unit cube (for Nerf-Synthetic) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random

def initialize_gaussians(N, bbox_min, bbox_max, method_type):
    """
    Initialize N Gaussians with uniform opacity based on the 3DGS/OTS policy.
    
    Args:
        N (int): Total number of Gaussians.
        bbox_min (list): Min coordinates of the bounding box [x, y, z].
        bbox_max (list): Max coordinates of the bounding box [x, y, z].
        method_type (str): '3dgs' or 'ots' to determine the exponent.
        
    Returns:
        dict: Dictionary containing initialized attributes.
    """
    # Step 1: Sample N random positions uniformly within the bounding box
    positions = []
    for _ in range(N):
        x = random.uniform(bbox_min[0], bbox_max[0])
        y = random.uniform(bbox_min[1], bbox_max[1])
        z = random.uniform(bbox_min[2], bbox_max[2])
        positions.append([x, y, z])
    
    # Step 2: Determine exponent beta based on the method
    beta = 0.35 if method_type == '3dgs' else 0.55
    
    # Step 3: Calculate uniform opacity/OTS weight
    alpha = 2.0 * (N ** beta)
    
    # Step 4: Initialize attributes
    # Rotations initialized to identity quaternion [G]
    rotations = [[1.0, 0.0, 0.0, 0.0] for _ in range(N)]
    # Scales initialized to small random values [G]
    scales = [[random.uniform(0.01, 0.1) for _ in range(3)] for _ in range(N)]
    # Colors initialized to neutral values [G]
    colors = [[1.0, 1.0, 1.0] for _ in range(N)]
    
    return {
        'positions': positions,
        'opacities': [alpha] * N,
        'rotations': rotations,
        'scales': scales,
        'colors': colors
    }
```

**Edge Cases and Pitfalls:**
*   **Precision:** For very large $N$ (e.g., 1m, 5m), the value $\alpha$ exceeds 1.0. Standard float32 represents the magnitude correctly, but rendering backends may require clamping or normalization depending on their opacity formulation [S1].
*   **Boundary Conditions:** The policy specifies a unit cube for Nerf-Synthetic; adapting to other bounding boxes requires adjusting the uniform sampling bounds accordingly [S1].
*   **Degenerate Input:** $N=0$ is invalid as the formula requires $N \ge 1$ [S1].

## Subtopics

### Geometric Correction via Neural Gaussians
A strategy to maximize overlap between cells and emitted neural Gaussians by dynamically aligning cell geometry with Gaussian properties [S2]. It involves three steps: (1) Dynamically defining the cell's quaternion as a weighted average of the quaternions associated with its emitted Gaussians [S2]. (2) Periodically updating cell centers using the formula $\mathbf{x}_n = \frac{\sum_{k=1}^K \boldsymbol{\mu}_{n,k}}{K}$, where $\mathbf{x}_n$ is the cell center, $K$ is the number of Gaussians in the cell, and $\boldsymbol{\mu}_{n,k}$ is the mean of the k-th Gaussian in cell n [S2]. (3) Learning the three-dimensional covariances and positions of first-layer cells in accordance with the updates to the second-layer Gaussians, leveraging the differentiability of cell scales [S2].

### Gaussian Pruning (Monocular)
A strategy that prunes randomly initialised 3D Gaussians to prevent incorrect initial geometry for other views, enhancing camera tracking performance in monocular SLAM [S3, unofficial]. On the TUM RGB-D dataset with monocular input, this pruning reduces the average ATE RMSE from 46.6 cm without pruning to 3.96 cm across the f1/desk, f2/xyz, and f3/office sequences [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Initialization is $O(N)$ for sampling positions and computing opacity [S1].
*   **Memory Complexity:** $O(N)$ to store the Gaussian attributes [S1].
*   **Performance:** The policy provides a reproducible baseline [S1]. The choice of exponent affects the initial opacity, which influences convergence speed and final quality [S1]. OTS methods require a higher exponent ($\beta=0.55$) compared to standard 3DGS ($\beta=0.35$) due to different image formation solutions [S1].
*   **Strengths:** Simple, deterministic, and reproducible [S1]. Ensures consistent initial conditions across experiments [S1].
*   **Limitations:** Does not account for scene geometry or structure [S1]. Relies on subsequent densification to adapt to the scene [S1].
*   **When to use:** Standard initialization for 3DGS and OTS-based methods, particularly for synthetic scenes like Nerf-Synthetic [S1].
*   **When not to use:** If the scene has significant structure that could be exploited for better initial placement, such as using point cloud data from SfM [S1].

## Background
The initialization policy was introduced in the context of evaluating whether 3D Gaussian Splatting requires accurate volumetric rendering [S1]. It was adopted from the original 3DGS codebase for training on Nerf-Synthetic datasets [S1]. The policy was adapted for OTS-based methods to account for their different image formation solutions, leading to the distinct exponents for opacity calculation [S1].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S2] Duplex-GS: Proxy-Guided Weighted Blending for Real-Time Order-Independent Gaussian Splatting, https://doi.org/10.1109/tcsvt.2026.3666188, fetched 2026-10-30
[S3] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
