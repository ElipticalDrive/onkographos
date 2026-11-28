# Distance-driven Interpolation

Also called: DD model

## Overview
Distance-driven (DD) interpolation is a geometrical discretization model for volume rendering that computes interpolation weights based on the overlap length between voxel and detector bin footprints, rather than relying on projection matrices or homographies [S1]. Unlike destination-driven models that use $\beta^1$ for both projection and backprojection, DD is neither source nor destination driven, resulting in a matched pair (up to normalization) for projection and backprojection [S1].

## How it works
**Inputs and Outputs:**
- **Input:** Voxel centers $x_j$, voxel width (implicitly 1 unit in the normalized domain), detector bin centers $u_i$, detector bin width $\Delta_i$, and the geometric mapping function $h^{-1}$ that maps detector coordinates to the common axis.
- **Output:** Interpolation weights $\xi_{\Delta_i}^{0,0}(h^{-1}(u_i) - x_j)$ representing the contribution of voxel $j$ to detector bin $i$.

**Data Structures:**
- **Voxel Grid:** Defined by centers $x_j$. In the 1D separable case, voxels are located by their edges. A voxel $j$ spans the interval $[x_j - 0.5, x_j + 0.5]$ assuming unit width [S1].
- **Detector Bins:** Defined by centers $u_i$ and widths $\Delta_i$. A bin $i$ spans the interval $[h^{-1}(u_i) - \Delta_i/2, h^{-1}(u_i) + \Delta_i/2]$ on the common axis [S1].

**Numbered Steps:**
1. **Locate Edges:** For planar parallel or fan-beam geometry (or when separability holds), identify the edges of each voxel and each detector bin on their respective axes [S1].
2. **Map to Common Axis:** Map the detector bin locations onto the same axis as the voxels using the inverse geometric mapping $h^{-1}$ [S1].
3. **Compute Overlap:** For each pair of voxel $j$ and detector bin $i$, compute the interpolation weight as the length of the overlapping segment between the voxel footprint and the mapped bin footprint [S1].

**Formula:**
The interpolation weight is given by the convolution of two order-0 B-splines (box functions):

$$
\xi_{\Delta_i}^{0,0}(h^{-1}(u_i) - x_j) = \int_{-\infty}^{+\infty} \beta_1^0(\tau) \beta_{\Delta_i}^0(h^{-1}(u_i) - x_j - \tau) d\tau = \int_{x_j - \frac{1}{2}}^{x_j + \frac{1}{2}} \beta_{\Delta_i}^0(h^{-1}(u_i) - \tau) d\tau
$$

Where:
- $\xi_{\Delta_i}^{0,0}$ is the DD kernel [S1].
- $h^{-1}(u_i)$ is the mapped detector bin center on the common axis [S1].
- $x_j$ is the voxel center [S1].
- $\beta_1^0$ is the order-0 B-spline (box function) of width 1, centered at $x_j$, defined as 1 on $[x_j - 0.5, x_j + 0.5]$ and 0 otherwise [S1].
- $\beta_{\Delta_i}^0$ is the order-0 B-spline of width $\Delta_i$, centered at $h^{-1}(u_i)$, defined as 1 on $[h^{-1}(u_i) - \Delta_i/2, h^{-1}(u_i) + \Delta_i/2]$ and 0 otherwise [S1].

The integral $\int_{x_j - \frac{1}{2}}^{x_j + \frac{1}{2}} \beta_{\Delta_i}^0(h^{-1}(u_i) - \tau) d\tau$ represents the intersection of the support of $\beta_1^0$ centered at $x_j$ and the support of $\beta_{\Delta_i}^0$ centered at $h^{-1}(u_i)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distance_driven_weight(x_j, h_inv_u_i, delta_i):
    """
    Computes the DD interpolation weight for a single voxel and detector bin pair.
    
    Args:
        x_j (float): Center of the voxel.
        h_inv_u_i (float): Mapped center of the detector bin on the common axis.
        delta_i (float): Width of the detector bin.
        
    Returns:
        float: The length of the overlapping segment between the voxel and the bin.
    """
    # Voxel footprint: [x_j - 0.5, x_j + 0.5]
    voxel_left = x_j - 0.5
    voxel_right = x_j + 0.5
    
    # Detector bin footprint on common axis: [h_inv_u_i - delta_i/2, h_inv_u_i + delta_i/2]
    bin_left = h_inv_u_i - delta_i / 2.0
    bin_right = h_inv_u_i + delta_i / 2.0
    
    # Calculate intersection interval
    overlap_left = max(voxel_left, bin_left)
    overlap_right = min(voxel_right, bin_right)
    
    # Calculate length of intersection
    overlap_length = overlap_right - overlap_left
    
    # If no overlap, weight is 0
    if overlap_length < 0:
        return 0.0
        
    return overlap_length
```

**Parameters:**
- $\Delta_i$: Detector bin width. Not given in the sources.
- $h^{-1}$: Geometric mapping function. Specific implementation depends on the system geometry (e.g., flat panel, cone beam). Not given in the sources.

**Edge Cases and Pitfalls:**
- **No Overlap:** If the voxel and bin footprints do not intersect, the weight is 0. The pseudocode handles this by returning 0.0 if `overlap_length < 0`.
- **Precision:** Floating-point precision may affect the calculation of overlap boundaries, especially for very small overlaps. [G]
- **Degenerate Input:** If $\Delta_i = 0$, the bin has no width, and the overlap length is 0 unless the bin center exactly coincides with the voxel center, in which case the overlap is a point (measure 0). [G]

## Subtopics

### [Optimized Trilinear Reconstruction](2437-optimized-trilinear-reconstruction.md)
Optimizes trilinear interpolation for hardware efficiency. Choose when hardware constraints limit computational resources.

### [Multi-Dimensional Interpolation Extension](2784-multi-dimensional-interpolation-extension.md)
Extends interpolation to higher dimensions. Choose when working with 3D or higher-dimensional data.

### [POCS Interpolation Hardware Implementation](6377-pocs-interpolation-hardware-implementation.md)
Implements interpolation using Projection Onto Convex Sets (POCS) on hardware. Choose for hardware-accelerated iterative reconstruction.

### [Seamless Interpolation](17470-seamless-interpolation.md)
Ensures seamless transitions between interpolated regions. Choose when visual continuity is critical.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:**
    - Geometrical discretization model that captures both voxel and detector bin sampling levels [S1].
    - Neither source nor destination driven, resulting in a matched pair for projection and backprojection [S1].
    - Not specific to flat panel detector and does not rely on projection matrices and homographies [S1].
- **Limitations:**
    - Requires geometric mapping $h^{-1}$ which may be complex for non-standard geometries [S1].
    - Computationally intensive if many voxel-bin pairs need to be evaluated [G].
- **When to Use:**
    - When a geometrical discretization model is preferred over matrix-based approaches [S1].
    - When a matched pair for projection and backprojection is required [S1].
- **When Not to Use:**
    - When projection matrices and homographies are already available and preferred [S1].
    - When computational efficiency is paramount and simpler interpolation methods suffice [G].

## Background
Distance-driven interpolation was proposed as an alternative to bilinear interpolation [S1]. It was introduced in the context of cone beam projection and backprojection, specifically for magnification-driven B-spline interpolation [S1]. The method is particularly relevant for flat panel cone beam geometry [S1].

## Sources
[S1] Magnification-driven B-spline Interpolation for Cone Beam Projection and Backprojection, https://doi.org/10.1002/mp.15179, fetched 2026-10-26
