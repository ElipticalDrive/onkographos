# Volume Density Stretching

Also called: Density stretching, mass-conserving density mapping

## Overview
Volume Density Stretching adjusts voxel density values based on geometric deformation to simulate mass conservation [S1, unofficial]. It ensures that when a volume primitive (such as a tube or cloud) is stretched over a greater distance, its density decreases proportionally, preventing visual artifacts where thicker parts incorrectly block more light than thinner parts [S1, unofficial]. Unlike standard voxelization which maps input density directly to grid cells regardless of local scale, this method dynamically scales density based on the Jacobian of the coordinate transformation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A parametric volume definition (e.g., spline tube, noise field), a world-space query point $P$, and the local geometric transformation parameters (scale, stretch factor).
*   **Output:** A scalar density value $\rho_{out}$ at the queried voxel.

**Data Structures:**
*   **Transformation Matrix/Scale Factor:** A scalar or vector representing the local expansion/contraction of the volume primitive relative to its rest state. For a tube, this is often the ratio of the current radius to the base radius, or the derivative of the spline curve length [S1, unofficial].

**Steps:**
1.  **Evaluate Base Density:** Calculate the base density $\rho_{base}$ at the deformed position using the underlying noise or attribute function (e.g., Perlin noise, SDF) [S1, unofficial].
2.  **Determine Stretch Factor:** Compute the local stretch factor $S$. For a volume element stretched by a factor $L$ in one dimension (conserving mass in 3D with incompressible flow assumption or specific artistic intent), $S$ represents the volume expansion ratio [S1, unofficial].
    *   *Note:* The source describes this as "particles stretched over a greater distance" implying an inverse relationship between length and density [S1, unofficial].
3.  **Apply Stretching:** Scale the base density by the inverse of the stretch factor.
    $$ \rho_{out} = \frac{\rho_{base}}{S} $$
    If the volume expands ($S > 1$), $\rho_{out}$ decreases. If it compresses ($S < 1$), $\rho_{out}$ increases [S1, unofficial].
4.  **Clamp/Filter:** Ensure $\rho_{out}$ remains within valid bounds (e.g., $[0, 1]$ or $[0, \infty)$) depending on the renderer's requirements [G].

**Parameters:**
*   **Stretch Factor ($S$):** Typically derived from the derivative of the parametric curve or the ratio of current to original dimensions. Not given in the sources as a fixed constant; it is dynamic [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Singularity:** If $S = 0$ (complete collapse), division by zero occurs. A small epsilon or clamping to a minimum density is required [G].
*   **Precision:** In highly stretched regions, density values may become very small, potentially falling below the rendering threshold or causing floating-point precision issues [S1, unofficial].
*   **Boundary:** The effect is most visible at the boundaries of the deformed volume where radius or thickness changes significantly [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_density_stretching(base_density, stretch_factor):
    """
    Computes density after applying volume density stretching.
    
    Args:
        base_density (float): The raw density value from the noise/SDF function.
        stretch_factor (float): The local volume expansion ratio (S). 
                                S > 1 means stretched/thinner.
                                S < 1 means compressed/thicker.
    
    Returns:
        float: The adjusted density value.
    """
    # Guard against division by zero or negative stretch factors
    if stretch_factor <= 0.0:
        stretch_factor = 1e-6
    
    # Apply mass conservation: density is inversely proportional to stretch
    stretched_density = base_density / stretch_factor
    
    # Optional: Clamp to valid range if required by the renderer
    # stretched_density = max(0.0, stretched_density)
    
    return stretched_density
```

## Subtopics

### Solar Flare Model
An animated solar flare composed of 4 Bezier segments arranged in a circle that grows and shrinks over time [S1, unofficial]. The density field stretches as the flare expands, rotates, and moves along the ring [S1, unofficial]. Perlin noise creates dust-cloud filaments that become anisotropic due to the stretching [S1, unofficial]. Density stretching is explicitly used to maintain visual consistency as the volume expands [S1, unofficial].

### Galaxy Model
A galaxy model inspired by Centaurus A, composed of 2 Bezier segments forming an S-shape with brighter orange centers and darker sides [S1, unofficial]. It uses dust-cloud noise that forms filaments when stretched [S1, unofficial]. Density stretching is applied to ensure the density distribution remains physically plausible relative to the geometric deformation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel query, assuming the stretch factor can be computed in constant time from local parameters [S1, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory per voxel, as no extra storage is required beyond the base density and local stretch parameters [S1, unofficial].
*   **Performance:** The method adds a single division operation per voxel. In the cited Shadertoy/WebGL implementation, performance was deemed sufficient for real-time rendering, though specific frame rates are not provided [S1, unofficial].
*   **Strengths:** Visually correct mass conservation; prevents "thick" parts from appearing unnaturally opaque compared to "thin" parts [S1, unofficial].
*   **Limitations:** Requires accurate computation of the local stretch factor; may require tuning for artistic control rather than strict physical accuracy [S1, unofficial].

## Background
The method was introduced in the context of real-time procedural volume rendering using Shadertoy/WebGL [S1, unofficial]. It was developed to address the visual artifact where standard density mapping fails to account for geometric deformation, specifically in animated splines and noise-based volumes [S1, unofficial]. The technique is demonstrated in the "Solar Flare" and "Galaxy" models, which are part of a broader toolkit for sculpting procedural noise [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
