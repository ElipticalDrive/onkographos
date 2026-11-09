# Volume Density Stretching

Also called: Density stretching, mass-conserving density mapping

## Overview
Volume Density Stretching adjusts the density values of a volumetric field based on spatial deformation to simulate mass conservation [S1, unofficial]. When a volume is stretched over a greater distance, its density decreases proportionally, ensuring that thicker parts do not artificially block more light than thinner parts [S1, unofficial]. This distinguishes it from sibling methods like [Interior Voxelize](995-interior-voxelize.md) or [Box-filtering Density Estimation](5987-box-filtering-density-estimation.md), which typically assign density based on geometric occupancy or filtering without accounting for the local stretching factor of the underlying parametric deformation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A parametric volume definition (e.g., a spline tube) where a point $\mathbf{p}$ is mapped from parametric coordinates $(u, v, \theta)$ to world space. The base density function $\rho_0$ defined in the undeformed parametric space.
*   **Output:** The stretched density value $\rho$ at the sampled point.

**Data Structures:**
*   Parametric coordinates $(u, v, \theta)$ representing position along the tube axis, radial distance, and angular position.
*   Deformation Jacobian or local scaling factors derived from the mapping function.

**Method:**
1.  **Map to World Space:** Compute the world position $\mathbf{P}$ from parametric coordinates using the volume's transformation function $T(u, v, \theta)$.
2.  **Compute Local Stretching:** Determine the local scaling factor $S$ resulting from the deformation. For a tube with varying radius $R(u)$, the stretching is related to the change in cross-sectional area or arc length. If the volume expands, $S > 1$; if it contracts, $S < 1$.
3.  **Apply Mass Conservation:** Adjust the base density $\rho_0$ by the inverse of the stretching factor to conserve mass. The stretched density $\rho$ is calculated as:
    $$ \rho = \frac{\rho_0}{S} $$
    Where $\rho_0$ is the density in the reference (undeformed) configuration [S1, unofficial].
4.  **Sample Noise (Optional):** If procedural noise (e.g., Perlin noise) is used, evaluate the noise in the parametric space before or after stretching depending on the desired anisotropy. Stretching the noise field anisotropically creates filamentary structures [S1, unofficial].

**Parameters:**
*   $\rho_0$: Base density value. Typical values depend on the material (e.g., gas vs. dust).
*   $S$: Local scaling factor. Computed from the derivative of the mapping function.

**Edge Cases and Pitfalls:**
*   **Singularity:** If $S \to 0$ (extreme compression), density may become unbounded. A clamp or minimum density threshold is required [G].
*   **Precision:** In WebGL/GLSL implementations, ensure floating-point precision is sufficient to handle small density variations in stretched regions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_stretched_density(u, v, theta, base_density_func, mapping_func, noise_func):
    """
    Computes density with mass conservation stretching.
    u: parametric coordinate along axis
    v: radial coordinate
    theta: angular coordinate
    base_density_func: function returning rho_0(u, v, theta)
    mapping_func: function returning (P, scale_factor) where P is world pos and scale_factor is local stretch S
    noise_func: function returning noise value at (u, v, theta)
    """
    # 1. Get base density
    rho_0 = base_density_func(u, v, theta)
    
    # 2. Compute local stretching factor S
    # The mapping function returns the world position and the local scaling factor
    # S represents how much the volume has been stretched relative to the reference
    P, S = mapping_func(u, v, theta)
    
    # Guard against division by zero or extreme compression
    if S < 1e-6:
        S = 1e-6
        
    # 3. Apply mass conservation: rho = rho_0 / S
    rho = rho_0 / S
    
    # 4. Optional: Apply noise for filaments
    # Noise is evaluated in parametric space, then the resulting field is stretched
    noise_val = noise_func(u, v, theta)
    rho = rho * (1.0 + noise_val)
    
    return rho
```

## Subtopics

### Solar Flare Model
An animated solar flare composed of 4 Bezier segments arranged in a circle that grows and shrinks with time [S1, unofficial]. The density field stretches as the flare expands, rotates, and moves along the ring [S1, unofficial]. Perlin noise creates dust-cloud filaments that become anisotropic due to the stretching [S1, unofficial].

### Galaxy Model
A galaxy inspired by Centaurus A, composed of 2 Bezier segments forming an S-shape [S1, unofficial]. It features brighter orange parts in the middle and darker sides [S1, unofficial]. Uses dust-cloud noise that forms filaments when stretched, with density stretching applied to maintain mass conservation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, assuming the mapping function and noise evaluation are constant time.
*   **Memory Complexity:** $O(1)$, as it is a procedural method requiring no voxel grid storage.
*   **Performance:** Implemented in WebGL/GLSL on Shadertoy, allowing real-time rendering [S1, unofficial]. Performance is limited by the complexity of the Bezier curve evaluation and noise function.
*   **Strengths:** Physically plausible density distribution for deforming volumes; creates realistic filamentary structures when combined with noise [S1, unofficial].
*   **Limitations:** Requires a parametric definition of the volume; not directly applicable to static voxel grids without inverse mapping [G].

## Background
Originated in the context of real-time procedural rendering of interstellar dust clouds and solar flares [S1, unofficial]. The method was implemented in WebGL/GLSL for Shadertoy, leveraging the ability to compute parametric mappings and noise on the GPU [S1, unofficial]. It addresses the visual artifact where undeformed density values cause thicker parts of a stretched volume to appear unnaturally opaque [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
