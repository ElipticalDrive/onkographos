# Marching Cubes Edge Processing Logic

Also called: ContourProcessEdge, Edge Processing Logic

## Overview
This method generates two triangles for a voxel edge based on sign changes between the corners of four adjacent nodes, using `processEdgeMask` to identify the specific edge [S1, unofficial]. It differs from standard Marching Cubes by relying on a directional mask and sign-change detection rather than a full 256-case lookup table, and is typically used in dual contouring or octree-based surface extraction where edge topology is managed via node indices [S1, unofficial].

## How it works
The method iterates over four nodes associated with a specific edge direction, identifies the two corners of the voxel edge for each node, checks for a sign change between these corners, and tracks the node with the smallest size to determine the primary vertex location [S1, unofficial]. Based on the sign change at this minimum-size node and a flip flag, it outputs two triangles [S1, unofficial].

### Inputs and Outputs
- **Inputs**:
  - `nodes`: An array of 4 node objects. Each node contains `drawInfo.corners` (integer bitmask of corner signs), `drawInfo.index` (vertex index), and `size` (voxel size/resolution) [S1, unofficial].
  - `dir`: An integer (0, 1, or 2) specifying the axis direction for the edge mask [G].
  - `processEdgeMask`: A 3×4 integer array mapping direction and index to edge indices [S1, unofficial].
  - `edgevmap`: A 12×2 integer array mapping edge indices to corner indices (0–7) within a cube [S1, unofficial].
  - `MATERIAL_AIR`: A constant representing the empty or negative sign state [S1, unofficial].
- **Outputs**:
  - A list of two triangles, each defined by 3 vertex indices, or an empty list if no intersection is detected [S1, unofficial].

### Data Structures
- **`processEdgeMask`**: `int[3][4]`. Maps `(dir, i)` to an edge index. Values: `{{3, 2, 1, 0}, {7, 5, 6, 4}, {11, 10, 9, 8}}` [S1, unofficial].
- **`edgevmap`**: `int[12][2]`. Maps edge index to two corner indices (0–7) within a cube. Used to extract corner signs from `drawInfo.corners` [S1, unofficial].
- **`drawInfo.corners`**: An integer bitmask where bit `k` represents the sign of corner `k` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_edge_logic(nodes, dir, process_edge_mask, edgevmap, material_air):
    """
    Generates two triangles for an edge based on sign changes between corners 
    of 4 adjacent nodes.
    
    Args:
        nodes: List of 4 node objects. Each has .draw_info.corners (int), 
               .draw_info.index (int), and .size (float).
        dir: Integer 0-2 specifying the axis direction.
        process_edge_mask: 3x4 array mapping dir and i to edge index.
        edgevmap: 12x2 array mapping edge index to corner indices.
        material_air: Constant representing the air/negative sign.
        
    Returns:
        List of two triangles (each a tuple of 3 vertex indices), or [].
    """
    min_size = 1000000
    min_index = 0
    indices = [-1, -1, -1, -1]
    flip = False
    sign_change = [False, False, False, False]
    
    for i in range(4):
        edge = process_edge_mask[dir][i]
        c1 = edgevmap[edge][0]
        c2 = edgevmap[edge][1]
        
        m1 = (nodes[i].draw_info.corners >> c1) & 1
        m2 = (nodes[i].draw_info.corners >> c2) & 1
        
        indices[i] = nodes[i].draw_info.index
        
        if nodes[i].size < min_size:
            min_size = nodes[i].size
            min_index = i
            flip = (m1 != material_air)
            
        sign_change[i] = (m1 == material_air and m2 != material_air) or \
                         (m1 != material_air and m2 == material_air)
    
    if sign_change[min_index]:
        if not flip:
            return [(indices[0], indices[1], indices[3]), (indices[0], indices[3], indices[2])]
        else:
            return [(indices[0], indices[3], indices[1]), (indices[0], indices[2], indices[3])]
    else:
        return []
```

### Parameters
- **`processEdgeMask`**: `{{3,2,1,0},{7,5,6,4},{11,10,9,8}}` [S1, unofficial].
- **`edgevmap`**: 12×2 array mapping edge indices to corner pairs [S1, unofficial].
- **`MATERIAL_AIR`**: Constant compared against extracted bits [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision**: If `nodes[i].size == min_size`, the first encountered node is kept [G].
- **Single-Edge Output**: The method only generates triangles for the edge at the finest-resolution node (`min_index`). If multiple edges cross the isosurface, only the highest-resolution one produces output, which can leave topological holes in the mesh [S1, unofficial].
- **Sign Extraction**: The bit shift `(corners >> c) & 1` assumes `corners` is an integer bitmask [S1, unofficial].
- **Flip Logic**: The `flip` variable is set based on `m1 != MATERIAL_AIR` and determines the winding order of the output triangles [S1, unofficial].

## Operating characteristics
- **Time Complexity**: O(1) per call, as it loops over a fixed 4 nodes [G].
- **Memory Complexity**: O(1) auxiliary space [G].
- **Strengths**: Simple implementation, low memory footprint, avoids large lookup tables [S1, unofficial].
- **Limitations**: Only processes the highest-resolution intersecting edge, which may result in non-manifold geometry or holes if lower-resolution edges also intersect the isosurface [S1, unofficial].
- **When to use**: In dual contouring or simplified marching cubes where edge processing is decoupled from a full lookup table [S1, unofficial].
- **When not to use**: When a complete, robust Marching Cubes implementation handling all 256 topological cases is required [S1, unofficial].

## Background
This logic originates from the Dual Contouring sample by Tao Ju, implemented by Nick Gildea [S1, unofficial]. It is used in octree-based surface extraction, specifically within the `SimplifyOctree` function, where nodes are processed to generate mesh geometry while respecting hierarchical resolution [S1, unofficial].

## Sources
[S1] DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
