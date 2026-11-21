# Gaussian Sampling for Surface Patch Anchors

Also called: Anchor sampling with Gaussian distribution, Pulled Queries, Surface Patch Generation, Surface Patch Sensing via Gaussian Sampling

## Overview
This method creates a denser, more continuous surface representation than single-point ray intersection by pulling randomly sampled queries around 3D anchor points derived from back-projected geometry cues [S1]. It solves the problem of sparse geometric supervision by defining an isotropic Gaussian distribution around each anchor to sample a local surface patch, which is significantly more geometry-aware than single-point intersection or simple depth patch back-projection [S1]. Unlike its siblings, it explicitly models the local surface neighborhood via statistical sampling rather than uniform grid or mesh-based distribution, enabling patch-level photo consistency which achieves better performance than pixel-level consistency [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D anchor points $q$ obtained by back-projecting geometry cues (predicted monocular depth or dataset-provided sensor cues) to 3D space [S1].
*   **Parameters:**
    *   $J$: Number of points sampled per surface patch. Typical value: 9 [S1].
    *   $\tau^2$: Variance of the isotropic Gaussian distribution controlling patch size. Baseline: distance between two other pixels in the 3D world coordinate [S1].
*   **Output:** A set of $J$ points $s = \{p'_j \mid j \in [1, J]\}$ forming the surface patch around anchor $q$ [S1].

**Data Structures:**
*   **Anchor Point $q$:** A 3D vector $[x, y, z]$ in world coordinates [S1].
*   **Surface Patch $s$:** A list or array of $J$ 3D vectors $p'_j$, each representing a sampled query point [S1].

**Steps:**
1.  **Obtain Anchors:** For each posed view, back-project geometry cues to obtain 3D anchor points $q$ [S1].
2.  **Define Distribution:** Define an isotropic Gaussian distribution $\mathcal{N}(q, \tau^2)$ centered at $q$ with variance $\tau^2$ [S1].
3.  **Sample Points:** Sample $J$ points from this distribution to form the surface patch $s$ [S1].
4.  **Initialize Variance:** Set initial $\tau^2$ to the distance between two other pixels in the 3D world coordinate [S1].
5.  **Refine (Optional):** Apply mask and weight filtering to the sampled points to exclude outliers and prioritize critical surface locations based on gradient alignment to ground truth normals [S1].

**Formulas:**
*   **Gaussian Sampling:**
    $$p'_j \sim \mathcal{N}(q, \tau^2)$$
    Where $p'_j$ is the $j$-th sampled point, $q$ is the anchor point, and $\tau^2$ is the variance [S1].
*   **Variance Initialization:**
    $$\tau^2 = \text{distance}(p_a, p_b)$$
    Where $p_a$ and $p_b$ are two other pixels in the 3D world coordinate [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_surface_patch(anchor_q, variance_tau2, num_points_J):
    """
    Generates a surface patch by sampling points from a Gaussian distribution
    around a 3D anchor point.

    Args:
        anchor_q (np.ndarray): 3D anchor point [x, y, z].
        variance_tau2 (float): Variance of the isotropic Gaussian distribution.
        num_points_J (int): Number of points to sample (e.g., 9).

    Returns:
        np.ndarray: Array of shape (J, 3) containing the sampled surface patch points.
    """
    # Define the isotropic Gaussian distribution parameters
    # Mean is the anchor point, covariance is identity scaled by variance
    mean = anchor_q
    cov = np.eye(3) * variance_tau2
    
    # Sample J points from the multivariate normal distribution
    # This covers the local surface patch around the anchor
    surface_patch = np.random.multivariate_normal(mean, cov, num_points_J)
    
    return surface_patch

def initialize_variance(world_coords):
    """
    Initializes the variance tau^2 as the distance between two other pixels
    in the 3D world coordinate.

    Args:
        world_coords (np.ndarray): Array of 3D coordinates of pixels.

    Returns:
        float: The squared distance between two arbitrary pixels.
    """
    # Select two arbitrary pixels (e.g., first two)
    p1 = world_coords[0]
    p2 = world_coords[1]
    
    # Calculate Euclidean distance
    dist = np.linalg.norm(p1 - p2)
    
    # Return squared distance as variance
    return dist ** 2
```

**Edge Cases and Pitfalls:**
*   **Variance Too Small:** If $\tau^2$ is too small, the patch does not cover a large enough area, degrading inference performance [S1].
*   **Variance Too Large:** If $\tau^2$ is too large, points may be ruled out by masks, decreasing efficiency and performance [S1].
*   **Point Count:** A small number of points (e.g., $J=1$) may not adequately represent the surface, while a too large number may not improve performance further but increases time complexity [S1].

## Subtopics

### [Mask and Weight Filtering for Patch Points](16626-mask-and-weight-filtering-for-patch-points.md)
Refines geometric constraints by filtering outliers and prioritizing critical surface locations using binary masks and scalar weights based on gradient alignment.

## Operating characteristics
*   **Time Complexity:** Increases with the number of sampled points $J$. A larger $J$ increases computational cost without necessarily improving performance [S1].
*   **Memory Complexity:** Proportional to $J$ per anchor, as $J$ points must be stored and processed for loss calculation [S1].
*   **Measured Performance:** On the Replica dataset, optimal sampling at $J=9, \tau^2=\tau^2$ yields Acc=0.036, Comp=0.039, CD-L1=0.037, Prec=0.820, Recall=0.777, F-score=0.797 [S1].
*   **Strengths:** Provides a denser and more continuous surface representation than single-point intersection; more geometry-aware; patch-level photo consistency achieves better performance than pixel-level [S1].
*   **Limitations:** Sensitive to the choice of variance $\tau^2$; requires back-projected geometry cues (monocular or sensor) [S1].
*   **When to Use:** When a denser surface representation is needed for volume rendering constraints and geometry-aware supervision is beneficial.
*   **When Not to Use:** When computational resources are limited and $J$ cannot be kept small, or when high-quality back-projected geometry cues are unavailable.

## Background
This method was introduced in the paper "Sensing Surface Patches in Volume Rendering for Inferring Signed Distance Functions" [S1]. It addresses the limitation of single-point ray intersection in differentiable rendering, where only one point gets supervised on the zero level set. By pulling randomly sampled queries, it leverages the characteristics of Signed Distance Fields (SDF) to move points onto the zero level set using gradients and signed distances [S1]. The method was evaluated on datasets such as Replica and ScanNet, demonstrating superiority over previous methods in real-world indoor scenes [S1].

## Sources
[S1] Sensing Surface Patches in Volume Rendering for Inferring Signed Distance Functions, https://doi.org/10.1609/aaai.v39i4.32426, fetched 2026-10-26
