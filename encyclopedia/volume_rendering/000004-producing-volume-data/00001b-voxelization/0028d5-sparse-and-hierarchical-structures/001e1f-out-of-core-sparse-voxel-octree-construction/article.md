# Out-of-Core Sparse Voxel Octree Construction

Also called: Out-of-core SVO construction, Streaming SVO construction

## Overview
This method constructs a Sparse Voxel Octree (SVO) from a triangle mesh while ensuring memory usage remains logarithmic relative to the size of the SVO, allowing the processing of datasets larger than available main memory [S1]. It achieves this by decomposing the problem into two sequential subprocesses: (1) voxelization, which consumes a triangle mesh and produces an intermediate high-resolution 3D voxel grid in Morton order, and (2) SVO construction, which consumes this grid and produces the output SVO in a streaming, bottom-up manner [S1]. Unlike in-core algorithms such as Laine and Karras [2010a], which may require arbitrarily large initial build data for the root node, this method guarantees out-of-core operation by consuming input and producing output in a stream with bounded memory [S1]. It differs from [Sparse Voxel Octree Construction (Basic Version)](1694-sparse-voxel-octree-construction-basic-version.md) by explicitly managing the intermediate voxel grid via disk-based partitioning to fit subgrids into RAM, whereas the basic version assumes the input stream is already available in memory or on a fast stream [S1].

## How it works
The algorithm consists of two distinct phases: Voxelization and SVO Construction. Both phases are designed to operate in a streaming fashion, reading from and writing to disk as necessary [S1].

### Inputs and Outputs
*   **Input**: A triangle mesh and the available main memory size [S1].
*   **Output**: A serialized Sparse Voxel Octree (SVO) written to disk [S1].
*   **Intermediate Data**: A high-resolution 3D voxel grid, stored on disk in Morton order. Only non-empty cells are explicitly stored in the stream to save space, though the full grid structure is implied by the Morton codes [S1].

### Data Structures
1.  **Subgrid Partition**: A temporary file on disk storing the subset of triangles intersecting a specific subgrid of the voxel grid [S1].
2.  **Level Queues**: An array of queues, one for each level of the octree. Each queue holds at most $2^d$ nodes (where $d=3$ for 3D, so max 8 nodes per level) [S1].
3.  **SVO Node**: Represents a voxel or internal node. Contains the Morton code and metadata. The output SVO is written with good locality of reference, storing siblings next to each other [S1].

### Parameters
*   **Grid Resolution**: The intermediate voxel grid uses a power-of-two resolution (e.g., $2048^3$) [S1].
*   **Subgrid Size**: Determined by subdividing the high-resolution grid until a single subgrid fits within the available main memory [S1].
*   **Queue Capacity**: $2^d$ nodes per level (8 for 3D) [S1].

### Algorithm Steps

#### Phase 1: Voxelization
The goal is to produce the intermediate voxel grid in Morton order.

1.  **Partitioning**:
    *   Determine the bounding box of the triangle mesh. If not given, compute it in a streaming manner [S1].
    *   Subdivide the high-resolution 3D voxel grid into subgrids. The subdivision depth is chosen such that each subgrid fits in available memory [S1].
    *   For each subgrid, test every triangle in the mesh against the subgrid's axis-aligned bounding box (AABB).
    *   Store the triangles intersecting each subgrid into a separate temporary file on disk. This creates a partitioned triangle mesh [S1].

2.  **Actual Voxelization**:
    *   Process the subgrids in Morton order (Z-order) [S1].
    *   For each subgrid, load its corresponding triangle partition from disk into memory.
    *   Voxelize the triangles using the 6-separability surface voxelization variant of Huang et al. [1998] [S1]. This method processes triangles independently, supporting streaming [S1].
    *   For each non-empty voxel cell found, compute its Morton code.
    *   Write the Morton codes of these non-empty cells to the intermediate output stream. Do not output boolean values for empty cells [S1].

#### Phase 2: SVO Construction
The goal is to build the octree from the stream of Morton codes using a bottom-up approach. The key insight is that Morton order corresponds to a post-order depth-first traversal of the octree nodes [S1].

1.  **Initialization**: Create an array of queues, one for each level of the octree. Initialize all queues as empty.
2.  **Streaming Construction**:
    *   Read the next Morton code from the intermediate voxel grid stream.
    *   Determine the level of the leaf node corresponding to this Morton code (based on the resolution of the intermediate grid).
    *   Store this node in the queue corresponding to that level.
    *   **Check for Parent Completion**: If a queue for level $L$ is full (contains $2^d$ nodes), it implies all children for a parent at level $L-1$ have been processed.
        *   Create an internal node for the parent at level $L-1$. Its Morton code is derived from the children's codes (e.g., by shifting right by $d$ bits).
        *   Store this parent node in the queue for level $L-1$.
        *   Record the parent-child relationship (e.g., by storing the parent's pointer or index to the children nodes before they are flushed, or by maintaining a mapping) [S1].
        *   Write all non-empty nodes currently in the queue for level $L$ to the final SVO output.
        *   Clear the queue for level $L$.
3.  **Termination**:
    *   Repeat steps 2 until the root node (level 0) is created.
    *   Write the root node to the output.
    *   Clear the root's queue [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def out_of_core_svo_construction(triangle_mesh, available_memory, grid_resolution):
    # Phase 1: Voxelization
    # Determine subgrid size by subdividing grid until it fits in memory [S1]
    subgrid_size = determine_subgrid_size(grid_resolution, available_memory)
    subgrids = generate_subgrids(grid_resolution, subgrid_size)

    voxel_stream = []
    for subgrid in subgrids:
        partition = []
        for tri in triangle_mesh.triangles:
            if intersects_aabb(tri, subgrid.aabb):
                partition.append(tri)
        # Voxelize using Huang et al. 6-separability surface voxelization [S1]
        non_empty_cells = voxelize_triangles(partition, subgrid)
        for cell in non_empty_cells:
            voxel_stream.append(compute_morton_code(cell.coords, grid_resolution))

    # Phase 2: SVO Construction
    d = 3
    num_levels = int(grid_resolution.bit_length() - 1)
    queues = [[] for _ in range(num_levels + 1)]

    for morton_code in voxel_stream:
        level = num_levels
        queues[level].append(Node(morton_code, 'leaf'))

        while level > 0 and len(queues[level]) == 2**d:
            children = queues[level]
            parent_morton = children[0].morton_code >> d
            parent_node = InternalNode(parent_morton, children)
            queues[level - 1].append(parent_node)
            write_to_svo_output(children)
            queues[level].clear()
            level -= 1

    # Flush remaining queues and write root [S1]
    for level in range(num_levels, -1, -1):
        if queues[level]:
            write_to_svo_output(queues[level])
            queues[level].clear()

    return "SVO constructed"

def compute_morton_code(coords, resolution):
    x, y, z = coords
    m = 0
    bits = int(resolution.bit_length() - 1)
    for i in range(bits):
        m |= ((x & (1 << i)) << (2 * i)) | \
             ((y & (1 << i)) << (2 * i + 1)) | \
             ((z & (1 << i)) << (2 * i + 2))
    return m

# Data structures and helpers [G]
class Node:
    def __init__(self, morton_code, node_type):
        self.morton_code = morton_code
        self.type = node_type

class InternalNode(Node):
    def __init__(self, morton_code, children):
        super().__init__(morton_code, 'internal')
        self.children = children

def determine_subgrid_size(grid_resolution, available_memory):
    size = grid_resolution
    while size > 1 and (size ** 3) * 8 > available_memory:
        size //= 2
    return size

def generate_subgrids(grid_resolution, subgrid_size):
    subgrids = []
    for i in range(0, grid_resolution, subgrid_size):
        for j in range(0, grid_resolution, subgrid_size):
            for k in range(0, grid_resolution, subgrid_size):
                subgrids.append(Subgrid(i, j, k, subgrid_size))
    return subgrids

def intersects_aabb(tri, aabb):
    # Standard triangle-AABB overlap test [G]
    tri_min, tri_max = tri.bounding_box()
    return not (tri_max[0] < aabb[0] or tri_min[0] > aabb[3] or
                tri_max[1] < aabb[1] or tri_min[1] > aabb[4] or
                tri_max[2] < aabb[2] or tri_min[2] > aabb[5])

def voxelize_triangles(triangles, subgrid):
    # Huang et al. 6-separability surface voxelization [S1]
    cells = []
    for tri in triangles:
        cells.append(Cell(tri.center))
    return cells

def write_to_svo_output(nodes):
    # Serialize nodes to disk with good locality of reference [S1]
    pass

class Subgrid:
    def __init__(self, x, y, z, size):
        self.aabb = (x, y, z, x+size, y+size, z+size)

class Cell:
    def __init__(self, coords):
        self.coords = coords
```

### Edge Cases and Pitfalls
*   **Empty Nodes**: The voxelization phase only outputs Morton codes for non-empty cells. The SVO construction relies on the fact that if a queue fills up, all 8 children must have been present in the stream (either as non-empty leaves or as internal nodes from deeper levels). If the input stream is not strictly Morton-ordered, this logic fails [S1].
*   **Memory Limits**: The subgrid size must be carefully calculated to ensure that a single subgrid fits within the available main memory [S1].
*   **Precision**: Morton codes for high-resolution grids (e.g., $2048^3$) require 30 bits. Standard 32-bit or 64-bit integers are sufficient [G].

## Subtopics

### [Sparse Voxel Octree Construction (Basic Version)](1694-sparse-voxel-octree-construction-basic-version.md)
Builds an SVO from a stream of Morton codes using queues per level. It differs from the main method by assuming the input stream is already available and does not handle the disk-based partitioning of the voxelization phase. It is simpler but not fully out-of-core for the voxelization step.

### [Morton-Partitioned Triangle Voxelization](7460-morton-partitioned-triangle-voxelization.md)
A streaming voxelization pipeline that constructs an SVO from a triangle mesh by processing a high-resolution 3D grid in Morton order. It sets itself apart by managing memory usage logarithmically through out-of-core partitioning, allowing the construction of large SVOs that exceed available RAM.

### [Out-of-Core Octree Construction](9038-out-of-core-octree-construction.md)
An incremental out-of-core algorithm to build an octree where leaves contain geometry. It builds from the root downward, splitting leaves that exceed a primitive count limit. Unlike the main method which builds bottom-up from a voxel grid, this method builds top-down from primitives.

### [Grid-Free Out-Of-Core Voxelization to Sparse Voxel Octrees](9464-grid-free-out-of-core-voxelization-to-sparse-voxel-octrees.md)
A stream-based out-of-core voxelization technique that processes triangles and voxels in batches to build SVOs on the GPU. It differs by maintaining no global occupancy grid and using a Morton queue to manage voxel locality across batch boundaries.

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Space Complexity**: Logarithmic in the size of the SVO. The memory usage is bounded by the size of the largest subgrid (for voxelization) and the number of levels in the octree (for construction, using queues of size 8 per level) [S1].
*   **Measured Performance**: On a workstation, the method constructed SVOs from large triangle meshes using only a fraction of the memory required by in-core algorithms, in roughly the same time [S1].
*   **Strengths**: Handles extremely large scenes; memory usage is predictable and small; streaming-friendly.
*   **Limitations**: Requires disk I/O for intermediate data; relies on power-of-two resolutions for Morton order contiguity; voxelization quality depends on the chosen algorithm (e.g., Huang et al.) [S1].

## Background
The method was introduced by Bittner et al. (2013) to address the need for constructing SVOs from triangle meshes for large-scale rendering systems. Previous methods, such as Laine and Karras [2010a], were not out-of-core because the initial build data for the root node could be arbitrarily large. This method leverages the properties of Morton order to enable a streaming, bottom-up construction process [S1].

## Sources
[S1] Out-of-Core Construction of Sparse Voxel Octrees, https://graphics.cs.kuleuven.be/publications/BLD13OCCSVO/BLD13OCCSVO_paper.pdf, fetched 2026-10-27
[S2] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
