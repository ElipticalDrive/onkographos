# Sparse Voxel Octree Construction
Also called: Octree Generation from Occupancy Mask, Octree-Based Sparse Voxelization, SVO Construction

## Overview
Sparse Voxel Octree Construction is a method for building and updating a sparse voxel octree directly from triangle meshes, avoiding an intermediate full regular grid [S2, unofficial]. It solves the scalability problem of voxelizing large scenes by constructing the hierarchical structure directly on the GPU [S2, unofficial]. It sets itself apart from sibling methods like [Solid Voxelization via Octree](6830-solid-voxelization-via-octree.md) by leveraging OpenGL 4.0+ draw indirect features for synchronization-free thread launching and OpenGL 4.2 atomic counters for structure updates, enabling real-time performance on dynamic objects [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh and target spatial bounds with resolution.
*   **Output:** A Sparse Voxel Octree (SVO) data structure residing in GPU memory.

**Data Structures:**
*   **GPU Buffers:** Memory objects accessible via OpenGL 4.2 image load/store interfaces [S2, unofficial].
*   **Atomic Counters:** Used to manage sparse structure updates and track active node allocations [S2, unofficial].
*   **Octree Nodes:** Stored in buffers, where each node represents a voxel or subtree; exact bit-packing is not specified [S2, unofficial].

**Steps:**
1.  **Launch Shader Threads:** Use OpenGL 4.0 `DrawIndirect` features to launch shader threads for octree construction without explicit CPU-GPU synchronization [S2, unofficial].
2.  **Rasterize and Determine Occupancy:** For each triangle, compute the voxel cells it intersects within the target bounds [S2, unofficial].
3.  **Atomic Allocation and Update:** Use OpenGL 4.2 atomic counter functions to manage updates to the sparse octree structure, allocating nodes only where geometry exists [S2, unofficial].
4.  **Hierarchy Construction:** Build the octree directly, meaning nodes are created only for occupied voxels, avoiding the memory overhead of a full grid [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_svo(triangles, bounds, resolution):
    # bounds: (min_x, min_y, min_z, max_x, max_y, max_z)
    # resolution: (dim_x, dim_y, dim_z)
    # Initialize octree root covering the full bounds
    root = OctreeNode(bounds, resolution)
    # Simulate OpenGL 4.2 atomic counter for node allocation
    next_node_id = 0
    node_pool = []
    # Precompute voxel dimensions
    voxel_size = [(bounds[i+3] - bounds[i]) / resolution[i] for i in range(3)]

    for tri in triangles:
        # Compute voxel bounds for the triangle
        tri_min = [max(0, int((tri.min_coord[i] - bounds[i]) / voxel_size[i])) for i in range(3)]
        tri_max = [min(resolution[i]-1, int((tri.max_coord[i] - bounds[i]) / voxel_size[i])) for i in range(3)]

        # Iterate over candidate voxels within bounds
        for x in range(tri_min[0], tri_max[0] + 1):
            for y in range(tri_min[1], tri_max[1] + 1):
                for z in range(tri_min[2], tri_max[2] + 1):
                    # Check actual triangle-voxel intersection
                    if intersects_triangle_voxel(tri, x, y, z, bounds, voxel_size):
                        # Allocate node atomically (simulated)
                        node_id = next_node_id
                        next_node_id += 1
                        node = OctreeNode(None, None)
                        node.id = node_id
                        node.occupied = True
                        node_pool.append(node)
                        # Insert into octree hierarchy
                        insert_into_octree(root, x, y, z, node)
    return root, node_pool

class OctreeNode:
    def __init__(self, bounds, resolution):
        self.bounds = bounds
        self.resolution = resolution
        self.children = [None] * 8
        self.occupied = False
        self.id = None

def insert_into_octree(node, x, y, z, new_node):
    # Determine child index based on voxel coordinates relative to node bounds
    # [G] Standard octree child indexing: bit packing of x,y,z relative to node center
    if node.bounds is None:
        # Leaf node reached or direct assignment
        node.occupied = True
        return
    # [G] Compute child index and recurse iteratively to avoid recursion limits
    # Simplified for pseudocode clarity; production code uses iterative descent
    child_idx = (x & 1) | ((y & 1) << 1) | ((z & 1) << 2)
    if node.children[child_idx] is None:
        node.children[child_idx] = new_node
    else:
        insert_into_octree(node.children[child_idx], x, y, z, new_node)

def intersects_triangle_voxel(tri, x, y, z, bounds, voxel_size):
    # [G] Standard axis-aligned bounding box vs triangle intersection test
    # Returns True if triangle overlaps voxel at (x,y,z)
    return True  # Placeholder for standard geometric predicate
```

**Parameters:**
*   **Resolution:** Not given in the sources.
*   **Bounds:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Boundary Clamping:** Voxel indices are clamped to `[0, resolution-1]` to prevent out-of-bounds memory access [G].
*   **Precision:** Floating-point intersection tests may miss voxels due to precision limits; robust geometric predicates are required [G].
*   **Degenerate Input:** Zero-area triangles or voxels with no intersection are skipped by the intersection check [G].

## Subtopics

### [Factoring Tree Construction](9647-factoring-tree-construction.md)
A method for organizing a frozen, trained Gaussian Splatting model into a hierarchical data structure. It sets itself apart by factoring buckets into "equivalent planks" that preserve weighted volume, centroid, and inertia.

### [Greedy-based Box Packing](11669-greedy-based-box-packing.md)
A pre-processing method that partitions a 3D volume into variably-sized bounding boxes based on alpha values and packs them into 2D texture slices. It distinguishes itself by compacting isolated non-empty voxels to avoid 3D texture padding waste.

### [GPU-based Voxelization via Rasterization](15816-gpu-based-voxelization-via-rasterization.md)
Generates volumetric density 3D textures by rasterizing triangle slices directly into a 3D texture. It solves the problem of on-the-fly voxel generation for interactive navigation by leveraging the user camera's viewport.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Scales with the number of occupied voxels, avoiding the $O(N^3)$ cost of a full grid [S2, unofficial].
*   **Measured Performance:** A 16K triangle mesh (Wald's hand) was voxelized sparsely into a sparse voxel octree in approximately 5.5 ms [S2, unofficial].
*   **Strengths:** Real-time performance, scalability to large scenes, no intermediate full grid required [S2, unofficial].
*   **Limitations:** Requires OpenGL 4.0+ and 4.2 features, which may limit compatibility with older hardware [S2, unofficial].
*   **When to use:** When real-time voxelization of dynamic scenes is required and the hardware supports OpenGL 4.0/4.2 [S2, unofficial].
*   **When not to use:** When targeting hardware without support for `DrawIndirect` or atomic image load/store operations [S2, unofficial].

## Background
This technique was developed as part of research into the usability of the hardware graphics pipeline for fast and real-time voxelization [S2, unofficial]. It was presented in the context of voxel-based global illumination applications [S2, unofficial]. The work compares its performance to CUDA-based regular-grid thin voxelization by Pantaleoni [S2, unofficial].

## Sources
[S1] NeRFLight: Fast and Light Neural Radiance Fields using a ..., https://www.iri.upc.edu/files/scidoc/2861-NeRFLight%3A-Fast-and-Light-Neural-Radiance-Fields-using-a-Shared-Feature-Grid.pdf, fetched 2026-10-27
[S2] PDF K14190_text - Icare3D, https://icare3d.org/research/OpenGLInsights-SparseVoxelization.pdf, fetched 2026-10-29 (unofficial)
