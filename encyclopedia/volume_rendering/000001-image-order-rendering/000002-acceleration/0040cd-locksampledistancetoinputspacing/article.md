# LockSampleDistanceToInputSpacing

Also called: LockSampleDistanceToInputSpacing Correction, Sampling distance equalization

## Overview
LockSampleDistanceToInputSpacing is a correction technique used in volume rendering mappers to equalize the ray sampling distance between CPU and GPU implementations, thereby reducing visible sampling seams in parallel or hybrid rendering pipelines [S1, unofficial]. It addresses the discrepancy where CPU mappers (e.g., `vtkFixedPointVolumeRayCastMapper`) previously used a finer sampling distance than GPU mappers, causing artifacts to become more evident when switching to or comparing with GPU rendering [S1, unofficial]. By enforcing a consistent, input-spacing-based sample distance, it ensures visual continuity across different rendering backends [S1, unofficial].

## How it works
The method modifies the ray marching step size calculation in the volume mapper to lock the sampling distance to the input data's physical spacing, rather than using a heuristic or fixed value that might differ between CPU and GPU paths.

**Inputs and Outputs:**
- **Input:** Volume dataset with defined voxel spacing (physical dimensions per voxel).
- **Output:** A ray sampling distance value used during front-to-back compositing.

**Data Structures:**
- The volume data structure contains spacing information (e.g., `spacing[0]`, `spacing[1]`, `spacing[2]` in VTK).
- The mapper maintains an internal mode flag `LockSampleDistanceToInputSpacing`.

**Steps:**
1. **Enable Mode:** The GPU mapper (e.g., `vtkSmartVolumeMapper` or `GPUMapper`) enables the internal mode `LockSampleDistanceToInputSpacing` [S1, unofficial].
2. **Calculate Distance:** The sampling distance is derived from the input volume's spacing. In standard implementations, this often involves using the input spacing directly or a function of it to ensure the sample step matches the physical resolution [S1, unofficial].
3. **Apply Step:** The calculated distance is passed to the ray marching loop as the step size $\Delta s$ [S1, unofficial].

**Formulas:**
Let $d_{input}$ be the input spacing (or a representative scalar derived from it, such as the mean voxel side or minimum spacing). The sampling distance $d_{sample}$ is set as:
$$ d_{sample} = d_{input} $$
Or, in some specific implementations like the CPU mapper's original mode, it may be a fraction of the input spacing to increase density, which the GPU mapper then matches [S1, unofficial].

**Parameters:**
- `LockSampleDistanceToInputSpacing`: Boolean flag, default `false` in older versions, `true` in corrected versions [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Block-Edge Seams:** Even after enabling this correction, seams may remain visible at block-edge planes in parallel rendering scenarios, indicating that equalizing sample distance alone is insufficient for all artifacts [S1, unofficial].
- **Point vs. Cell Data:** When using point data, a transformation is required to align points with texture cell centers. If the ray step is not scaled accordingly with this transformation, errors persist [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sampling_distance(volume_data, use_lock_mode):
    """
    Computes the ray sampling distance based on volume data and lock mode.
    
    Args:
        volume_data: Object containing spacing information.
        use_lock_mode: Boolean flag for LockSampleDistanceToInputSpacing.
        
    Returns:
        float: The sampling distance.
    """
    # Get spacing from volume data
    # Assuming volume_data has a 'spacing' attribute, e.g., [dx, dy, dz]
    spacing = volume_data.spacing
    
    if use_lock_mode:
        # Lock to input spacing. 
        # In VTK CPU mapper, this mode further decreases sampling distance.
        # For GPU mapper, enabling it equalizes with CPU.
        # Implementation detail: Often uses the input spacing directly or 
        # a specific fraction defined by the mapper's internal logic.
        # Here we assume direct locking to the input spacing for consistency.
        sample_distance = min(spacing) # Or mean, depending on specific mapper implementation
        
        # Note: Some CPU implementations might use a factor. 
        # If the CPU mapper uses a factor k < 1, the GPU mapper must use the same k.
        # The source implies the GPU mapper was adjusted to match the CPU mapper's 
        # behavior when this mode is on.
        return sample_distance
    else:
        # Default behavior: may use a different heuristic or fixed value
        # Not specified in sources, so we return a placeholder or default
        return volume_data.default_sample_distance

def render_volume(volume_data, mapper_config):
    """
    Simulates the volume rendering loop with the corrected sampling distance.
    """
    use_lock = mapper_config.get('LockSampleDistanceToInputSpacing', False)
    step_size = compute_sampling_distance(volume_data, use_lock)
    
    # Ray marching loop
    # ...
    # For each ray:
    #   t = 0
    #   while t < ray_length:
    #       sample = texture_lookup(ray_origin + t * ray_direction, step_size)
    #       composite(sample)
    #       t += step_size
    # ...
```

## Subtopics

### Grid-Aware Sample Distance Calculation
A variant method used in streaming or multi-resolution volume rendering (e.g., in `vtk.js`) to compute the ray sampling distance based on the active texture grid rather than the full-resolution volume [S2, unofficial]. This prevents excessive sample counts and performance degradation when rendering reduced-resolution textures [S2, unofficial].
- **Definition:** The sample distance is calculated as half the mean voxel side of the grid that the current texture holds [S2, unofficial].
- **Rationale:** A reduced texture has a larger spacing, and using a finer sample (based on full resolution) adds no detail [S2, unofficial].
- **Constraint:** The distance never falls below the diagonal of the volume divided by 4000, capping the maximum samples per ray at 4000 [S2, unofficial].
- **Pitfall:** Using half the mean voxel side of the *full-resolution* volume for long series can cause the sample count to exceed the 4000 limit (e.g., ~5575 samples), triggering warnings and potentially degrading performance [S2, unofficial]. The fix is to use the grid's spacing instead [S2, unofficial].
- **Parameter:** `MAXIMUM_SAMPLES_PER_RAY` is set to 4000 [S2, unofficial].

## Operating characteristics
- **Visual Quality:** Reduces sampling seams between CPU and GPU mappers by equalizing sample density [S1, unofficial]. However, it does not eliminate seams at block-edge planes in parallel rendering [S1, unofficial].
- **Performance:** By locking the sample distance to input spacing, it ensures consistent computational load across different mappers [S1, unofficial]. The grid-aware variant prevents excessive sample counts, improving performance for large or multi-resolution volumes [S2, unofficial].
- **Complexity:** The correction adds minimal computational overhead, primarily involving a flag check and a distance calculation [S1, unofficial].

## Background
The technique originated in the VTK (Visualization Toolkit) and ParaView ecosystem to address sampling artifacts in parallel volume rendering [S1, unofficial]. It was introduced in ParaView 5.2 (OpenGL2) to correct discrepancies between the `vtkFixedPointVolumeRayCastMapper` (CPU) and the GPU mapper [S1, unofficial]. The CPU mapper had an internal mode `LockSampleDistanceToInputSpacing` that decreased the sampling distance, making artifacts more evident in the GPU mapper; enabling this mode in the GPU mapper corrected the issue [S1, unofficial]. A related variant, Grid-Aware Sample Distance Calculation, was developed in `vtk.js` to handle multi-resolution streaming volumes [S2, unofficial].

## Sources
[S1] Correcting Sampling Seams in Parallel Rendered Volumes, https://www.kitware.com/sampling-seams-in-parallel-rendered-volumes/, fetched 2026-10-29 (unofficial)
[S2] feat(core): multi-resolution volume textures, the base for task T2, https://github.com/cornerstonejs/cornerstone3D/pull/2931, fetched 2026-10-26 (unofficial)
