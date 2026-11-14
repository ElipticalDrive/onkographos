# Visibility Field Computation

Also called: Visibility Contribution Calculation, Visibility Field, Voxel Visibility

## Overview
Visibility Field Computation calculates the specific contribution of each voxel to the final 2D image, accounting for its own opacity and the occlusion caused by preceding voxels along the viewing ray [S1, unofficial][S2, unofficial]. It builds on [Front-to-Back Compositing (Over Operator)](3360-front-to-back-compositing-over-operator.md) by explicitly storing these per-voxel visibility weights in a data structure, enabling analysis of which parts of the volume contribute most to the image and how tissues occlude one another [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume dataset, a transfer function mapping scalar values to color and opacity, and a camera viewpoint [S1, unofficial][S2, unofficial].
*   **Output:** A visibility field $V$, where each voxel $i$ has an associated visibility value $v_i$ [S2, unofficial].

**Data Structures**
*   **Volume Data:** A 3D grid of scalar values [G].
*   **Visibility Field:** A 3D grid of real values (floats) with the same dimensions as the volume, storing $v_i$ [S2, unofficial].
*   **Accumulated Opacity Buffer:** A per-ray temporary buffer or register holding $A_i$, the accumulated opacity up to voxel $i$ [G].

**Formulas**
The accumulated opacity $A_i$ at voxel $i$ is computed using front-to-back compositing [S2, unofficial]:
$$ A_i = (1 - A_{i-1})a_i + A_{i-1} $$
where $a_i$ is the opacity of voxel $i$, and $A_{i-1}$ is the accumulated opacity of the voxel immediately preceding $i$ along the ray [S2, unofficial]. The base case is $A_{-1} = 0$ [G].

The visibility $v_i$ of voxel $i$ is the difference between the accumulated opacity at $i$ and the accumulated opacity at the previous voxel [S2, unofficial]:
$$ v_i = A_i - A_{i-1} $$
Substituting the accumulation formula, this simplifies to:
$$ v_i = (1 - A_{i-1})a_i $$
This value represents the fraction of the final pixel color contributed by voxel $i$ [S1, unofficial]. A fully opaque, unoccluded voxel has a visibility of 1 [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_visibility_field(volume, transfer_function, viewpoint):
    # Initialize visibility field with zeros
    visibility_field = zeros_like(volume)
    # Per-ray accumulated opacity buffer
    ray_accumulated_opacity = zeros(num_rays)
    
    # Render a series of quads parallel to the viewing plane, one for each slice
    for slice_idx in range(num_slices):
        for ray_idx in range(num_rays):
            fragment = slice_quads[slice_idx][ray_idx]
            
            # Discard fragments that do not belong to the volume
            if fragment is None or not volume.contains(fragment.position):
                continue
                
            # Retrieve opacity from transfer function
            a_i = transfer_function.opacity(volume[fragment.position])
            
            A_prev = ray_accumulated_opacity[ray_idx]
            # Compute current accumulated opacity
            A_curr = (1.0 - A_prev) * a_i + A_prev
            # Clamp to [0, 1] to prevent numerical drift [G]
            A_curr = max(0.0, min(1.0, A_curr))
            
            # Calculate visibility
            v_i = A_curr - A_prev
            
            # Store visibility value at the voxel's location
            # Use atomic operations if multiple rays map to the same voxel [S1, unofficial]
            visibility_field[fragment.position] += v_i
            
            # Update accumulated opacity for the next voxel along the ray
            ray_accumulated_opacity[ray_idx] = A_curr
            
    return visibility_field
```

**Parameters**
*   The method supports computation for up to 32 features [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Viewpoint/Transfer Function Dependence:** The visibility field is strictly dependent on the viewpoint and the transfer function. It cannot be precomputed independently of these parameters [S2, unofficial].
*   **Race Conditions:** In GPU implementations, multiple rays may map to the same voxel. Atomic operations are required for correctness [S1, unofficial].
*   **Precision:** Visibility values are real numbers. Accumulated opacity must be clamped to $[0, 1]$ to prevent numerical drift, though the sources do not specify a particular clamping strategy [G].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels sampled along rays. GPU implementation allows real-time performance [S2, unofficial].
*   **Memory Complexity:** Requires storage for the visibility field (same size as volume) plus temporary buffers for accumulated opacity [G].
*   **Strengths:** Enables analysis of volume structure, occlusion relationships, and feature contribution [S2, unofficial]. Useful for saliency-based visualization [S2, unofficial].
*   **Limitations:** Must be recomputed for every change in viewpoint or transfer function [S2, unofficial]. Not suitable for static precomputation [S2, unofficial].

## Background
The concept of voxel visibility as a contribution metric originates from early volume rendering research, specifically Levoy (1988) [S1, unofficial]. The term "Visibility Field" and its use in saliency analysis were formalized by Luo and Dingliana (2015) [S2, unofficial]. The method is implemented using CUDA for ray casting [S1, unofficial] and OpenGL 2.0+ for slice-based rendering [S2, unofficial].

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S2] Visibility-Weighted Saliency for Volume Visualization, https://doi.org/10.2312/cgvc.20151243, fetched 2026-10-26 (unofficial)
