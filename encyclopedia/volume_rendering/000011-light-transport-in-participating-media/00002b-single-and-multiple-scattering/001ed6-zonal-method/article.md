# Zonal Method

Also called: Isotropic Volume Radiosity, Method of diffuse radiosity for volumes, Multiple Scattering (Radiosity Method), Radiosity Method for Volumes, Simultaneous linear equations for g(Xi), Volume-to-volume scattering, Volumetric Radiosity

## Overview
The Zonal Method is a deterministic technique for computing multiple scattering in participating media by assuming isotropic scattering, where the source term depends only on position [S2, unofficial]. It extends the diffuse radiosity method for interreflecting surfaces to volumes, solving for the isotropic scattering source term by setting up and solving a system of simultaneous linear equations [S1][S2, unofficial]. Unlike stochastic methods such as [Delta tracking](985-delta-tracking.md) or [Volumetric Radiance Estimators](818-volumetric-radiance-estimators.md), it provides a global solution for multiple bounces but is restricted to isotropic phase functions and suffers from high computational complexity for dense volumes [S1][S2, unofficial].

## How it works
The method discretizes the volume into voxels and assumes the source term $g(X)$ is piecewise constant within each voxel. It calculates the total isotropic scattering contribution by determining form factors between voxel pairs and solving for the unknown source terms.

### Inputs and Outputs
*   **Inputs:**
    *   A 3D volume of $n^3$ voxels [S2, unofficial].
    *   Albedo $a(X_i)$ for each voxel $X_i$, representing the fraction of intercepted flux that is scattered [S2, unofficial].
    *   Emissivity $E(X_i)$ for each voxel, containing the first bounce of external illumination propagated as in [Single Scattering Approximations](5340-single-scattering-approximations.md) (specifically Kajiya and Von Herzen) [S2, unofficial].
    *   Extinction coefficient $\tau$ distribution for calculating transparency [S2, unofficial].
*   **Outputs:**
    *   The source term $g(X_i)$ for each voxel [S1].
    *   The final rendered image, computed in a subsequent view-dependent pass using the determined source terms [S2, unofficial].

### Data Structures
*   **Voxel Grid:** A 3D array of size $n \times n \times n$. Each voxel $X_i$ stores $a(X_i)$, $E(X_i)$, and $g(X_i)$ [S2, unofficial].
*   **Form Factor Matrix:** A dense matrix $F$ of size $n^3 \times n^3$, where $F_{ij}$ is the form factor from voxel $X_j$ to voxel $X_i$ [S1][S2, unofficial].

### Formulas
1.  **Scattering Contribution:** The total isotropic scattering $S(X_i)$ at voxel $X_i$ from all other voxels is:
    $$ S(X_i) = a(X_i) \sum_j F_{ij} g(X_j) $$
    where $a(X_i)$ is the albedo at $X_i$ [S2, unofficial].

2.  **Linear System:** The source term $g(X_i)$ is the sum of external emissivity and scattered light:
    $$ g(X_i) = E(X_i) + a(X_i) \sum_j F_{ij} g(X_j) $$
    This forms a system of simultaneous linear equations for the unknowns $g(X_i)$ [S2, unofficial].

3.  **Form Factor ($F_{ij}$):** Represents the fraction of flux originating at voxel $X_j$ intercepted by voxel $X_i$ [S2, unofficial]. It is theoretically a 7-dimensional integral over the voxels and rays between them [S2, unofficial]. Rushmeier approximates this using a single 1D integral along the ray connecting the voxel centers [S2, unofficial]. For each pair of points, transparency is calculated by integrating $\tau$ across intervening voxels [S2, unofficial].

4.  **Final Rendering:** Once $g(X_i)$ is determined, the image is produced using the standard volume rendering integral, interpolating the constant voxel values $g(X_i)$ for smoother rendering [S2, unofficial].

### Parameters
*   **Voxel Volume:** Assumed to be unit volume for simplicity in the derivation [S2, unofficial].
*   **Albedo Bound:** For iterative convergence, albedos $a(X_i)$ must be bounded by a constant $r < 1$ [S2, unofficial].

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
    # Extract voxel properties
    albedo = [volume[i][j][k].albedo for i in range(n) for j in range(n) for k in range(n)]
    emissivity = compute_first_bounce_illumination(volume) # [G] Standard single-scattering pass from external lights
    
    # 1. Compute Form Factors Fij
    F = [[0.0] * num_voxels for _ in range(num_voxels)]
    for i in range(num_voxels):
        for j in range(num_voxels):
            center_i = voxel_center(i, n)
            center_j = voxel_center(j, n)
            direction = normalize(center_i - center_j)
            distance = length(center_i - center_j)
            # Integrate transparency along the ray between centers
            optical_depth = integrate_tau(volume, center_j, direction, distance, n)
            transmittance = exp(-optical_depth)
            # Rushmeier's 1D approximation along voxel centers
            F[i][j] = transmittance * (1.0 / (distance ** 2))
            
    # 2. Solve for Source Terms g(Xi)
    # System: g = E + diag(a) * F * g  =>  (I - diag(a) * F) * g = E
    A = identity(num_voxels)
    for i in range(num_voxels):
        for j in range(num_voxels):
            A[i][j] -= albedo[i] * F[i][j]
    b = emissivity
    g = solve_linear_system(A, b) # [G] Standard linear solver; Gauss-Seidel iterative method also works
            
    # 3. Final View-Dependent Pass
    g_grid = reshape(g, (n, n, n))
    image = render_volume(volume, g_grid)
    return image

def compute_first_bounce_illumination(volume):
    # [G] Standard single-scattering integration from external light sources
    return [0.0] * (n ** 3)

def voxel_center(idx, n):
    i, j, k = idx // (n ** 2), (idx // n) % n, idx % n
    return [i + 0.5, j + 0.5, k + 0.5]

def normalize(v):
    mag = sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return [v[0]/mag, v[1]/mag, v[2]/mag]

def length(v):
    return sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def integrate_tau(volume, origin, direction, distance, n):
    optical_depth = 0.0
    steps = int(distance * n)
    for t in range(steps):
        pos = [origin[d] + t * direction[d] for d in range(3)]
        if 0 <= pos[0] < n and 0 <= pos[1] < n and 0 <= pos[2] < n:
            optical_depth += volume[int(pos[0])][int(pos[1])][int(pos[2])].extinction
    return optical_depth

def identity(size):
    return [[1.0 if i == j else 0.0 for j in range(size)] for i in range(size)]

def solve_linear_system(A, b):
    # [G] Standard Gaussian elimination or iterative solver
    return [0.0] * len(b)

def render_volume(volume, g_grid):
    # [G] Standard ray casting compositing source terms along view rays
    return [[0.0] * 3 for _ in range(100) for _ in range(100)]
```

### Edge Cases and Pitfalls
*   **Anisotropic Scattering:** The method assumes isotropic scattering. It cannot handle directional phase functions directly [S2, unofficial].
*   **High Albedo:** If albedo $a(X_i)$ is not bounded by $r < 1$, iterative solvers may not converge [S2, unofficial].
*   **Complexity:** The $O(n^7)$ time complexity for form factor calculation makes it impractical for high-resolution volumes [S1][S2, unofficial].
*   **Boundary Conditions:** The method assumes an isolated volume or requires additional terms for surface interactions (surface-to-surface, surface-to-volume, volume-to-surface) as proposed by Rushmeier [S2, unofficial].

## Subtopics

### Hierarchical Zonal Method
Extends the basic method by grouping elements to reduce the number of form factors from $O(N^2)$ to $O(N)$, where $N$ is the total number of elements [S1][S2, unofficial].

### Diffuse Hemisphere Scattering
A generalization by Sobierajski that allows voxels to scatter diffusely into a hemisphere instead of a full sphere [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the computation of form factors, which takes $O(n^7)$ time for a cubic volume of $n$ voxels per side [S1][S2, unofficial]. This is because there are $O(n^6)$ form factors to compute, and each requires $O(n)$ work to integrate $\tau$ across intervening voxels [S1][S2, unofficial].
*   **Memory Complexity:** $O(n^6)$ to store the dense form factor matrix [S1][S2, unofficial].
*   **Convergence:** Iterative methods converge in $O(1)$ iterations if albedos are bounded by a constant $r < 1$ [S2, unofficial].
*   **Strengths:** Provides a global solution for multiple scattering; efficient for low-resolution volumes with isotropic scattering; view-independent source term calculation allows for multiple viewpoints [S1][S2, unofficial].
*   **Limitations:** Extremely high computational cost for dense volumes; restricted to isotropic scattering; requires solving a large linear system [S1][S2, unofficial].
*   **When to Use:** Suitable for low-resolution volumes with high albedo (e.g., atmospheric clouds) where multiple scattering is significant and isotropic scattering is a valid approximation [S2, unofficial].
*   **When Not to Use:** High-resolution volumes; anisotropic scattering; real-time applications [S1][S2, unofficial].

## Background
The Zonal Method was introduced by Rushmeier and Torrance [23] and Hottel and Sarofim [24], extending the method of diffuse radiosity for interreflecting surfaces to volumes [S2, unofficial]. It was originally developed in the field of thermal radiation heat transport [S2, unofficial]. Sobierajski [26] extended the method to include diffuse hemisphere scattering, and Hanrahan et al. [27] proposed hierarchical grouping to reduce computational cost [S1][S2, unofficial].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] [PDF] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S3] Optical models for direct volume rendering (PDF) - Academia.edu, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
[S4] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
