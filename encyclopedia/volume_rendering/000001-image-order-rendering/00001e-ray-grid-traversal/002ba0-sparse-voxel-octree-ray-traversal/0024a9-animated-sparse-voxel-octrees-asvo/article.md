# Animated Sparse Voxel Octrees (ASVO)

## Overview
Animated Sparse Voxel Octrees (ASVO) is a GPU-based rendering technique that applies rigid transformations or skinning to voxel-based 3D models in real-time [S1, unofficial]. It differs from standard Sparse Voxel Octree (SVO) ray traversal by decoupling the traversal logic into a parallel job queue system that processes animated leaf nodes, allowing for efficient handling of skeletal animation without reconstructing the octree structure per frame [S1, unofficial].

## How it works
ASVO operates in two main phases: a traversal phase that identifies visible, animated voxels and a drawing phase that resolves pixel coverage and lighting. The method relies on a breadth-first search (BFS) job queue to manage traversal state across the octree hierarchy.

### Inputs and Outputs
**Inputs:**
*   **Octree Data:**
    *   `innerNodes`: Array of `BFSInnerNode` structures. Each contains a `mask` (bitmask of children) and `childPtr` (index offset to children).
    *   `leaves`: Array of `VisualData` structures. Each contains visual properties (normal, tangent, position, texCoord, eyeVec, originalCenter) and animation data (`boneWeights`, `boneIndex0-3`).
    *   `animation`: Array of `float4x4` matrices representing bone transformations for the current frame. Size is `frameCount * boneCount`.
    *   `jobs`: Initial queue of `BFSJob` structures. Each job contains `index` (node index), `x`, `y`, `z` (subdivision coordinates).
    *   `dim`: The total dimension of the octree space.
    *   `level`: The maximum depth of the octree.
    *   `boneCount`: Number of bones in the skeleton.
*   **Camera/View:**
    *   `world`, `view`, `projection`: Transformation matrices.
    *   `camPos`: Camera position in world space.
*   **Lighting:**
    *   `shadowMap`: Depth texture for shadow testing.
    *   `diffuse`, `illum`, `spec`, `normal`: Texture maps for material properties.

**Outputs:**
*   `depthBuffer`: Buffer storing the minimum depth for each screen pixel.
*   `voxelBuffer`: Buffer storing visual data for visible voxels.
*   `colorBuffer`: Final rendered image.

### Data Structures
*   **BFSJob**:
    *   `index`: `uint32_t`, index into `innerNodes` or `leaves`.
    *   `x`, `y`, `z`: `uint32_t`, subdivision coordinates within the parent node.
*   **BFSInnerNode**:
    *   `mask`: `uint64_t`, bitmask where bit $j$ is set if child $j$ exists.
    *   `childPtr`: `uint32_t`, offset to the first child in the `innerNodes` or `leaves` array.
*   **VisualData** (Leaf Node):
    *   `vd.boneWeights`: `float4`, weights for up to 4 bones. If `boneWeights.x > 0.0`, the node is skinned.
    *   `vd.boneIndex0-3`: `uint32_t`, indices of the bones affecting this leaf.
    *   `vd.normal`, `vd.tangent`, `vd.pos`, `vd.texCoord`, `vd.eyeVec`, `vd.originalCenter`: `float3` or `float4`, visual attributes.
    *   `mask`: `uint64_t`, bitmask indicating if the leaf has children (used for traversal continuation).

### Algorithm Steps

1.  **Initialization**:
    *   Load octree data, animation matrices, and initial BFS jobs from the root.
    *   Initialize `depthBuffer` with maximum depth values.
    *   Initialize `sharedTravQueuePtr` to 0.

2.  **Traversal Kernel (Per Job)**:
    Each thread processes one job from the queue.
    
    a. **Grid Cell Calculation**:
       Calculate the dimensions of the current voxel in world space based on the octree level.
       $$ gridCellDim = \frac{dim}{2^{level}} $$
       $$ gridCellHalfDim = gridCellDim \times 0.5 $$
       
    b. **Voxel Center Computation**:
       Determine the center of the voxel in world space.
       *   If the node is a leaf with skinning (`node.vd.boneWeights.x > 0.0`):
           *   Retrieve bone transformations $M_0, M_1, M_2, M_3$ from the `animation` buffer using `boneIndex0-3`.
           *   Interpolate the center position $P_{orig}$ using bone weights $w_0, w_1, w_2, w_3$:
               $$ P_{world} = \sum_{i=0}^{3} w_i \cdot (M_i \cdot P_{orig}) $$
           *   Adjust the half-dimension to counteract stretching during skinning:
               $$ involvedBones = \sum_{i=0}^{3} w_i $$
               $$ gridCellHalfDim = gridCellHalfDim \times \min(2.0, involvedBones) $$
       *   Otherwise, use the original center position from the leaf data.

    c. **Clip Space Transformation**:
       Transform the center and dimensions into clip space.
       $$ P_{clip} = projection \times view \times world \times P_{world} $$
       Compute the eye vector:
       $$ eyeVec = \frac{camPos - P_{world}}{\|camPos - P_{world}\|} $$

    d. **Culling**:
       *   **Frustum Culling**: Check if the voxel's clip-space bounds are within $[-1, 1]$.
       *   **Back-face Culling**: If `level > 8`, check the dot product of the normal and eye vector:
           $$ dot(node.vd.normal, eyeVec) \ge -0.4 $$
       If either culling test fails, discard the voxel.

    e. **Screen Projection and Depth**:
       Project the voxel center to screen coordinates $(x, y)$.
       Compute depth:
       $$ depth = P_{clip}.z \times INT\_MAX $$
       
    f. **Queue Management or Buffer Update**:
       *   If the voxel has children (`d_getChildCountFromMask(node.mask) != 0`) AND spans more than 1 pixel (`dimVec.x > 1.0`):
           *   Generate child jobs for the 8 sub-voxels.
           *   Add child jobs to the queue using `atomicAdd` on `sharedTravQueuePtr`.
       *   Otherwise (leaf node or too small):
           *   Update the depth buffer:
               $$ atomicMin(depthBuffer + index, depth) $$

    g. **Visual Data Storage**:
       If the voxel was drawn (`w==1` in clip space) and its depth matches the updated depth buffer value:
       *   Write visual data (`normal`, `tangent`, `position`, `texCoord`, `eyeVec`, `originalCenter`) to `voxelBuffer` at the screen index.

3.  **Drawing Kernel (Per Pixel)**:
    For each pixel $(x, y)$:
    
    a. **Neighborhood Scan**:
       Scan a $3 \times 3$ neighborhood in the `depthBuffer` to find the voxel with the minimum depth that covers the pixel.
       A voxel covers the pixel if:
       $$ |vd.pos.x - 0.5 - x| \le vd.pos.z $$
       $$ |vd.pos.y - 0.5 - y| \le vd.pos.w $$
       
    b. **Lighting**:
       *   Transform the voxel center to light space.
       *   Check shadow mapping against `shadowMap`.
       *   Compute diffuse and specular lighting using `diffuse`, `illum`, `normal`, and `spec` textures.
       *   Update the `colorBuffer`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Traversal Kernel
def traverse_job(job, innerNodes, leaves, animation, dim, level, camPos, world, view, projection, 
                 depthBuffer, voxelBuffer, sharedTravQueuePtr):
    
    # 1. Grid Cell Calculation
    gridCellDim = dim / (1 << level)
    gridCellHalfDim = gridCellDim * 0.5
    
    # 2. Voxel Center Computation
    node = innerNodes[job.index] if job.index < innerNodeCount else leaves[job.index - innerNodeCount]
    
    if node.is_leaf and node.vd.boneWeights.x > 0.0:
        # Skinning
        P_orig = node.vd.originalCenter
        w = node.vd.boneWeights
        idx0 = node.vd.boneIndex0
        idx1 = node.vd.boneIndex1
        idx2 = node.vd.boneIndex2
        idx3 = node.vd.boneIndex3
        
        M0 = animation[idx0]
        M1 = animation[idx1]
        M2 = animation[idx2]
        M3 = animation[idx3]
        
        P_world = w.x * (M0 * P_orig) + w.y * (M1 * P_orig) + w.z * (M2 * P_orig) + w.w * (M3 * P_orig)
        
        involvedBones = w.x + w.y + w.z + w.w
        gridCellHalfDim *= min(2.0, involvedBones)
    else:
        P_world = node.vd.originalCenter
        
    # 3. Clip Space Transformation
    P_clip = projection * view * world * P_world
    eyeVec = normalize(camPos - P_world)
    
    # 4. Culling
    if not is_in_frustum(P_clip):
        return
    
    if level > 8:
        if dot(node.vd.normal, eyeVec) < -0.4:
            return
            
    # 5. Screen Projection and Depth
    x_screen = P_clip.x / P_clip.w
    y_screen = P_clip.y / P_clip.w
    depth = P_clip.z * INT_MAX
    
    # 6. Queue Management or Buffer Update
    childCount = d_getChildCountFromMask(node.mask)
    dimVec = compute_screen_dimensions(P_clip, gridCellHalfDim)
    
    if childCount != 0 and dimVec.x > 1.0:
        # Add child jobs
        for j in range(8):
            if (node.mask >> j) & 1:
                new_job = BFSJob(
                    index=node.childPtr + child_index,
                    x=2*job.x + (j & 1),
                    y=2*job.y + ((j & 2) >> 1),
                    z=2*job.z + ((j & 4) >> 2)
                )
                atomicAdd(sharedTravQueuePtr, 1)
                queue[sharedTravQueuePtr] = new_job
    else:
        # Update depth buffer
        index = screen_to_index(x_screen, y_screen)
        old_depth = atomicMin(depthBuffer + index, depth)
        
        # 7. Visual Data Storage
        if P_clip.w == 1.0 and depth == old_depth:
            voxelBuffer[index] = node.vd

# Drawing Kernel
def draw_pixel(x, y, depthBuffer, voxelBuffer, shadowMap, diffuse, illum, spec, normal):
    # 1. Neighborhood Scan
    min_depth = INF
    best_voxel = None
    
    for dx in range(-1, 2):
        for dy in range(-1, 2):
            idx = (x + dx) + (y + dy) * width
            d = depthBuffer[idx]
            if d < min_depth:
                min_depth = d
                best_voxel = voxelBuffer[idx]
                
    if best_voxel is None:
        return
        
    # Check coverage
    if abs(best_voxel.pos.x - 0.5 - x) <= best_voxel.pos.z and \
       abs(best_voxel.pos.y - 0.5 - y) <= best_voxel.pos.w:
        
        # 2. Lighting
        P_light = light_view_projection * world * best_voxel.originalCenter
        shadow = shadowMap.sample(P_light.xy / P_light.w, P_light.z / P_light.w)
        
        color = compute_diffuse(best_voxel, diffuse, illum)
        color += compute_specular(best_voxel, spec)
        color *= shadow
        
        colorBuffer[x + y * width] = color
```

### Parameters and Edge Cases
*   **Level Threshold**: Back-face culling is only applied if `level > 8` [S1, unofficial].
*   **Bone Weights**: Skinning is applied only if `boneWeights.x > 0.0` [S1, unofficial].
*   **Precision**: Depth is computed as `center.z * INT_MAX`, which may introduce precision issues for very large scenes or deep octrees [S1, unofficial].
*   **Queue Size**: The traversal uses a dynamic queue with `atomicAdd`, requiring careful management of shared memory or global memory contention [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Traversal is proportional to the number of visible nodes and their children. The drawing phase is $O(1)$ per pixel but involves a $3 \times 3$ neighborhood scan.
*   **Memory Complexity**: Requires storage for `innerNodes`, `leaves`, `animation` matrices, and buffers (`depthBuffer`, `voxelBuffer`, `colorBuffer`). The queue size depends on the branching factor and visibility.
*   **Performance**: High performance on CUDA-capable GPUs (Compute Capability 3.0+) due to parallel job processing [S1, unofficial].
*   **Strengths**: Efficient handling of animated voxel models; no need to rebuild the octree for animation frames.
*   **Limitations**: Limited to rigid transformations and skinning; complex deformations may require more sophisticated approaches. Precision issues with depth buffering in large scenes.

## Background
ASVO was developed as a CUDA/C++ implementation for rendering animated voxel models. It builds on the concept of Sparse Voxel Octrees, extending them to support skeletal animation by applying bone transformations to leaf nodes during traversal. The project includes a high-performance CUDA implementation and an XNA/C# prototype for algorithm design [S1, unofficial].

## Sources
[S1] denniskb/asvo, https://github.com/denniskb/asvo, fetched 2026-10-26 (unofficial)
