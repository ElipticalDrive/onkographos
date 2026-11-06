# Zonal Method

Also called: Isotropic Volume Radiosity, Method of diffuse radiosity for volumes, Multiple Scattering (Radiosity Method), Radiosity Method for Volumes, Simultaneous linear equations for g(Xi), Volume-to-volume scattering

## Overview
The Zonal Method is a technique for calculating multiple scattering in participating media by assuming isotropic scattering, where the source term $g(X)$ depends only on position $X$ [S1, unofficial][S2]. This assumption allows the extension of diffuse radiosity methods, originally developed for interreflecting surfaces, to volumetric domains [S1, unofficial][S2]. It is specifically designed for high-albedo media, such as atmospheric clouds, where single-scattering approximations fail [S2]. Unlike general integral equation solvers that handle directional phase functions, the Zonal Method simplifies the problem to solving a system of simultaneous linear equations for isotropic source terms, distinguishing it from siblings like the Discrete Ordinates Method which retain angular dependence [S1, unofficial][S2].

## How it works
The method discretizes the volume into voxels and solves for the isotropic source term $g(X_i)$ at each voxel $X_i$ by accounting for external illumination and multiple scattering bounces [S1, unofficial][S2].

### Inputs and Outputs
*   **Inputs:**
    *   A 3D volume grid of $n \times n \times n$ voxels [S1, unofficial][S2].
    *   Albedo $a(X_i)$ for each voxel, representing the fraction of intercepted flux that is scattered [S1, unofficial][S2].
    *   Extinction coefficient $\tau$ (or density) for transparency calculations [S1, unofficial][S2].
    *   External illumination field [S1, unofficial][S2].
*   **Outputs:**
    *   The source term $g(X_i)$ for each voxel [S1, unofficial][S2].
    *   A final rendered image produced via a view-dependent pass using $g(X_i)$ [S1, unofficial][S2].

### Data Structures
*   **Voxel Grid:** A 3D array representing the volume. The sources assume unit volume for simplicity, but the method applies to arbitrary voxel sizes [S1, unofficial][S2].
*   **Form Factor Matrix ($F_{ij}$):** A matrix where $F_{ij}$ represents the fraction of flux originating at voxel $X_j$ that is intercepted by voxel $X_i$ [S1, unofficial][S2]. For a cubic volume of $n$ voxels per side, there are $O(n^6)$ form factors [S1, unofficial][S2].
*   **Source Vector ($g$):** A vector of size $n^3$ storing the unknown isotropic source terms [S1, unofficial][S2].
*   **Emissivity Vector ($E$):** A vector of size $n^3$ storing the initial illumination contribution [S1, unofficial][S2].

### Formulas
The core relationship is derived from the radiative transfer equation under the assumption of isotropic scattering [S1, unofficial][S2].

1.  **Scattering Contribution:**
    The total isotropic scattering contribution $S(X_i)$ at voxel $X_i$ from all other voxels is:
    $$ S(X_i) = a(X_i) \sum_j F_{ij} g(X_j) $$
    where $a(X_i)$ is the albedo at $X_i$ and $F_{ij}$ is the form factor [S1, unofficial][S2].

2.  **Linear System for Source Term:**
    The source term $g(X_i)$ is the sum of external illumination (emissivity $E(X_i)$) and the scattered light:
    $$ g(X_i) = E(X_i) + a(X_i) \sum_j F_{ij} g(X_j) $$
    This forms a system of simultaneous linear equations for the unknowns $g(X_i)$ [S1, unofficial][S2].

3.  **Form Factor ($F_{ij}$):**
    $F_{ij}$ is theoretically a 7-dimensional integral over voxels $X_i$, $X_j$, and the rays between them [S1, unofficial][S2]. It accounts for the geometry and attenuation between voxels [S1, unofficial][S2].
    *   **Approximation:** Rushmeier approximates this 7D integral using a single 1D integral along the ray connecting the centers of voxels $X_i$ and $X_j$ [S1, unofficial][S2].
    *   **Transparency:** For each pair of points (one in $X_i$, one in $X_j$), transparency is calculated by integrating $\tau$ across intervening voxels along the ray [S1, unofficial][S2].

4.  **Final Rendering:**
    Once $g(X_i)$ is determined, the final image is computed using the standard volume rendering integral (Equation 7 in [S1, unofficial][S2]):
    $$ I(X, \omega) = I_0(X - D\omega, \omega) e^{-\int_0^D \tau(X - t\omega) dt} + \int_0^D \left[ \int_{4\pi} r(X - s\omega, \omega, \omega') I(X - s\omega, \omega') d\omega' \right] e^{-\int_0^s \tau(X - t\omega) dt} ds $$
    In the Zonal Method, the source term $g(X)$ replaces the complex scattering integral, and constant voxel values $g(X_i)$ are interpolated for smoother rendering [S1, unofficial][S2].

### Parameters
*   **Albedo ($a(X_i)$):** Must be bounded by a constant $r < 1$ for iterative convergence [S1, unofficial][S2].
*   **Emissivity ($E(X_i)$):** Includes the first bounce of external illumination, propagated as in the first pass of the Kajiya and Von Herzen method [S1, unofficial][S2].

### Steps
1.  **Discretize Volume:** Divide the participating medium into voxels $X_i$. Assume $g(X)$ is piecewise constant on these voxels [S1, unofficial][S2].
2.  **Compute Emissivity ($E$):** Propagate external illumination into the volume (e.g., using the first pass of Kajiya and Von Herzen) and store the result in $E(X_i)$ [S1, unofficial][S2].
3.  **Compute Form Factors ($F_{ij}$):**
    *   For every pair of voxels $(X_i, X_j)$, calculate the form factor $F_{ij}$ [S1, unofficial][S2].
    *   Approximate the 7D integral by integrating along the ray between voxel centers [S1, unofficial][S2].
    *   Calculate transparency along this ray by integrating the extinction coefficient $\tau$ across all intervening voxels [S1, unofficial][S2].
4.  **Solve Linear System:**
    *   Set up the system $g(X_i) = E(X_i) + a(X_i) \sum_j F_{ij} g(X_j)$ [S1, unofficial][S2].
    *   Solve for $g(X_i)$ using iterative methods (e.g., Jacobi or Gauss-Seidel) [S1, unofficial][S2].
    *   *Note:* Iterative methods compute one scattering bounce per iteration. Convergence is guaranteed in $O(1)$ iterations if albedos are bounded by $r < 1$ [S1, unofficial][S2].
5.  **Render Image:**
    *   Use the computed $g(X_i)$ values in the volume rendering integral [S1, unofficial][S2].
    *   Interpolate the constant voxel values $g(X_i)$ to produce a smooth final image [S1, unofficial][S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zonal_method(volume, extinction, albedo, external_illumination):
    """
    volume: 3D array of voxels
    extinction: 3D array of extinction coefficients tau
    albedo: 3D array of albedo values a(Xi)
    external_illumination: function or array providing initial light E0
    
    Returns:
    g: 3D array of source terms
    image: rendered image
    """
    n = volume.shape[0]
    num_voxels = n * n * n
    
    # Step 1: Compute Emissivity E(Xi)
    # Propagate external illumination as in Kajiya and Von Herzen first pass
    E = compute_first_pass_illumination(volume, extinction, external_illumination)
    
    # Step 2: Compute Form Factors Fij
    # Fij is a matrix of size num_voxels x num_voxels
    # Optimization: Only compute for pairs where visibility is possible
    F = compute_form_factors(volume, extinction)
    
    # Step 3: Solve for g(Xi)
    # System: g = E + diag(a) * F * g
    # Rearranged: (I - diag(a) * F) * g = E
    # Use iterative solver since matrix is large and sparse-ish
    
    g = E.copy()
    max_iterations = 100 # Convergence depends on albedo < 1
    for iteration in range(max_iterations):
        # Compute scattering contribution: S = diag(a) * F * g
        # Vectorized: S_i = a_i * sum_j (F_ij * g_j)
        scattering = albedo * (F @ g)
        
        # Update g
        g_new = E + scattering
        
        # Check convergence (optional, sources state O(1) if albedo < 1)
        if norm(g_new - g) < tolerance:
            break
        g = g_new
        
    # Step 4: Render Image
    # Use g in the volume rendering integral with interpolation
    image = render_volume(volume, extinction, g, camera_params)
    
    return g, image

def compute_form_factors(volume, extinction):
    """
    Computes Fij for all voxel pairs.
    Approximates 7D integral with 1D integral along ray between centers.
    """
    n = volume.shape[0]
    num_voxels = n * n * n
    F = zeros((num_voxels, num_voxels))
    
    voxel_centers = get_voxel_centers(volume)
    
    for i in range(num_voxels):
        for j in range(num_voxels):
            if i == j:
                F[i, j] = 0.0
                continue
                
            # Ray from center of j to center of i
            ray_start = voxel_centers[j]
            ray_end = voxel_centers[i]
            
            # Calculate transparency along ray
            # Integrate extinction tau across intervening voxels
            transparency = compute_ray_transparency(ray_start, ray_end, extinction)
            
            # Geometric factor (approximation of 7D integral)
            # Rushmeier's approximation involves distance and solid angle
            distance = norm(ray_end - ray_start)
            if distance == 0:
                F[i, j] = 0.0
                continue
                
            # Simplified geometric term for unit voxels
            # Actual formula involves solid angle subtended by voxel i at voxel j
            # and vice versa, modulated by transparency
            geometric_factor = compute_geometric_factor(voxel_centers[i], voxel_centers[j], volume)
            
            F[i, j] = geometric_factor * transparency
            
    return F

def compute_ray_transparency(start, end, extinction):
    """
    Integrates extinction along the ray between start and end.
    """
    # Sample points along the ray
    # Integrate tau(x) dx
    # Return exp(-integral)
    pass

def compute_geometric_factor(center_i, center_j, volume):
    """
    Computes the geometric part of the form factor.
    For unit voxels, this is related to the solid angle.
    """
    # Rushmeier's approximation uses a 1D integral
    # Exact implementation details for the 1D integral are not fully specified in sources
    # but it approximates the 7D integral over voxel volumes
    pass
```

### Edge Cases and Pitfalls
*   **Albedo Bound:** The iterative solver requires albedos $a(X_i)$ to be bounded by a constant $r < 1$. If albedo approaches 1, convergence may be slow or fail [S1, unofficial][S2].
*   **High Computational Cost:** Computing $O(n^6)$ form factors takes $O(n^7)$ time for a cubic volume of $n$ voxels per side, due to the $O(n)$ intervening voxels along each ray [S1, unofficial][S2]. This makes the method expensive for high-resolution volumes.
*   **Isotropic Assumption:** The method assumes isotropic scattering. It is not suitable for highly anisotropic phase functions without modification [S1, unofficial][S2].
*   **Boundary Conditions:** The method assumes an isolated volume or handles boundaries via the emissivity term. Surface interactions (surface-to-volume, volume-to-surface) were added by Rushmeier and Torrance, but the basic formulation focuses on volume-to-volume scattering [S1, unofficial][S2].

## Subtopics

### Hierarchical Zonal Method
An extension by Hanrahan et al. that groups interactions to reduce form factors from $O(N^2)$ to $O(N)$, where $N$ is the number of elements [S1, unofficial][S2]. Choose this for large volumes where performance is critical. [Hierarchical Zonal Method](link)

### Sobierajski Extension
Extends the method to include diffuse hemisphere scattering instead of full sphere scattering [S1, unofficial][S2]. Choose this when modeling surfaces with diffuse scattering properties within the volume context. [Sobierajski Extension](link)

## Operating characteristics
*   **Time Complexity:** Dominated by the computation of form factors, which takes $O(n^7)$ time for a cubic volume of $n$ voxels per side [S1, unofficial][S2]. The iterative solution converges in $O(1)$ iterations if albedos are bounded by $r < 1$ [S1, unofficial][S2].
*   **Memory Complexity:** Requires storing $O(n^6)$ form factors, which is significant for large $n$ [S1, unofficial][S2].
*   **Strengths:**
    *   Accurately handles multiple scattering in high-albedo media [S2].
    *   Extends well-established surface radiosity concepts to volumes [S1, unofficial][S2].
    *   Provides smooth rendering through interpolation of source terms [S1, unofficial][S2].
*   **Limitations:**
    *   Extremely expensive for high-resolution volumes due to $O(n^7)$ complexity [S1, unofficial][S2].
    *   Limited to isotropic scattering [S1, unofficial][S2].
    *   Overkill for scientific visualization applications where single scattering is sufficient [S2].
*   **When to Use:** Realistic rendering of high-albedo participating media like clouds where multiple scattering is visually significant [S2].
*   **When Not to Use:** Low-albedo media, high-resolution volumes where performance is critical, or media with strong anisotropic scattering [S1, unofficial][S2].

## Background
The Zonal Method was introduced by Rushmeier and Torrance [23] and Hottel and Sarofim [24] in the context of thermal radiation heat transport [S1, unofficial][S2]. It was adapted for computer graphics by Rushmeier to handle volume-to-volume scattering [S1, unofficial][S2]. Sobierajski [26] extended the method to include diffuse hemisphere scattering [S1, unofficial][S2]. Hanrahan et al. [27] proposed hierarchical grouping to reduce computational cost [S1, unofficial][S2]. Bhate [28] and Sobierajski [26] further extended these hierarchies to volume scattering [S1, unofficial][S2]. The method is primarily used in realistic volume rendering of atmospheric phenomena [S2].

## Sources
[S1] Optical Models for Direct Volume Rendering, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Optical models for direct volume rendering, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
