# Sparse Voxel Octree Construction from Heightmap

Also called: Heightmap-based SVO Construction, Octree Construction via Heightmap Sampling, Top-down SVO Construction from Heightmap

## Overview
This method constructs a Sparse Voxel Octree (SVO) by sampling a noise heightmap at the eight corners of a voxel bounding box, creating a child node only if the height value at a corner lies within the vertical span of the voxel [S1, unofficial]. It differs from general brute-force SVO construction by deriving the 3D structure directly from 2D terrain data without requiring a pre-existing 3D volume or point cloud [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A heightmap object (`heightMap`) providing width and value retrieval; maximum recursion depth (`maxDepth`); octree scale factor (`octreeScale`); initial bounding box (`minBB`, `maxBB`) [S1, unofficial].
*   **Output:** A root voxel node (`root`) representing the constructed SVO [S1, unofficial].

**Data Structures**
The voxel node structure contains the following fields [S1, unofficial]:
*   `childPointers`: A dynamic array (e.g., `std::vector<Voxel*>`) storing pointers to child nodes.
*   `validMask`: An integer bitmask indicating which of the 8 potential children exist. Bit $i$ corresponds to child index $i$.
*   `leafMask`: An integer bitmask indicating which children are leaf nodes.

**Parameters**
*   `octreeScale`: Set to 8, defining the maximum recursion depth for octree construction [S1, unofficial].
*   `minBB`: Initial minimum bounding box, defaulting to $(0,0,0)$ [S1, unofficial].
*   `maxBB`: Initial maximum bounding box, defined as $(\text{heightMap.GetWidth()}, \text{heightMap.GetWidth()}, \text{heightMap.GetWidth})$ [S1, unofficial].

**Algorithm Steps**
1.  **Initialization:** Create the root node [S1, unofficial]. Call the recursive construction function `constructChildren(&root, heightMap, 1, octreeScale, minBB, maxBB)` [S1, unofficial].
2.  **Recursive Construction (`constructChildren`):**
    *   Compute the midpoint of the current bounding box: $mid = (min + max) / 2$ [S1, unofficial].
    *   If `currDepth` $\le$ `maxDepth`, iterate through 8 octants (indices 0-7) [S1, unofficial]. For each octant, define a sub-bounding box using an offset of $0.001f$ to prevent boundary overlap between adjacent voxels [S1, unofficial].
    *   Call `constructNode` for each octant to determine occupancy [S1, unofficial].
    *   If `currDepth` == `maxDepth`, set the `leafMask` [S1, unofficial]. Iterate $i$ from 0 to 7. If bit $i$ is set in `validMask` (checked via `node->validMask & exp2(i)`), set the corresponding bit in `leafMask` (`node->leafMask += exp2(i)`) [S1, unofficial].
    *   Initialize `childCounter` to 0. Iterate through bits 1, 2, 4, 8, 16, 32, 64, 128 of `node->validMask` [S1, unofficial]. If a bit is set, recursively call `constructChildren` on `node->childPointers[childCounter]` with updated bounding box coordinates $(min, mid, max)$, then increment `childCounter` [S1, unofficial].
3.  **Node Construction (`constructNode`):**
    *   Calculate the vertical span of the voxel: $pointSpan = c1.x - c0.x$ (where $c0$ and $c1$ are the min and max coordinates of the voxel) [S1, unofficial].
    *   For each of the eight corners (c0..c7) of the voxel bounding box:
        *   Retrieve the height value from the heightmap [S1, unofficial].
        *   Check if the height value lies within the voxel's vertical span: if $heightVal > c.y$ AND $heightVal < (c.y + pointSpan)$, then the voxel is occupied [S1, unofficial].
        *   If occupied, push a new `Voxel` to `node->childPointers` and add $exp2(idx)$ to `node->validMask` [S1, unofficial].

**Height Value Calculation**
The height value for a corner vertex $c$ is calculated as [S1, unofficial]:
$$heightVal = \left( \frac{\text{heightmap.GetValue}(c.x, c.z) + 1}{2} \right) \times \text{Chunk::CHUNK\_SIZE}$$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Voxel:
    def __init__(self):
        self.childPointers = []
        self.validMask = 0
        self.leafMask = 0

def exp2(n):
    return 1 << n

def construct_children(node, height_map, curr_depth, max_depth, min_bb, max_bb, octree_scale):
    mid_x = (min_bb[0] + max_bb[0]) / 2.0
    mid_y = (min_bb[1] + max_bb[1]) / 2.0
    mid_z = (min_bb[2] + max_bb[2]) / 2.0
    
    if curr_depth <= max_depth:
        for i in range(8):
            x_start = min_bb[0] if (i & 1) == 0 else mid_x
            x_end = mid_x if (i & 1) == 0 else max_bb[0]
            y_start = min_bb[1] if (i & 2) == 0 else mid_y
            y_end = mid_y if (i & 2) == 0 else max_bb[1]
            z_start = min_bb[2] if (i & 4) == 0 else mid_z
            z_end = mid_z if (i & 4) == 0 else max_bb[2]
            
            offset = 0.001
            sub_min = [x_start + offset, y_start + offset, z_start + offset]
            sub_max = [x_end - offset, y_end - offset, z_end - offset]
            
            construct_node(node, height_map, sub_min, sub_max)
            
    if curr_depth == max_depth:
        for i in range(8):
            if (node.validMask & exp2(i)):
                node.leafMask |= exp2(i)
                
    child_counter = 0
    for i in range(8):
        if (node.validMask & exp2(i)):
            x_start = min_bb[0] if (i & 1) == 0 else mid_x
            x_end = mid_x if (i & 1) == 0 else max_bb[0]
            y_start = min_bb[1] if (i & 2) == 0 else mid_y
            y_end = mid_y if (i & 2) == 0 else max_bb[1]
            z_start = min_bb[2] if (i & 4) == 0 else mid_z
            z_end = mid_z if (i & 4) == 0 else max_bb[2]
            
            offset = 0.001
            sub_min = [x_start + offset, y_start + offset, z_start + offset]
            sub_max = [x_end - offset, y_end - offset, z_end - offset]
            
            construct_children(node.childPointers[child_counter], height_map, curr_depth + 1, max_depth, sub_min, sub_max, octree_scale)
            child_counter += 1

def construct_node(node, height_map, min_bb, max_bb):
    point_span = max_bb[1] - min_bb[1]
    CHUNK_SIZE = 8.0
    
    corners = [
        (min_bb[0], min_bb[1], min_bb[2]),
        (max_bb[0], min_bb[1], min_bb[2]),
        (min_bb[0], min_bb[1], max_bb[2]),
        (max_bb[0], min_bb[1], max_bb[2]),
        (min_bb[0], max_bb[1], min_bb[2]),
        (max_bb[0], max_bb[1], min_bb[2]),
        (min_bb[0], max_bb[1], max_bb[2]),
        (max_bb[0], max_bb[1], max_bb[2])
    ]
    
    for idx, c in enumerate(corners):
        raw_val = height_map.get_value(c[0], c[2])
        height_val = ((raw_val + 1) * CHUNK_SIZE) / 2.0
        
        if height_val > c[1] and height_val < (c[1] + point_span):
            if idx >= len(node.childPointers):
                node.childPointers.append(Voxel())
            node.validMask |= exp2(idx)

def build_svo(height_map, max_depth=8):
    min_bb = [0.0, 0.0, 0.0]
    width = height_map.get_width()
    max_bb = [width, width, width]
    
    root = Voxel()
    construct_children(root, height_map, 1, max_depth, min_bb, max_bb, 8)
    return root
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Strengths:** Not given in the sources [S1, unofficial].
*   **Limitations:** Not given in the sources [S1, unofficial].

## Background
This method is described in a senior project on volumetric rendering [S1, unofficial]. It represents a practical application of SVO construction for real-time terrain rendering, leveraging the simplicity of heightmaps to drive the octree hierarchy [S1, unofficial].

## Sources
[S1] Dean904/Senior-Project-Volumetric-Rendering, https://github.com/Dean904/Senior-Project-Volumetric-Rendering, fetched 2026-10-26 (unofficial)
