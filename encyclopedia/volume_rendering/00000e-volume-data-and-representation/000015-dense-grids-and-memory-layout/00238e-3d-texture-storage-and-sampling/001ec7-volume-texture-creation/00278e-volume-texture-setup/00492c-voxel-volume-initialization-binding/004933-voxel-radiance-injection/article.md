# Voxel Radiance Injection

Also called: Radiance Injection Pass, Voxel Radiance Injection Pass

## Overview
Voxel Radiance Injection is the compute-shader-based step that transfers precomputed lighting information from a shadow map volume into a dense voxel grid, enabling subsequent cone-tracing or ray-marching passes to access indirect illumination data. It differs from standard rasterization-based voxelization by operating entirely in GPU compute, allowing for flexible data transformation and direct injection of radiance values into the voxel volume's memory layout.

## How it works
The method executes a single compute shader dispatch that iterates over the voxel volume dimensions, sampling from the shadow map volume and writing the resulting radiance values into the voxel volume.

**Inputs and Outputs:**
*   **Input:** `ShadowMapVolume` (read-only), `VoxelVolume` (read-only for metadata, write-only for radiance).
*   **Output:** Updated `VoxelVolume` containing injected radiance.

**Data Structures:**
*   **ShadowMapVolume:** A 3D texture or buffer storing shadow/occlusion data. The exact internal format is not specified in the sources, but it is sampled via standard texture fetch operations.
*   **VoxelVolume:** A 3D texture or buffer storing voxel data. The sources indicate it is bound for writing. The specific layout of the radiance data (e.g., RGB, RGBA, half-float) is not explicitly defined in the provided notes, but it is the target of the injection.

**Parameters:**
*   **Grid Size:** The sources imply a fixed grid dimension of 256 units for the calculation of work groups.
*   **Work Group Size:** 4x4x4 threads per work group (derived from `256.0f / 4.0f`).
*   **Settings:**
    *   `ContinousVoxelizationPass`: Boolean flag. If disabled, the pass returns early.
    *   `RadianceInjection`: Boolean flag. If disabled, the pass returns early.

**Numbered Steps:**
1.  Check if the setting `ContinousVoxelizationPass` is enabled. If not, terminate the pass [S1, unofficial].
2.  Check if the setting `RadianceInjection` is enabled. If not, terminate the pass [S1, unofficial].
3.  Set the active compute shader to `VOXEL_RADIANCE_INJECTION_PASS_COMPUTE_SHADER` [S1, unofficial].
4.  Bind the `ShadowMapVolume` render volume for reading [S1, unofficial].
5.  Bind the `VoxelVolume` render volume for reading (likely for metadata or existing data) [S1, unofficial].
6.  Send custom attributes associated with the `ShadowMapVolume` to the pipeline [S1, unofficial].
7.  Send custom attributes associated with the `VoxelVolume` to the pipeline [S1, unofficial].
8.  Bind the `VoxelVolume` render volume for writing [S1, unofficial].
9.  Calculate the number of work groups per dimension: $N_{wg} = \lceil 256.0 / 4.0 \rceil = 64$ [S1, unofficial].
10. Dispatch the compute shader with dimensions $(64, 64, 64)$ [S1, unofficial].
11. Insert a memory barrier for texture fetch and shader image access to ensure writes are visible to subsequent reads [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_radiance_injection_pass(settings, shadow_map_volume, voxel_volume):
    """
    Executes the Voxel Radiance Injection pass.
    
    Args:
        settings: Object containing 'ContinousVoxelizationPass' and 'RadianceInjection' flags.
        shadow_map_volume: Read-only resource containing shadow/occlusion data.
        voxel_volume: Read/Write resource for the voxel grid.
    """
    # Step 1: Check settings
    if not settings.ContinousVoxelizationPass:
        return
    if not settings.RadianceInjection:
        return

    # Step 2: Setup Compute Shader
    shader = get_compute_shader("VOXEL_RADIANCE_INJECTION_PASS_COMPUTE_SHADER")
    
    # Step 3: Bind Resources
    # Note: The sources specify binding ShadowMapVolume and VoxelVolume for reading,
    # then VoxelVolume for writing. This implies a specific binding order or descriptor set update.
    bind_resource(shadow_map_volume, usage=READ_ONLY)
    bind_resource(voxel_volume, usage=READ_ONLY)
    
    # Send custom attributes (implementation detail of the engine's attribute system)
    send_custom_attributes(shadow_map_volume)
    send_custom_attributes(voxel_volume)
    
    bind_resource(voxel_volume, usage=WRITE_ONLY)
    
    # Step 4: Calculate Work Groups
    # Grid size is fixed at 256 units, work group size is 4
    grid_size = 256
    work_group_size = 4
    num_work_groups = ceil(grid_size / work_group_size) # 64
    
    # Step 5: Dispatch
    dispatch_compute(num_work_groups, num_work_groups, num_work_groups, shader)
    
    # Step 6: Memory Barrier
    # Ensure image writes complete before subsequent reads
    memory_barrier(
        flags=TEXTURE_FETCH_BARRIER_BIT | SHADER_IMAGE_ACCESS_BARRIER_BIT
    )
```

**Edge Cases and Pitfalls:**
*   **Early Termination:** If either `ContinousVoxelizationPass` or `RadianceInjection` is disabled, the pass does nothing. This must be checked before resource binding to avoid unnecessary overhead or errors if resources are not prepared.
*   **Memory Barriers:** The explicit barrier `GL_TEXTURE_FETCH_BARRIER_BIT | GL_SHADER_IMAGE_ACCESS_BARRIER_BIT` is critical. Without it, subsequent passes reading from the `VoxelVolume` may see stale data or undefined behavior due to GPU pipeline ordering.
*   **Fixed Grid Size:** The work group calculation assumes a fixed grid dimension of 256. If the actual voxel volume dimensions differ, the dispatch may access out-of-bounds memory or leave parts of the volume uninitialized. [G] In a general implementation, the grid size should be derived from the actual voxel volume dimensions, not hardcoded to 256.

## Operating characteristics
*   **Time Complexity:** $O(N^3)$ where $N$ is the voxel grid dimension (256 in the source). The complexity is linear with respect to the number of voxels.
*   **Memory Complexity:** $O(N^3)$ for storing the voxel volume. The pass itself requires minimal additional memory beyond the shader state and temporary buffers for attributes.
*   **Performance:** The use of a compute shader allows for parallel processing of all voxels. The fixed work group size (4x4x4) and grid size (256) suggest an optimized, fixed-resolution implementation.
*   **Strengths:** Efficient parallel injection of radiance data; decouples the injection process from rasterization; allows for custom data transformation via the compute shader.
*   **Limitations:** Fixed grid size (256) limits flexibility; requires careful management of memory barriers; dependent on the availability of a shadow map volume.

## Background
The method is part of the Voxel Cone Tracing pipeline, specifically the step that populates the voxel volume with radiance information for global illumination. It is implemented in the `Voxel-Cone-Tracing` project by maritim.

## Sources
[S1] maritim/Voxel-Cone-Tracing, https://github.com/maritim/Voxel-Cone-Tracing, fetched 2026-10-26 (unofficial)
