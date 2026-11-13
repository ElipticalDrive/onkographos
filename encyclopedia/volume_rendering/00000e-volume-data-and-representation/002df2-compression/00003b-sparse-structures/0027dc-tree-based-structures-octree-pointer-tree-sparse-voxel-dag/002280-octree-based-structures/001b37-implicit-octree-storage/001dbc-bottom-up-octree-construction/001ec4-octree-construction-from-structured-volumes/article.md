# Octree Construction from Structured Volumes

Also called: Adaptive-resolution octree, Lossless-compression octree representation, Octree Volume Construction and Compression, Voxel Consolidation, Zero-variance Octree Construction

## Overview
This method compresses large structured scalar volumes into an adaptive-resolution octree to reduce memory footprint while serving as an acceleration structure for ray tracing [S1, S2, unofficial]. It generates an octree from a 3D grid by consolidating voxels with zero inter-voxel variance, extending the principle of quadtree generation from 2D grids (consolidating pixels with zero inter-pixel variance) to 3D [S1, S2, unofficial]. Unlike [Min/Max Tree Construction](1623-min-max-tree-construction.md) which focuses on the hierarchy bounds, this method focuses on the data consolidation and tree topology. It differs from [Velasco and Torres](https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes) by encapsulating actual individual voxels rather than eight-voxel cells [S1, unofficial].

## How it works
**Inputs:** A 3D scalar array (structured volume) of dimensions $W \times H \times D$.
**Outputs:** An octree structure where leaf nodes store scalar values and interior nodes define the hierarchy.

**Data Structures:**
The octree is stored as a hash map or array where each node is identified by its spatial coordinates and depth.
*   **Node Layout:** Each node contains a scalar value (for leaves) or child pointers (for interior nodes). In the static-resolution implementation, nodes store eight-value arrays for child pointers and scalar leaves [S3, unofficial].
*   **Voxel Mapping:** Voxels are mapped to the lower-left corner of a cell to facilitate ray tracing [S2, unofficial]. Values outside the octree data are defined to be zero [S2, unofficial].

**Algorithm Steps:**
1.  **Initialize:** Start with the finest resolution grid where each voxel is a leaf node.
2.  **Bottom-Up Consolidation:** Iterate from the finest level up to the root. For every group of 8 sibling voxels (a $2 \times 2 \times 2$ block):
    *   Calculate the inter-voxel variance of the 8 children.
    *   If the variance is zero (all 8 voxels have identical scalar values), consolidate them into a single parent node.
    *   The parent node becomes a leaf storing that common scalar value.
    *   If the variance is non-zero, the parent remains an interior node with pointers to its 8 children.
3.  **Repeat:** Continue this process recursively up the hierarchy until the root is reached or no further consolidation is possible.
4.  **Min/Max Tree Construction:** (See [Min/Max Tree Construction](1623-min-max-tree-construction.md) for the specific algorithm, but note that this method embeds the acceleration structure within the same octree framework).

**Formulas:**
*   **Inter-voxel Variance ($\sigma^2$):** For a block of 8 voxels with values $v_1, ..., v_8$:
    $$ \sigma^2 = \frac{1}{8} \sum_{i=1}^{8} (v_i - \mu)^2 $$
    where $\mu = \frac{1}{8} \sum_{i=1}^{8} v_i$.
*   **Consolidation Condition:** A parent node is created if $\sigma^2 = 0$ [S1, S2, unofficial].

**Parameters:**
*   **Variance Threshold:** 0 for lossless compression [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating point precision issues may prevent exact zero variance detection for theoretically identical values. [G] A small epsilon tolerance might be required in implementation, though the source specifies zero variance [S1, unofficial].
*   **Boundary Conditions:** Values outside the octree data are defined to be zero [S2, unofficial]. This affects variance calculations for blocks at the volume boundary if not handled explicitly.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_octree(volume_grid):
    """
    volume_grid: 3D array of scalar values
    Returns: Octree structure (hash map or nested objects)
    """
    # Initialize leaves at finest resolution
    # Each voxel is a leaf node
    leaves = create_leaves(volume_grid)
    
    # Bottom-up construction
    current_level = leaves
    depth = max_depth
    
    while depth > 0:
        next_level = {}
        # Iterate over all possible parent positions at this depth
        for parent_pos in get_parent_positions(current_level):
            children = get_children(current_level, parent_pos)
            
            if len(children) == 8:
                # Calculate variance of the 8 children
                values = [child.value for child in children]
                mean = sum(values) / 8.0
                variance = sum((v - mean)**2 for v in values) / 8.0
                
                if variance == 0:
                    # Consolidate into a leaf node
                    parent_node = Node(type='leaf', value=values[0], pos=parent_pos, depth=depth-1)
                    next_level[parent_pos] = parent_node
                else:
                    # Keep as interior node
                    parent_node = Node(type='interior', children=children, pos=parent_pos, depth=depth-1)
                    next_level[parent_pos] = parent_node
            else:
                # Partial blocks (boundary conditions) remain as children
                for child in children:
                    next_level[child.pos] = child
        
        current_level = next_level
        depth -= 1
        
    return current_level
```

## Subtopics

### Multiresolution Octree Construction with Variance Threshold
A method for building octree volumes where coarser-resolution consolidated voxels are computed and stored in interior nodes, regardless of whether they are leaves, allowing for lossless compression (threshold zero) or lossy compression (variance threshold) [S3, unofficial]. It constructs the octree bottom-up, merging voxels with a variance beneath a given threshold [S3, unofficial]. Nodes store eight-value arrays containing either child pointers or scalar leaves, identical to the static-resolution implementation [S3, unofficial]. It requires a slightly looser min-max tree computation that includes wider neighbors at coarser resolutions [S3, unofficial].

## Operating characteristics
*   **Compression:** The data occupies less than one quarter the size of the original 3D array [S1, unofficial].
*   **Performance:** On an Intel Core Duo 2.16 GHz laptop with 2 GB RAM, frame rates were 2.4, 1.3, and 3.3 fps for three test scenes. On a 16-core NUMA 2.4 GHz Opteron workstation, frame rates were 17.9, 9.8, and 22.0 fps [S1, unofficial].
*   **Strengths:** Reduces memory footprint significantly, allowing storage of large data in main memory. Embeds acceleration tree and scalar data in a single structure [S1, unofficial].
*   **Limitations:** Performance impact of the looser min/max tree in multiresolution mode is negligible but present [S3, unofficial].

## Background
This technique was presented for ray tracing isosurfaces of large compressed structured volumes [S1, S2, unofficial]. It draws inspiration from previous works using min/max trees for extraction and rasterization [S1, S2, unofficial]. The approach is similar to Velasco and Torres but encapsulates actual voxels as opposed to eight-voxel cells [S1, unofficial]. It employs optimized octree hash schemes to achieve competitive frame rates on common multicore architectures [S1, unofficial].

## Sources
[S1] Interactive isosurface ray tracing of large octree volumes, https://www.researchgate.net/publication/310054812_Interactive_isosurface_ray_tracing_of_large_octree_volumes, fetched 2026-10-29 (unofficial)
[S2] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
