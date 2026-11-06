# Bounding Volume Optimization

Also called: Adaptive Voxel-Array-Größe, BV Optimisation, Bounding Volume Optimisation, Bounding-Volume-Optimierung, SDF Bounding Volume Culling, Volumen-Größenanpassung

## Overview
Bounding Volume Optimization reduces the computational cost of volume rendering by resizing the 3D voxel array to match the actual spatial extents of the data, thereby eliminating unused regions that cause false positives during traversal [S1, unofficial]. Additionally, it minimizes the cost of Signed Distance Field (SDF) map functions by skipping evaluations for objects that do not contribute to the minimal distance field, using bounding volumes to cull expensive geometry [S2]. It sets itself apart from sibling methods like [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md) by focusing on spatial extent reduction and geometric culling rather than value interpolation or binary compression.

## How it works
**Inputs and Outputs:**
*   **Input:** A discrete 3D voxel grid generated via [Voxelization](12-voxelization.md), defined by indices $(i, j, k)$, and a set of geometric primitives or SDF sources.
*   **Output:** A resized voxel array with dimensions matching the data's minimal bounding box, and a traversal/raycasting routine that utilizes bounding volume tests to skip non-contributing geometry.

**Data Structures:**
1.  **Voxel Array:** A 3D array of scalar, vector, or binary values. The dimensions $(D_x, D_y, D_z)$ are determined by the minimal bounding box of the active data.
    *   *Layout:* Contiguous memory or texture format.
    *   *Size:* $D_x \times D_y \times D_z$ voxels.
2.  **Bounding Volumes:** Axis-Aligned Bounding Boxes (AABBs) associated with individual primitives or groups of primitives.
    *   *Fields:* `min` $(x, y, z)$ and `max` $(x, y, z)$ coordinates in world or local space.

**Numbered Steps:**
1.  **Determine Data Extents:** Analyze the voxel grid to find the minimum and maximum indices $(i_{min}, j_{min}, k_{min})$ and $(i_{max}, j_{max}, k_{max})$ that contain non-empty or relevant data [S1, unofficial].
2.  **Resize Array:** Allocate a new voxel array with dimensions:
    $$ D_x = i_{max} - i_{min} + 1 $$
    $$ D_y = j_{max} - j_{min} + 1 $$
    $$ D_z = k_{max} - k_{min} + 1 $$
    Copy the relevant voxel data into this new array, discarding the unused padding [S1, unofficial].
3.  **Compute Bounding Volumes:** For each primitive or SDF source in the scene, compute its AABB [S2].
4.  **Traversal/Culling:** During raymarching or voxel traversal:
    *   Check if the current ray segment or voxel block intersects the bounding volume of a primitive [S2].
    *   If no intersection, skip the evaluation of the primitive's SDF or contribution to the density field [S2].
    *   If intersection, proceed with the expensive SDF evaluation or density calculation [S2].
5.  **Handle Geometry:** For dense voxel models, the bounding box can be interpreted as a rigid body, where voxels serve as contact points and mass parts [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounding_volume_optimization(voxel_grid, primitives):
    # Step 1: Determine data extents
    min_idx = [infinity, infinity, infinity]
    max_idx = [-infinity, -infinity, -infinity]
    
    for i, j, k in voxel_grid.active_voxels():
        min_idx = min(min_idx, [i, j, k])
        max_idx = max(max_idx, [i, j, k])
        
    # Step 2: Resize array
    new_dims = [
        max_idx[0] - min_idx[0] + 1,
        max_idx[1] - min_idx[1] + 1,
        max_idx[2] - min_idx[2] + 1
    ]
    new_grid = allocate_3d_array(new_dims)
    
    for i, j, k in voxel_grid.active_voxels():
        new_grid[i - min_idx[0], j - min_idx[1], k - min_idx[2]] = voxel_grid[i, j, k]
        
    # Step 3: Compute bounding volumes for primitives
    bounding_boxes = []
    for prim in primitives:
        bb = compute_aabb(prim)
        bounding_boxes.append(bb)
        
    return new_grid, bounding_boxes

def raymarch_with_bv_optimization(ray, grid, bounding_boxes):
    # Step 4: Traverse and cull
    for t in ray.step_range():
        pos = ray.position_at(t)
        
        # Check against bounding volumes
        for bb in bounding_boxes:
            if intersects_aabb(bb, ray, t):
                # Evaluate SDF only if bounding volume is hit
                dist = evaluate_sdf(pos, bb.primitive)
                if dist < ray.min_dist:
                    ray.min_dist = dist
            else:
                # Skip expensive SDF evaluation
                continue
                
    return ray.min_dist
```

**Parameters:**
*   **Resolution:** The resolution of the resized array depends on the original voxelization step. Typical values are not specified, but the goal is to minimize $D_x \times D_y \times D_z$ while preserving data fidelity [S1, unofficial].
*   **Bounding Volume Precision:** AABBs are used for simplicity. More complex bounding volumes (e.g., OBBs) are not mentioned in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **False Positives:** A fixed-size array (e.g., $512^3$) loads unnecessary data into GPU memory and causes false positives during traversal if the data does not fill the volume [S1, unofficial].
*   **GPU Architecture:** Modern GPUs are MIMD (Multiple Instruction, Multiple Data) rather than pure SIMD. Different Streaming Multiprocessors (SMs) can process different instruction streams simultaneously. This allows the GPU to finish computation of cheaper parts of the frame early and distribute more expensive threads across available SMs [S2].
*   **Lighting Artifacts:** Using AABBs for voxel models can lead to hard, physically implausible lighting because the normals of the cube faces do not allow for smooth shading [S1, unofficial].

## Subtopics

### 3D Voxel Grid Smoothing
Focuses on interpolating voxel values to reduce noise, not on spatial extent reduction. [Link](833-3d-voxel-grid-smoothing.md)

### Binary Voxel RLE Encoding
Compresses binary voxel data using run-length encoding, potentially within a bounding box, but does not resize the array itself. [Link](1071-binary-voxel-rle-encoding.md)

### Occupancy Voxel Creation
Determines voxel occupancy based on geometric intersection, a prerequisite for defining the data extents. [Link](2628-occupancy-voxel-creation.md)

### Voxelization Optimization
Optimizes the initial generation of the voxel grid, whereas Bounding Volume Optimization optimizes the subsequent storage and traversal. [Link](3191-voxelization-optimization.md)

## Operating characteristics
*   **Time Complexity:** Reduces the number of SDF evaluations and voxel traversals by skipping empty regions and non-contributing primitives. In a scene with 20 expensive primitives at 720p, enabling BV optimization reduced average frame duration from 41 ms (24 fps) to 21 ms (48 fps) [S2].
*   **Memory Complexity:** Reduces memory usage by resizing the voxel array to the minimal bounding box of the data. A fixed $512^3$ array is replaced by a tighter fit (e.g., $512 \times 321 \times 512$), saving significant GPU memory [S1, unofficial].
*   **Scaling:** Performance gains are most significant in scenes with sparse data or many expensive primitives where culling can skip large portions of the computation [S2].
*   **Strengths:** Eliminates false positives, reduces memory bandwidth, and localizes expensive computations to areas with complex geometry [S1, unofficial][S2].
*   **Limitations:** AABBs can cause visually unappealing, hard lighting on voxel models [S1, unofficial]. The optimization requires an initial pass to determine extents and bounding volumes.

## Background
*   **Origin:** The concept of resizing voxel arrays to fit data extents is discussed in the context of raycasting-based voxel rendering for game worlds, where it helps separate game worlds from entities and reduces memory load [S1, unofficial]. The use of bounding volumes to cull SDF evaluations is explored in CUDA-based raymarching distance field renderers [S2].
*   **History:** The technique is part of the broader family of Voxel Grid Optimization methods, which assume a discrete 3D voxel grid has already been generated [S1, unofficial][S2].
*   **Usage:** Used in real-time raymarching renderers and voxel-based physics simulations, where dense voxel models can be interpreted as bounding boxes for rigid bodies [S1, unofficial].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
