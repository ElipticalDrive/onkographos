# Cell Reconstruction via Octree Hashing

Also called: Frisken and Perry hashing scheme, Octree Hashing Scheme

## Overview
Cell Reconstruction via Octree Hashing is a method for locating leaf nodes and retrieving adjacent "forward neighbor" voxels in a sparse octree volume representation, enabling the reconstruction of 2x2x2 cells required for isosurface intersection primitives [S1, unofficial][S2]. It solves the problem of needing 8 corner scalar values for a cell intersection when only 1 scalar value is stored per leaf node, by using a bitwise arithmetic hash scheme to map 3D coordinates to integer indices for O(log N) point location and O(1) average-case neighbor finding [S1, unofficial][S2].

## How it works
**Inputs:**
- A sparse octree structure where nodes are defined by their spatial extent and subdivision depth [S2].
- A query point $P = (x, y, z)$ in the global coordinate space, or a reference leaf node for neighbor finding [S1, unofficial][S2].
- The maximum depth of the octree, $d_{max}$ [S2].

**Outputs:**
- The leaf node containing the query point $P$ (Point Location) [S1, unofficial][S2].
- The scalar values or references to the 7 forward neighbors of a given leaf node (Neighbor Finding) [S1, unofficial][S2].

**Data Structures:**
The octree nodes are implicitly defined by their Morton codes (Z-order curves) or spatial coordinates [S2]. The hash scheme maps the 3D integer coordinates of a voxel center to a unique integer index [S2].
- **Morton Code / Hash Index:** An integer formed by interleaving the bits of the x, y, and z coordinates [S2].
- **Node Types:**
  - **Interior Node:** Contains pointers to 8 children [S2].
  - **Cap Node:** At depth $d_{max}-1$, contains 8 finest-level voxels [S2].
  - **Scalar Leaf:** Stores a single scalar value [S2].

**Steps:**
1. **Coordinate Mapping (Point Location):**
   a. Map the query point $P(x, y, z)$ to integer voxel indices within the range $[0, 2^{d_{max}}]^3$ [S2].
   b. Compute the Morton code $M$ by interleaving the bits of the x, y, and z coordinates [S2].
   c. Traverse the octree from the root. At each level $d$ from $0$ to $d_{max}-1$, extract the 3-bit octant index $o_d$ from the bits of $M$ corresponding to that level using a bitwise-AND with a left-shifted mask [S2].
   d. Move to the child at index $o_d$. The traversal ends at the leaf node [S1, unofficial][S2].
2. **Neighbor Finding:**
   a. Given an origin leaf and a target direction, compute the target coordinates and its Morton code $M_{target}$ [S2].
   b. Find the nearest common ancestor (NCA) of the origin and target leaves by comparing their Morton code bits from the most significant bit downwards [S2].
   c. Perform point location for the target leaf starting from the root (or the NCA) to retrieve it [S2].
3. **Optimization:**
   a. Precompute masks and use constant interleaving for interactivity [S2].

**Formulas:**
- Morton Code: $M = \text{interleave}(x, y, z)$, where $\text{interleave}$ packs the bits of $x$, $y$, and $z$ into a single integer [S2].
- Octant Index: $o_d = ((M \gg (3(d_{max} - 1 - d))) \ \& \ 7)$, extracting the 3-bit path at level $d$ [S2].
- Target Coordinates: $P_{target} = P_{origin} + \vec{d}$, where $\vec{d}$ is the unit direction vector scaled by the voxel size at the target depth [G].

**Parameters:**
- $d_{max}$: The maximum depth of the octree. Typical values depend on the resolution of the volume data [S2].
- Voxel resolution: The size of the voxel at the root level [G].

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interleave_bits(x, y, z, max_bits=10):
    """Interleave bits of x, y, z to form a Morton code [G]."""
    m = 0
    for i in range(max_bits):
        m |= (x & (1 << i)) << (2 * i)
        m |= (y & (1 << i)) << (2 * i + 1)
        m |= (z & (1 << i)) << (2 * i + 2)
    return m

def find_leaf(root, m, d_max):
    """Find the leaf node for a given Morton code m in the octree."""
    node = root
    for d in range(d_max):
        # Extract 3-bit octant index for level d (MSB first)
        octant = (m >> (3 * (d_max - 1 - d))) & 7
        if node is None:
            return None
        node = node.children[octant]
    return node

def find_neighbor(root, origin_coords, direction, d_max):
    """Find the forward neighbor of origin_leaf in the given direction [S2]."""
    # Compute target coordinates based on direction and origin position
    target_coords = (origin_coords[0] + direction[0], origin_coords[1] + direction[1], origin_coords[2] + direction[2])
    
    # Compute Morton codes
    m_origin = interleave_bits(*origin_coords)
    m_target = interleave_bits(*target_coords)
    
    # Find NCA level by comparing bits from MSB
    nca_level = -1
    for d in range(d_max - 1, -1, -1):
        if ((m_origin >> (3 * d)) & 7) == ((m_target >> (3 * d)) & 7):
            nca_level = d
            break
            
    # Perform point location for target
    return find_leaf(root, m_target, d_max)
```

## Subtopics

### [Cell Retrieval via Neighbor-Finding](1822-cell-retrieval-via-neighbor-finding.md)
This method enables isosurface reconstruction from sparse octree volumes that store only a single scalar value per leaf voxel, avoiding the 8x storage expansion required to store full 8-voxel cells explicitly. It solves the problem of needing 8 corner values for a cell intersection primitive when only 1 scalar value is stored per leaf node by retrieving the 7 forward neighbors at intersection time. Unlike [Cell Reconstruction via Octree Hashing](1595-cell-reconstruction-via-octree-hashing.md), which uses a bitwise arithmetic hash scheme for point location, this method relies on neighbor-finding techniques to retrieve forward-neighboring voxels, skewing heavily toward O(1) performance when neighbors share a parent node.

### [Octree Hashing for Point Location and Neighbor Finding](1984-octree-hashing-for-point-location-and-neighbor-finding.md)
This method performs point location and neighbor finding in an implicit octree by mapping 3D coordinates to integer indices within a fixed interval $[0, 2^{d_{max}}]$. It locates leaf nodes and their neighbors without storing explicit parent pointers, relying instead on bitwise arithmetic and a traversal stack. It differs from Frisken and Perry [5] by omitting the distinction of "left" or "right" midplane neighbors and by precomputing bit-depth shifts to avoid expensive arbitrary left-shifts.

### [Min/Max Tree Construction with Forward Neighbors](2020-min-max-tree-construction-with-forward-neighbors.md)
This method constructs a bounding volume hierarchy on an octree where each node stores the minimum and maximum isosurface values within its spatial extent, specifically accounting for cells defined by forward neighbors. It differs from standard min/max trees by ensuring that a parent node's bounding box considers not only its 8 direct children but also their forward neighbors, effectively evaluating 27 voxel values per parent. This allows for efficient ray traversal rejection without traversing child nodes, reducing cache misses during volume rendering.

### [Neighbor Cell Access via Hash Table and Morton Code](6715-neighbor-cell-access-via-hash-table-and-morton-code.md)
This method retrieves face-adjacent cells in an irregular (generalized) octree where subdivision is not uniform across all three axial directions. It solves the problem that standard Morton code arithmetic fails to identify true geometric neighbors in anisotropic structures, where cells close in Morton space may be far apart geometrically. It sets itself apart from sibling methods by explicitly validating adjacency through geometric limits rather than relying solely on bitwise offsets or parent pointers, ensuring correctness despite irregular subdivision.

## Operating characteristics
- **Time Complexity:** Point location is O(log N) in the worst case, where N is the number of nodes. Neighbor finding is O(log N) in the worst case but O(1) in the best and average cases when neighbors share a parent node [S2].
- **Memory Complexity:** The method requires storing the octree structure [S2].
- **Performance:** The method is optimized for interactivity by precomputing masks and using constant interleaving [S2].
- **Strengths:** Fast point location and neighbor finding; suitable for real-time ray tracing [S1, unofficial][S2].
- **Limitations:** Not given in the sources.
- **When to Use:** When rendering large sparse octree volumes and needing efficient access to neighbor voxels for cell reconstruction [S1, unofficial][S2].
- **When Not to Use:** Not given in the sources.

## Background
Point location and neighbor-finding algorithms were pioneered by Glassner and Samet for use in ray tracing [S1, unofficial]. Frisken and Perry proposed an efficient hashing scheme using binary arithmetic on integer coordinates, which this method builds upon [S1, unofficial]. The approach is used for interactive isosurface ray tracing of large octree volumes [S1, unofficial][S2].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S2] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://pdfs.semanticscholar.org/aef9/6e0dd17920d0c70cbf0dd43f494cab1c0bcc.pdf, fetched 2026-10-28
