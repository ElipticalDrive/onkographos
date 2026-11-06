# Solid Voxelization via Octree

Also called: Solid Voxelization using Octree

## Overview
Solid Voxelization via Octree is a technique to convert a set of 3D line segments (typically derived from triangle meshes) into a solid voxel representation using an octree data structure [S1, unofficial]. This method enables efficient ray intersection testing against the solid volume, serving as a bridge between polygonal geometry and volume rendering [S1, unofficial]. It distinguishes itself from sibling methods like [Out-of-Core Sparse Voxel Octree Construction](1689-out-of-core-sparse-voxel-octree-construction.md) by focusing on generating a "solid" (filled) representation suitable for direct ray casting, rather than just storing sparse density fields or neural features.

## How it works

### Inputs and Outputs
*   **Input**: A set of 3D line segments (edges) extracted from a mesh, and a target `resolution` for the bounding box.
*   **Output**: An octree data structure representing the solid volume, and optionally a generated mesh of the voxelized surface.

### Data Structures

**Octree Node**
The node is a struct aligned to 8 bytes containing three fields [S1, unofficial]:
1.  `_child_offset` (`uint32_t`): Offset to the first child node in a flat array or pointer.
2.  `_child_masks` (`uint8_t`): An 8-bit mask representing the occupancy of the 8 children. The bits correspond to children in the order: 000, 001, 011, 010, 110, 111, 101, 100 (Morton order) [S1, unofficial].
3.  `_full` (`bool`): Indicates if the node is completely filled with geometry.

**Cube Vertices**
A unit cube at the origin $(0,0,0)$ to $(1,1,1)$ is defined by 8 vertices. Indices define 12 triangles (36 indices) forming the cube surface [S1, unofficial].

### Parameters
*   **resolution**: The side length of the bounding box in voxels. Must be a power of two between 2 and 4096 inclusive [S1, unofficial].

### Algorithm Steps

1.  **Build Phase**: Recursively build the octree.
    *   If the list of segments intersecting the node's bounding box is empty, the node is marked empty.
    *   If the current resolution is 2 (leaf level), determine the occupancy of the 8 sub-cubes by checking if any segment intersects the Z-range of the sub-cube.
    *   If the resolution is greater than 2, partition the segments by X, then Y, then Z coordinates to distribute them to the 8 children. Recursively build the children. Set the `_child_masks` based on which children are non-empty.

2.  **Mesh Generation Phase** (Optional, for visualization):
    *   Traverse the octree.
    *   If a node is marked `_full`, add a single cube mesh for the node's bounding box.
    *   If the node is a leaf (resolution 2), iterate over the 8 children. If a child's mask bit is set, add a unit cube mesh at the child's offset.
    *   If the node is an inner node, iterate over the 8 children. If a child's mask bit is set, recursively traverse the child node at the offset plus the child's index.

3.  **Ray Traversal Phase**:
    *   Use a stack-based traversal starting with the root node.
    *   For each node, compute the intersection with its bounding box.
    *   If the hit is valid and closer than the current closest hit:
        *   If the node is `_full`, update the closest hit to the entry point of the node.
        *   If the node is a leaf, check the 8 sub-cubes for intersections.
        *   If the node is an inner node, push its children to the stack for further processing.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class OctreeNode:
    def __init__(self):
        self.child_offset = 0
        self.child_masks = 0
        self.full = False
        self.children = []

def build_octree(segments, min_coord, max_coord, resolution):
    """
    Recursively builds the octree.
    segments: List of line segments (p1, p2) intersecting the current bounding box.
    min_coord: (x, y, z) minimum of current node.
    max_coord: (x, y, z) maximum of current node.
    resolution: Current resolution (side length in voxels).
    """
    node = OctreeNode()
    
    # Base case: Leaf node
    if resolution == 2:
        # Determine occupancy of 8 sub-cubes
        # Check if any segment intersects the Z-range of each sub-cube
        # This is a simplified check based on source notes
        for i in range(8):
            # Calculate sub-cube bounds
            sub_min, sub_max = get_subcube_bounds(min_coord, max_coord, i)
            if any(segment_intersects_z_range(seg, sub_min[2], sub_max[2]) for seg in segments):
                node.child_masks |= (1 << i)
        return node

    # Recursive case: Inner node
    mid = (min_coord + max_coord) / 2.0
    
    # Partition segments by X, Y, Z
    # This step distributes segments to children
    child_segments = [[] for _ in range(8)]
    for seg in segments:
        # Determine which children the segment intersects
        # Based on source: partition by X, then Y, then Z
        child_indices = get_intersecting_children(seg, min_coord, max_coord)
        for idx in child_indices:
            child_segments[idx].append(seg)
            
    # Recursively build children
    for i in range(8):
        if child_segments[i]:
            sub_min, sub_max = get_subcube_bounds(min_coord, max_coord, i)
            child_node = build_octree(child_segments[i], sub_min, sub_max, resolution // 2)
            node.children.append(child_node)
            node.child_masks |= (1 << i)
            
    # Check if node is full
    if node.child_masks == 0xFF: # All 8 children present
        # Further check if all children are full or if the volume is solid
        # Source implies setting mask based on non-empty children
        pass
        
    return node

def ray_traverse(node, ray_origin, ray_direction, current_min_hit):
    """
    Stack-based ray traversal.
    """
    stack = [node]
    closest_hit = current_min_hit
    
    while stack:
        current_node = stack.pop()
        
        # Compute intersection with bounding box
        hit = intersect_box(current_node.bounds, ray_origin, ray_direction)
        
        if hit and hit.t < closest_hit:
            if current_node.full:
                closest_hit = hit.t
                # Return or store hit
            elif current_node.children:
                # Push children to stack
                for i, child in enumerate(current_node.children):
                    if current_node.child_masks & (1 << i):
                        stack.append(child)
            else:
                # Leaf node check
                # Check 8 sub-cubes
                for i in range(8):
                    if current_node.child_masks & (1 << i):
                        sub_hit = intersect_subcube(current_node, i, ray_origin, ray_direction)
                        if sub_hit and sub_hit.t < closest_hit:
                            closest_hit = sub_hit.t
                            
    return closest_hit

def intersect_box_akari(box_min, box_max, ray_o, ray_d):
    """
    Simplified slab method using component-wise max/min.
    Uses a tolerance of 1e-5f for floating point comparison.
    """
    tol = 1e-5
    t_min = -math.inf
    t_max = math.inf
    
    for i in range(3):
        if abs(ray_d[i]) < tol:
            if ray_o[i] < box_min[i] or ray_o[i] > box_max[i]:
                return None
        else:
            t1 = (box_min[i] - ray_o[i]) / ray_d[i]
            t2 = (box_max[i] - ray_o[i]) / ray_d[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
            
    if t_min > t_max:
        return None
        
    return t_min
```

### Edge Cases and Pitfalls
*   **Floating Point Comparison**: Box intersection uses a tolerance of $10^{-5}$ to determine which face was hit [S1, unofficial].
*   **Resolution Constraints**: The resolution must be a power of two between 2 and 4096. Non-power-of-two resolutions are not supported [S1, unofficial].
*   **Segment Intersection**: The leaf occupancy check relies on Z-range intersection, which may be sensitive to the orientation of segments relative to the voxel grid.

## Operating characteristics

*   **Time Complexity**: Building the octree is $O(N \log N)$ where $N$ is the number of segments, due to the recursive partitioning. Ray traversal is $O(\log V)$ where $V$ is the number of voxels, but can be slower in complex scenes with many small voxels.
*   **Memory Complexity**: The octree stores only non-empty nodes, making it sparse. However, the "solid" nature means that even empty space inside a bounding box may be filled if it is enclosed by geometry, potentially increasing memory usage compared to purely sparse representations.
*   **Strengths**: Efficient ray intersection testing; suitable for real-time applications; handles complex geometry well.
*   **Limitations**: Resolution is fixed and must be a power of two; may consume significant memory for large, detailed scenes if not carefully managed.
*   **When to use**: When you need to convert mesh data into a voxel format for ray casting or volume rendering, and performance is critical.
*   **When not to use**: For scenes with extremely high detail requiring adaptive resolution beyond the fixed octree levels, or when memory is extremely constrained.

## Background

This method is implemented in the [Mike-Leo-Smith/SolidVoxelization](https://github.com/Mike-Leo-Smith/SolidVoxelization) repository [S1, unofficial]. It is designed as a practical tool for converting 3D models into voxel data for rendering purposes. The approach builds on general principles of sparse and hierarchical structures, specifically using octrees for spatial decomposition.

## Sources

[S1] Mike-Leo-Smith/SolidVoxelization, https://github.com/Mike-Leo-Smith/SolidVoxelization, fetched 2026-10-27 (unofficial)
