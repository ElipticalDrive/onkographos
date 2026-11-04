# Sparse Voxel Octree with Bricks

Also called: SVO with Bricks, SVOC, Sparse Voxel Octree with 3x3x3 Bricks, Voxel Cone Tracing Octree

## Overview
Sparse Voxel Octree with Bricks is a hierarchical scene representation that stores filtered lighting information (energy intensity, direction, occlusion, and Normal Distribution Functions) in small 3x3x3 voxel volumes, called bricks, attached to octree nodes. It solves the memory inefficiency of inter-brick interpolation in sparse octrees by positioning voxel centers at node corners, allowing hardware trilinear interpolation to cover 2x2x2 nodes without expensive one-voxel borders. This method enables smooth, geometry-independent indirect illumination at real-time frame rates, differing from naive node representations by using texture volumes per node rather than single values.

## How it works
**Inputs and Outputs:**
*   **Input:** A scene mesh or volume data; viewing and lighting configuration.
*   **Output:** A sparse voxel octree structure stored in linear GPU memory, with bricks stored in texture memory, enabling filtered scene queries with increasing precision.

**Data Structures:**
1.  **Octree Node:**
    *   Stored in linear GPU memory.
    *   Nodes are grouped into 2x2x2 tiles.
    *   Each tile shares a single pointer to its 2x2x2 children.
    *   Contains a pointer to a 3x3x3 brick in texture memory.
    *   Contains neighbor pointers (to spatially neighboring nodes and parent) [S1].
    *   Empty nodes are collapsed to save memory [S1].
2.  **Brick:**
    *   A 3x3x3 voxel volume stored in texture memory [S1].
    *   Voxel centers are located at the corners of the node's bounding box, not the center [S1].
    *   Approximates the scene section represented by the node [S1].
    *   Allocated in a shared brick buffer [S1].

**Numbered Steps:**
1.  **Tree Construction:** The root node represents the entire scene. Children represent an eighth of the parent's volume. This hierarchy allows querying filtered scene information (energy intensity, direction, occlusion, NDFs) with increasing precision by descending the tree [S1].
2.  **Brick Assignment:** Each non-empty node is assigned a 3x3x3 brick. The brick stores the filtered scene information for that spatial region [S1].
3.  **Voxel Center Placement:** To enable correct inter-brick interpolation using hardware trilinear filtering, the 3x3x3 brick is configured such that its voxel centers align with the corners of the node's bounding box. This ensures that interpolated values for any point within a 2x2x2 node group are computed entirely within a single brick covering those nodes [S1].
4.  **Memory Optimization:** Empty nodes are collapsed. This reduces memory occupancy but requires pointer-based traversal logic to handle the sparse nature of the tree [S1].
5.  **Neighbor Pointers:** Due to memory savings from the 3x3x3 brick design (using less than half the memory of border-based approaches), neighbor pointers are added to nodes. These pointers enable quick visits to spatially neighboring nodes and the parent [S1].
6.  **Grouping:** Nodes are grouped into 2x2x2 tiles. A single pointer stores the children for all 2x2x2 nodes in the tile, optimizing memory layout [S1].

**Formulas:**
*   **Memory Reduction:** The 3x3x3 brick approach uses less than half the memory compared to adding a one-voxel boundary for the same sampling precision [S1].
*   **Precision:** Filtered scene information is queried with increasing precision by descending the tree hierarchy [S1].

**Parameters:**
*   **Brick Size:** 3x3x3 voxels [S1].
*   **Node Tile Size:** 2x2x2 nodes [S1].
*   **Max Octree Resolution:** Typically 512x512 pixels for a 512^3 octree [S1].

**Edge Cases and Pitfalls:**
*   **Inter-brick Interpolation Continuity:** The original approach [CNLE09] used a one-voxel border duplicating neighbors to ensure correct interpolation. This is expensive for small bricks. The 3x3x3 brick with corner-aligned voxel centers solves this by ensuring interpolated values are computed inside a brick covering 2x2x2 nodes [S1].
*   **Empty Nodes:** Collapsing empty nodes reduces memory but requires careful pointer-based traversal logic to avoid accessing invalid memory [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
class Node:
    def __init__(self):
        self.brick_ptr = None  # Pointer to 3x3x3 brick in texture memory
        self.children_ptr = None  # Single pointer to 2x2x2 children tile
        self.parent_ptr = None  # Pointer to parent node
        self.neighbor_ptrs = []  # Pointers to spatially neighboring nodes

class Brick:
    def __init__(self):
        self.voxels = [[[[0.0 for _ in range(3)] for _ in range(3)] for _ in range(3)] for _ in range(3)]
        # 3x3x3 voxels stored in texture memory
        # Voxel centers are at node corners

def query_scene(node, point):
    """
    Query filtered scene information at a point.
    Uses hardware trilinear interpolation on the brick.
    """
    if node is None:
        return None  # Empty node collapsed

    # Determine which brick contains the point
    # The brick covers a 2x2x2 node group
    # Voxel centers are at node corners
    
    # Get brick data
    brick = node.brick_ptr
    
    # Perform trilinear interpolation using hardware texture fetch
    # The interpolation is guaranteed to be within the brick
    # because voxel centers are at node corners
    value = hardware_trilinear_interpolate(brick, point)
    
    return value

def build_octree(scene_data):
    """
    Build the sparse voxel octree with bricks.
    """
    root = Node()
    root.brick_ptr = allocate_brick()
    
    # Recursive construction
    build_recursive(root, scene_data, 0, 0)
    
    return root

def build_recursive(node, scene_data, depth, max_depth):
    """
    Recursively build the octree.
    """
    if depth >= max_depth:
        return
    
    # Check if region is empty
    if is_empty(scene_data, node.bounding_box):
        node.children_ptr = None  # Collapse empty node
        return
    
    # Allocate children tile (2x2x2 nodes)
    node.children_ptr = allocate_tile()
    
    # For each child in the 2x2x2 tile
    for i in range(2):
        for j in range(2):
            for k in range(2):
                child = Node()
                child.parent_ptr = node
                child.brick_ptr = allocate_brick()
                
                # Add neighbor pointers
                add_neighbor_pointers(child, node)
                
                # Recurse
                build_recursive(child, scene_data, depth + 1, max_depth)
                
                # Add to tile
                add_to_tile(node.children_ptr, i, j, k, child)

def add_neighbor_pointers(node, parent):
    """
    Add pointers to spatially neighboring nodes and parent.
    """
    node.parent_ptr = parent
    
    # Find and add neighbors
    # This requires traversing the tree to find adjacent nodes
    # Details omitted as per source [S1]
    pass
```

## Subtopics

### Naive Octree Node Representation
Stores a single value per node. Simpler but lacks the spatial interpolation capabilities of bricks. Choose when memory is extremely constrained and interpolation is not needed. [Naive Octree Node Representation](2445-naive-octree-node-representation.md)

### SVRaster
A rasterization-based approach for indirect illumination. Faster for direct visibility but may lack the smoothness of cone tracing. Choose for real-time direct illumination. [SVRaster](2464-svraster.md)

### Dynamic Update Optimization
Techniques for updating the octree structure efficiently. Choose when the scene changes frequently. [Dynamic Update Optimization](4120-dynamic-update-optimization.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Uses less than half the memory of border-based approaches for the same sampling precision [S1]. Empty nodes are collapsed to save memory [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Handles large and complex scenes through adaptivity [S1].
*   **Strengths:** Smooth results without light undersampling or geometric LOD issues [S1]. Efficient memory usage for small bricks [S1].
*   **Limitations:** Requires pointer-based traversal logic due to collapsed empty nodes [S1].
*   **When to Use:** When smooth, geometry-independent indirect illumination is required at real-time frame rates [S1].
*   **When Not to Use:** When memory is extremely constrained and interpolation is not needed (use Naive Octree Node Representation) [S1].

## Background
The method originates from the paper "Interactive Indirect Illumination Using Voxel Cone Tracing" by [CNLE09] and subsequent work [S1]. It was developed to provide smooth results without missing information like light undersampling or geometric LOD, which are issues in path-tracing or photon mapping [Jen96]. The specific 3x3x3 brick design with corner-aligned voxel centers was proposed to solve the memory inefficiency of inter-brick interpolation in sparse octrees [S1].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
