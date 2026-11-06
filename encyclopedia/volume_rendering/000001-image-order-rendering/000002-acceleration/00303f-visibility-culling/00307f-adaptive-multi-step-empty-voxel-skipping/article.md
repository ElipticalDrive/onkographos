# Adaptive Multi-Step Empty Voxel Skipping
Also called: Improved Ray Casting with Empty Voxel Skipping and Multi-Step Length, Space leap empty voxel ray-casting algorithm, Three-step jump algorithm

## Overview
Adaptive Multi-Step Empty Voxel Skipping is an acceleration technique for direct volume rendering that combines space-leaping (empty voxel skipping) with adaptive variable step-size sampling to reduce redundant computation in sparse regions [S1]. It distinguishes itself from standard ray casting by dynamically adjusting the integration step length based on local voxel density—using longer steps for empty areas and shorter steps for dense regions—thereby minimizing invalid computations without altering the input data structure [S1].

## How it works
The method modifies the standard front-to-back ray casting loop by introducing a state machine for ray processing and a density-dependent step size calculation.

### Inputs and Outputs
*   **Inputs:** A 3D volume dataset (typically a 3D texture or grid of voxels), camera parameters (origin, direction, field of view), and a ray traversal range.
*   **Outputs:** A 2D image where each pixel contains the accumulated color and opacity of the ray passing through it.

### Data Structures
The algorithm is designed to be integrated into a volume rendering pipeline, specifically inheriting from the `vtkVolumeRayCastingFunction` base class in the Visualization Toolkit (VTK) [S1].
*   **Ray State:** Each ray maintains a state indicating whether it is currently searching for non-empty voxels or accumulating color/transparency from non-empty voxels [S1].
*   **Volume Data:** The volume is accessed via standard 3D indexing. The definition of "empty" is not explicitly defined in the sources, but implies voxels with zero or negligible density/opacity [S1].

### Algorithm Steps
1.  **Ray Initialization:** For each pixel, generate a ray. Initialize the ray's state to "Search" [S1].
2.  **Pre-check Validity:** Determine if the ray intersects any non-empty voxels within the volume bounds. If the ray passes entirely through empty space (no non-empty voxels exist along the path), mark the ray as "invalid" and skip all further processing for this ray, proceeding to the next pixel [S1].
3.  **Adaptive Step Selection:** Calculate the sampling step length ($\Delta s$) based on the local voxel density at the current ray position:
    *   If the local area contains many empty voxels, set $\Delta s$ to a longer value [S1].
    *   If the voxel density is high, set $\Delta s$ to a shorter value to increase accuracy [S1].
    *   *Note:* The sources refer to this as a "three-step length" method, implying a discrete set of step sizes or a specific heuristic for switching between them, but do not provide the exact mathematical formula for the step size selection [S1].
4.  **Resample and Synthesize:** Advance the ray by $\Delta s$.
    *   If the current voxel is empty, continue advancing by $\Delta s$ without accumulating color (maintaining "Search" state or skipping directly to the next non-empty check) [S1].
    *   If the current voxel is non-empty, accumulate its color ($C$) and opacity ($\alpha$) into the accumulated color ($C_{acc}$) and accumulated opacity ($\alpha_{acc}$) using the standard front-to-back compositing equations [S1]:
        $$ C_{acc} = C_{acc} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
        $$ \alpha_{acc} = \alpha_{acc} + \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
    *   Continue until the ray exits the volume or a termination condition is met [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_adaptive_multi_step(volume, ray_origin, ray_direction, volume_bounds):
    """
    Renders a single ray using Adaptive Multi-Step Empty Voxel Skipping.
    
    Args:
        volume: 3D array of voxel data (density/opacity).
        ray_origin: 3D vector (x, y, z).
        ray_direction: 3D vector (x, y, z), normalized.
        volume_bounds: Object defining the min/max coordinates of the volume.
        
    Returns:
        accumulated_color: 3D vector (R, G, B).
        accumulated_opacity: float.
    """
    
    # 1. Initialize Ray States
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    current_pos = ray_origin
    
    # 2. Pre-check Validity
    # Check if the ray intersects any non-empty voxel within bounds.
    # This requires a traversal or bounding box check.
    if not intersects_non_empty_volume(volume, current_pos, ray_direction, volume_bounds):
        return accumulated_color, accumulated_opacity
    
    # 3. Adaptive Step Selection & Resample Loop
    while current_pos is within volume_bounds:
        
        # Get voxel density at current position
        density = get_voxel_density(volume, current_pos)
        
        # Determine step length based on local density
        # Note: Exact formula for step length selection is not provided in sources [S1]
        # Heuristic: Longer step for empty, shorter for dense.
        if density == 0:
            step_length = LONG_STEP_SIZE # Placeholder for adaptive logic
        else:
            step_length = SHORT_STEP_SIZE # Placeholder for adaptive logic
            
        # Advance ray
        next_pos = current_pos + ray_direction * step_length
        
        # Check if we crossed a voxel boundary or if the next pos is still in empty space
        # The source implies skipping empty voxels entirely.
        # We sample at the new position or along the segment.
        
        # For implementation completeness [G]:
        # In a grid-based volume, we typically step through voxels.
        # If the source implies "jumping" over empty regions, we might need to
        # find the next non-empty voxel index. However, the description suggests
        # a continuous step size adjustment.
        
        # Sample the voxel at the current position (or interpolated)
        voxel_data = sample_volume(volume, current_pos)
        color_sample = voxel_data.color
        alpha_sample = voxel_data.opacity
        
        # 4. Resample and Synthesize
        # Only accumulate if non-empty (alpha > 0)
        if alpha_sample > 0:
            # Front-to-back compositing
            transmittance = 1.0 - accumulated_opacity
            contribution = color_sample * alpha_sample * transmittance
            accumulated_color = accumulated_color + contribution
            accumulated_opacity = accumulated_opacity + alpha_sample * transmittance
            
            # Early termination [G] (Standard practice, though not explicitly detailed in S1's steps,
            # it is implied by "accumulate... of non-empty voxels" and standard ray casting)
            if accumulated_opacity >= 1.0:
                break
        
        current_pos = next_pos
        
    return accumulated_color, accumulated_opacity
```

### Parameters
*   **Step Length:** A variable step size. It is not fixed. Typical values are not given, but the logic dictates it is longer for empty areas and shorter for dense areas [S1].
*   **Empty Threshold:** The definition of an "empty" voxel is not explicitly defined in the sources, but typically corresponds to zero density or opacity [S1].

### Edge Cases and Pitfalls
*   **Visual Artifacts:** The combination of empty voxel jumping and uneven (adaptive) sampling can cause "porphyritic" (patchy or mottled) appearance in the rendered image, particularly noticeable during rotation interactions [S1].
*   **Invalid Rays:** Rays that do not intersect any non-empty voxels must be correctly identified and skipped to avoid unnecessary computation [S1].
*   **Precision:** The adaptive step size may lead to aliasing or visual inconsistencies if the step size changes abruptly between adjacent pixels or frames [S1].

## Operating characteristics
*   **Time Complexity:** Reduced compared to standard equidistant ray casting, particularly for sparse volumes with large empty regions, as empty voxels are skipped and step sizes are increased in those regions [S1].
*   **Memory Complexity:** Similar to standard ray casting, as it does not require additional large acceleration structures beyond the volume data itself, though it may require state management for the ray [S1].
*   **Performance:** Tested on a Windows 7 system with Visual Studio 2010 and VTK 5.10.1, using virtual human head CT scan data. The method achieved faster rendering speeds (measured by rotation time) while maintaining visual quality almost identical to the original method [S1].
*   **Strengths:** Reduces computational overhead by skipping empty voxels and adapting step size; does not require changing the input 2D sequence images [S1].
*   **Limitations:** Can introduce visual artifacts ("porphyritic" appearance) due to uneven sampling [S1].

## Background
The method was proposed as an improvement to the standard ray-casting algorithm, which traditionally uses equidistant sampling for all voxels, leading to high computational overhead in medical images with many independent organs and empty voxels [S1]. It was implemented using the Visualization Toolkit (VTK) by creating a new class `vtkMyVolumeRayCastFunction` that inherits from `vtkVolumeRayCastingFunction` [S1]. The algorithm was tested on virtual human body head CT scan data [S1].

## Sources
[S1] The research of improved ray casting algorithm on VTK, https://doi.org/10.2991/iccsae-15.2016.172, fetched 2026-10-26
