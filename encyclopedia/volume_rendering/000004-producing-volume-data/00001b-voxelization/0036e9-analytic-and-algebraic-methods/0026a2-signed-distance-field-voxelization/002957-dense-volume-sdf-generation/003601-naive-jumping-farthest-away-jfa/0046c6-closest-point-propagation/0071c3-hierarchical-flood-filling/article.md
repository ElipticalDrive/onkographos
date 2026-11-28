# Hierarchical Flood-Filling

Also called: Bottom-up Sign Assignment, Hierarchical Flood-Fill, Hierarchical sign flood-fill, Sign Propagation

## Overview
Hierarchical Flood-Filling is a bottom-up algorithm that assigns correct signed distance values to inactive voxels outside the narrow band by propagating signs from active voxels within the narrow band [S1, unofficial]. It solves the problem of determining the inside/outside topology for voxels not explicitly computed during the narrow-band construction [S1, unofficial]. Unlike methods that propagate signs between nodes at the same level, this approach uses the hierarchical structure to ensure strict bottom-up propagation, avoiding cross-node propagation at the same level and reducing overhead [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A hierarchical volume data structure (VDB) where LeafNodes contain signed distance values and a validity mask (`mValueMask`). The narrow-band construction guarantees that every LeafNode contains at least one active value (a voxel with a valid signed distance) [S1, unofficial].
*   **Output:** The same data structure with all inactive voxels assigned a sign consistent with their nearest active neighbor, effectively filling the volume with signed distances outside the narrow band.

**Data Structures:**
*   **LeafNode:** Contains a dense array of values (`mLeafDAT.values`) and a bit mask (`mValueMask`) indicating which voxels are active (part of the narrow band).
*   **Mask Operations:**
    *   `findFirst0n()`: Returns the index of the first inactive (0) bit in the mask.
    *   `is0n(index)`: Returns true if the bit at `index` is inactive (0).

**Algorithm Steps:**
The algorithm operates recursively from the leaf level up to the root. At each level, it performs a scanline flood-fill within each node, using signs from active voxels to initialize inactive ones.

1.  **Leaf Node Processing:** For each LeafNode, perform the following scanline traversal to propagate signs within the leaf:
    *   Initialize `i` to the first inactive bit index: `i = mValueMask.findFirst0n()`.
    *   Iterate `x` from 0 to $2^{\text{Log2X}} - 1$:
        *   Calculate base index `x00 = x << (\text{Log2Y} + \text{Log2Z})`.
        *   If `mValueMask.is0n(x00)` is true, update `i = x00`.
        *   Initialize `j = i`.
        *   Iterate `y` from 0 to $2^{\text{Log2Y}} - 1$:
            *   Calculate index `xy0 = x00 + (y << \text{Log2Z})`.
            *   If `mValueMask.is0n(xy0)` is true, update `j = xy0`.
            *   Initialize `k = j`.
            *   Iterate `z` from 0 to $2^{\text{Log2Z}} - 1$:
                *   Calculate index `xyz = xy0 + z`.
                *   If `mValueMask.is0n(xyz)` is true, update `k = xyz`.
                *   Else (voxel is active):
                    *   Assign the sign of the active voxel at `k` to the current voxel `xyz` using `copysign`. Specifically: `values[xyz] = copysign(values[xyz], values[k])`.
2.  **Recursive Propagation:** After processing all LeafNodes, apply the same scanline logic recursively to parent nodes. The signs computed at the lower level serve as boundary values for the higher level.
3.  **Termination:** The process terminates at the top level (RootNode). Because the algorithm is strictly bottom-up, no sign propagation occurs between sibling nodes at the same level [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_flood_fill(node):
    """
    Recursively flood-fill signs in a VDB node.
    node: A node object with 'children' (if internal) or 'values' and 'mask' (if leaf).
    """
    if node.is_leaf():
        # Process leaf node
        # Find first inactive bit to initialize search
        i = node.mask.find_first_0n()
        
        # Dimensions are derived from node configuration (Log2X, Log2Y, Log2Z)
        dim_x = 1 << node.Log2X
        dim_y = 1 << node.Log2Y
        dim_z = 1 << node.Log2Z
        
        for x in range(dim_x):
            x00 = x << (node.Log2Y + node.Log2Z)
            if node.mask.is_0n(x00):
                i = x00
            
            j = i
            for y in range(dim_y):
                xy0 = x00 + (y << node.Log2Z)
                if node.mask.is_0n(xy0):
                    j = xy0
                
                k = j
                for z in range(dim_z):
                    xyz = xy0 + z
                    if node.mask.is_0n(xyz):
                        k = xyz
                    else:
                        # Active voxel: propagate sign from the last seen inactive source 'k'
                        # copysign(y, x) returns y with the sign of x
                        node.values[xyz] = copysign(node.values[xyz], node.values[k])
    else:
        # Process internal node: recurse to children first (bottom-up)
        for child in node.children:
            hierarchical_flood_fill(child)
        
        # Note: The source describes the scanline algorithm applied recursively to parent nodes.
        # The exact implementation for internal nodes mirrors the leaf logic but operates on 
        # the aggregated values/signs from children. The key constraint is that propagation 
        # is strictly bottom-up, so parent nodes only use values finalized by their children.
        # [G] Implementation detail: Internal nodes typically aggregate child results. 
        # The scanline logic above applies to the dense representation within a node. 
        # For internal nodes, the "values" are often pointers or aggregated metadata, 
        # but the sign propagation logic remains: use the sign of the nearest active 
        # descendant to fill inactive slots in the parent's representation.
```

**Parameters:**
*   **VDB Configuration:** Defined by the number of levels and the leaf size (e.g., `[7,6,5,4,3]`). The algorithm's performance depends heavily on this configuration [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty LeafNodes:** The algorithm relies on the guarantee that all LeafNodes contain at least one active value from the narrow band construction. If a LeafNode is entirely inactive, the initialization of `i` and subsequent propagation may fail or produce incorrect signs [S1, unofficial].
*   **Topology:** The algorithm is designed for closed manifold narrow bands. While the VDB dilation algorithm has no topology limitations, the sign propagation assumes a connected active region to propagate signs from [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   The algorithm is efficient, with performance scaling well with resolution. It is described as incurring negligible computational overhead in the overall scan conversion [S1, unofficial].
*   The strict bottom-up nature avoids the overhead of cross-node propagation at the same level [S1, unofficial].

**Measured Performance:**
*   **512^3 Resolution (794,720 active voxels):**
    *   Configurations `[7,6,5,4,3]`, `[6,5,4,3]`, `[6,5,3]`: < 0.01s [S1, unofficial].
    *   Configuration `[10,3]`: 1.87s [S1, unofficial].
*   **16,384^3 Resolution (815,936,095 active voxels):**
    *   Configurations `[7,6,5,4,3]`, `[6,5,4,3]`: 3.45s [S1, unofficial].
    *   Configuration `[6,5,3]`: 3.38s [S1, unofficial].
    *   Configuration `[10,3]`: N/A (likely due to memory constraints or resolution limits) [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Orders of magnitude faster than shallow configurations like `[10,3]` at low resolutions because it avoids the bottleneck of flood-filling a large dense RootNode [S1, unofficial]. Near real-time performance [S1, unofficial].
*   **Limitations:** Performance degrades for shallow configurations (e.g., `[10,3]`) where the root node is large and dense, as the hierarchical advantage is lost [S1, unofficial].

**When to Use:**
*   Use when you need to quickly assign signs to inactive voxels in a hierarchical volume structure (VDB) after narrow-band construction.
*   Prefer deeper hierarchical configurations (e.g., `[7,6,5,4,3]`) over shallow ones (e.g., `[10,3]`) for better performance on lower resolutions.

## Background
**Origin:**
*   The algorithm was introduced in the VDB (Volumetric Dynamic Binary) paper by Ken Museth [S1, unofficial].
*   It is part of the VDB data structure, which is designed for high-resolution sparse volumes with dynamic topology [S1, unofficial].

**History and Development:**
*   The VDB structure won an Academy Award [S1, unofficial].
*   The hierarchical flood-fill is a key component of VDB's ability to handle constructive solid geometry (CSG) and other operations efficiently [S1, unofficial].

**Where it is Used:**
*   Used in VDB for defining the signs of voxels outside the narrow band of a level set [S1, unofficial].
*   Essential for CSG operations, where consistent signed distances are required for boolean operations between level sets [S1, unofficial].

## Sources
[S1] VDB: High-Resolution Sparse Volumes with Dynamic Topology (Academy Award Winner) [pdf], http://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf, fetched 2026-10-26 (unofficial)
