# InnerGS Volumetric Rendering

Also called: Factorized 3D Gaussian Splatting for Inner Scenes

## Overview
InnerGS is a volumetric rendering method that models continuous 3D density using 3D Gaussians to synthesize novel 2D slices at arbitrary depths, avoiding the discrete projection artifacts of standard 3D Gaussian Splatting (3DGS) [S1, unofficial]. Unlike standard 3DGS which renders pixels via perspective projection of splats onto a camera plane, InnerGS calculates the contribution of each Gaussian directly to a pixel on a 2D slice at a specific depth $t$ along a given axis [S1, unofficial]. This approach enables seamless volumetric interpolation and explicit internal structure reconstruction, making it suitable for applications like medical imaging where internal slices are required [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of $N$ 3D Gaussians, each defined by position, covariance (geometry), opacity $\alpha_i$, and color $c_i$ (derived from [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md)). A target slice depth $t$ and pixel coordinates $(u, v)$.
*   **Outputs:** The final color $I(u, v, t)$ for the pixel on the slice at depth $t$.

**Data Structures**
The method operates on a list of $N$ Gaussians. Each Gaussian $i$ contains:
*   $\mathbf{\mu}_i$: 3D center position.
*   $\mathbf{\Sigma}_i$: 3D covariance matrix (defining shape/orientation).
*   $\alpha_i$: Scalar opacity.
*   $c_i$: Color value (e.g., RGB or SH coefficients).

**Algorithm Steps**
1.  **Candidate Selection:** Identify candidate Gaussians for the target slice. The method sorts Gaussians based on the distance between their centers $\mathbf{\mu}_i$ and the slice plane at depth $t$ [S1, unofficial]. Two methods are described for bounding box adaptation:
    *   *Method 1:* Projects a fixed cubic bounding box onto every slice.
    *   *Method 2:* Computes an adapted bounding box for each slice [S1, unofficial]. Method 2 is preferred for efficiency, reducing average bounding box area and false positives, though it introduces a small rate of false negatives [S1, unofficial].
2.  **Contribution Calculation:** For each pixel $(u, v)$ on the slice at depth $t$, calculate the scalar contribution $p_i(u, v, t)$ of each candidate Gaussian $i$. This value represents the volumetric density contribution of Gaussian $i$ at that specific 2D location and depth [S1, unofficial].
3.  **Volumetric Accumulation:** Compute the final color $I(u, v, t)$ using alpha compositing, accounting for the transmittance of preceding Gaussians [S1, unofficial].

**Formulas**
The final color is computed as [S1, unofficial]:

$$
I(u, v, t) = \sum_{i=1}^{N} p_i(u, v, t)\alpha_i c_i \prod_{j=1}^{i-1} (1 - p_j(u, v, t)\alpha_j)
$$

Where:
*   $I(u, v, t)$: Final color of pixel $(u, v)$ on the slice at depth $t$.
*   $N$: Total number of Gaussians.
*   $p_i(u, v, t)$: Contribution (density/weight) of Gaussian $i$ at pixel $(u, v)$ and depth $t$.
*   $\alpha_i$: Opacity of Gaussian $i$.
*   $c_i$: Color of Gaussian $i$.
*   The product term $\prod_{j=1}^{i-1} (1 - p_j(u, v, t)\alpha_j)$ represents the transmittance, ensuring Gaussians are composited in sorted order (typically by distance to the slice) [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_inner_slice(gaussians, slice_depth_t, pixel_u, pixel_v):
    """
    Renders a single pixel on a 2D slice at depth t using InnerGS.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
                   - center: (x, y, z)
                   - cov: 3x3 covariance matrix
                   - alpha: float opacity
                   - color: float3 color
        slice_depth_t: Float, depth along the axis perpendicular to the slice.
        pixel_u, pixel_v: Integers, pixel coordinates on the slice.
        
    Returns:
        color: Float3, the accumulated color for the pixel.
    """
    # Step 1: Filter and Sort Candidates
    # Calculate distance from each Gaussian center to the slice plane.
    # Note: The exact geometric projection of the Gaussian onto the slice 
    # to compute p_i(u,v,t) is implementation-specific to the Gaussian's 
    # covariance and the slice orientation. We assume a helper function exists.
    
    candidates = []
    for i, g in enumerate(gaussians):
        # Calculate distance from Gaussian center to the slice plane at t
        dist = distance_to_slice(g.center, slice_depth_t)
        candidates.append((dist, i))
    
    # Sort by distance (closest first) for correct alpha compositing
    candidates.sort(key=lambda x: x[0])
    
    # Step 2: Volumetric Accumulation
    transmittance = 1.0
    accumulated_color = (0.0, 0.0, 0.0)
    
    for dist, i in candidates:
        g = gaussians[i]
        
        # Calculate contribution p_i(u, v, t)
        # This function computes the Gaussian's density/weight at the specific 
        # pixel (u,v) on the slice at depth t.
        p_i = compute_gaussian_contribution(g, pixel_u, pixel_v, slice_depth_t)
        
        # If contribution is negligible, skip (optimization)
        if p_i < 1e-6:
            continue
            
        # Alpha compositing
        alpha_contribution = p_i * g.alpha
        
        # Update accumulated color
        # I = I_prev + (1 - T_prev) * alpha * color
        accumulated_color += transmittance * alpha_contribution * g.color
        
        # Update transmittance
        transmittance *= (1.0 - alpha_contribution)
        
        # Early exit if fully opaque
        if transmittance < 1e-4:
            break
            
    return accumulated_color

def compute_gaussian_contribution(gaussian, u, v, t):
    """
    Computes the scalar contribution p_i(u, v, t) of a Gaussian to a pixel 
    on a slice at depth t.
    
    Note: The exact mathematical formulation for p_i depends on how the 
    3D Gaussian is projected/intersected with the 2D slice plane. 
    Standard 3DGS uses 2D projection; InnerGS uses volumetric density 
    evaluation at the slice location.
    """
    # [G] Placeholder for the specific density evaluation function.
    # In a typical implementation, this involves evaluating the 3D Gaussian 
    # PDF at the 3D point corresponding to pixel (u,v) at depth t, 
    # potentially adjusted for the slice's normal vector.
    # For example, if the slice is the XY plane at Z=t:
    # point = (u, v, t)
    # p_i = exp(-0.5 * (point - mu)^T Sigma^-1 (point - mu))
    # This value is then normalized or scaled appropriately.
    pass
```

**Parameters**
*   **Density Threshold:** 0.01. Used during simulation analysis to decide whether a Gaussian is considered "active" at a given pixel [S1, unofficial].
*   **Initial Positions:** Uniform 3D grid of 423 points. Used as initialization for Gaussian centers in MRI reconstruction experiments [S1, unofficial].
*   **Training Iterations:** Maximum 1,000 iterations or until absolute loss decrease over 100 iterations falls below $1 \times 10^{-4}$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **False Negatives:** Method 2 (adapted bounding boxes) introduces false negatives (0.126 per pixel) compared to Method 1, meaning some high-density Gaussians might be missed [S1, unofficial].
*   **Precision:** The method relies on accurate distance calculations between Gaussian centers and the slice plane to ensure correct sorting and candidate selection [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time is reduced compared to standard projection methods. In simulation tests on a $20 \times 20 \times 20$ volume with 50 Gaussians, Method 2 (InnerGS) took 0.10s per slice compared to 0.69s for Method 1 (3D Ellipsoid Projection) [S1, unofficial].
*   **Memory/Processing:** Method 2 processes fewer candidate Gaussians per pixel (2.62 vs 16.70) and has a smaller average bounding box area (63.09 vs 209.80) [S1, unofficial].
*   **Strengths:** Enables synthesis of novel slices at arbitrary depths without relying on projection; provides continuous volumetric interpolation; explicit and explainable representation suitable for medical imaging [S1, unofficial].
*   **Limitations:** Introduces a small rate of false negatives in candidate selection; requires careful handling of the contribution function $p_i(u, v, t)$ which differs from standard 2D rasterization [S1, unofficial].
*   **Hardware:** Experiments conducted on NVIDIA RTX 4070 GPU [S1, unofficial].

## Background
InnerGS was introduced to address the limitation of standard 3DGS, which fails to capture continuous volumetric information between discrete slices, leading to incomplete internal representations [S1, unofficial]. It combines the continuous nature of NeRF-like interpolation with the explicit efficiency of discrete Gaussian representations [S1, unofficial]. The method was specifically designed for internal scene reconstruction and segmentation, with applications in medical imaging such as Brain MRI reconstruction [S1, unofficial].

## Sources
[S1] InnerGS: Internal Scenes Reconstruction and Segmentation via Factorized 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2508.13287, fetched 2026-10-26 (unofficial)
