# Solid Voxelization via Octree

Also called: Solid Voxelization using Octree

## Overview
Solid Voxelization via Octree converts a set of 3D line segments into a solid voxel representation using an octree data structure, allowing for efficient ray intersection testing against the solid volume [S1, unofficial]. It distinguishes itself from dense voxelization by using hierarchical subdivision to store only occupied regions, and from standard sparse voxelization by explicitly computing solid occupancy rather than just surface presence or density [S1, unofficial].

## How it works
The method consists of three phases: building the octree from input segments, generating a mesh representation for rendering, and traversing the tree for ray intersection [S1, unofficial].

### Inputs and Outputs
*   **Input**: A list of 3D line segments [S1, unofficial].
*   **Output**: An octree data structure representing the solid volume, and optionally a generated mesh for visualization [S1, unofficial].

### Data Structures

**Octree Node**
Each node is a struct aligned to 8 bytes containing three fields [S1, unofficial]:
1.  `_child_offset` (`uint32_t`): Offset to the children array in memory.
2.  `_child_masks` (`uint8_t`): An 8-bit mask representing the occupancy of the 8 children. The bits correspond to children in the order: 000, 001, 011, 010, 110, 111, 101, 100 [S1, unofficial].
3.  `_full` (`bool`): Indicates if the node is completely filled with geometry [S1, unofficial].

**Cube Vertices**
A unit cube is defined by 8 vertices at coordinates $(0,0,0)$ to $(1,1,1)$. The surface is composed of 12 triangles, requiring 36 indices [S1, unofficial].

### Parameters
*   **Resolution**: The side length of the bounding box in voxels. Must be a power of two between 2 and 4096 inclusive [S1, unofficial].

### Algorithm Steps

#### 1. Build Phase
Recursively build the octree. The process depends on the current resolution level [S1, unofficial]:

1.  **Empty Check**: If the list of segments intersecting the current node's bounding box is empty, mark the node as empty and return.
2.  **Leaf Case (Resolution = 2)**:
    *   Divide the current bounding box into 8 sub-cubes.
    *   For each sub-cube, determine occupancy by checking if any segment in the list intersects the Z-range of that sub-cube.
    *   Set the corresponding bit in `_child_masks` if occupied.
    *   Mark `_full` as false.
3.  **Inner Node Case (Resolution > 2)**:
    *   Partition the segment list into 8 subsets based on which sub-cube they intersect. This is done by checking X, then Y, then Z coordinates relative to the node's center.
    *   Recursively build the 8 children using their respective segment subsets.
    *   Compute `_child_masks` by checking if each child is non-empty.
    *   If all 8 children are non-empty and fully filled, mark `_full` as true. Otherwise, `_full` is false.

#### 2. Mesh Generation Phase
Traverse the octree to generate a renderable mesh [S1, unofficial]:

1.  Start traversal at the root node.
2.  For each node:
    *   If `_full` is true, add a single cube mesh for the node's bounding box.
    *   If the node is a leaf (Resolution = 2):
        *   Iterate through the 8 children.
        *   If the corresponding bit in `_child_masks` is set, add a unit cube mesh at the child's local offset.
    *   If the node is an inner node:
        *   Iterate through the 8 children.
        *   If the corresponding bit in `_child_masks` is set, recursively traverse the child node, applying the child's offset to the current position.

#### 3. Ray Traversal Phase
Perform ray intersection testing using a stack-based traversal [S1, unofficial]:

1.  Initialize a stack with the root node.
2.  While the stack is not empty:
    *   Pop the top node.
    *   Compute the intersection of the ray with the node's bounding box.
    *   If the intersection is valid and closer than the current closest hit:
        *   If the node is marked `_full`, update the closest hit to the entry point of the bounding box.
        *   If the node is a leaf (Resolution = 2), check the 8 sub-cubes defined by `_child_masks` for intersection.
        *   If the node is an inner node, push its 8 children (where `_child_masks` bits are set) onto the stack.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Tuple, Optional

TOLERANCE = 1e-5

class Node:
    def __init__(self):
        self.child_offset: int = 0
        self.child_masks: int = 0
        self.full: bool = False
        self.children: List[Node] = []

class Segment:
    def __init__(self, p1: Tuple[float, float, float], p2: Tuple[float, float, float]):
        self.p1 = p1
        self.p2 = p2

class BoundingBox:
    def __init__(self, min_pt: Tuple[float, float, float], max_pt: Tuple[float, float, float]):
        self.min = min_pt
        self.max = max_pt

def build_octree(segments: List[Segment], bbox: BoundingBox, resolution: int) -> Node:
    """
    Recursively build the octree.
    resolution: current side length in voxels.
    """
    node = Node()
    
    # Filter segments intersecting current bbox
    intersecting_segs = []
    for seg in segments:
        if segment_intersects_bbox(seg, bbox):
            intersecting_segs.append(seg)
            
    if not intersecting_segs:
        return node # Empty node

    if resolution == 2:
        # Leaf case: check occupancy of 8 sub-cubes
        sub_cubes = get_sub_cubes(bbox)
        mask = 0
        for i, sub_cube in enumerate(sub_cubes):
            # Check if any segment intersects the Z-range of the sub-cube
            if segment_intersects_z_range(intersecting_segs, sub_cube):
                mask |= (1 << i)
        node.child_masks = mask
        node.full = False
    else:
        # Inner node case: partition segments by X, then Y, then Z relative to center
        sub_cubes = get_sub_cubes(bbox)
        child_segments = [[] for _ in range(8)]
        
        for seg in intersecting_segs:
            mid_x, mid_y, mid_z = (bbox.min[0] + bbox.max[0]) / 2, (bbox.min[1] + bbox.max[1]) / 2, (bbox.min[2] + bbox.max[2]) / 2
            seg_mid = ((seg.p1[0] + seg.p2[0]) / 2, (seg.p1[1] + seg.p2[1]) / 2, (seg.p1[2] + seg.p2[2]) / 2)
            child_idx = 0
            if seg_mid[0] >= mid_x: child_idx |= 1
            if seg_mid[1] >= mid_y: child_idx |= 2
            if seg_mid[2] >= mid_z: child_idx |= 4
            child_segments[child_idx].append(seg)
        
        # Recursively build children
        for i in range(8):
            child_node = build_octree(child_segments[i], sub_cubes[i], resolution // 2)
            node.children.append(child_node)
            if child_node.child_masks != 0:
                node.child_masks |= (1 << i)
        
        # Determine if node is full
        node.full = all(child.full for child in node.children)

    return node

def segment_intersects_bbox(seg: Segment, bbox: BoundingBox) -> bool:
    """Check if a segment intersects a bounding box."""
    min_x, min_y, min_z = bbox.min
    max_x, max_y, max_z = bbox.max
    
    if (seg.p1[0] < min_x and seg.p2[0] < min_x) or (seg.p1[0] > max_x and seg.p2[0] > max_x): return False
    if (seg.p1[1] < min_y and seg.p2[1] < min_y) or (seg.p1[1] > max_y and seg.p2[1] > max_y): return False
    if (seg.p1[2] < min_z and seg.p2[2] < min_z) or (seg.p1[2] > max_z and seg.p2[2] > max_z): return False
    return True

def segment_intersects_z_range(segments: List[Segment], bbox: BoundingBox) -> bool:
    """Check if any segment intersects the Z-range of the bbox."""
    min_z, max_z = bbox.min[2], bbox.max[2]
    for seg in segments:
        seg_min_z = min(seg.p1[2], seg.p2[2])
        seg_max_z = max(seg.p1[2], seg.p2[2])
        if seg_max_z >= min_z and seg_min_z <= max_z:
            return True
    return False

def get_sub_cubes(bbox: BoundingBox) -> List[BoundingBox]:
    """Divide bbox into 8 sub-cubes."""
    min_x, min_y, min_z = bbox.min
    max_x, max_y, max_z = bbox.max
    mid_x, mid_y, mid_z = (min_x + max_x) / 2, (min_y + max_y) / 2, (min_z + max_z) / 2
    
    sub_cubes = []
    for i in range(2):
        for j in range(2):
            for k in range(2):
                new_min = (min_x + i * (mid_x - min_x), min_y + j * (mid_y - min_y), min_z + k * (mid_z - min_z))
                new_max = (mid_x + i * (max_x - mid_x), mid_y + j * (max_y - mid_y), mid_z + k * (max_z - mid_z))
                sub_cubes.append(BoundingBox(new_min, new_max))
    return sub_cubes

def ray_intersect_box_akari(ray_origin: Tuple[float, float, float], ray_dir: Tuple[float, float, float], bbox: BoundingBox) -> Optional[float]:
    """
    Akari's method for ray-box intersection (simplified slab method).
    Returns t_min if hit, else None.
    """
    min_x, min_y, min_z = bbox.min
    max_x, max_y, max_z = bbox.max
    
    tx1 = (min_x - ray_origin[0]) / ray_dir[0]
    tx2 = (max_x - ray_origin[0]) / ray_dir[0]
    ty1 = (min_y - ray_origin[1]) / ray_dir[1]
    ty2 = (max_y - ray_origin[1]) / ray_dir[1]
    tz1 = (min_z - ray_origin[2]) / ray_dir[2]
    tz2 = (max_z - ray_origin[2]) / ray_dir[2]
    
    t_enter = max(min(tx1, tx2), min(ty1, ty2), min(tz1, tz2))
    t_exit = min(max(tx1, tx2), max(ty1, ty2), max(tz1, tz2))
    
    if t_enter > t_exit or t_exit < 0:
        return None
    
    return t_enter

def traverse_octree_for_mesh(node: Node, bbox: BoundingBox, mesh: List) -> None:
    """Generate mesh from octree."""
    if node.full:
        mesh.append(create_cube_mesh(bbox))
        return
    
    if node.child_masks == 0:
        return

    if node.child_masks != 0 and len(node.children) == 0:
        # Leaf case (resolution 2)
        sub_cubes = get_sub_cubes(bbox)
        for i in range(8):
            if node.child_masks & (1 << i):
                mesh.append(create_cube_mesh(sub_cubes[i]))
    else:
        # Inner node case
        sub_cubes = get_sub_cubes(bbox)
        for i in range(8):
            if node.child_masks & (1 << i):
                traverse_octree_for_mesh(node.children[i], sub_cubes[i], mesh)

def create_cube_mesh(bbox: BoundingBox) -> List:
    # Sources do not provide mesh generation implementation details.
    pass

def ray_traverse_octree(ray_origin: Tuple[float, float, float], ray_dir: Tuple[float, float, float], root: Node, bbox: BoundingBox) -> Optional[float]:
    """
    Stack-based ray traversal.
    Returns closest t hit, or None.
    """
    stack = [(root, bbox)]
    closest_t = float('inf')
    
    while stack:
        node, node_bbox = stack.pop()
        
        t_hit = ray_intersect_box_akari(ray_origin, ray_dir, node_bbox)
        if t_hit is not None and t_hit < closest_t:
            if node.full:
                closest_t = t_hit
            elif node.child_masks != 0:
                if len(node.children) == 0:
                    # Leaf case
                    sub_cubes = get_sub_cubes(node_bbox)
                    for i in range(8):
                        if node.child_masks & (1 << i):
                            t_sub = ray_intersect_box_akari(ray_origin, ray_dir, sub_cubes[i])
                            if t_sub is not None and t_sub < closest_t:
                                closest_t = t_sub
                else:
                    # Inner node case
                    sub_cubes = get_sub_cubes(node_bbox)
                    for i in range(8):
                        if node.child_masks & (1 << i):
                            stack.append((node.children[i], sub_cubes[i]))
                            
    return closest_t if closest_t != float('inf') else None
```

### Edge Cases and Pitfalls
*   **Floating Point Precision**: Box intersection uses a tolerance of $10^{-5}$ to determine which face was hit, which can affect accuracy in tight intersections [S1, unofficial].
*   **Resolution Limits**: Resolution must be a power of two between 2 and 4096. Values outside this range are invalid [S1, unofficial].
*   **Segment Intersection Logic**: The leaf occupancy check relies on Z-range intersection, which may not fully capture 3D intersection for segments parallel to the Z-axis or lying exactly on boundaries [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Each node uses 8 bytes [S1, unofficial].
*   **Strengths**: Enables efficient ray intersection testing against the solid volume [S1, unofficial].
*   **Limitations**: Resolution is limited to powers of two up to 4096; floating-point precision issues in intersection tests [S1, unofficial].
*   **When to Use**: Suitable for real-time rendering of sparse solid volumes where ray intersection performance is critical [S1, unofficial].
*   **When Not to Use**: Not given in the sources.

## Background
Solid Voxelization via Octree is a technique to convert a set of 3D line segments into a solid voxel representation using an octree data structure [S1, unofficial]. It is used in real-time rendering applications, as demonstrated by the accompanying GPU-based ray tracer in the source code [S1, unofficial].

## Sources
[S1] Mike-Leo-Smith/SolidVoxelization, https://github.com/Mike-Leo-Smith/SolidVoxelization, fetched 2026-10-27 (unofficial)
