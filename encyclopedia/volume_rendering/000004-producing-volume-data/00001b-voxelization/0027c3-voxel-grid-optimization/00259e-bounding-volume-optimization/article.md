# Bounding Volume Optimization

Also called: Adaptive Voxel-Array-Größe, BV Optimisation, Bounding Volume Optimisation, Bounding-Volume-Optimierung, SDF Bounding Volume Culling, Volumen-Größenanpassung

## Overview
Bounding Volume Optimization reduces the computational cost of volume rendering by eliminating false positives during traversal and skipping expensive Signed Distance Field (SDF) evaluations for objects that do not contribute to the minimal distance field [S1, unofficial][S2]. It sets itself apart from sibling methods like [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md) or [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) by focusing on spatial culling and array dimensionality reduction rather than data compression or noise reduction [S1, unofficial][S2]. Unlike [Voxelization Optimization](3191-voxelization-optimization.md), which addresses the generation phase, this method operates on the final voxel data structure to minimize the cost of the map function during rendering [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A populated 3D voxel array (occupancy, density, or SDF values) defined on a discrete integer grid $(i, j, k)$ [G].
*   **Output:** An optimized voxel data structure with reduced dimensions or associated bounding volumes that allow early exit during traversal [S1, unofficial][S2].

**Data Structures:**
1.  **Adaptive Bounding Box:** A 3D array of size $W \times H \times D$ where $W, H, D$ correspond to the actual spatial extents of the non-empty voxels, replacing a fixed-size container (e.g., $512^3$) [S1, unofficial].
2.  **Primitive Bounding Volumes:** Axis-Aligned Bounding Boxes (AABBs) associated with individual geometric primitives or object clusters to gate SDF evaluation [S2].

**Steps:**
1.  **Determine Spatial Extents:** Scan the voxel grid to find the minimum and maximum indices $(i_{min}, j_{min}, k_{min})$ and $(i_{max}, j_{max}, k_{max})$ containing non-empty data [S1, unofficial].
2.  **Resize Array (Adaptive Sizing):** Allocate a new 3D texture or array with dimensions:
    $$ W = i_{max} - i_{min} + 1 $$
    $$ H = j_{max} - j_{min} + 1 $$
    $$ D = k_{max} - k_{min} + 1 $$
    Copy the relevant voxel data into this new structure [S1, unofficial]. This eliminates unused regions that cause false positives during traversal [S1, unofficial].
3.  **Compute Primitive Bounding Volumes:** For each primitive contributing to the scene's SDF, compute its AABB in world or local space [S2].
4.  **Traversal with Culling:** During raymarching, for each step:
    a. Check if the current ray position intersects the global adaptive bounding box. If not, terminate traversal [S1, unofficial].
    b. For each primitive, check if the ray intersects the primitive's bounding volume.
    c. If the ray does not intersect the bounding volume, skip the evaluation of that primitive's SDF [S2].
    d. Evaluate the SDF only for primitives whose bounding volumes are intersected [S2].

**Formulas:**
*   **Array Reduction Factor:** The memory and traversal cost reduction is proportional to the ratio of the optimized volume to the original fixed volume:
    $$ \text{Reduction} = \frac{W \cdot H \cdot D}{W_{fixed} \cdot H_{fixed} \cdot D_{fixed}} $$
    Example: Reducing from $512^3$ to $512 \times 321 \times 512$ saves significant memory and traversal steps [S1, unofficial].

**Parameters:**
*   **Fixed Grid Size:** Often a power of two (e.g., $512^3$) before optimization [S1, unofficial].
*   **Resolution:** Dependent on the scene; e.g., 720p rendering [S2].

**Edge Cases and Pitfalls:**
*   **False Positives:** A fixed-size array (e.g., $512^3$) loads unnecessary data into GPU memory and causes false positives during traversal if the actual data occupies a smaller subspace (e.g., $512 \times 191 \times 512$) [S1, unofficial].
*   **Lighting Artifacts:** Using AABBs for bounding volumes can result in hard, physically implausible lighting because the normals of the cube faces do not align with the curved geometry, leading to harsh shadows or incorrect diffuse lighting [S1, unofficial].
*   **GPU Architecture:** Modern GPUs are MIMD (Multiple Instruction, Multiple Data) rather than pure SIMD. Different Streaming Multiprocessors (SMs) can process different instruction streams. This allows the GPU to finish cheaper computations (culling) early and distribute expensive SDF evaluations across available SMs [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounding_volume_optimization(voxel_data, primitives):
    # Step 1: Determine extents of non-empty voxels
    min_i, max_i, min_j, max_j, min_k, max_k = find_extents(voxel_data)
    
    # Step 2: Resize array
    w = max_i - min_i + 1
    h = max_j - min_j + 1
    d = max_k - min_k + 1
    optimized_voxels = resize_array(voxel_data, w, h, d, min_i, min_j, min_k)
    
    # Step 3: Compute bounding volumes for primitives
    bounding_volumes = []
    for prim in primitives:
        bv = compute_aabb(prim)
        bounding_volumes.append(bv)
        
    return optimized_voxels, bounding_volumes

def raymarch_with_bv_optimization(ray, optimized_voxels, bounding_volumes, primitives):
    t = ray.origin
    while t < ray.max_distance:
        # Check global adaptive bounding box
        if not point_in_box(t, optimized_voxels.bounds):
            break
            
        # Check primitive bounding volumes
        active_primitives = []
        for i, bv in enumerate(bounding_volumes):
            if ray_intersects_aabb(ray, bv):
                active_primitives.append(primitives[i])
                
        # Evaluate SDF only for active primitives
        min_dist = infinity
        for prim in active_primitives:
            dist = evaluate_sdf(prim, t)
            if dist < min_dist:
                min_dist = dist
                
        if min_dist < epsilon:
            return hit(t, min_dist)
            
        t += min_dist
        
    return miss()
```

## Subtopics

### Adaptive Voxel-Array-Größe
Focuses on resizing the 3D array dimensions to match the actual data extent, reducing memory load and traversal false positives [S1, unofficial]. Link: [Bounding Volume Optimization](52-bounding-volume-optimization.md).

### SDF Bounding Volume Culling
Focuses on using AABBs to skip SDF evaluations for primitives not intersected by the ray, localizing expensive computations [S2]. Link: [Bounding Volume Optimization](52-bounding-volume-optimization.md).

## Operating characteristics
*   **Time Complexity:** Reduces the number of SDF evaluations per frame. In a scene with 20 expensive primitives at 720p, enabling BV optimization reduced average frame duration from 41 ms (24 fps) to 21 ms (48 fps) [S2].
*   **Memory Complexity:** Reduces memory footprint by allocating only the necessary volume dimensions (e.g., $512 \times 321 \times 512$ instead of $512^3$) [S1, unofficial].
*   **Strengths:** Eliminates false positives from unused array regions; localizes expensive computations to areas with complex geometry; leverages GPU MIMD capabilities to distribute workload [S1, unofficial][S2].
*   **Limitations:** AABBs can cause visually unappealing, hard lighting artifacts due to mismatched normals [S1, unofficial].
*   **When to use:** When the voxel data occupies a small fraction of a fixed-size grid, or when the scene contains many expensive SDF primitives [S1, unofficial][S2].
*   **When not to use:** When lighting quality is critical and AABB artifacts are unacceptable without further post-processing [S1, unofficial].

## Background
The concept of adapting voxel array sizes to reduce redundancy was discussed in the context of raycasting-based voxel rendering, particularly for separating game worlds from entities and optimizing memory usage [S1, unofficial]. The application of bounding volume optimization to minimize SDF evaluation costs in raymarchers was explored in CUDA-based distance field rendering [S2].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
