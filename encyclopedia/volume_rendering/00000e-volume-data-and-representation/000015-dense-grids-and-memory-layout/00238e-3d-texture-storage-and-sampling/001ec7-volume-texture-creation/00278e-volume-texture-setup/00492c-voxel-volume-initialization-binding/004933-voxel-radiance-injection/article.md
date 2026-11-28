# Voxel Radiance Injection

Also called: Radiance Injection Pass, Voxel Radiance Injection Pass

## Overview
Voxel Radiance Injection is a compute-shader pass that transfers lighting data from a shadow map volume into a dense voxel grid for subsequent cone-tracing or ray-marching passes [S1, unofficial]. It operates entirely in GPU compute, dispatching a single shader kernel over a fixed 256³ grid [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `ShadowMapVolume` (read-only) and `VoxelVolume` (read-only for metadata) [S1, unofficial].
*   **Output:** Updated `VoxelVolume` containing injected radiance [S1, unofficial].

**Data Structures:**
*   **ShadowMapVolume:** Stores shadow/occlusion data and is bound for reading [S1, unofficial].
*   **VoxelVolume:** Stores voxel data, is bound for reading initially and writing subsequently, and its exact internal format is not specified in the sources [S1, unofficial].

**Parameters:**
*   **Grid Size:** Fixed at 256 units per dimension [S1, unofficial].
*   **Work Group Size:** 4 threads per dimension [S1, unofficial].
*   **Settings:** `ContinousVoxelizationPass` (boolean) and `RadianceInjection` (boolean); if either is disabled, the pass returns early [S1, unofficial].

**Numbered Steps:**
1. Check if `ContinousVoxelizationPass` is enabled; if not, return early [S1, unofficial].
2. Check if `RadianceInjection` is enabled; if not, return early [S1, unofficial].
3. Set the compute shader to `VOXEL_RADIANCE_INJECTION_PASS_COMPUTE_SHADER` [S1, unofficial].
4. Bind `ShadowMapVolume` for reading [S1, unofficial].
5. Bind `VoxelVolume` for reading [S1, unofficial].
6. Send custom attributes for `ShadowMapVolume` to the pipeline [S1, unofficial].
7. Send custom attributes for `VoxelVolume` to the pipeline [S1, unofficial].
8. Bind `VoxelVolume` for writing [S1, unofficial].
9. Calculate work groups per dimension: $N_{wg} = \lceil 256.0 / 4.0 \rceil = 64$ [S1, unofficial].
10. Dispatch the compute shader with $(64, 64, 64)$ work groups [S1, unofficial].
11. Insert a memory barrier for texture fetch and shader image access to ensure writes complete before subsequent reads [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxel_radiance_injection_pass(settings, shadow_map_volume, voxel_volume):
    if not settings.ContinousVoxelizationPass:
        return
    if not settings.RadianceInjection:
        return

    # Set compute shader
    active_shader = "VOXEL_RADIANCE_INJECTION_PASS_COMPUTE_SHADER"

    # Bind resources
    shadow_map_volume.bind(usage="READ_ONLY")
    voxel_volume.bind(usage="READ_ONLY")
    shadow_map_volume.send_custom_attributes()
    voxel_volume.send_custom_attributes()
    voxel_volume.bind(usage="WRITE_ONLY")

    # Dispatch compute
    num_work_groups = math.ceil(256.0 / 4.0)
    # Dispatch shader with (num_work_groups, num_work_groups, num_work_groups)
    # Not given in the sources. [G]

    # Memory barrier
    # Ensure image writes complete before subsequent reads
    # Not given in the sources. [G]
```

**Edge Cases and Pitfalls:**
*   **Early Termination:** If either `ContinousVoxelizationPass` or `RadianceInjection` is disabled, the pass returns early before resource binding [S1, unofficial].
*   **Memory Barriers:** The explicit barrier `GL_TEXTURE_FETCH_BARRIER_BIT | GL_SHADER_IMAGE_ACCESS_BARRIER_BIT` ensures image writes complete before subsequent reads [S1, unofficial].
*   **Fixed Grid Size:** The work group calculation assumes a fixed grid dimension of 256 [S1, unofficial]. If the actual voxel volume dimensions differ, the dispatch may access out-of-bounds memory or leave parts of the volume uninitialized [G].

## Operating characteristics
*   **Time Complexity:** $O(N^3)$ where $N=256$ [S1, unofficial].
*   **Memory Complexity:** $O(N^3)$ for storing the voxel volume [S1, unofficial].
*   **Performance:** Parallel dispatch over the fixed 256³ grid [S1, unofficial].
*   **Strengths:** Efficient parallel injection of radiance data; decouples the injection process from rasterization [S1, unofficial].
*   **Limitations:** Fixed grid size limits flexibility; requires correct memory barrier placement [S1, unofficial].
*   **Not given in the sources:** Measured performance metrics, hardware specifics, and scaling behavior beyond the fixed grid.

## Background
Originates from the Voxel Cone Tracing pipeline by maritim [S1, unofficial]. Used to populate the voxel volume with radiance for global illumination [S1, unofficial].

## Sources
[S1] maritim/Voxel-Cone-Tracing, https://github.com/maritim/Voxel-Cone-Tracing, fetched 2026-10-26 (unofficial)
