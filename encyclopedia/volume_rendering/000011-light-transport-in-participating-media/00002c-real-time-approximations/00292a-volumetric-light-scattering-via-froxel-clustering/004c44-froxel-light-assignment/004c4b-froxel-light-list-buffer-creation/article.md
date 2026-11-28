# Froxel Light List Buffer Creation

Also called: Froxel Light List Buffer Allocation

## Overview
This method allocates and populates the GPU buffers that store variable-length light lists for each froxel after spatial assignment [S1, unofficial]. It prepares the `Counts`, `Offsets`, `TotalIndices`, and `LightIndices` buffers required for subsequent volumetric rendering passes [S1, unofficial].

## How it works
The method consists of buffer allocation, a reduction pass to determine the total light count, an atomic assignment pass to populate indices, and a three-pass prefix-sum scan to compute offsets [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- `Grid`: 3D grid dimensions $(G_X, G_Y, G_Z)$ defining the froxel structure [S1, unofficial].
- `NumLights_RT`: Number of real-time lights in the scene [S1, unofficial].
- `MaxLightsPerFroxel`: Maximum number of lights assigned to any single froxel [S1, unofficial].

**Outputs:**
- `FroxelLists`: A structure containing four buffers (see Layout) [S1, unofficial].

### Data Structures and Layout
The `FFroxelLists` structure contains the following buffers, all created as `RWStructuredBuffer<uint32>` with `BUF_ShaderResource | BUF_UnorderedAccess` flags [S1, unofficial]:

1. **Counts**: `RWStructuredBuffer<uint32>`, size $G_X \times G_Y \times G_Z$. Stores the number of lights assigned to each froxel [S1, unofficial].
2. **Offsets**: `RWStructuredBuffer<uint32>`, size $G_X \times G_Y \times G_Z$. Stores the starting index in `LightIndices` for each froxel [S1, unofficial].
3. **TotalIndices**: `RWStructuredBuffer<uint32>`, size 1. Stores the total number of light indices across all froxels [S1, unofficial].
4. **LightIndices**: `RWStructuredBuffer<uint32>`, size $(G_X \times G_Y \times G_Z) \times \text{MaxLightsPerFroxel}$. Stores the actual light indices for all froxels, packed sequentially [S1, unofficial].

### Steps

1. **Calculate Froxel Count**:
    $$N_{\text{froxel}} = G_X \times G_Y \times G_Z$$ [S1, unofficial].

2. **Allocate Buffers**:
    Allocate the four buffers defined in the Layout section with sizes derived from $N_{\text{froxel}}$ and `MaxLightsPerFroxel` [S1, unofficial].

3. **Compute Total Indices (Reduction)**:
    - Clear `TotalIndices` UAV to 0 [S1, unofficial].
    - Execute the `FFroxelTotalIndicesCS` compute shader with group count $(1, 1, 1)$ [S1, unofficial].
    - This pass sums the `Counts` buffer to determine the total number of light entries [S1, unofficial].

4. **Assign Light Indices**:
    - Clear `LightIndices` UAV to 0 [S1, unofficial].
    - Reuse the `Counts` buffer as `FroxelHeads` (atomic head pointer) and clear it to 0 [S1, unofficial].
    - Execute the `FFroxelAssignLightsCS` compute shader with group count $\lceil \text{NumLights\_RT} / 64 \rceil$ [S1, unofficial].
    - Each light thread atomically increments the head for its assigned froxel and writes the light index to `LightIndices` at the returned position [S1, unofficial].

5. **Compute Offsets (Prefix Sum)**:
    The offsets are computed using a three-pass Blelloch scan to handle the 3D grid efficiently [S1, unofficial]:
    - **Pass A (Intra-Slice Scan)**: Perform a prefix sum on `Counts` within each Z-slice (size $G_X \times G_Y$) [S1, unofficial]. Store intermediate results in `ZSliceTotals` and final offsets in `Offsets` [S1, unofficial].
    - **Pass B (Inter-Slice Scan)**: Perform a prefix sum on the `ZSliceTotals` buffer (size $G_Z$) to get cumulative sums for each Z-slice [S1, unofficial]. Store in `ZSliceOffsets` [S1, unofficial].
    - **Pass C (Add Z-Slice Offsets)**: Add the corresponding `ZSliceOffsets` value to each froxel's offset in the `Offsets` buffer [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_froxel_light_list_buffers(grid: tuple, num_lights: int, max_lights_per_froxel: int):
    """
    Allocates and populates buffers for froxel light lists.
    grid: (Gx, Gy, Gz)
    """
    Gx, Gy, Gz = grid
    N_froxel = Gx * Gy * Gz
    Z_slice_size = Gx * Gy

    # 1. Allocate Buffers
    counts = [0] * N_froxel
    offsets = [0] * N_froxel
    total_indices = [0]
    light_indices = [0] * (N_froxel * max_lights_per_froxel)
    z_slice_totals = [0] * Gz
    z_slice_offsets = [0] * Gz

    # 2. Compute Total Indices (Reduction)
    # Simulates FFroxelTotalIndicesCS with group count (1, 1, 1)
    total_indices[0] = sum(counts)

    # 3. Assign Light Indices
    # Reuse counts as FroxelHeads (atomic heads)
    froxel_heads = [0] * N_froxel
    # Simulates FFroxelAssignLightsCS: one thread per light
    for light_idx in range(num_lights):
        froxel_idx = light_idx % N_froxel  # Simplified assignment logic
        pos = froxel_heads[froxel_idx]
        froxel_heads[froxel_idx] += 1
        if pos < max_lights_per_froxel:
            light_indices[froxel_idx * max_lights_per_froxel + pos] = light_idx

    # 4. Compute Offsets (3-Pass Blelloch Scan)
    # Pass A: Prefix sum within each Z-slice
    for z in range(Gz):
        base = z * Z_slice_size
        current_sum = 0
        for i in range(Z_slice_size):
            offsets[base + i] = current_sum
            current_sum += counts[base + i]
        z_slice_totals[z] = current_sum

    # Pass B: Prefix sum of Z-slice totals
    for z in range(1, Gz):
        z_slice_offsets[z] = z_slice_offsets[z-1] + z_slice_totals[z-1]

    # Pass C: Add Z-slice offsets to individual froxel offsets
    for z in range(Gz):
        base = z * Z_slice_size
        for i in range(Z_slice_size):
            offsets[base + i] += z_slice_offsets[z]

    return {
        'Counts': counts,
        'Offsets': offsets,
        'TotalIndices': total_indices,
        'LightIndices': light_indices
    }
```

### Parameters and Edge Cases
- **MaxLightsPerFroxel**: Determines the size of `LightIndices` as $(G_X \times G_Y \times G_Z) \times \text{MaxLightsPerFroxel}$ [S1, unofficial].
- **NumLights_RT**: Determines the group count for the assignment shader as $\lceil \text{NumLights\_RT} / 64 \rceil$ [S1, unofficial].
- **Grid Layout**: The prefix sum assumes a Z-slice major order where each slice contains $G_X \times G_Y$ froxels [S1, unofficial].

## Subtopics

### [Froxel Assign Lights Compute Shader Parameters](13208-froxel-assign-lights-compute-shader-parameters.md)
Defines the shader parameter structure for the compute shader that assigns lights to froxels during the volumetric light scattering pass. It provides the GPU kernel with references to the froxel grid state, view parameters, light data, offsets, and atomic counters required for populating the light index buffer. Choose this when configuring the GPU kernel dispatch for the assignment phase. See [Froxel Assign Lights Compute Shader Parameters](13208-froxel-assign-lights-compute-shader-parameters.md).

### Froxel Light Index Assignment
Computes the list of light indices for each froxel, storing results in `FroxelLists.LightIndices` [S1, unofficial]. It first computes the total number of indices using a reduction pass, then assigns indices using an atomic counter [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N_{\text{froxel}})$ for allocation and offset computation, and $O(\text{NumLights\_RT})$ for index assignment [S1, unofficial].
- **Memory Complexity**: $O(N_{\text{froxel}} \times \text{MaxLightsPerFroxel})$ for `LightIndices`, plus $O(N_{\text{froxel}})$ for `Counts` and `Offsets`, and $O(G_Z)$ for temporary prefix sum buffers [S1, unofficial].
- **Strengths**: Handles variable-length light lists efficiently on the GPU [G].
- **Limitations**: Requires temporary buffers for the three-pass prefix sum [G].

## Background
This method is part of the Unreal Engine 5 Froxel Lighting system for volumetric light scattering via froxel clustering [S1, unofficial].

## Sources
[S1] ArminTajik/ue5-froxel-lighting, https://github.com/ArminTajik/ue5-froxel-lighting, fetched 2026-10-27 (unofficial)
