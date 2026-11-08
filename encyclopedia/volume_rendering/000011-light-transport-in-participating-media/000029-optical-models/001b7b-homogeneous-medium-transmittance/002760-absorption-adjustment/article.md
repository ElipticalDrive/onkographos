# Absorption Adjustment

Also called: Opacity Adjustment

## Overview
Absorption Adjustment corrects the opacity parameter $\alpha$ used in layered smoke rendering to account for non-orthogonal viewing angles and variable plane spacing. It ensures that the visual density of smoke remains consistent regardless of the camera's orientation relative to the simulation grid or the distance between rendered slices. It differs from standard homogeneous transmittance by dynamically scaling the absorption coefficient based on geometric projection factors rather than assuming a fixed path length through a uniform medium.

## How it works
**Inputs and Outputs:**
- Input: Base absorption parameter $\alpha_0$ (derived from the medium's extinction coefficient and plane spacing), viewing direction vector $\mathbf{v}$, grid axis vector $\mathbf{a}$ (orthogonal to the viewing planes), and current inter-plane distance $d$.
- Output: Adjusted absorption parameter $\alpha_{adj}$ suitable for blending.

**Data Structures:**
- $\alpha$: A scalar float representing opacity, typically clamped to $[0.0, 1.0]$.
- $\mathbf{v}$: Normalized view direction vector.
- $\mathbf{a}$: Normalized axis vector orthogonal to the smoke planes.

**Steps:**
1. Calculate the angle $\theta$ between the viewing direction $\mathbf{v}$ and the grid axis $\mathbf{a}$. This is done using the dot product:
   $$ \cos \theta = \mathbf{v} \cdot \mathbf{a} $$
2. Determine the geometric scaling factor. When viewing along the axis ($\theta = 0$), the path length through a plane interval is $d$. When viewing at an angle, the effective path length through the same interval increases by a factor of $1 / |\cos \theta|$.
3. Adjust the absorption coefficient. The Beer-Lambert law states transmittance $T = e^{-\sigma_t L}$. Since $\alpha$ is derived from transmittance ($\alpha = 1 - T$), and the effective path length $L$ scales with $1/|\cos \theta|$, the exponent in the transmittance calculation must be scaled. Consequently, the absorption parameter $\alpha$ used for blending must be adjusted to reflect the increased optical depth.
   If $\alpha_0$ corresponds to the opacity for a path length $d$ along the axis, the adjusted opacity $\alpha_{adj}$ for a path length $d / |\cos \theta|$ is derived from:
   $$ 1 - \alpha_{adj} = (1 - \alpha_0)^{1 / |\cos \theta|} $$
   Solving for $\alpha_{adj}$:
   $$ \alpha_{adj} = 1 - (1 - \alpha_0)^{1 / |\cos \theta|} $$
   [G] Note: If $\cos \theta$ is zero (viewing parallel to the planes), the path length through the interval becomes infinite, and the medium should be treated as fully opaque for that interval, or the plane should be skipped depending on the specific implementation's handling of degenerate cases.
4. Adjust for plane skipping or variable spacing. If the distance between adjacent smoke planes changes to $d'$, or if viewing planes are skipped (effectively increasing the step size), the base absorption parameter must be scaled proportionally to the change in physical distance. If the original $\alpha_0$ was calibrated for distance $d_{ref}$, and the current step is $d_{curr}$:
   $$ \alpha_{step} = 1 - (1 - \alpha_0)^{d_{curr} / d_{ref}} $$
   This $\alpha_{step}$ is then further adjusted by the viewing angle factor from Step 3.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def adjust_absorption(alpha_0, view_dir, grid_axis, current_dist, ref_dist=1.0):
    """
    Adjusts the absorption parameter alpha for view direction and plane spacing.
    
    Args:
        alpha_0 (float): Base opacity for the reference distance along the grid axis.
        view_dir (tuple): Normalized view direction vector (vx, vy, vz).
        grid_axis (tuple): Normalized axis vector orthogonal to the planes (ax, ay, az).
        current_dist (float): Actual distance between the current and next plane.
        ref_dist (float): Reference distance used to calibrate alpha_0.
        
    Returns:
        float: Adjusted absorption parameter alpha_adj.
    """
    # Step 1: Calculate cosine of the angle between view direction and grid axis
    cos_theta = view_dir[0]*grid_axis[0] + view_dir[1]*grid_axis[1] + view_dir[2]*grid_axis[2]
    
    # Handle degenerate case where view is parallel to planes
    if abs(cos_theta) < 1e-6:
        # Path length is effectively infinite through the interval
        return 1.0
    
    # Step 2 & 3: Adjust for viewing angle
    # The optical depth scales by 1/|cos_theta|
    # Transmittance T = 1 - alpha
    # T_adj = T_0 ^ (1/|cos_theta|)
    # alpha_adj = 1 - T_adj
    
    # Clamp alpha_0 to avoid numerical issues with log/exp if it's exactly 0 or 1
    if alpha_0 <= 0.0:
        return 0.0
    if alpha_0 >= 1.0:
        return 1.0
        
    t_0 = 1.0 - alpha_0
    angle_factor = 1.0 / abs(cos_theta)
    t_adj = t_0 ** angle_factor
    alpha_angle = 1.0 - t_adj
    
    # Step 4: Adjust for distance scaling
    # If current_dist differs from ref_dist, scale the optical depth further
    if ref_dist != 0:
        dist_factor = current_dist / ref_dist
        # Apply distance scaling to the transmittance
        # T_final = T_adj ^ dist_factor
        t_final = t_adj ** dist_factor
        alpha_adj = 1.0 - t_final
    else:
        alpha_adj = alpha_angle
        
    # Clamp result to [0.0, 1.0]
    return max(0.0, min(1.0, alpha_adj))
```

**Parameters:**
- $\alpha_0$: Base absorption/opacity. Typical values depend on the simulation's extinction coefficient $\sigma_t$ and reference plane spacing.
- `ref_dist`: Reference plane spacing. Default is often the uniform grid spacing of the simulation.

**Edge Cases:**
- **Parallel View:** If $\mathbf{v} \cdot \mathbf{a} = 0$, the ray travels parallel to the planes. The path length through any single plane interval is undefined/infinite in the projection model. The method returns 1.0 (fully opaque) or requires a different handling strategy (e.g., skipping the plane).
- **Numerical Precision:** When $\alpha_0$ is very close to 0 or 1, floating-point precision in the exponentiation can lead to artifacts. Clamping or using logarithmic forms ($\ln(1-\alpha)$) may be necessary for stability.

## Subtopics

### Extinction Falloff
A shortcut that approximates the effect of hundreds of additional scattering bounces by gently reducing extinction with each real bounce. It uses a default Extinction Falloff Value of 0.2, described as a deliberate sweet spot that produces brighter, more realistic results while rendering faster than scenes without it. It is distinct from Absorption Adjustment as it modifies the scattering behavior over multiple bounces rather than correcting geometric projection for single-plane blending.

## Operating characteristics
- **Time Complexity:** The adjustment requires one dot product, one absolute value, one division, and one exponentiation per sample. The source notes that displaying smoke with corrected $\alpha$'s at 10 frames per second for a $100 \times 100 \times 100$ grid requires ten million exponential operations per second.
- **Memory Complexity:** Minimal; requires storing only the current $\alpha$ and geometric vectors.
- **Strengths:** Ensures visual consistency of smoke density across different camera angles and plane spacings.
- **Limitations:** Computationally expensive due to the need for exponential operations, though modern hardware makes this feasible.

## Background
The method originates from the visualization of fire dynamics simulation data, specifically in the Smokeview tool. It addresses the visual artifacts that arise when rendering layered smoke volumes using the Beer-Lambert law without accounting for the geometric relationship between the viewer and the simulation grid.

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S2] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
