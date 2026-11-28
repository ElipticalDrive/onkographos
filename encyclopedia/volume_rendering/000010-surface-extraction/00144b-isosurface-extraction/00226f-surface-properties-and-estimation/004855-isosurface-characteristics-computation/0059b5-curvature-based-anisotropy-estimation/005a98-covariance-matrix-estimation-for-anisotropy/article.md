# Covariance Matrix Estimation for Anisotropy

Also called: Ambient Sphere Covariance Estimation

## Overview
This method estimates anisotropy by sampling a scalar field within an ambient sphere around each voxel to construct a 2D covariance matrix on the local tangent plane [S1]. It differs from curvature-based approaches by evaluating feature extent in both spatial and data domains, allowing it to capture anisotropy in volumetric structures rather than just sharp isosurfaces [S1].

## How it works
**Inputs:** Scalar field volume, radius $r$, interval size $\varepsilon$, number of samples $n$, boolean filter flag $f$.
**Outputs:** Symmetric 2x2 covariance matrix $C$ stored at each voxel.

**Data Structures:**
*   **Normal Vector:** Stored per voxel.
*   **Covariance Matrix $C$:** Symmetric 2x2 matrix. Stored as 3 values per voxel (quantized to 8 bits per channel) [S1].

**Steps:**
1.  **Normal Estimation:** For all voxels $x$, estimate a high-quality normal vector $N(x)$ using the Sobel operator [S1]. Store $N(x)$ at voxel $x$.
2.  **Covariance Initialization:** For all voxels $x$, sample the scalar field value $s$ at $x$. Initialize a symmetric 2x2 covariance matrix $C$ to zero [S1].
3.  **Ambient Sampling:** Loop $i$ from 1 to $n$:
    *   Draw a uniform sample point $x_i$ on a sphere of radius $r$ centered at $x$ [S1].
    *   Sample the scalar field value $s_i$ at $x_i$ [S1].
    *   If $s - \varepsilon \le s_i \le s + \varepsilon$:
        *   Project $x_i$ onto the tangent plane at $x$ using normal $N(x)$ to get 3D point $x_p$ [S1].
        *   Transform $x_p$ to a 2D point $x_2$ in the tangent plane basis defined by $N(x)$ [S1].
        *   Add $x_2$ to the covariance matrix $C$ [S1].
4.  **Storage:** Store the resulting symmetric 2x2 matrix $C$ at voxel $x$ [S1].
5.  **Filtering (Optional):** If flag $f$ is true, for all voxels $x$, read $C$, apply a Gaussian filter, and store the filtered $C$ back [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_anisotropy(volume, r, epsilon, n, filter_flag):
    # Step 1: Estimate normals
    normals = {}
    for x in volume.voxels:
        normals[x] = sobel_normal(volume, x)

    # Step 2-4: Estimate covariance matrices
    covariances = {}
    for x in volume.voxels:
        s = volume.sample(x)
        C = zero_matrix(2, 2)
        
        for i in range(n):
            # Draw uniform sample on sphere of radius r around x
            xi = uniform_sphere_sample(center=x, radius=r)
            si = volume.sample(xi)
            
            # Check if sample lies within the isosurface interval
            if s - epsilon <= si <= s + epsilon:
                # Project to tangent plane
                xp = project_to_tangent_plane(xi, normals[x])
                # Transform to 2D coordinates in tangent basis
                x2 = transform_to_2d(xp, normals[x])
                
                # Update covariance matrix
                C = add_point_to_covariance(C, x2)
        
        covariances[x] = C

    # Step 5: Optional Gaussian filtering
    if filter_flag:
        for x in volume.voxels:
            C = covariances[x]
            C = gaussian_filter(C)
            covariances[x] = C
            
    return covariances
```

**Parameters:**
*   $r$: Radius of the ambient sphere. Controls feature size in the spatial domain. Small $r$ provides local curvature info but requires filtering to avoid discrete artifacts. Large $r$ accounts for large-scale structures but may make estimation overly isotropic and requires more samples $n$ to maintain constant sample density [S1].
*   $\varepsilon$: Interval size in the data domain. Controls feature size in the scalar field. Small $\varepsilon$ samples locally for sharp isosurfaces but causes noise artifacts. Large $\varepsilon$ samples wide intervals, beneficial for volumetric structures, but may lose anisotropic details [S1].
*   $n$: Number of samples. Must be increased if $r$ is large to maintain constant sample density [S1].

**Pitfalls:**
*   Too few accepted samples (due to small $\varepsilon$ or large $r$) can result in non-uniform clusters, falsifying anisotropy estimation because spatial position influences the covariance matrix [S1].

## Operating characteristics
*   **Memory:** Requires 3 values per voxel for the gradient (normal) and 3 values per voxel for the symmetric 2x2 covariance matrix. This is more efficient than storing a symmetric 3x3 Hessian matrix (6 values) [S1].
*   **Performance:** The method involves $n$ samples per voxel. Performance scales linearly with $n$ and the number of voxels.
*   **Strengths:** Estimates anisotropy for structures with finite extent in both spatial and data domains. It can classify local isotropic areas as partially anisotropic if they are part of a larger anisotropic feature, making shading less susceptible to artifacts from strong variations or low signal-to-noise ratio [S1].
*   **Limitations:** Requires careful tuning of $r$ and $\varepsilon$. Small parameters require extensive filtering or high $n$ to avoid noise. Large parameters may lose anisotropic details [S1].

## Background
This method was introduced in "Anisotropic Ambient Volume Shading" [S1]. It was developed to provide BRDF-independent anisotropy estimation that works for both sharp isosurfaces and volumetric structures, addressing limitations of derivative-based curvature computation [S1].

## Sources
[S1] Anisotropic Ambient Volume Shading, https://doi.org/10.1109/tvcg.2015.2467963, fetched 2026-10-28
