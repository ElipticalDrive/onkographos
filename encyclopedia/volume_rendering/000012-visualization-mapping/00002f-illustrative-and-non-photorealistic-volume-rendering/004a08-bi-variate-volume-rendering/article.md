# Bi-variate Volume Rendering

Also called: LIC texture on scalar regions, Primary/Secondary data blending

## Overview
Bi-variate volume rendering is a non-photorealistic technique that visualizes vector flow fields by blending a primary scalar data set (defining opacity and geometric regions) with a secondary Line Integral Convolution (LIC) texture data set (defining color and flow direction) [S1, unofficial]. It solves the problem of visualizing vector fields within volumetric regions where standard LIC outputs are noise-like and lack physical meaning, allowing users to interactively track scalar features while simultaneously viewing vector flow patterns [S1, unofficial]. It differs from standard volume rendering by decoupling the geometric definition (opacity) from the textural definition (color), and from surface-based LIC by rendering the texture volumetrically within scalar-defined regions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Primary Data:** A scalar volume (e.g., velocity magnitude, pressure, or vorticity) with an associated opacity map [S1, unofficial].
*   **Secondary Data:** A 3D LIC texture volume derived from the vector field [S1, unofficial].
*   **Output:** A final 2D image where scalar geometry and vector texture are blended [S1, unofficial].

**Data Structures:**
*   **Primary Volume:** 3D grid of scalar values defining opacity via a transfer function [S1, unofficial].
*   **Secondary Volume:** 3D grid storing LIC-derived color values [S1, unofficial].

**Parameters:**
*   $\beta \in [0, 1]$: User-specified blending weight between primary and secondary data [S1, unofficial]. $\beta=1$ renders only the scalar volume; $\beta=0$ renders only the LIC volume [S1, unofficial].

**Algorithm Steps:**
1.  **Voxel Color Mixture:** For each sample point $V_i$ along a ray, compute the mixed color $C_{v_i}$ using the blending weight $\beta$:
    $$ C_{v_i} = \beta C_{v_i(p)} + (1 - \beta) C_{v_i(s)} $$
    where $C_{v_i(p)}$ is the color contribution from the primary data at $V_i$, and $C_{v_i(s)}$ is the color contribution from the secondary LIC data at $V_i$ [S1, unofficial].

2.  **Light Attenuation:** Compute the light attenuation factor $W(i)$ for the $i$-th sample point, which determines how much light reaches the current sample based on the opacity of preceding samples [S1, unofficial]. The opacity $\alpha(V_i)$ is defined by the primary data set's opacity map and the data value at $V_i$ [S1, unofficial]:
    $$ W(i) = \alpha(V_i) \left[ 1 - \sum_{j=0}^{i-1} W(j) \right] $$
    where the summation runs over all preceding sample indices $j$ [S1, unofficial].

3.  **Front-to-Back Compositing:** Render the final pixel color $C_p$ using standard front-to-back compositing:
    $$ C_p = \sum_{i=1}^{N-1} W(i) C_{v_i} $$
    where $N$ is the total number of sample points along the ray [S1, unofficial].

4.  **Decoupled Rendering:** By substituting the mixture formula into the compositing equation, the rendering process separates into two independent passes due to linearity:
    $$ C_p = \beta \sum_{i=1}^{N-1} W(i) C_{v_i(p)} + (1 - \beta) \sum_{i=1}^{N-1} W(i) C_{v_i(s)} $$
    This allows the primary scalar volume and secondary LIC volume to be rendered separately into intermediate buffers, then blended in 2D space using $\beta$ [S1, unofficial].

**Pseudocode:**
The following pseudocode implements the single-pass mixture approach, which directly follows Equations 8 and 9 and avoids the indexing/step-size bugs present in earlier drafts [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_bi_variate(primary_grid, secondary_grid, beta, ray_origin, ray_direction, step_size, num_samples, bounds_min, bounds_max):
    """
    Renders a bi-variate volume using single-pass front-to-back compositing.
    Implements Equation 9 combined with Equation 8 from the source [S1, unofficial].
    """
    final_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    
    for i in range(num_samples):
        # Calculate sample position with correct step size and midpoint offset
        pos = ray_origin + (i + 0.5) * step_size * ray_direction
        
        # Check volume bounds to prevent out-of-bounds sampling
        if not (bounds_min[0] <= pos[0] <= bounds_max[0] and
                bounds_min[1] <= pos[1] <= bounds_max[1] and
                bounds_min[2] <= pos[2] <= bounds_max[2]):
            continue
            
        # Sample primary data for opacity
        alpha_p = primary_grid.sample_opacity(pos)
        
        # Compute transmittance (1 - sum of previous W(j))
        transmittance = 1.0 - accumulated_opacity
        
        # Compute light attenuation factor W(i)
        W_i = alpha_p * transmittance
        
        # Early exit if ray is fully opaque
        if transmittance < 1e-6:
            break
            
        # Sample colors from both volumes
        color_p = primary_grid.sample_color(pos)
        color_s = secondary_grid.sample_color(pos)
        
        # Voxel color mixture (Equation 8)
        C_v = [beta * color_p[k] + (1.0 - beta) * color_s[k] for k in range(3)]
        
        # Front-to-back compositing (Equation 9)
        for k in range(3):
            final_color[k] += W_i * C_v[k]
        
        accumulated_opacity += W_i
        
    return final_color
```

**Edge Cases and Pitfalls:**
*   **LIC Noise Interpretation:** The values in the LIC volume are primarily noise signals used to construct vector textures and do not have physical meaning, so standard isosurface extraction or volume rendering cannot be directly applied to the LIC output alone [S1, unofficial].
*   **Opacity Definition:** The opacity for the secondary LIC data is not defined by its own transfer function but is derived entirely from the primary data set's opacity map [S1, unofficial].
*   **Precision and Boundaries:** Rays starting outside the volume bounds or stepping through empty regions must be handled to avoid sampling errors [G]. The transmittance check prevents over-compositing when the ray exits the volume [G].

## Subtopics

### Dye Rendering with Opacity
An extension of bi-variate rendering where the LIC volume (secondary data) contributes to the total opacity, allowing visualization of dye propagation through the volume, not just color blending [S1, unofficial]. In this variant, the opacity at each sample point is calculated as the sum of the primary data opacity and the LIC volume (dye) opacity [S1, unofficial]. The color contribution of the LIC voxel is then computed using this combined opacity [S1, unofficial]. Additionally, the dye color can be determined by additional scalar variables to visualize flow direction and scalar distribution simultaneously [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of rays (pixels) and $R$ is the number of samples per ray [S1, unofficial]. The decoupled rendering approach allows two separate volume rendering passes followed by a constant-time 2D blend, effectively doubling the volume rendering cost but simplifying the final compositing [S1, unofficial].
*   **Memory Complexity:** Requires storage for two 3D volumes (primary scalar and secondary LIC) plus intermediate buffers if using the decoupled 2D blend approach [S1, unofficial].
*   **Strengths:** Allows interactive adjustment of the blending parameter $\beta$ to track features from one variable and visualize the counterparts of the other [S1, unofficial]. It effectively visualizes vector flow within scalar-defined regions [S1, unofficial].
*   **Limitations:** The LIC volume values lack physical meaning, so they cannot be used for geometric isolation alone without the primary data [S1, unofficial]. The method relies on the linearity of the compositing equation, which may not hold for non-linear transfer functions or lighting models [S1, unofficial].

## Background
Bi-variate volume rendering was introduced in a 1997 NASA Technical Report titled "Visualizing Vector Fields Using Line Integral Convolution and Dye Advection" [S1, unofficial]. It was developed to address the limitations of applying standard visualization techniques to 3D LIC outputs, which consist of noise signals without physical meaning [S1, unofficial]. The method enables the interactive tracking of scalar features while visualizing vector flow, a capability not directly supported by standard volume rendering or surface-based LIC [S1, unofficial].

## Sources
[S1] Visualizing Vector Fields Using Line Integral Convolution and Dye Advection, https://archive.org/details/NASA_NTRS_Archive_19970011183, fetched 2026-10-26 (unofficial)
