# Octree Hashing for Point Location and Neighbor Finding

Also called: Integer coordinate hashing

## Overview
This method performs point location and neighbor finding in an implicit octree by mapping 3D coordinates to integer indices within a fixed interval $[0, 2^{d_{max}}]$ [S1]. It locates leaf nodes and their neighbors without storing explicit parent pointers, relying instead on bitwise arithmetic and a traversal stack [S1]. It differs from Frisken and Perry [5] by omitting the distinction of "left" or "right" midplane neighbors and by precomputing bit-depth shifts to avoid expensive arbitrary left-shifts [S1].

## How it works
**Inputs:**
- $d_{max}$: The maximum depth of the octree [S1].
- $P_{origin}$: The 3D integer coordinates of the current node [S1].
- $P_{dest}$: The 3D integer coordinates of the target node [S1].
- `parent_trace[]`: An array filled during ray traversal containing the indices of parent nodes for the current cap [S1].

**Outputs:**
- The scalar value of the leaf node at $P_{dest}$ (for point location) [S1].
- The coordinates or index of the neighbor node (for neighbor finding) [S1].

**Data Structures:**
- **Coordinate Space:** Vectors are defined on the interval $[0, 2^{d_{max}}]$ and components are cast to integers [S1].
- **Precomputed Table:** `child_bit_depth[d]` stores the bit mask for depth $d$ [S1].
- **Parent Trace:** An array `parent_trace[]` storing parent indices, used to replace explicit parent pointers in nodes [S1].

**Formulas:**
1.  **Bit Depth Precomputation:**
    $$ \text{child\_bit\_depth}[d] = 1 \ll (d_{max} - d - 1) $$
    where $d$ is the current depth [S1].

2.  **Octant Offset Calculation:**
    The target child octant offset $o \in [0, 7]$ is computed by masking the destination coordinates with the depth bit and shifting by constants for X, Y, and Z components:
    $$ o = \left( (P_{dest}.x \ \& \ \text{child\_bit\_depth}[d]) \gg \text{SHIFT\_X} \right) + \left( (P_{dest}.y \ \& \ \text{child\_bit\_depth}[d]) \gg \text{SHIFT\_Y} \right) + \left( (P_{dest}.z \ \& \ \text{child\_bit\_depth}[d]) \gg \text{SHIFT\_Z} \right) $$
    where $\text{SHIFT\_X}$, $\text{SHIFT\_Y}$, and $\text{SHIFT\_Z}$ are constant shift amounts corresponding to the interleaving of X, Y, and Z components [S1].

**Steps:**

**Point Location:**
1.  Start at an initial node (root for full location, or a deeper node for neighbor-finding context) with current coordinates $P_{curr}$ and depth $d$ [S1].
2.  Compute the target child octant offset $o$ using the formula above with $P_{dest}$ and `child_bit_depth[d]` [S1].
3.  Determine the child node index based on $o$ [S1].
4.  If the child is a leaf, return the scalar value (either a scalar leaf in an interior node or a voxel within a cap node) [S1].
5.  If the child is an interior node, update $P_{curr}$ to the child's coordinates, increment $d$, and repeat from step 2 [S1].

**Neighbor Finding:**
1.  Begin at the depth of the origin node's parent [S1]. Let $d = d_{origin} - 1$ [S1].
2.  Recurse up the octree to find a common parent:
    a.  Check if the origin and destination nodes share the same parent at depth $d$ by &-masking both $P_{origin}$ and $P_{dest}$ with `child_bit_depth[d]` and performing integer equality [S1].
    b.  If they are equal, a common parent is found [S1]. Proceed to point location from this common parent to $P_{dest}$ [S1].
    c.  If not equal, decrement $d$ and repeat [S1].
3.  To navigate up the tree without explicit parent pointers, use the `parent_trace[]` array filled during the initial ray traversal to retrieve parent indices [S1].
4.  Once the common parent is identified, use the Point Location algorithm to find the leaf at $P_{dest}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants for interleaving X, Y, Z components
SHIFT_X = 0
SHIFT_Y = 1
SHIFT_Z = 2

def precompute_child_bit_depth(max_depth):
    """Precompute bit masks for each depth level."""
    child_bit_depth = [1 << (max_depth - d - 1) for d in range(max_depth + 1)]
    return child_bit_depth

def get_octant_offset(dest_coords, depth, child_bit_depth):
    """Compute the 0-7 octant offset for a destination coordinate at a given depth."""
    mask = child_bit_depth[depth]
    x_part = (dest_coords[0] & mask) >> SHIFT_X
    y_part = (dest_coords[1] & mask) >> SHIFT_Y
    z_part = (dest_coords[2] & mask) >> SHIFT_Z
    return x_part + y_part + z_part

def point_location(initial_node_idx, dest_coords, child_bit_depth, max_depth, nodes, initial_depth=0):
    """
    Perform top-down search to find the leaf value at dest_coords.
    nodes: list of dicts with keys 'children' (list of 8 indices) and 'value' (for leaves).
    """
    current_node_idx = initial_node_idx
    current_depth = initial_depth
    
    while current_depth < max_depth:
        octant = get_octant_offset(dest_coords, current_depth, child_bit_depth)
        child_node_idx = nodes[current_node_idx]['children'][octant]
        
        if nodes[child_node_idx]['value'] is not None:
            return nodes[child_node_idx]['value']
        
        current_node_idx = child_node_idx
        current_depth += 1
        
    return None

def find_neighbor(origin_node_idx, origin_coords, dest_coords, child_bit_depth, parent_trace, nodes, max_depth, origin_depth):
    """Find the neighbor of origin_node in the direction of dest_coords."""
    d = origin_depth - 1
    
    while d >= 0:
        mask = child_bit_depth[d]
        origin_parent_coord = origin_coords & mask
        dest_parent_coord = dest_coords & mask
        
        if origin_parent_coord == dest_parent_coord:
            common_parent_idx = parent_trace[d]
            return point_location(common_parent_idx, dest_coords, child_bit_depth, max_depth, nodes, d)
        
        d -= 1
        
    return None
```

**Parameters:**
- $d_{max}$: Maximum octree depth. Typical values depend on the volume resolution and required detail [S1].
- `SHIFT_X`, `SHIFT_Y`, `SHIFT_Z`: Constants for interleaving X, Y, Z components. Exact values depend on the bit-interleaving scheme used [S1].

**Edge Cases and Pitfalls:**
- **Precision:** The method relies on integer arithmetic, avoiding floating-point issues. Coordinates must be cast to integers within the $[0, 2^{d_{max}}]$ range [S1].
- **Boundaries:** Neighbor finding assumes the destination is within the octree bounds. Out-of-bounds checks are not explicitly described in the source [S1].
- **Parent Trace:** The method requires `parent_trace[]` to be correctly filled during ray traversal. If the trace is incomplete or incorrect, neighbor finding will fail [S1].

## Subtopics

### [Cell Retrieval via Neighbor-Finding](1822-cell-retrieval-via-neighbor-finding.md)
A sibling method that focuses on retrieving cell data once neighbor coordinates are resolved. Choose this when neighbor indices are already computed and only data fetch is required.

### [Cell Reconstruction via Octree Hashing](1595-cell-reconstruction-via-octree-hashing.md)
The parent method that uses this technique for point location to find forward neighbors for cell reconstruction. Choose this when building the octree structure itself.

## Operating characteristics
- **Time Complexity:** Point location is $O(d_{max})$ as it performs a top-down search from the root (or a deeper starting node) to a leaf [S1]. Neighbor finding involves recursing up the tree to find a common parent, which is also bounded by the depth $d_{max}$, resulting in $O(d_{max})$ complexity [S1].
- **Memory Footprint:** The method omits parent pointers from nodes to minimize memory footprint, relying on the `parent_trace[]` array instead [S1].
- **Performance:** The precomputation of `child_bit_depth` and use of bitwise operations ($\&$, $\gg$) are optimized for integer arithmetic, avoiding expensive arbitrary left-shifts [S1].
- **Strengths:** Fast point location and neighbor finding without explicit parent pointers; simplified logic compared to Frisken and Perry [5] [S1].
- **Limitations:** Requires precomputation of bit depths; requires `parent_trace[]` to be maintained during traversal; limited to integer coordinate spaces [S1].

## Background
This method is based on the octree hashing scheme described in "PDF Interactive Isosurface Ray Tracing of Large Octree Volumes" [S1]. It adapts the approach of Frisken and Perry [5] but simplifies the neighbor-finding logic by omitting the "left"/"right" midplane distinction and removes parent pointers to reduce memory usage [S1].

## Sources
[S1] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://collections.lib.utah.edu/dl_files/1d/58/1d589a03a725c9d894a3866ad32f1386f0613f9d.pdf, fetched 2026-10-28
