# Level 2 Triangle-Voxel Classification
Also called: CUDAClassifyTessellationLevel2Kernel, Level 2 Inside/Outside Determination

## Overview
Refines the coarse triangle classification from Level 1 into a finer 3D voxel grid by re-evaluating triangle overlaps and determining inside/outside status for sub-voxels [S1, unofficial]. It builds on [Level 1 Triangle-Voxel Classification](4872-level-1-triangle-voxel-classification.md) by using Level 1 results to limit the search space, processing each Level 2 box against only the triangles assigned to its parent Level 1 box [S1, unofficial]. Unlike Level 1, which maps vertices to coarse indices, Level 2 computes precise geometric intersections (overlap or ray casting) within the finer grid to resolve topology [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Triangle data (`trianglesCUDAData`), Level 1 voxel centers (`level1MidPointCUDAData`), Level 1 triangle counts (`voxelTriCountCUDAData`), Level 1 triangle flat indices (`level1TriFlatIndexCUDAData`), Level 1 triangle flat list (`level1TriFlatCUDAData`), Level 2 to Level 1 box mapping (`level2IndexCUDAData`), grid resolutions (`numDiv`, `numDiv2`), bounding boxes (`objBoxMin`, `objBoxMax`), and voxel extents (`boxExtentsLevel1`, `boxExtentsLevel2`) [S1, unofficial].
*   **Outputs:** Level 2 voxel buffers (`level2InOutCUDAData`, `level2NormalCUDAData`) storing inside/outside flags and/or normals [S1, unofficial].

**Data Structures:**
*   `trianglesCUDAData`: Flat array of floats, 9 values per triangle (3 vertices × 3 coordinates) [S1, unofficial].
*   `level2IndexCUDAData`: Array mapping Level 2 box index to Level 1 box index [S1, unofficial].
*   `level1TriFlatCUDAData`: Flat array of triangle indices, concatenated per Level 1 box [S1, unofficial].
*   `level1TriFlatIndexCUDAData`: Array of starting indices into `level1TriFlatCUDAData` for each Level 1 box [S1, unofficial].
*   `voxelTriCountCUDAData`: Array of triangle counts per Level 1 box [S1, unofficial].
*   `level1MidPointCUDAData`: Array of float3 centers for Level 1 boxes [S1, unofficial].

**Algorithm Steps:**
1.  Each thread `b` processes one Level 2 box.
2.  Retrieve the parent Level 1 box index: `level1BoxIndex = level2IndexCUDAData[b]` [S1, unofficial].
3.  Get the number of triangles in the parent box: `numTriangles = voxelTriCountCUDAData[level1BoxIndex]` [S1, unofficial].
4.  Get the starting index in the flat triangle list: `localIndex = level1TriFlatIndexCUDAData[level1BoxIndex]` [S1, unofficial].
5.  Retrieve the Level 1 box center: `midPoint = level1MidPointCUDAData[b * 3 : b * 3 + 3]` [S1, unofficial].
6.  Compute the Level 2 box coordinates `(p, q, r)` from the global Level 2 box index `localBoxIndex`:
    *   `r = localBoxIndex / (numDiv2.x * numDiv2.y)`
    *   `pqIndex = localBoxIndex - (r * numDiv2.x * numDiv2.y)`
    *   `q = pqIndex / numDiv2.x`
    *   `p = pqIndex % numDiv2.x`
    [S1, unofficial]
7.  Compute the Level 2 box midpoint `boxMidPoint`:
    *   `boxMidPoint.x = (2 * p + 1) * boxExtentsLevel2.x + midPoint.x - boxExtentsLevel1.x`
    *   `boxMidPoint.y = (2 * q + 1) * boxExtentsLevel2.y + midPoint.y - boxExtentsLevel1.y`
    *   `boxMidPoint.z = (2 * r + 1) * boxExtentsLevel2.z + midPoint.z - boxExtentsLevel1.z`
    [S1, unofficial]
8.  Iterate through each triangle `t` in the Level 1 box (from `localIndex` to `localIndex + numTriangles`):
    a. Extract vertices `v0, v1, v2` from `trianglesCUDAData` [S1, unofficial].
    b. Check for overlap between the triangle and the Level 2 box using `TriBoxOverlapCUDA` [S1, unofficial].
    c. If overlapping, compute the face normal:
       *   `e01 = v1 - v0`
       *   `e02 = v2 - v0`
       *   `faceNormal = normalize(cross(e01, e02))`
       [S1, unofficial]
    d. Add the point and normal to the Level 2 voxel buffer via `AddPointNormalToVoxelLevel2` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def classify_level_2(triangles, level1_midpoints, level1_tri_counts, level1_tri_flat_indices, level1_tri_flat, level2_to_level1_map, num_div2, box_extents_l1, box_extents_l2, num_level2_boxes):
    # Output buffers
    level2_normals = [None] * num_level2_boxes
    level2_inside = [False] * num_level2_boxes

    for b in range(num_level2_boxes):
        # 1. Retrieve parent Level 1 box info
        level1_box_idx = level2_to_level1_map[b]
        num_tris = level1_tri_counts[level1_box_idx]
        local_idx = level1_tri_flat_indices[level1_box_idx]
        mid_point = level1_midpoints[b * 3 : b * 3 + 3]

        # 2. Compute Level 2 box coordinates (p, q, r)
        # Assuming localBoxIndex is b for simplicity in this global loop context
        r = b // (num_div2.x * num_div2.y)
        pq_index = b - (r * num_div2.x * num_div2.y)
        q = pq_index // num_div2.x
        p = pq_index % num_div2.x

        # 3. Compute Level 2 box midpoint
        box_mid = [
            (2 * p + 1) * box_extents_l2.x + mid_point.x - box_extents_l1.x,
            (2 * q + 1) * box_extents_l2.y + mid_point.y - box_extents_l1.y,
            (2 * r + 1) * box_extents_l2.z + mid_point.z - box_extents_l1.z
        ]

        # 4. Iterate Level 1 triangles
        for i in range(num_tris):
            tri_idx = level1_tri_flat[local_idx + i]
            # Extract vertices (9 floats per triangle)
            v0 = [triangles[tri_idx * 9], triangles[tri_idx * 9 + 1], triangles[tri_idx * 9 + 2]]
            v1 = [triangles[tri_idx * 9 + 3], triangles[tri_idx * 9 + 4], triangles[tri_idx * 9 + 5]]
            v2 = [triangles[tri_idx * 9 + 6], triangles[tri_idx * 9 + 7], triangles[tri_idx * 9 + 8]]

            # 5. Check overlap
            if tri_box_overlap(v0, v1, v2, box_mid, box_extents_l2):
                # 6. Compute normal
                e01 = [v1[j] - v0[j] for j in range(3)]
                e02 = [v2[j] - v0[j] for j in range(3)]
                normal = normalize(cross(e01, e02))
                level2_normals[b] = normal
                level2_inside[b] = True # Simplified: presence implies inside for this variant

    return level2_inside, level2_normals

def tri_box_overlap(v0, v1, v2, box_mid, box_extents):
    # Placeholder for TriBoxOverlapCUDA logic
    # Returns True if triangle intersects the box centered at box_mid with size box_extents
    return True # [G] Implementation detail omitted in source, assumed to exist
```

**Parameters:**
*   `numDiv2`: Resolution of the Level 2 grid (int3) [S1, unofficial].
*   `boxExtentsLevel1/2`: Size of Level 1 and Level 2 voxels [S1, unofficial].

**Edge Cases:**
*   Precision issues may occur if the ray origin is exactly on a coplanar edge during inside/outside determination [S1, unofficial].
*   Vertices exactly on the object bounding box max limits require clamping to `numDiv - 1` to avoid out-of-bounds indices [S1, unofficial].

## Subtopics

### Level 2 Inside/Outside Determination
Determines inside/outside status for Level 2 voxels by casting a ray along the Z-axis from the voxel center and counting triangle intersections [S1, unofficial]. It differs from the normal-based classification by using a topological ray-casting approach rather than geometric overlap [S1, unofficial]. Choose this when topological correctness (winding number) is required over surface normal estimation [S1, unofficial]. A pitfall is precision errors at coplanar edges; a small epsilon offset to the ray origin is suggested to avoid degenerate intersections [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of Level 2 boxes times the average number of triangles per parent Level 1 box [S1, unofficial].
*   **Memory Complexity:** Linear in the number of Level 2 boxes for output buffers and Level 1 triangle lists [S1, unofficial].
*   **Performance:** Optimized for GPU execution via CUDA kernels, leveraging parallel processing of Level 2 boxes [S1, unofficial].
*   **Strengths:** High precision due to finer grid; efficient culling via Level 1 parent mapping [S1, unofficial].
*   **Limitations:** Higher memory usage than Level 1; sensitive to ray-casting precision issues [S1, unofficial].

## Background
Originated in the GPView project by idealab-isu, implemented as a CUDA kernel for real-time volume rendering tessellation [S1, unofficial]. It extends the coarse voxelization method to provide detailed surface representation for volume rendering pipelines [S1, unofficial].

## Sources
[S1] idealab-isu/GPView, https://github.com/idealab-isu/GPView, fetched 2026-10-26 (unofficial)
