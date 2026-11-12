# Adaptive Multi-Step Empty Voxel Skipping

Also called: Improved Ray Casting with Empty Voxel Skipping and Multi-Step Length, Space leap empty voxel ray-casting algorithm, Three-step jump algorithm

## Overview
Adaptive Multi-Step Empty Voxel Skipping is an acceleration technique for direct volume rendering that combines space leaping (empty voxel skipping) with adaptive variable step sampling to reduce redundant computation in empty regions while maintaining image quality [S1]. It sets itself apart from traditional equidistant ray casting by dynamically adjusting the sampling interval based on local voxel density and explicitly culling rays that traverse only empty space, whereas siblings like [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md) rely on precomputed opacity maps and [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) focus on alpha-based termination rather than geometric space leaping.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (typically medical CT/MRI represented as a grid of voxels with scalar values mapped to color and opacity), camera parameters (origin, direction, field of view), and the volume rendering transfer function.
*   **Output:** A 2D image where each pixel is synthesized from the volume data along the corresponding view ray.

**Data Structures:**
The method operates on the standard volume grid. In the referenced VTK implementation, the algorithm is encapsulated within a class `vtkMyVolumeRayCastFunction` that inherits from `vtkVolumeRayCastingFunction` [S1]. The core data structure is the volume grid itself, where each voxel is either "empty" (background/no data) or "non-empty" (foreground/data). The sources do not specify a separate acceleration structure (like a sparse voxel octree or hash grid) for the skipping logic, implying the check is performed against the base grid or a simple validity flag per voxel [S1].

**Algorithm Steps:**
1.  **Ray Initialization:** For each pixel, cast a ray into the volume. Initialize the ray state to "Search for non-empty voxels." [S1]
2.  **Pre-check Validity (Global Empty Check):** Before sampling, determine if the ray intersects any non-empty voxels within the volume bounds. This is done by checking the bounding box of the non-empty data or the volume extent. If the ray passes entirely through empty space (no non-empty voxels exist along the ray direction), mark the ray as "invalid" and skip all further operations, proceeding to the next ray. [S1]
3.  **Adaptive Step Selection:** Determine the sampling step length $s$ based on the local voxel density encountered along the ray.
    *   If the current region contains many empty voxels, set a **longer** step value $s_{large}$ to skip over empty space rapidly. [S1]
    *   If the voxel density is high (non-empty voxels are present), set a **shorter** step value $s_{small}$ to increase sampling accuracy and preserve detail. [S1]
    *   *Note:* The source refers to this as a "three-step length" method, implying a discrete set of step sizes or a specific logic for transitioning between them, but does not provide the exact formula for calculating $s$ from density. [S1]
4.  **Resample and Synthesize:** Advance the ray position by the selected step length $s$.
    *   If the current voxel is empty, continue to the next step using the "empty" step size (or skip multiple voxels if the implementation supports multi-voxel jumps, though the text emphasizes "skipping invalid computations" rather than explicit multi-voxel indexing) [S1].
    *   If the current voxel is non-empty, sample the scalar value, map it to color and opacity using the transfer function, and accumulate the result into the ray's color and transparency buffer using standard volume compositing (e.g., front-to-back or back-to-front blending) [S1].
5.  **State Transition:** The ray transitions between the "Search" state (skipping empty) and "Accumulate" state (compositing non-empty) as it enters and exits dense regions. [S1]
6.  **Termination:** Stop when the ray exits the volume or the accumulated opacity reaches a threshold (if early termination is enabled, though not explicitly detailed as part of the core "multi-step" logic in the source, it is a common companion). [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_multi_step_empty_voxel_skipping(ray_origin, ray_direction, volume, transfer_function):
    """
    ray_origin: (x, y, z)
    ray_direction: (dx, dy, dz) normalized
    volume: 3D array or grid with is_empty(x,y,z) function
    transfer_function: function mapping scalar value to (color_rgba, opacity)
    """
    # 1. Pre-check validity: Check if ray intersects any non-empty voxel in the volume bounds
    # Note: The source implies a check for "any non-empty voxels along the ray".
    # A simple implementation checks if the ray's bounding box intersects the volume's non-empty bounding box.
    if not ray_intersects_non_empty_volume(ray_origin, ray_direction, volume):
        return (0.0, 0.0, 0.0, 0.0) # Invalid ray, return transparent black

    # Initialize accumulation
    accumulated_color = (0.0, 0.0, 0.0)
    accumulated_opacity = 0.0
    
    # Current ray position
    t = 0.0
    # Step size variables
    step_large = Not given in the sources. # Typical value for empty regions
    step_small = Not given in the sources. # Typical value for dense regions
    
    # Determine initial step size based on entry point
    # If entry point is empty, use large step; if non-empty, use small step
    current_voxel = get_voxel_at(ray_origin, ray_direction, t, volume)
    if is_empty(current_voxel):
        current_step = step_large
    else:
        current_step = step_small

    # 2. Adaptive Loop
    while t < max_distance:
        # Get voxel at current position
        voxel = get_voxel_at(ray_origin, ray_direction, t, volume)
        
        if is_empty(voxel):
            # State 1: Search for non-empty voxels
            # Use larger step to skip empty regions
            current_step = step_large
            t += current_step
        else:
            # State 2: Accumulate color and transparency
            # Use smaller step for accuracy in dense regions
            current_step = step_small
            
            # Sample and map
            scalar_value = get_scalar(voxel)
            color, opacity = transfer_function(scalar_value)
            
            # Front-to-back compositing
            # C_out = C_in + C_new * (1 - A_in)
            # A_out = A_in + A_new * (1 - A_in)
            accumulated_color = accumulated_color + color * (1.0 - accumulated_opacity)
            accumulated_opacity = accumulated_opacity + opacity * (1.0 - accumulated_opacity)
            
            # Optional: Early termination if opacity is high enough
            # if accumulated_opacity > threshold: break
            
            t += current_step
            
        # Re-evaluate step size for next iteration based on next voxel
        # This creates the "adaptive" behavior
        next_voxel = get_voxel_at(ray_origin, ray_direction, t, volume)
        if is_empty(next_voxel):
            current_step = step_large
        else:
            current_step = step_small

    return accumulated_color, accumulated_opacity
```

**Parameters:**
*   **Step Lengths:** Two primary step sizes are implied: one for empty regions ($s_{empty}$) and one for non-empty regions ($s_{dense}$). The source does not provide specific numerical values or the formula for deriving them from density. [S1]
*   **Thresholds:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Porphyritic Artifacts:** The combination of empty voxel jumping and uneven (adaptive) sampling can cause a "porphyritic" (patchy or mottled) appearance in the rendered image. This is particularly noticeable during rotation interactions. [S1]
*   **Precision:** Adaptive step sizes may lead to inconsistent sampling rates, potentially causing aliasing or visual discontinuities if the transition between step sizes is not smooth. [S1]
*   **Boundary Conditions:** The pre-check for "invalid rays" must correctly handle rays that graze the volume boundary or pass through thin structures. [S1]

## Operating characteristics
*   **Time Complexity:** Reduces computational overhead by skipping empty voxels entirely and reducing the number of samples in empty regions. The complexity is proportional to the number of non-empty voxels intersected by the ray, rather than the total number of voxels along the ray path. [S1]
*   **Memory Complexity:** Requires standard volume storage. The VTK implementation adds minimal overhead by inheriting from `vtkVolumeRayCastingFunction`. [S1]
*   **Measured Performance:** Tested on a Windows 7 system with Visual Studio 2010 and VTK 5.10.1 using virtual human head CT scan data. Performance was measured by rotation time. The method achieved significant speedup compared to traditional ray casting. [S1]
*   **Visual Quality:** The visual quality is reported to be "almost the same" as the original equidistant sampling, despite the presence of porphyritic artifacts. The artifacts do not significantly affect the interpretability of the result. [S1]
*   **Strengths:** Simple to implement (simpler than adaptive resampling schemes), does not require changing the input 2D sequence images, and effectively handles large datasets with significant empty space (common in medical images with independent organs). [S1]
*   **Limitations:** The "porphyritic" visual artifact may be unacceptable for high-quality visualization. The lack of a defined formula for step size selection may lead to suboptimal performance if not tuned carefully. [S1]
*   **When to use:** Suitable for real-time or interactive volume rendering of large medical datasets where significant portions of the volume are empty, and slight visual artifacts are tolerable in exchange for speed. [S1]
*   **When not to use:** When high-fidelity, artifact-free visualization is required, or when the volume has very high density with little empty space. [S1]

## Background
The method was proposed in the context of improving real-time volume rendering for large datasets, specifically medical images. It builds on the traditional Ray-Casting algorithm proposed by Marc Levoy [S1]. The specific paper introducing this method is "The research of improved ray casting algorithm on VTK" [S1]. It was implemented using the Visualization Toolkit (VTK) [S10], a widely used C++ library for 3D computer graphics and visualization. The work addresses the computational bottleneck of processing large amounts of data where many voxels are empty (e.g., in CT scans of the human body where organs are separated by empty space). [S1]

## Sources
[S1] The research of improved ray casting algorithm on VTK, https://doi.org/10.2991/iccsae-15.2016.172, fetched 2026-10-26
