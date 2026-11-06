# Cell Reconstruction via Octree Hashing

Also called: Frisken and Perry hashing scheme, Octree Hashing Scheme

## Overview
Cell Reconstruction via Octree Hashing is a method for locating leaf nodes and retrieving adjacent "forward neighbor" voxels in a sparse octree volume representation, enabling the reconstruction of 2x2x2 cells required for isosurface intersection primitives. It solves the problem of needing 8 corner scalar values for a cell intersection when only 1 scalar value is stored per leaf node, by using a bitwise arithmetic hash scheme to map 3D coordinates to integer indices for O(log N) point location and O(1) average-case neighbor finding [S1, S2, S3]. It sets itself apart from sibling methods like [Pointerless Octree for Seed Placement](5280-pointerless-octree-for-seed-placement.md) by focusing specifically on the bitwise arithmetic mapping for neighbor retrieval rather than seed placement, and differs from [Cell Retrieval via Neighbor-Finding](1822-cell-retrieval-via-neighbor-finding.md) by relying on direct coordinate hashing rather than geometric neighbor-finding techniques that skew toward O(1) only when neighbors share a parent [S1, S2, S3].

## How it works
**Inputs:**
- A sparse octree structure where nodes are defined by their spatial extent and subdivision depth.
- A query point $P = (x, y, z)$ in the global coordinate space, or a reference leaf node for neighbor finding.
- The maximum depth of the octree, $d_{max}$.

**Outputs:**
- The leaf node containing the query point $P$ (Point Location).
- The scalar values or references to the 7 forward neighbors of a given leaf node (Neighbor Finding).

**Data Structures:**
The octree nodes are implicitly defined by their Morton codes (Z-order curves) or spatial coordinates. The hash scheme maps the 3D integer coordinates of a voxel center to a unique integer index.
- **Morton Code / Hash Index:** An integer formed by interleaving the bits of the x, y, and z coordinates. For a coordinate $(x, y, z)$, the hash $H$ is computed by interleaving the bits of $x$, $y$, and $z$.
- **Node Types:**
  - **Interior Node:** Contains pointers to 8 children.
  - **Cap Node:** At depth $d_{max}-1$, contains 8 finest-level voxels.
  - **Scalar Leaf:** Stores a single scalar value.

**Steps:**

1. **Coordinate Mapping (Point Location):**
   To find the leaf node for a given point $P(x, y, z)$:
   a. Convert the continuous coordinates to integer voxel indices $(ix, iy, iz)$ based on the voxel resolution at the root or a specified grid.
   b. Compute the Morton code $M$ for $(ix, iy, iz)$ by interleaving the bits:
      $$ M = \text{interleave}(ix, iy, iz) $$
      The `interleave` function packs the bits of $x, y, z$ into a single integer. For example, if $x=1 (01_2), y=2 (10_2), z=3 (11_2)$, the interleaved bits are $x_0 y_0 z_0 x_1 y_1 z_1 \dots$.
   c. Traverse the octree from the root using the bits of $M$ to determine the path to the leaf. At each level $d$ from $0$ to $d_{max}-1$:
      - Extract the 3-bit octant index $o_d$ from the bits of $M$ corresponding to level $d$.
      - The octant index is determined by:
        $$ o_d = ((M \gg (3d)) \ \& \ 7) $$
      - Move to the child at index $o_d$.
   d. The traversal ends at the leaf node.

2. **Neighbor Finding:**
   To find the forward neighbor of a leaf node at position $P_{origin}$ in a specific direction (e.g., +X):
   a. **Find Nearest Common Ancestor (NCA):**
      - Compute the Morton codes for the origin leaf $M_{origin}$ and the target neighbor position $M_{target}$.
      - The NCA is the deepest node that is an ancestor of both. This can be found by comparing the bits of $M_{origin}$ and $M_{target}$ from the most significant bit downwards. The first bit where they differ indicates the level of the NCA.
   b. **Point Location for Target:**
      - Perform point location for $M_{target}$ as described in Step 1.
      - If the target position is within the bounds of the octree and the node exists, return the leaf.
   c. **Optimization:**
      - Precompute masks and use constant interleaving for interactivity [S2].
      - If the neighbor shares the same parent as the origin, the neighbor can be found in O(1) by computing the sibling index directly from the Morton code [S1, S2, S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interleave_bits(x, y, z):
    """Interleave bits of x, y, z to form a Morton code."""
    # This is a simplified representation; actual implementation uses bitwise ops
    # to pack bits into a single integer.
    # Example for 3-bit coordinates:
    # x = x0 x1 x2, y = y0 y1 y2, z = z0 z1 z2
    # M = x0 y0 z0 x1 y1 z1 x2 y2 z2
    m = 0
    for i in range(3): # Assuming 3-bit precision for simplicity
        m |= (x & (1 << i)) << (2 * i)
        m |= (y & (1 << i)) << (2 * i + 1)
        m |= (z & (1 << i)) << (2 * i + 2)
    return m

def find_leaf(root, x, y, z, d_max):
    """Find the leaf node for point (x, y, z) in the octree."""
    # Convert to integer indices
    ix, iy, iz = int(x), int(y), int(z)
    
    # Compute Morton code
    m = interleave_bits(ix, iy, iz)
    
    node = root
    for d in range(d_max):
        # Extract octant index at level d
        octant = (m >> (3 * d)) & 7
        if node is None:
            return None
        node = node.children[octant]
    return node

def find_neighbor(root, origin_leaf, direction, d_max):
    """Find the forward neighbor of origin_leaf in the given direction."""
    # Get Morton code of origin
    m_origin = origin_leaf.morton_code
    
    # Calculate target Morton code based on direction
    # This requires knowing the voxel size and position
    # For simplicity, assume we can compute target coordinates
    target_coords = get_target_coords(origin_leaf, direction)
    m_target = interleave_bits(*target_coords)
    
    # Find NCA
    # Compare bits from MSB to LSB
    nca_level = 0
    for d in range(d_max - 1, -1, -1):
        if ((m_origin >> (3 * d)) & 7) == ((m_target >> (3 * d)) & 7):
            nca_level = d
            break
    
    # If NCA is root, traverse from root
    # Otherwise, traverse from NCA
    # For simplicity, traverse from root
    return find_leaf(root, *target_coords, d_max)
```

**Parameters:**
- $d_{max}$: The maximum depth of the octree. Typical values depend on the resolution of the volume data.
- Voxel resolution: The size of the voxel at the root level.

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** If the neighbor position is outside the octree bounds, the method should return null or handle the boundary appropriately [S2].
- **Precision:** Bitwise operations assume integer coordinates. Floating-point coordinates must be converted to integer voxel indices before hashing [S2].
- **Degenerate Input:** If the octree is empty or the root is null, the method should return null [S2].

## Subtopics

### [Cell Retrieval via Neighbor-Finding](1822-cell-retrieval-via-neighbor-finding.md)
This method enables isosurface reconstruction from sparse octree volumes that store only a single scalar value per leaf voxel, avoiding the 8x storage expansion required to store full 8-voxel cells explicitly. It solves the problem of needing 8 corner values for a cell intersection primitive when only 1 scalar value is stored per leaf node by retrieving the 7 forward neighbors at intersection time. Unlike [Cell Reconstruction via Octree Hashing](1595-cell-reconstruction-via-octree-hashing.md), which uses a bitwise arithmetic hash scheme for point location, this method relies on neighbor-finding techniques to retrieve forward-neighboring voxels, skewing heavily toward O(1) performance when neighbors share a parent node.

### [Octree Hashing for Point Location and Neighbor Finding](1984-octree-hashing-for-point-location-and-neighbor-finding.md)
This method performs point location and neighbor finding in an implicit octree by mapping 3D coordinates to integer indices within a fixed interval $[0, 2^{d_{max}}]$. It locates leaf nodes and their neighbors without storing explicit parent pointers, relying instead on bitwise arithmetic and a traversal stack. It differs from Frisken and Perry [5] by omitting the distinction of "left" or "right" midplane neighbors and by precomputing bit-depth shifts to avoid expensive arbitrary left-shifts.

### [Min/Max Tree Construction with Forward Neighbors](2020-min-max-tree-construction-with-forward-neighbors.md)
This method constructs a bounding volume hierarchy on an octree where each node stores the minimum and maximum isosurface values within its spatial extent, specifically accounting for cells defined by forward neighbors. It differs from standard min/max trees by ensuring that a parent node's bounding box considers not only its 8 direct children but also their forward neighbors, effectively evaluating 27 voxel values per parent. This allows for efficient ray traversal rejection without traversing child nodes, reducing cache misses during volume rendering.

### [Neighbor Cell Access via Hash Table and Morton Code](6715-neighbor-cell-access-via-hash-table-and-morton-code.md)
A method to find face-adjacent cells in an irregular octree using spatial coordinates, Morton codes, and a hash table for constant-time access. It handles anisotropic subdivision by checking geometric limits to ensure true adjacency.

## Operating characteristics
- **Time Complexity:** Point location is O(log N) in the worst case, where N is the number of nodes. Neighbor finding is O(log N) in the worst case but O(1) in the best and average cases when neighbors share a parent node [S2].
- **Memory Complexity:** The method requires storing the octree structure. The hash scheme itself does not require additional storage beyond the Morton codes, which can be computed on the fly or stored with each node [S1, unofficial][S2].
- **Performance:** The method is optimized for interactivity by precomputing masks and using constant interleaving [S2]. It is efficient for sparse volumes where most of the space is empty [S1, unofficial].
- **Strengths:** Fast point location and neighbor finding; low memory overhead for sparse data; suitable for real-time ray tracing [S1, unofficial][S2].
- **Limitations:** Performance degrades if the octree is deep and neighbors do not share parents; requires integer coordinates for hashing [S2].
- **When to Use:** When rendering large sparse octree volumes and needing efficient access to neighbor voxels for cell reconstruction [S1, unofficial][S2].
- **When Not to Use:** When the volume is dense and the overhead of the octree traversal outweighs the benefits; when floating-point precision is critical and cannot be mapped to integer coordinates [S2].

## Background
Cell Reconstruction via Octree Hashing builds on the work of Frisken and Perry, who proposed an efficient hashing scheme using binary arithmetic on integer coordinates for point location and neighbor finding in octrees [S1, unofficial]. This approach was pioneered by Glassner and Samet for use in ray tracing and has been extended to other applications involving regular binary trees [S1, unofficial]. The method is particularly useful for interactive isosurface ray tracing of large octree volumes, where efficient access to neighbor voxels is crucial for performance [S1, unofficial][S2].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S2] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://pdfs.semanticscholar.org/aef9/6e0dd17920d0c70cbf0dd43f494cab1c0bcc.pdf, fetched 2026-10-28
