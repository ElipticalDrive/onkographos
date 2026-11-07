# Zonal Method

Also called: Isotropic Volume Radiosity, Method of diffuse radiosity for volumes, Multiple Scattering (Radiosity Method), Radiosity Method for Volumes, Simultaneous linear equations for g(Xi), Volume-to-volume scattering

## Overview
The Zonal Method is a deterministic technique for computing multiple scattering in participating media by assuming isotropic scattering, where the source term depends only on position. It extends the diffuse radiosity method for interreflecting surfaces to volumes, solving for the isotropic scattering source term by setting up and solving a system of simultaneous linear equations [S1, unofficial][S2]. Unlike stochastic methods such as [Delta tracking](985-delta-tracking.md) or [Volumetric Radiance Estimators](818-volumetric-radiance-estimators.md), it provides a global solution for multiple bounces but is restricted to isotropic phase functions and suffers from high computational complexity for dense volumes [S1, unofficial][S2].

## How it works
The method discretizes the volume into voxels and assumes the source term $g(X)$ is piecewise constant within each voxel. It calculates the total isotropic scattering contribution by determining form factors between voxel pairs and solving for the unknown source terms.

### Inputs and Outputs
*   **Inputs:**
    *   A 3D volume of $n^3$ voxels.
    *   Albedo $a(X_i)$ for each voxel $X_i$, representing the fraction of intercepted flux that is scattered.
    *   Emissivity $E(X_i)$ for each voxel, containing the first bounce of external illumination propagated as in [Single Scattering Approximations](5340-single-scattering-approximations.md) (specifically Kajiya and Von Herzen).
    *   Extinction coefficient $\tau$ distribution for calculating transparency.
*   **Outputs:**
    *   The source term $g(X_i)$ for each voxel.
    *   The final rendered image, computed in a subsequent view-dependent pass using the determined source terms.

### Data Structures
*   **Voxel Grid:** A 3D array of size $n \times n \times n$. Each voxel $X_i$ stores:
    *   $a(X_i)$: Scalar albedo.
    *   $E(X_i)$: Scalar emissivity (first-bounce illumination).
    *   $g(X_i)$: Scalar source term (unknown to be solved).
*   **Form Factor Matrix:** A dense matrix $F$ of size $n^3 \times n^3$.
    *   $F_{ij}$: The form factor from voxel $X_j$ to voxel $X_i$.
    *   Note: The sources describe computing $O(n^6)$ form factors for a cubic volume, implying a full pairwise matrix [S1, unofficial][S2].

### Formulas
1.  **Scattering Contribution:** The total isotropic scattering $S(X_i)$ at voxel $X_i$ from all other voxels is:
    $$ S(X_i) = a(X_i) \sum_j F_{ij} g(X_j) $$
    where $a(X_i)$ is the albedo at $X_i$ [S1, unofficial][S2].

2.  **Linear System:** The source term $g(X_i)$ is the sum of external emissivity and scattered light:
    $$ g(X_i) = E(X_i) + a(X_i) \sum_j F_{ij} g(X_j) $$
    This forms a system of simultaneous linear equations for the unknowns $g(X_i)$ [S1, unofficial][S2].

3.  **Form Factor ($F_{ij}$):** Represents the fraction of flux originating at voxel $X_j$ intercepted by voxel $X_i$. It is theoretically a 7-dimensional integral over the voxels and rays between them. Rushmeier approximates this using a single 1D integral along the ray connecting the voxel centers [S1, unofficial][S2].
    *   For each pair of points (one in $X_i$, one in $X_j$), transparency is calculated by integrating $\tau$ across intervening voxels [S1, unofficial][S2].

4.  **Final Rendering:** Once $g(X_i)$ is determined, the image is produced using the standard volume rendering integral (Equation 7 in [S1, unofficial]), interpolating the constant voxel values $g(X_i)$ for smoother rendering [S1, unofficial][S2].

### Parameters
*   **Voxel Volume:** Assumed to be unit volume for simplicity in the derivation [S2].
*   **Albedo Bound:** For iterative convergence, albedos $a(X_i)$ must be bounded by a constant $r < 1$ [S1, unofficial][S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zonal_method(volume, n):
    """
    volume: 3D array of shape (n, n, n) containing extinction coefficients and albedos.
    n: side length of the cubic volume.
    Returns: Rendered image.
    """
    num_voxels = n ** 3
    
    # 1. Initialize Emissivity E(Xi)
    # Propagate external illumination as in the first pass of Kajiya and Von Herzen.
    # Include the first bounce of external illumination in E(Xi).
    E = compute_first_bounce_illumination(volume)
    
    # 2. Compute Form Factors Fij
    # F is a dense matrix of size num_voxels x num_voxels
    F = zeros((num_voxels, num_voxels))
    
    for i in range(num_voxels):
        for j in range(num_voxels):
            # Approximate the 7D integral using a single 1D integral 
            # along the ray between voxel centers.
            # Calculate transparency by integrating tau across intervening voxels.
            F[i, j] = compute_form_factor_1d_approx(volume, i, j)
            
    # 3. Solve for Source Terms g(Xi)
    # System: g = E + diag(a) * F * g
    # Rearranged: (I - diag(a) * F) * g = E
    
    albedo_vector = get_albedo_vector(volume)
    A = identity(num_voxels) - diag(albedo_vector) @ F
    b = E
    
    # Solve linear system. Iterative methods converge in O(1) iterations 
    # if albedos are bounded by r < 1.
    g = solve_linear_system(A, b)
    
    # 4. Final View-Dependent Pass
    # Use equation (7) from [S1, unofficial] to produce the output image.
    # Interpolate constant voxel values g(Xi) for smoother rendering.
    image = render_volume(volume, g)
    
    return image

def compute_form_factor_1d_approx(volume, idx_i, idx_j):
    """
    Approximates the form factor Fij using a 1D integral along the ray 
    between voxel centers.
    """
    center_i = voxel_center(idx_i, n)
    center_j = voxel_center(idx_j, n)
    
    # Ray from center_j to center_i
    direction = normalize(center_i - center_j)
    distance = length(center_i - center_j)
    
    # Integrate transparency along the ray
    # Transparency T = exp(-integral of tau dt)
    # The form factor involves geometric terms and this transparency.
    # Rushmeier's approximation integrates tau across intervening voxels.
    
    # Note: Exact geometric integration details for the 1D approximation 
    # are not fully specified in the provided sources beyond "single 1D integral".
    # Implementation requires integrating the geometric projection factor 
    # and the exponential extinction along the segment.
    
    # Placeholder for the specific geometric integration logic 
    # as described by Rushmeier [S1, unofficial][S2].
    # This step computes the fraction of flux from j intercepted by i.
    
    return 0.0 # Placeholder: Exact 1D integration formula not detailed in sources.
```

### Edge Cases and Pitfalls
*   **Anisotropic Scattering:** The method assumes isotropic scattering. It cannot handle directional phase functions directly [S1, unofficial][S2].
*   **High Albedo:** If albedo $a(X_i)$ is not bounded by $r < 1$, iterative solvers may not converge [S1, unofficial][S2].
*   **Complexity:** The $O(n^7)$ time complexity for form factor calculation makes it impractical for high-resolution volumes [S1, unofficial][S2].
*   **Boundary Conditions:** The method assumes an isolated volume or requires additional terms for surface interactions (surface-to-surface, surface-to-volume, volume-to-surface) as proposed by Rushmeier [S1, unofficial][S2].

## Subtopics

### Hierarchical Zonal Method
Extends the basic method by grouping elements to reduce the number of form factors from $O(N^2)$ to $O(N)$, where $N$ is the total number of elements [S1, unofficial][S2].

### Diffuse Hemisphere Scattering
A generalization by Sobierajski that allows voxels to scatter diffusely into a hemisphere instead of a full sphere [S1, unofficial][S2].

## Operating characteristics
*   **Time Complexity:** Dominated by the computation of form factors, which takes $O(n^7)$ time for a cubic volume of $n$ voxels per side [S1, unofficial][S2]. This is because there are $O(n^6)$ form factors to compute, and each requires $O(n)$ work to integrate $\tau$ across intervening voxels [S1, unofficial][S2].
*   **Memory Complexity:** $O(n^6)$ to store the dense form factor matrix [S1, unofficial][S2].
*   **Convergence:** Iterative methods converge in $O(1)$ iterations if albedos are bounded by a constant $r < 1$ [S1, unofficial][S2].
*   **Strengths:** Provides a global solution for multiple scattering; efficient for low-resolution volumes with isotropic scattering; view-independent source term calculation allows for multiple viewpoints [S1, unofficial][S2].
*   **Limitations:** Extremely high computational cost for dense volumes; restricted to isotropic scattering; requires solving a large linear system [S1, unofficial][S2].
*   **When to Use:** Suitable for low-resolution volumes with high albedo (e.g., atmospheric clouds) where multiple scattering is significant and isotropic scattering is a valid approximation [S2].
*   **When Not to Use:** High-resolution volumes; anisotropic scattering; real-time applications [S1, unofficial][S2].

## Background
The Zonal Method was introduced by Rushmeier and Torrance [23] and Hottel and Sarofim [24], extending the method of diffuse radiosity for interreflecting surfaces to volumes [S1, unofficial][S2]. It was originally developed in the field of thermal radiation heat transport [S2]. Sobierajski [26] extended the method to include diffuse hemisphere scattering, and Hanrahan et al. [27] proposed hierarchical grouping to reduce computational cost [S1, unofficial][S2].

## Sources
[S1] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Optical models for direct volume rendering (PDF) - Academia.edu, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
