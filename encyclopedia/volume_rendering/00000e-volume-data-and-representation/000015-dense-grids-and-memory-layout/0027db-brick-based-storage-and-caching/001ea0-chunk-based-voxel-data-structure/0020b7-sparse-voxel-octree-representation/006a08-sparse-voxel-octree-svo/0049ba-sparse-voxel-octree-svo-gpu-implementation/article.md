# Sparse Voxel Octree (SVO) GPU Implementation

Also called: GPU Voxel Octree, Linked nodes in linear video memory, Octree Pool / Brick Pool, Octree terminology and utils, SVO structure, Sparse Voxel Octree (SVO) Data Layout

## Overview
This method implements a Sparse Voxel Octree (SVO) directly on the GPU to support real-time voxel cone tracing and other volume rendering techniques [S2, unofficial]. It solves the memory inefficiency of dense grids by using linked nodes in linear video memory, organized into 2x2x2 node tiles, while storing actual voxel data in a 3D texture (brick pool) to enable hardware tri-linear interpolation [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Scene geometry (meshes) [S2, unofficial].
- Target octree depth (e.g., 9 levels for a $512^3$ resolution) [S2, unofficial].
- Voxelization resolution [S2, unofficial].

**Outputs:**
- An SVO structure ready for traversal, consisting of:
  - `OCTREE_NODE_POOL`: Buffer texture of node pointers [S1, unofficial].
  - `OCTREE_LEVEL_START_INDICES`: Buffer of start indices per level [S1, unofficial].
  - `OCTREE_NODE_POSITIONS`: Buffer of node positions [S1, unofficial].
  - `OCTREE_NODE_POOL_BRICK_POINTERS`: Buffer of pointers to brick data [S1, unofficial].
  - `OCTREE_NODE_POOL_NEIGHBOR_N`: Six buffer textures for neighbor information (X, X_NEGATIVE, Y, Y_NEGATIVE, Z, Z_NEGATIVE) [S1, unofficial].
  - `BRICK_POOL`: 3D Texture storing voxel data [S2, unofficial].

### Data Structures
The implementation uses two primary pools in linear video memory [S2, unofficial]:

1.  **Octree Pool**: Contains linked nodes organized in 2x2x2 node tiles [S2, unofficial]. Each node contains 1 pointer to a node-tile [S2, unofficial]. Each node has 8 elements, each being a pointer to a child node [S1, unofficial]. If the buffer holds pointers, its layout is `r32ui`; if it holds coordinates, it is `rgb10_a2ui` [S1, unofficial].
2.  **Brick Pool**: A 3D Texture storing the actual voxel data (e.g., RGBA32F) [S2, unofficial]. This allows hardware tri-linear interpolation [S2, unofficial].

**Auxiliary Buffers:**
-   `OCTREE_LEVEL_START_INDICES`: Stores the indices of the first node in each level. For an 8-level example, values are `[0, 1, 5, 9, 22, 65, 216, 775]` [S1, unofficial].
-   `OCTREE_NODE_POSITIONS`: Stores the spatial position of each node [S1, unofficial].
-   `OCTREE_NODE_POOL_BRICK_POINTERS`: Stores pointers to the brick data for each node, accessed by the node's index [S1, unofficial].
-   `OCTREE_NODE_POOL_NEIGHBOR_N`: Set of buffer textures storing neighbor information for each node in six directions [S1, unofficial].

### Construction Steps
Sparse voxelization is performed in two steps: 1. Octree subdivision, and 2. Values MIP-mapping [S2, unofficial]. Top-down octree construction uses Compute and Graphics pipelines [S2, unofficial]:

1.  **Voxelize Mesh**: Voxelize the mesh at the target level resolution [S2, unofficial]. Execution uses 1 thread per voxel-fragment [S2, unofficial].
2.  **Tag Octree Nodes**: Process the voxel fragments to tag nodes in the octree [S2, unofficial]. Execution uses 1 thread per node [S2, unofficial].
3.  **Create New Node Tiles**: Allocate and link new node tiles in the Octree Pool [S2, unofficial].

**Constraint:** Every node has at least one child node unless it is the last level [S1, unofficial]. This means there are no leaves on higher levels; leaves are only located at the last level of the octree [S1, unofficial].

### Traversal
Traversal logic is implemented in `_octreeTraversal.glsl` [S1, unofficial]. Functions query the SVO with coordinates and return the node where the query coordinates were found, or `NODE_NOT_FOUND` if not found [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
NODE_NOT_FOUND = -1

def query_svo(node_pool, level_starts, node_positions, coords):
    """
    Queries the SVO with coordinates.
    Returns the node index where coordinates were found, or NODE_NOT_FOUND.
    Based on _octreeTraversal.glsl [S1, unofficial].
    """
    if not level_starts:
        return NODE_NOT_FOUND

    current_idx = level_starts[0]
    for level in range(len(level_starts)):
        # Check if coordinates are within the current node's bounding box [G]
        if not coords_in_node(node_positions[current_idx], coords, level):
            return NODE_NOT_FOUND

        # If not the last level, traverse to the child containing the coordinates
        if level < len(level_starts) - 1:
            child_idx = find_child_node(node_pool[current_idx], coords, level)
            if child_idx == NODE_NOT_FOUND:
                return NODE_NOT_FOUND
            current_idx = child_idx
        else:
            return current_idx
    return NODE_NOT_FOUND

def coords_in_node(node_pos, coords, level):
    """Check if coordinates fall within the node's spatial bounds [G]."""
    # Node size scales with level; implementation depends on coordinate space [G]
    return True

def find_child_node(child_pointers, coords, level):
    """
    Determine which of the 8 children contains the coordinate.
    Returns the child index or NODE_NOT_FOUND if empty.
    """
    # Calculate child index from coordinate offsets relative to node [G]
    child_idx = compute_child_index(coords, level)
    return child_pointers[child_idx] if child_pointers[child_idx] != NODE_NOT_FOUND else NODE_NOT_FOUND

def compute_child_index(coords, level):
    """Map coordinates to one of the 8 octree children [G]."""
    # Standard octree child indexing based on coordinate signs/offsets [G]
    return 0
```

### Edge Cases and Pitfalls
-   **Leaves:** Leaves are only located at the last level of the octree. No leaves exist on higher levels, meaning every node has at least one child unless it is the final level [S1, unofficial].
-   **Precision:** Not given in the sources [S1, unofficial].
-   **Boundaries:** Neighbor buffers (`OCTREE_NODE_POOL_NEIGHBOR_N`) must handle boundary conditions where a node has no neighbor in a specific direction [G].

## Operating characteristics
-   **Time Complexity:** Not given in the sources [S1, unofficial].
-   **Memory Complexity:** Not given in the sources [S1, unofficial].
-   **Measured Performance:** On Kepler GK104 hardware, the implementation is 30% - 58% faster than Fermi GF100, with atomic merging up to 80% faster [S2, unofficial].
-   **Scaling:** Not given in the sources [S2, unofficial].
-   **Strengths:** Avoids plain grid allocation; allows hardware tri-linear interpolation via 3D texture brick pool; flexible and scalable for forward or deferred rendering [S2, unofficial].
-   **Limitations:** Requires explicit management of neighbor buffers and level-start indices; complex construction pipeline involving both Compute and Graphics shaders [S2, unofficial].
-   **When to use:** Real-time voxel cone tracing and indirect illumination where hardware interpolation is beneficial [S2, unofficial].
-   **When not to use:** Not given in the sources [S2, unofficial].

## Background
The Sparse Voxel Octree (SVO) GPU Implementation was presented by Cyril Crassin and Simon Green at Siggraph 2012 in the paper "Octree-Based Sparse Voxelization Using The GPU Hardware Rasterizer" [S2, unofficial]. It builds on earlier work by Olick (2008), Laine and Karras (NVIDIA, 2010), and Crassin et al. (2009, GigaVoxels) [S2, unofficial]. The method was demonstrated in the context of Voxel Cone Tracing for real-time illumination [S2, unofficial].

## Sources
[S1] franciscoaguirre/voxel-cone-tracing, https://github.com/franciscoaguirre/voxel-cone-tracing, fetched 2026-10-26 (unofficial)
[S2] Voxel Cone Tracing and Sparse Voxel Octree for Real-time ..., https://developer.download.nvidia.com/GTC/PDF/GTC2012/PresentationPDF/SB134-Voxel-Cone-Tracing-Octree-Real-Time-Illumination.pdf, fetched 2026-10-30 (unofficial)
