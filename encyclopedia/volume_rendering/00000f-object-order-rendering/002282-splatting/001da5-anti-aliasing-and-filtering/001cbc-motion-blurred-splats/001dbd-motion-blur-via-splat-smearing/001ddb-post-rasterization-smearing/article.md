# Post-rasterization Smearing

Also called: Accumulated Motion Blur, Post-Rasterization Velocity Smearing, Velocity Buffer Smear, Velocity buffer smearing

## Overview
Post-rasterization smearing is a technique to approximate motion blur in voxel buffers by deferring the blurring operation until after the rasterization/splatting phase is complete [S1, unofficial]. Unlike methods that smear individual samples during rasterization, this approach accumulates velocity information in a separate buffer and applies a single, global smear pass [S1, unofficial]. It is chosen when performance is prioritized over handling complex overlapping primitives with divergent motion vectors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel buffer containing density/intensity values, and a separate velocity buffer storing accumulated motion vectors for each voxel.
*   **Output:** A motion-blurred voxel buffer.

**Data Structures:**
1.  **Density Buffer:** Standard voxel grid storing scalar values (e.g., density, color).
2.  **Velocity Buffer:** A voxel grid of the same dimensions as the density buffer. Each voxel stores a 3D vector ($\vec{v}$) representing the accumulated motion.
    *   *Layout:* Typically stored as three floating-point values (x, y, z) per voxel.
    *   *Initialization:* Zeroed before the frame begins.

**Steps:**
1.  **Rasterization/Splatting with Velocity Accumulation:**
    *   For each primitive (or sample) being rasterized:
        *   Determine its motion vector $\vec{m}$ for the current frame.
        *   Rasterize the primitive into the density buffer as usual.
        *   Add $\vec{m}$ to the corresponding voxels in the velocity buffer.
        *   *Note:* If multiple primitives overlap in a voxel, their velocity vectors are summed (accumulated) [S1, unofficial].
2.  **Velocity Normalization/Averaging (Optional Variant):**
    *   To resolve direction ambiguity when multiple primitives overlap, calculate the average motion vector per voxel:
        $$ \vec{v}_{avg} = \frac{\sum \vec{m}_i}{N} $$
        where $N$ is the number of primitives contributing to that voxel [S1, unofficial].
    *   *Pitfall:* This averaging can cause visual artifacts if primitives have drastically different velocities [S1, unofficial].
3.  **Global Smear Pass:**
    *   Iterate through every voxel in the density buffer.
    *   Retrieve the accumulated velocity $\vec{v}$ from the velocity buffer for that voxel.
    *   Smear the voxel's value along $\vec{v}$. This distributes the value evenly across the voxels touched by the motion vector, preserving the sum of values [S1, unofficial].
    *   *Implementation Note:* The smear direction is derived directly from the accumulated velocity. If $\vec{v}$ is zero, no smear occurs.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def post_rasterization_smearing(density_buffer, velocity_buffer, primitives):
    """
    density_buffer: 3D array of floats, initialized to 0
    velocity_buffer: 3D array of vec3, initialized to (0,0,0)
    primitives: list of objects, each with a method get_motion_vector() and a rasterize_into() method
    """
    # Step 1: Accumulate velocities during rasterization
    for primitive in primitives:
        motion_vec = primitive.get_motion_vector()
        primitive.rasterize_into(density_buffer)
        
        # Accumulate motion vector into the velocity buffer for affected voxels
        affected_voxels = primitive.get_affected_voxel_indices()
        for voxel_idx in affected_voxels:
            velocity_buffer[voxel_idx] += motion_vec

    # Step 2: Global smear pass
    blurred_buffer = copy(density_buffer)
    
    for voxel_idx in all_voxels_in_buffer:
        v = velocity_buffer[voxel_idx]
        
        # If no motion, skip
        if length(v) == 0:
            continue
            
        # Smear the density value along the velocity vector
        # This distributes the value evenly across the path defined by v
        smear_value = blurred_buffer[voxel_idx]
        clear_affected_voxels(blurred_buffer, voxel_idx, v)
        distribute_evenly(blurred_buffer, voxel_idx, v, smear_value)
        
    return blurred_buffer
```

**Edge Cases and Pitfalls:**
*   **Overlapping Primitives with Divergent Velocities:** If input primitives overlap and have drastically different motion vectors, it becomes impossible to calculate a valid single direction to smear in [S1, unofficial].
*   **Visual Artifacts:** Keeping track of the average motion vector in overlapping voxels can cause visual artifacts in the final render [S1, unofficial].
*   **Camera Motion Cancellation:** If the camera moves at the same speed as the primitives, motion should cancel out, resulting in a sharp image. However, because motion blur is baked into the voxel buffer via accumulated velocity, this cancellation is not possible, leading to incorrect blur [S1, unofficial].

## Subtopics

### Velocity Buffer Smear Variant
Retains both density and velocity buffers and calculates motion blur during rendering. This avoids reduced shading detail in motion-blurred areas but still suffers from the overlapping primitive problem [S1, unofficial].

### Post-Lighting Smearing
If the lit volume is voxelized, motion blur can be applied after lighting. This solves the problem of lighting a voxel buffer with motion blur baked in, but still shows artifacts where overlapping primitives have different velocity vectors [S1, unofficial].

## Operating characteristics
*   **Performance:** Often faster than smearing each sample as it is written to the buffer, as it requires only a single global smear pass after rasterization [S1, unofficial].
*   **Memory:** Requires an additional velocity buffer of the same size as the density buffer, increasing memory usage [S1, unofficial].
*   **Limitations:** Poor handling of complex scenes with overlapping primitives having divergent motion vectors. Cannot correctly handle camera motion cancellation [S1, unofficial].
*   **Use Cases:** Suitable for simple scenes or when performance is critical and visual artifacts from overlapping divergent motions are acceptable. Microvoxel-based volume renderers may benefit from the variant that retains separate buffers [S1, unofficial].

## Background
This method is described in the context of production volume rendering fundamentals, specifically as an alternative to per-sample smearing during rasterization [S1, unofficial]. It is part of a broader discussion on motion blur techniques in voxel-based rendering pipelines [S1, unofficial].

## Sources
[S1] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
