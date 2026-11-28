# Anisotropic Filtering of Volume Data for LPV

Also called: Central Differencing for LPV Light Leaks, Radiance Gradient Dampening

## Overview
Anisotropic filtering of volume data for Light Propagation Volumes (LPV) is a post-processing mitigation technique applied during the LPV lighting stage to reduce light bleeding (leaks) through thin walls or gaps [S1, unofficial]. It detects if radiance is flowing away from the surface normal by computing a directional derivative of the SH coefficients and dampens the sampled radiance if a leak is identified [S1, unofficial]. Unlike sibling methods such as Octree LPV which alter the data structure to improve spatial coherence, this method operates as a heuristic filter on the existing grid data to suppress artifacts caused by the limited angular resolution of the SH representation [S1, unofficial].

## How it works
The method mitigates light leaks by comparing the directional derivative of radiance along the surface normal against the actual radiance distribution. If the derivative indicates that light intensity drops sharply when moving into the wall (opposite the normal), the radiance is dampened.

### Inputs and Outputs
*   **Input:** Surface position $x$, surface normal $n$, LPV grid data containing Spherical Harmonic (SH) coefficients, cell size $h$.
*   **Output:** Dampened radiance value $L_{out}$.

### Data Structures
The LPV grid stores SH coefficients. For standard LPV implementations using 2-band SH (as noted in sources), each cell stores coefficients for the DC term and the first-order directional terms. The source notes that LPV typically uses 3 separate SH sets for RGB color channels [S1, unofficial].

### Steps
1.  **Evaluate Radiance at Current Position:**
    Evaluate the SH coefficients at the current surface location $x$ using the surface normal $n$ to obtain a scalar radiance value $c(x)$. This represents the projected radiance in the direction of the normal [S1, unofficial].

2.  **Compute Radiance Gradient:**
    Compute the radiance gradient (directional derivative) along the surface normal $n$ using a central difference scheme. This involves sampling the SH coefficients at positions offset from the current cell center by half the cell size ($h/2$) in the direction of $n$ and $-n$. Let $c(x + n/2)$ and $c(x - n/2)$ be the scalar radiance values evaluated at these offset positions [S1, unofficial].

3.  **Detect Leak Direction:**
    Compare the computed gradient with the actual radiance direction. If the differential implies that light flow is going away from the current normal (i.e., the radiance is diminishing in the direction opposed to the normal, indicating a leak into the wall), apply a dampening factor [S1, unofficial]. Specifically, if moving into the wall ($-n$) shows a slower decay or different behavior than moving out ($+n$), it suggests the light is propagating through the air on the other side rather than being blocked [S1, unofficial].

4.  **Apply Dampening:**
    Down-tone the sampled radiance using an empirical factor based on the differential. This reduces the intensity of the leaked light [S1, unofficial].

5.  **Mitigate Hue Shifts:**
    To prevent hue shifts (e.g., pink or green artifacts) caused by the central difference yielding different results for individual RGB color channels, evaluate the central difference dampener using greyscale lightness instead of individual color channels. Apply a common reduction factor to all color channels [S1, unofficial].

### Formulas
The gradient $G$ is computed as:
$$ G = c(x + \frac{n}{2}) - c(x - \frac{n}{2}) $$
The sign of this gradient indicates if radiance increases or decreases in the direction of the normal $n$ [S1, unofficial].

The dampened radiance $L_{out}$ for a color channel can be expressed as:
$$ L_{out} = L_{in} \times \max(0, 1 - \alpha \cdot G) $$
Where $\alpha$ is an empirical dampening factor. The source notes that the exact empirical factor is not specified, but the logic relies on the sign and magnitude of $G$ [S1, unofficial].

### Parameters
*   **Constant Ambient Weight:** A suggested mitigation strategy is to mix LPV output with a constant ambient term. A typical weight is 0.5 for the constant term and 0.5 for LPV [S1, unofficial].
*   **Cell Size Offset:** The offset for the central difference is half the cell size ($h/2$) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Black Holes:** Applying this technique carelessly can create 'black holes' (dark artifacts) or hue shifts (pink, green) on very dark areas containing light leaks [S1, unofficial].
*   **Hue Shifts:** Because LPV uses 3 separate SH sets for colors, the central difference tends to give different results for each channel. Evaluating the dampener per-channel without a unified luminance metric causes color distortion [S1, unofficial].
*   **DC Component Artifacts:** Without this technique, LPV with 2-band SH often produces strong DC components that cause powerful leaks. The technique mitigates this but does not eliminate it [S1, unofficial].
*   **Geometry Dependency:** LPV remains impractical for geometries not tightly controlled by an LPV-aware artist. Manual blocker walls are often required to kill leaks, similar to the geometry volume approach [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def anisotropic_filter_lpv(surface_pos, surface_normal, lpv_grid, cell_size):
    """
    Applies anisotropic filtering to LPV radiance to reduce light leaks.
    
    Args:
        surface_pos: 3D vector, position of the surface point.
        surface_normal: 3D vector, normalized surface normal.
        lpv_grid: Object providing access to SH coefficients at grid positions.
        cell_size: Float, size of the LPV voxel/cell.
        
    Returns:
        Vector3, dampened radiance.
    """
    # Step 1: Evaluate radiance at current position
    # c(x) is the radiance projected along the normal n
    c_x = evaluate_sh_radiance(lpv_grid, surface_pos, surface_normal)
    
    # Step 2: Compute radiance gradient using central difference
    # Offset by half cell size in direction of normal and opposite
    offset = surface_normal * (cell_size / 2.0)
    
    pos_forward = surface_pos + offset
    pos_backward = surface_pos - offset
    
    c_forward = evaluate_sh_radiance(lpv_grid, pos_forward, surface_normal)
    c_backward = evaluate_sh_radiance(lpv_grid, pos_backward, surface_normal)
    
    # Gradient calculation
    gradient = c_forward - c_backward
    
    # Step 3 & 4: Detect leak and apply dampening
    # If gradient implies light flow goes away from normal (leak into wall), dampen.
    # The source suggests using greyscale lightness to evaluate the dampener to avoid hue shifts.
    
    # Evaluate lightness (greyscale) for dampening logic
    # Assuming standard RGB to luminance conversion
    lightness_x = rgb_to_luminance(c_x)
    lightness_forward = rgb_to_luminance(c_forward)
    lightness_backward = rgb_to_luminance(c_backward)
    
    gradient_lightness = lightness_forward - lightness_backward
    
    # Empirical dampening factor
    # The source does not specify the exact formula for the factor, 
    # but implies a reduction based on the differential.
    # A common implementation would clamp the result to avoid negative light.
    dampening_factor = max(0.0, 1.0 - gradient_lightness)
    
    # Apply dampening to all color channels uniformly to preserve hue
    dampened_radiance = c_x * dampening_factor
    
    return dampened_radiance

def evaluate_sh_radiance(grid, pos, normal):
    """
    Evaluates the SH coefficients at pos projected along normal.
    This is a placeholder for the specific SH evaluation routine 
    used by the LPV implementation.
    """
    # Get SH coefficients at pos
    coeffs = grid.get_sh_coefficients(pos)
    # Project coeffs along normal
    return project_sh(coeffs, normal)

def rgb_to_luminance(rgb):
    """
    Converts RGB vector to greyscale luminance.
    Standard Rec. 709 weights.
    """
    return 0.2126 * rgb.x + 0.7152 * rgb.y + 0.0722 * rgb.z
```

## Subtopics

### Octree Light Propagation Volumes
Uses an octree structure instead of a uniform grid to improve spatial coherence and reduce leaks by adapting resolution to geometry. Choose when geometry is sparse or highly detailed. [Link](12437-octree-light-propagation-volumes.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per surface point, as it requires a constant number of SH evaluations (3 positions: current, forward, backward).
*   **Memory Complexity:** $O(1)$ additional memory, as it does not require storing extra data structures.
*   **Measured Performance:** The source notes that the technique works in approximately 70% of cases [S1, unofficial].
*   **Strengths:** Simple to implement; effectively reduces light bleeding through walls without modifying the core LPV data structure.
*   **Limitations:** Does not eliminate leaks entirely; prone to creating black holes or hue shifts if applied carelessly; requires tight geometric control (e.g., minimum wall thickness) to be effective.
*   **When to Use:** When using standard LPV with 2-band SH and encountering light leaks through thin walls.
*   **When Not to Use:** When high-fidelity indirect illumination is required for complex, uncontrolled geometry; consider Octree LPV or hybrid approaches with constant ambient terms instead.

## Background
The central difference technique was introduced by AMD during a demo featuring crystals flying in a tunnel [S1, unofficial]. It was developed as a heuristic to address the limitations of 2-band SH in LPV, which often produce strong DC components leading to powerful light leaks. Square Enix has been noted to use manually designed blocker walls to kill leaks, similar to the geometry volume approach, indicating industry adoption of complementary mitigation strategies [S1, unofficial].

## Sources
[S1] Light Propagation Volumes: Anisotropic filtering of volume data, https://gamedev.stackexchange.com/questions/68017, fetched 2026-10-29 (unofficial)
