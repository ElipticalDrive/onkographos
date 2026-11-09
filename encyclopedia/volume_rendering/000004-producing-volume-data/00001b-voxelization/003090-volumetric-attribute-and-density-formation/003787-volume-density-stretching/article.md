# Volume Density Stretching

Also called: Density stretching, mass-conserving density mapping

## Overview
Volume Density Stretching adjusts voxel density values based on geometric deformation to simulate mass conservation, ensuring that density decreases when "particles" are stretched over a greater distance [S1, unofficial]. Unlike standard voxelization which maps primitive attributes directly to grid cells regardless of local expansion or compression, this method scales the stored density inversely to the local volume expansion factor [S1, unofficial]. This sets it apart from siblings like [Interior Voxelize](995-interior-voxelize.md) or [Box-filtering Density Estimation](5987-box-filtering-density-estimation.md), which typically assume uniform or static density distributions, by allowing dynamic density modulation that preserves the visual consistency of expanding or contracting volumes [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A parametric volume definition (e.g., a spline tube or noise field), a deformation map (scaling factors or Jacobian determinant of the coordinate transformation), and a base density function $\rho_0$.
*   **Output**: A voxel grid where each cell contains a density value $\rho$ adjusted for local stretching.

**Data Structures**
*   **Voxel Grid**: A 3D array of floats. Each element stores the final density $\rho$.
*   **Deformation Field**: A 3D array of floats (or a scalar field) representing the local volume expansion factor $J$. If the transformation is defined by a Jacobian matrix $J_{mat}$, then $J = |\det(J_{mat})|$ [S1, unofficial].

**Steps**
1.  **Evaluate Base Density**: For each voxel center $x$, compute the undeformed density $\rho_0(x)$ using the source primitive (e.g., noise value, tube radius function) [S1, unofficial].
2.  **Compute Expansion Factor**: Determine the local volume expansion factor $J(x)$. For a 1D stretching along a curve, if the arc length element changes from $ds_0$ to $ds$, then $J = ds/ds_0$ [S1, unofficial]. In general, $J$ is the determinant of the Jacobian of the mapping from the reference configuration to the deformed configuration.
3.  **Apply Stretching**: Calculate the stretched density $\rho(x)$ using the mass conservation principle:
    $$ \rho(x) = \frac{\rho_0(x)}{J(x)} $$
    If $J(x) > 1$ (expansion), $\rho(x) < \rho_0(x)$. If $J(x) < 1$ (compression), $\rho(x) > \rho_0(x)$ [S1, unofficial].
4.  **Clamp and Store**: Clamp $\rho(x)$ to valid range $[0, \rho_{max}]$ and store in the voxel grid.

**Parameters**
*   $\rho_{max}$: Maximum density cap. Typical value: 1.0 or based on light absorption limits [G].
*   $J_{min}$: Minimum expansion factor to prevent division by zero. Typical value: $10^{-6}$ [G].

**Edge Cases and Pitfalls**
*   **Singularity**: If $J(x) \approx 0$ (extreme compression), density becomes infinite. Guard against this by clamping $J$ to $J_{min}$ [G].
*   **Discontinuities**: Sharp changes in deformation can cause visible artifacts in density. Smoothing $J$ may be required [G].
*   **Coordinate Mapping**: Ensure the deformation field is evaluated at the same resolution and coordinate system as the voxel grid [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_stretched_density(base_density_field, expansion_field, max_density=1.0, min_expansion=1e-6):
    """
    Computes density with mass-conserving stretching.
    
    Args:
        base_density_field: 3D array of floats, rho_0(x)
        expansion_field: 3D array of floats, J(x)
        max_density: float, upper bound for density
        min_expansion: float, lower bound for expansion factor to avoid division by zero
        
    Returns:
        3D array of floats, rho(x)
    """
    stretched_density = base_density_field.copy()
    
    # Guard against division by zero
    safe_expansion = max(expansion_field, min_expansion)
    
    # Apply mass conservation: rho = rho_0 / J
    stretched_density = base_density_field / safe_expansion
    
    # Clamp to valid range
    stretched_density = min(stretched_density, max_density)
    stretched_density = max(stretched_density, 0.0)
    
    return stretched_density
```

## Subtopics

### Solar Flare Model
An animated solar flare composed of 4 Bezier segments arranged in a circle. The circle grows and shrinks with time. The density field stretches as it grows, rotates, and moves along the ring. Perlin noise creates dust-cloud filaments. Density stretching is used to simulate the expansion of the flare [S1, unofficial].

### Galaxy Model
A galaxy inspired by Centaurus A, composed of 2 Bezier segments forming an S-shape. Brighter orange parts in the middle, darker on sides. Uses dust-cloud noise forming filaments when stretched. Density stretching is used to maintain visual consistency during deformation [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels, assuming constant-time evaluation of base density and expansion field.
*   **Memory Complexity**: $O(N)$ for storing the voxel grid and expansion field.
*   **Performance**: Depends on the complexity of the deformation field. Simple analytic deformations are fast; complex noise-based deformations may require additional sampling [S1, unofficial].
*   **Strengths**: Physically plausible density modulation for expanding/contracting volumes; prevents visual artifacts from uniform density in deforming shapes [S1, unofficial].
*   **Limitations**: Requires computation of the expansion factor (Jacobian), which may be non-trivial for complex deformations; potential for numerical instability if expansion factors are very small [G].
*   **When to use**: When simulating gases, fluids, or procedural noise fields that undergo significant deformation or expansion/compression [S1, unofficial].
*   **When not to use**: For static volumes or volumes with negligible deformation, where the overhead is unjustified [G].

## Background
Originated in the context of real-time rendering of procedural noise fields, specifically for interstellar dust clouds and solar flares, as described in "Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares" [S1, unofficial]. The method was implemented in WebGLSL on Shadertoy to demonstrate the visual impact of mass-conserving density mapping in animated volumetric scenes [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
