# Ray Origin Jitter for Mesh Voxelization

Also called: Jittered Ray Origins, STL Voxelization Artifacts Fix

## Overview
This method eliminates vertical slice artifacts in mesh voxelization by introducing a small, direction-aware offset to ray origins before performing ray-triangle intersection tests [S1, unofficial]. It specifically addresses numerical instability in the Möller-Trumbore algorithm when rays align exactly with mesh vertices or edges, a common issue in meshes with regular grid topology such as terrain height maps [S1, unofficial]. Unlike alternative approaches that bypass ray-tracing entirely (e.g., using Signed Distance Fields), this technique preserves the geometric fidelity of the original mesh while ensuring robust intersection detection [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Voxel grid position (`xyz`), voxel size, ray direction (`direction`), and an initial offset (`offset`) derived from the voxel position.
*   **Output:** A jittered ray origin vector (`r_origin`) used for the subsequent Möller-Trumbore intersection test.

**Data Structures**
*   `xyz`: Integer or float coordinates of the voxel center or corner.
*   `direction`: An integer or boolean mask indicating the primary axis of the ray (0 for X, 1 for Y, 2 for Z).
*   `offset`: A vector representing the displacement from the voxel center to the ray start point (often half-voxel or zero depending on the specific voxelization scheme).

**Parameters**
*   `jitter`: The magnitude of the offset. The standard value is `0.001f`, which represents 0.1% of the voxel size [S1, unofficial]. This value is small enough to not affect the geometric accuracy of the voxelization but large enough to break exact coplanarity [S1, unofficial].

**Steps**
1.  Calculate the base ray origin by adding the voxel position and any predefined offset:
    $$ r_{origin} = position(xyz) + offset $$
2.  Define the jitter magnitude constant:
    $$ jitter = 0.001 \times voxel\_size $$
3.  Apply the jitter to the components of `r_origin` that are perpendicular to the ray direction. The jitter is added to component $N$ only if the ray direction is not aligned with axis $N$ [S1, unofficial].
    *   If `direction` is 0 (X-ray), jitter is applied to Y and Z components.
    *   If `direction` is 1 (Y-ray), jitter is applied to X and Z components.
    *   If `direction` is 2 (Z-ray), jitter is applied to X and Y components.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jitter_ray_origin(position_xyz, offset, direction, voxel_size):
    """
    Calculate a jittered ray origin for mesh voxelization.
    
    Args:
        position_xyz: Vector3, the base position of the voxel.
        offset: Vector3, additional offset (e.g., half-voxel).
        direction: int, 0 for X-aligned, 1 for Y-aligned, 2 for Z-aligned.
        voxel_size: float, the side length of a single voxel.
        
    Returns:
        Vector3, the jittered ray origin.
    """
    # Step 1: Calculate base origin
    r_origin = position_xyz + offset
    
    # Step 2: Define jitter magnitude (0.1% of voxel size)
    jitter = 0.001 * voxel_size
    
    # Step 3: Apply jitter perpendicular to ray direction
    # direction != 0 checks if the ray is NOT aligned with the X-axis
    # If true, add jitter to the X component of the origin
    if direction != 0:
        r_origin.x += jitter
        
    # If the ray is NOT aligned with the Y-axis, add jitter to the Y component
    if direction != 1:
        r_origin.y += jitter
        
    # If the ray is NOT aligned with the Z-axis, add jitter to the Z component
    if direction != 2:
        r_origin.z += jitter
        
    return r_origin
```

**Edge Cases and Pitfalls**
*   **Regular Grid Topology:** The primary failure case occurs with meshes generated from height maps or regular grids (e.g., `hill.stl` with 421 Y-aligned planes) where vertices lie on exact integer or half-integer coordinates [S1, unofficial]. Without jitter, rays often start exactly on vertex planes, leading to undefined intersection results or missed voxels [S1, unofficial].
*   **Precision:** The jitter magnitude must be chosen carefully. Too small, and it fails to break coplanarity due to floating-point precision limits; too large, and it introduces visible artifacts or inaccuracies in the voxel boundary [S1, unofficial]. The value `0.001f` (0.1%) has been empirically shown to be imperceptible in final simulations while effectively solving the issue [S1, unofficial].
*   **Direction Alignment:** The jitter must *not* be applied along the ray's primary direction of travel, as this would shift the ray along its own path without addressing the coplanarity issue with the triangle's plane [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel. The jitter calculation adds a constant number of arithmetic operations and comparisons to the existing voxelization loop.
*   **Memory Complexity:** $O(1)$ additional memory. No new data structures are required; the jitter is computed on the fly.
*   **Performance:** Negligible performance impact. The added operations are simple integer comparisons and floating-point additions, which are extremely fast on modern GPUs and CPUs [S1, unofficial].
*   **Strengths:**
    *   Effectively eliminates vertical slice artifacts in voxelized terrain and regular meshes [S1, unofficial].
    *   Simple to implement and integrate into existing ray-triangle intersection kernels (e.g., Möller-Trumbore) [S1, unofficial].
    *   Preserves the original mesh geometry, unlike SDF-based approaches which may smooth or alter details [S1, unofficial].
*   **Limitations:**
    *   Only addresses coplanarity issues related to ray origins. It does not fix other numerical instabilities in ray-triangle intersection (e.g., near-parallel rays) [S1, unofficial].
    *   Requires knowledge of the ray direction and voxel size.
    *   May not be sufficient for highly irregular or non-manifold meshes where other intersection failures occur [S1, unofficial].
*   **When to use:** Use this method when voxelizing STL meshes (especially terrain/heightmap data) using ray-triangle intersection and observing vertical gaps or missing voxels. It is particularly effective for meshes with regular grid topology [S1, unofficial].
*   **When not to use:** If the mesh is highly irregular or if the voxelization pipeline already uses a robust intersection algorithm with epsilon handling, this may be unnecessary. For complex geometries where ray-tracing is too slow, consider SDF-based voxelization instead [S1, unofficial].

## Background
This technique was proposed in a GitHub pull request to the FluidX3D project, a lattice Boltzmann CFD software, to fix vertical slice artifacts in STL mesh voxelization [S1, unofficial]. The issue was identified when voxelizing terrain meshes with regular grid structures, where the Möller-Trumbore algorithm failed due to rays aligning exactly with mesh vertices [S1, unofficial]. The solution, introduced by user `meet-brad-ch` in December 2025, adds a small jitter to ray origins perpendicular to the ray direction [S1, unofficial]. It was tested on a `hill.stl` terrain mesh with 421 Y-aligned planes, showing a transition from visible artifacts to clean voxelization [S1, unofficial].

## Sources
[S1] Fix vertical slice artifacts in mesh voxelization, https://github.com/ProjectPhysX/FluidX3D/pull/323, fetched 2026-10-26 (unofficial)
