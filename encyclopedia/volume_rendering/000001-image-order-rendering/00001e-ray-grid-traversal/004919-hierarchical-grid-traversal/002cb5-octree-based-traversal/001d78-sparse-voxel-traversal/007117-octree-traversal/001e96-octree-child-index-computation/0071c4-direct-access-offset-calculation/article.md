# Direct Access Offset Calculation

Also called: InternalNode offset calculation, LeafNode offset calculation, VDB Direct Access Offset Calculation

## Overview
Direct Access Offset Calculation computes the index of a child node within a parent node's data structure based on global 3D grid coordinates $(x, y, z)$ [S1, unofficial]. It solves the problem of mapping spatial positions to hierarchical tree indices without using division, multiplication, or modulus operations, which are computationally expensive [S1, unofficial]. It sets itself apart from sibling methods like [Point Location](2076-point-location.md) by relying on compile-time constant block sizes to perform bitwise arithmetic, and from [Voxel Child Address Calculation](2495-voxel-child-address-calculation.md) by specifically targeting the sparse VDB internal node and leaf node structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Global grid coordinates $(x, y, z)$; compile-time constants `sLog2X`, `sLog2Y`, `sLog2Z` representing the base-2 logarithm of the child block dimensions in X, Y, and Z; `Log2Y` and `Log2Z` (derived constants); `mChildMask` (bitmask of existing children); `mInternalDAT` (data array for internal nodes).
*   **Outputs:** An unsigned integer offset into the child data structure; for InternalNodes, a boolean indicating if the location is a tile (constant value) or a child node.

**Data Structures:**
*   `mChildMask`: A bitmask where each bit corresponds to a potential child octant. If the bit is 0, the child is a tile; if 1, it is a sub-node.
*   `mInternalDAT`: An array storing tile values or child node pointers/indices, indexed by the calculated offset.

**Formulas:**
The offset is calculated using bitwise operations to extract the relative position within the child block and pack it into a Morton-like index.

For an **InternalNode**, the offset is:
$$
\text{internalOffset} = \left( \left( (x \ \& \ ((1 \ll \text{sLog2X}) - 1)) \gg \text{Child::sLog2X} \right) \ll \text{Log2YZ} \right) + \left( \left( (y \ \& \ ((1 \ll \text{sLog2Y}) - 1)) \gg \text{Child::sLog2Y} \right) \ll \text{Log2Z} \right) + \left( (z \ \& \ ((1 \ll \text{sLog2Z}) - 1)) \gg \text{Child::sLog2Z} \right)
$$
Where $\text{Log2YZ} = \text{Log2Y} + \text{Log2Z}$.

For a **LeafNode**, the offset is:
$$
\text{leafOffset} = \left( (x \ \& \ ((1 \ll \text{sLog2X}) - 1)) \ll (\text{Log2Y} + \text{Log2Z}) \right) + \left( (y \ \& \ ((1 \ll \text{sLog2Y}) - 1)) \ll \text{Log2Z} \right) + (z \ \& \ ((1 \ll \text{sLog2Z}) - 1))
$$

**Steps:**
1.  **Compute Internal Offset:** Calculate `internalOffset` using the formula above. This involves masking the coordinates to the block size, shifting to align bits, and summing them.
2.  **Check Child Mask:** Check if the bit at index `internalOffset` in `mChildMask` is 0.
3.  **Handle Tile:** If the bit is 0, the coordinate $(x, y, z)$ lies within a constant tile. Return the tile value from `mInternalDAT` at index `internalOffset` and terminate traversal [S1, unofficial].
4.  **Handle Child Node:** If the bit is 1, extract the child node from `mInternalDAT` at index `internalOffset` and continue traversal recursively or iteratively [S1, unofficial].
5.  **Compute Leaf Offset (if LeafNode reached):** If a `LeafNode` is reached, calculate `leafOffset` using the LeafNode formula to index directly into the leaf's data array [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_internal_offset(x, y, z, sLog2X, sLog2Y, sLog2Z, Log2Y, Log2Z):
    # Calculate block-local coordinates
    # Masking: (1 << sLog2N) - 1 creates a mask of sLog2N ones
    x_local = x & ((1 << sLog2X) - 1)
    y_local = y & ((1 << sLog2Y) - 1)
    z_local = z & ((1 << sLog2Z) - 1)
    
    # Shift to align bits for Morton-like packing
    # Note: Child::sLog2X is typically 1 for standard octrees, but generalized here
    x_idx = x_local >> sLog2X
    y_idx = y_local >> sLog2Y
    z_idx = z_local >> sLog2Z
    
    # Combine indices
    Log2YZ = Log2Y + Log2Z
    offset = (x_idx << Log2YZ) + (y_idx << Log2Z) + z_idx
    return offset

def calculate_leaf_offset(x, y, z, sLog2X, sLog2Y, sLog2Z, Log2Y, Log2Z):
    # Calculate block-local coordinates
    x_local = x & ((1 << sLog2X) - 1)
    y_local = y & ((1 << sLog2Y) - 1)
    z_local = z & ((1 << sLog2Z) - 1)
    
    # Pack into offset
    # Note: The source formula uses left shifts for leaf offset, implying 
    # x, y, z are treated as indices within the leaf block directly
    offset = (x_local << (Log2Y + Log2Z)) + (y_local << Log2Z) + z_local
    return offset

def traverse_internal_node(x, y, z, mChildMask, mInternalDAT, sLog2X, sLog2Y, sLog2Z, Log2Y, Log2Z):
    offset = calculate_internal_offset(x, y, z, sLog2X, sLog2Y, sLog2Z, Log2Y, Log2Z)
    
    # Check if child exists (bit is 1) or is a tile (bit is 0)
    if not (mChildMask & (1 << offset)):
        # Tile value
        return mInternalDAT[offset]
    else:
        # Child node exists, continue traversal
        child_node = mInternalDAT[offset]
        # Implementation detail: [G] The specific method to pass coordinates 
        # to the child node (e.g., subtracting base coordinates) is not 
        # detailed in the source, but standard octree traversal requires 
        # adjusting x, y, z relative to the child's origin.
        return traverse(child_node, x, y, z)
```

**Parameters:**
*   `sLog2X`, `sLog2Y`, `sLog2Z`: Compile-time constants. Typical values depend on the VDB configuration, often corresponding to the log2 of the number of voxels per child block dimension.
*   `Log2Y`, `Log2Z`: Derived constants used for bit shifting.

**Edge Cases and Pitfalls:**
*   **Negative Coordinates:** Bit-wise AND is well-defined for signed integers in two's complement representation, allowing these operations to work correctly for negative $(x, y, z)$ [S1, unofficial]. However, bit-wise shifts must be performed on unsigned integers to ensure defined behavior [S1, unofficial].
*   **Compile-Time Constants:** The efficiency relies on `sLog2X`, `sLog2Y`, `sLog2Z` being known at compile time, allowing the compiler to optimize shifts and masks [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** InternalNode offset calculation reduces to three bitwise ANDs, five bit shifts, and two additions [S1, unofficial]. LeafNode offset calculation reduces to three bitwise ANDs, two left shifts, and two additions [S1, unofficial]. This is $O(1)$ with very low constant factors.
*   **Memory Complexity:** $O(1)$ auxiliary space.
*   **Strengths:** Extremely fast, avoiding expensive division/multiplication/modulus operations. Works for negative coordinates. Deterministic performance.
*   **Limitations:** Requires compile-time knowledge of block sizes. Only applicable to regular hierarchical structures like VDB octrees.
*   **When to use:** When traversing sparse voxel grids with known, fixed block sizes at compile time, such as in VDB-based rendering or simulation.
*   **When not to use:** For dynamic structures where block sizes are not known at compile time, or for non-hierarchical data structures.

## Background
Direct Access Offset Calculation was introduced in the VDB (Voxel Database) system by Ken Museth et al. [S1, unofficial]. It is part of the VDB architecture designed for high-resolution sparse volumes with dynamic topology. The method leverages the height-balanced nature of VDB octrees, where the tree depth and branching factors are fixed at runtime, allowing for level-independent, non-recursive lookup logic [S1, unofficial].

## Sources
[S1] VDB: High-Resolution Sparse Volumes with Dynamic Topology (Academy Award Winner) [pdf], http://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf, fetched 2026-10-26 (unofficial)
