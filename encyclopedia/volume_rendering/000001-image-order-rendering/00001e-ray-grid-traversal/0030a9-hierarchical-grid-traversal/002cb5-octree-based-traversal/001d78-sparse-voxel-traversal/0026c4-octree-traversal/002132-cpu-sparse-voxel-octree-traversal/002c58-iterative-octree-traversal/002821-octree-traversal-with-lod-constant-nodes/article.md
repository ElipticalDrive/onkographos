# Octree Traversal with LOD & Constant Nodes

Also called: Octree Brick Traversal with LOD & Constant Nodes, VoxViz Traversal, main traversal loop

## Overview
This method renders a Sparse Voxel Octree (SVO) on the CPU by traversing the hierarchy to project voxels into a color buffer, using depth-sorted traversal and hierarchical culling based on projected size [S1, unofficial]. It distinguishes itself from sibling methods by supporting Level of Detail (LOD) switching between constant-value nodes and textured bricks, and by using a hybrid rendering primitive strategy (points vs. rectangles) based on the projected voxel size relative to screen resolution [S1, unofficial]. It builds on [Iterative Octree Traversal](2618-iterative-octree-traversal.md) by adding specific logic for constant node handling and brick texture sampling [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   `nodes`: A flat vector of `Node` structures representing the SVO.
*   `eye`: Camera position (`Vec3A`).
*   `camera`: Camera object providing projection matrix and ray generation.
*   `fb`: Framebuffer (`ColorBuffer`) with width, height, and pixel data.
*   `config`: `DrawConfig` parameters controlling rendering thresholds.

**Outputs:**
*   `fb`: Updated color buffer with rendered voxels.

### Data Structures
**Node Layout**
Each node in the flat `nodes` vector is defined as:
*   `occupancy`: `u8` bitmask where bit `i` corresponds to child index `i`.
*   `children`: Array of 8 `NodeRef` (usize indices into `nodes` vector).
*   `data`: `Payload` containing `color` (`u32` in ABGR format: Alpha 24-31, Red 16-23, Green 8-15, Blue 0-7).
*   `is_leaf()`: Returns true if `occupancy == 0`.
*   `is_occupied(mask)`: Returns true if `(occupancy & mask) != 0`.

**Brick Data**
*   `BrickData`: Holds texture references and GPU upload flags for textured nodes.

### Parameters
*   `min_rect: f32`: Minimum projected Z to draw as a rectangle.
*   `max_rect: f32`: Maximum projected Z to draw as a rectangle; beyond this, culling is tested.
*   `max_test: usize`: Maximum pixel area for culling test.
*   `k_MaxNodesPerPixel`: 12.
*   `k_BrickStepSizeScalar`: Scalar for brick step size calculation.
*   `LodScalar`: Scalar for LOD calculation.
*   `OctTreeDepthMinusOne`: Constant derived from tree depth.

### Algorithm Steps
1.  **Initialization**: Initialize traversal at root: `octTreeNodePtr = ivec3(0)`, fetch root node, set `nodeExtentMin/Max` to volume bounds, `nodeSize = 1.0f`, `voxelHalfSize = RootVoxelHalfSize` [S1, unofficial].
2.  **Projected Voxel Size**: Compute `projVoxelSize = ComputeVoxelSize(rayWorldPosition, voxelHalfSize) * LodScalar` [S1, unofficial].
3.  **LOD Decision**: The condition `PixelSize > projVoxelSize` triggers LOD switching [S1, unofficial].
4.  **Constant Node Handling**: If constant: compute `sampleRatio = pow(k_BrickStepSizeScalar, OctTreeDepthMinusOne - curDepth)`, fetch constant value, call `TraverseConstant` [S1, unofficial].
5.  **Textured Brick Handling**: If textured: compute `sampleRatio = pow(k_BrickStepSizeScalar, OctTreeDepthMinusOne - curBrickDepth)`, compute `worldRayStep = rayDir * RayStepSize * sampleRatio`, call `TraverseBrick` [S1, unofficial].
6.  **Depth Sorting**: Children are traversed in a specific order defined by `HILBERT_ORDER` for general iteration, and `SORTED_ORDER_INDICES` for depth-sorted rendering [G].
7.  **Framebuffer Blending**: If destination pixel is 0, write new pixel. Else, extract destination alpha `dst_a`. If `dst_a < 255`, blend using `a = 256 - dst_a` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_octree(root_node, ray_origin, ray_dir, config):
    # Initialize traversal at root
    oct_tree_node_ptr = 0
    node = root_node
    node_extent_min = config.volume_bounds_min
    node_extent_max = config.volume_bounds_max
    node_size = 1.0
    voxel_half_size = config.root_voxel_half_size

    stack = [(node, 0, 0.0)]  # (node, depth, t)
    while stack:
        node, cur_depth, t = stack.pop()

        # Compute projected voxel size
        proj_voxel_size = compute_voxel_size(ray_origin, voxel_half_size) * config.LodScalar

        # LOD Decision
        if config.PixelSize > proj_voxel_size:
            # LOD switching triggers here
            pass

        if node.is_constant():
            sample_ratio = config.k_BrickStepSizeScalar ** (config.OctTreeDepthMinusOne - cur_depth)
            constant_value = node.data.color
            traverse_constant(constant_value, sample_ratio)
        else:
            sample_ratio = config.k_BrickStepSizeScalar ** (config.OctTreeDepthMinusOne - cur_depth)
            world_ray_step = ray_dir * config.RayStepSize * sample_ratio
            traverse_brick(node, world_ray_step, sample_ratio)

        # Push children in depth-sorted order
        children = get_sorted_children(node, ray_origin)
        for child in children:
            stack.append((child, cur_depth + 1, t))

def compute_voxel_size(ray_pos, voxel_half_size):
    # Standard projection size calculation
    return voxel_half_size / max(1.0, ray_pos.z)

def get_sorted_children(node, eye):
    # Determine dominant axis of eye - stem
    stem = node.center
    diff = eye - stem
    a = abs(diff)
    if a.x > a.y and a.x > a.z:
        order = 0
    elif a.y > a.z:
        order = 1
    else:
        order = 2
    # Return children indices sorted by depth relative to eye
    return sorted(node.children, key=lambda c: dot(c.center - eye, eye))

def traverse_constant(color, sample_ratio):
    # Apply constant color with computed sample ratio
    pass

def traverse_brick(node, world_ray_step, sample_ratio):
    # Sample textured brick along world ray step
    pass
```

### Edge Cases and Pitfalls
*   **LOD Interpolation**: The condition `PixelSize > projVoxelSize` triggers LOD switching, but the source code shows commented-out interpolation logic (`TraverseBrickInterp`) intended to blend between parent/child bricks. This blending is not implemented, potentially causing visual artifacts at LOD boundaries [S1, unofficial].
*   **Precision**: The `depth_to_offset` function uses bit manipulation to approximate powers of two. This may introduce small precision errors for deep trees [G].
*   **Culling**: The `test_rect` function reserves space in the framebuffer if the projected area is below `max_test`. If multiple nodes project to the same area, only the first one (in traversal order) will be drawn, potentially occluding valid geometry if traversal order is not strictly depth-sorted [G].

## Subtopics

### [Sparse Voxel Octree CPU Rasterizer](5183-sparse-voxel-octree-cpu-rasterizer.md)
A CPU-based rasterizer that traverses a Sparse Voxel Octree (SVO) to render voxels directly into a color buffer, utilizing depth-sorted traversal and hierarchical culling strategies that switch between rectangle and point primitives based on projected depth. It distinguishes itself from GPU-based SVO renderers by performing the entire traversal and rasterization pipeline on the CPU, relying on a flat node array and explicit stack management for memory efficiency.

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of visited nodes. Depth sorting adds overhead proportional to the number of internal nodes traversed.
*   **Memory Complexity**: $O(D)$ for the traversal stack, where $D$ is the maximum depth of the octree.
*   **Performance**: The method is CPU-based and relies on efficient stack-based traversal and SIMD-friendly pixel format conversions (ABGR packing/unpacking) [S1, unofficial].
*   **Strengths**:
    *   Supports LOD switching between constant and textured nodes.
    *   Efficient culling using projected size thresholds.
    *   Depth-sorted traversal ensures correct alpha blending.
*   **Limitations**:
    *   Lack of LOD interpolation may cause visual artifacts.
    *   CPU-bound, potentially slower than GPU-based ray marching for very large datasets.
    *   Fixed stack size may limit traversal depth.

## Background
This method originates from the VoxViz project, a CPU-based rasterizer for Sparse Voxel Octrees [S1, unofficial]. The traversal logic is designed to handle both constant-value nodes (for distant or low-detail rendering) and textured bricks (for high-detail rendering), with LOD switching based on projected voxel size [S1, unofficial]. The depth sorting algorithm is based on the work by Iñigo Quilez for volume sorting [S1, unofficial].

## Sources
[S1] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
