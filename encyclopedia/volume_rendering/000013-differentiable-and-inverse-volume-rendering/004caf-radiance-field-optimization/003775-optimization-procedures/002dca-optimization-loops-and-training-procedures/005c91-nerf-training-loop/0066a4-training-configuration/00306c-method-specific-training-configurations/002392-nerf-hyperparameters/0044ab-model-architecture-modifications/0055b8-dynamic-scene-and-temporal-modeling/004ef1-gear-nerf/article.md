# Gear-NeRF

Also called: Gear-NeRF: Free-Viewpoint Rendering and Tracking with Motion-aware Spatio-Temporal Sampling

## Overview
Gear-NeRF is a dynamic scene representation method that stratifies a 4D scene into multiple "gear" levels based on semantic motion scales, allowing for region-specific spatio-temporal sampling resolution [S1]. It differs from sibling methods like [NeRF with Time Parameter](14705-nerf-with-time-parameter.md) by using semantic embeddings to drive adaptive resolution rather than uniform time-conditioning, and from [KFD-NeRF](15551-kfd-nerf.md) by avoiding explicit deformation fields in favor of semantic-aware gear assignment [S1]. The method solves the inefficiency of uniform sampling in dynamic scenes by allocating higher resolution to regions with larger motion scales [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $W$ input videos $V = \{V_1, V_2, \dots, V_W\}$ of a dynamic scene with calibrated camera poses [S1].
*   **Output:** Volume density $\sigma$ and view-dependent color $\mathbf{c}$ for a ray $\mathbf{r}(s)$ at time $t$, computed via the standard volume rendering integral [S1].

**Data Structures:**
The representation consists of a series of 4D feature volumes, one for each gear level $G \in \{1, \dots, N_{\text{gear}}\}$ [S1].
*   **Shared Spatial Features:** Functions $h_j(\cdot, \cdot)$ map 2D coordinate tuples to features. These are shared across all gears.
*   **Gear-Specific Temporal Features:** Functions $k^G_j(\cdot, \cdot)$ map 2D coordinate tuples to $M$-dimensional feature vectors. Each gear $G$ has its own set of these embeddings [S1].
*   **Linear Transforms:** Matrices $B_j$ map the element-wise products to feature vectors of dimension $M$ [S1].
*   **Semantic Embedding:** A 4D semantic embedding is optimized using features from a powerful image segmentation model (specifically SAM) to profile the scene into semantic regions [S1].

**Formulas:**
The feature vector $f^G(\mathbf{x}, t)$ for gear $G$ at space-time coordinate $(\mathbf{x}, t)$ is computed as [S1]:

$$
f^G(\mathbf{x}, t) = B_1 (h_1(x, y) \bowtie k^G_1(z, t)) + B_2 (h_2(x, z) \bowtie k^G_2(y, t)) + B_3 (h_3(y, z) \bowtie k^G_3(x, t))
$$

Where:
*   $\mathbf{x} = (x, y, z)$ are spatial coordinates.
*   $t$ is the time coordinate.
*   $h_j(\cdot, \cdot)$ and $k^G_j(\cdot, \cdot)$ are evaluated via bilinear interpolation on regularly spaced 2D feature grids [S1].
*   $\bowtie$ denotes element-wise product.
*   $B_j$ denotes a linear transform [S1].

The gear level $g(\mathbf{x}, t)$ at any spatio-temporal coordinate is computed from a planar-factorized 4D feature volume [S1]:

$$
g(\mathbf{x}, t) = \mathbf{1}^\top (h'_1(x, y) \bowtie k'_1(z, t) + \dots)
$$

(Note: The source provides the start of this formula but truncates the remaining terms. The structure implies a similar planar factorization to the feature vector, summed and projected to a scalar score [S1].)

**Parameters:**
*   $N_{\text{gear}}$: The total number of gear levels used to stratify the scene representation based on motion scales [S1].
*   $M$: The dimension of the feature space for the temporal embeddings $k^G_j$ [S1].

**Steps:**
1.  **Semantic Embedding Optimization:** Optimize a 4D semantic embedding for the scene using features extracted from a powerful image segmentation model (e.g., SAM) [S1].
2.  **Gear Assignment:** Determine the gear level $g(\mathbf{x}, t)$ for each space-time coordinate based on the semantic embedding and motion scale [S1]. Regions with larger motion scales are assigned higher gear levels [S1].
3.  **Feature Computation:** For a given coordinate $(\mathbf{x}, t)$ and assigned gear $G$, compute the feature vector $f^G(\mathbf{x}, t)$ using the formula above [S1].
4.  **Density and Color Prediction:** Pass the feature vector $f^G(\mathbf{x}, t)$ through a tiny MLP to predict volume density $\sigma$ and view-dependent color $\mathbf{c}$ given the viewing direction $\mathbf{d}$ [S1].
5.  **Spatio-Temporal Sampling:** Perform spatio-temporal sampling for ray integration. Higher gear levels (larger motion) receive higher-resolution sampling [S1].
6.  **Rendering:** Compute the final color $C(\mathbf{r}, t)$ using the volume rendering integral [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gear_nerf_forward(xyz, t, direction, gear_params, shared_params):
    """
    xyz: (N, 3) spatial coordinates
    t: (N,) time coordinates
    direction: (N, 3) viewing directions
    gear_params: Dictionary containing k_G_j grids for each gear G
    shared_params: Dictionary containing h_j grids and B_j matrices
    """
    # 1. Determine gear level for each point
    # Note: The exact formula for g(x,t) is truncated in sources.
    # Implementation requires the full planar factorization sum.
    gear_level = compute_gear_level(xyz, t, shared_params['h_prime'], shared_params['k_prime'])
    
    # 2. Retrieve gear-specific temporal embeddings and compute features
    # f^G(x,t) = B1(h1(x,y) * k1_G(z,t)) + B2(h2(x,z) * k2_G(y,t)) + B3(h3(y,z) * k3_G(x,t))
    
    # Extract spatial coordinates
    x, y, z = xyz[:, 0], xyz[:, 1], xyz[:, 2]
    
    # Initialize feature vector
    features = torch.zeros_like(xyz)
    
    for G in range(len(gear_params)):
        # Get gear-specific temporal grids
        k1_G = gear_params['k1'][G]
        k2_G = gear_params['k2'][G]
        k3_G = gear_params['k3'][G]
        
        # Get shared spatial grids
        h1 = shared_params['h1']
        h2 = shared_params['h2']
        h3 = shared_params['h3']
        
        # Get shared linear transforms
        B1 = shared_params['B1']
        B2 = shared_params['B2']
        B3 = shared_params['B3']
        
        # Interpolate spatial features
        # h1(x, y) -> (N, M)
        feat_h1 = interpolate_2d(h1, x, y)
        feat_h2 = interpolate_2d(h2, x, z)
        feat_h3 = interpolate_2d(h3, y, z)
        
        # Interpolate gear-specific temporal features
        # k1_G(z, t) -> (N, M)
        feat_k1 = interpolate_2d(k1_G, z, t)
        feat_k2 = interpolate_2d(k2_G, y, t)
        feat_k3 = interpolate_2d(k3_G, x, t)
        
        # Element-wise product and linear transform
        term1 = B1 @ (feat_h1 * feat_k1).T # Assuming B1 is (M, M) or similar
        term2 = B2 @ (feat_h2 * feat_k2).T
        term3 = B3 @ (feat_h3 * feat_k3).T
        
        # Accumulate features for the assigned gear
        # In practice, only the features for the specific gear G assigned to each point are used
        # This loop structure is illustrative; actual implementation would mask by gear_level
        features[G] = term1 + term2 + term3

    # Select features based on assigned gear level
    final_features = features[range(len(xyz)), gear_level]
    
    # 3. Predict density and color
    # tiny MLP maps feature vector to sigma and c
    sigma, color = predict_sigma_color(final_features, direction)
    
    return sigma, color

def compute_gear_level(xyz, t, h_prime, k_prime):
    """
    Computes the gear level g(x,t) for each point.
    Formula: g(x,t) = 1^T (h'_1(x,y) * k'_1(z,t) + ...)
    Note: The full formula is not provided in the source.
    """
    # Placeholder for the truncated formula implementation
    # Requires h_prime grids and k_prime grids
    pass

def interpolate_2d(grid, u, v):
    """
    Performs bilinear interpolation on a 2D grid.
    grid: (H, W, M)
    u, v: (N,) coordinates in [0, 1]
    Returns: (N, M)
    """
    # Standard bilinear interpolation implementation
    pass

def predict_sigma_color(features, direction):
    """
    Tiny MLP to predict density and color.
    """
    # Implementation of the tiny MLP
    pass
```

**Edge Cases and Pitfalls:**
*   **Truncated Formula:** The formula for the gear level assignment $g(\mathbf{x}, t)$ is truncated in the source material [S1]. Implementers must infer the remaining terms based on the symmetry of the feature vector formula or consult the full paper [S1].
*   **Semantic Dependency:** The method relies on a powerful image segmentation model (SAM) for semantic embedding. Errors in segmentation may propagate to incorrect gear assignments [S1].
*   **Gear Assignment Ambiguity:** The exact thresholding or soft-assignment mechanism for determining gear levels from the semantic embedding is not detailed in the provided notes [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Achieves state-of-the-art rendering and tracking performance on multiple challenging datasets [S1].
*   **Strengths:** Enables free-viewpoint tracking of objects of interest almost for free; improves photo-realistic dynamic novel view synthesis through motion-aware sampling [S1].
*   **Limitations:** Requires calibrated camera poses and input videos; relies on external semantic segmentation models [S1].
*   **When to use:** When high-quality dynamic novel view synthesis is required for scenes with varying motion scales, and free-viewpoint tracking is desired [S1].
*   **When not to use:** For static scenes or scenes with uniform motion where uniform sampling is sufficient [S1].

## Background
Gear-NeRF was introduced in the paper "Gear-NeRF: Free-Viewpoint Rendering and Tracking with Motion-aware Spatio-Temporal Sampling" [S1]. The method builds on the principles of differentiable volume rendering and radiance field optimization [S1]. It addresses the limitations of existing NeRF-based methods in handling dynamic scenes by introducing a semantic-aware stratification of the scene into motion-based "gears" [S1].

## Sources
[S1] Gear-NeRF: Free-Viewpoint Rendering and Tracking with Motion-aware Spatio-Temporal Sampling, https://arxiv.org/abs/2406.03723v1, fetched 2026-10-26
