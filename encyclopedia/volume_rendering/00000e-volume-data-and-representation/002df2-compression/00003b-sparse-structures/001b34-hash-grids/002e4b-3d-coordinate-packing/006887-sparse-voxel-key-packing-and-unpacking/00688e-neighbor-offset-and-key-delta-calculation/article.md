# Neighbor Offset and Key Delta Calculation


## Overview
This method generates constant additive deltas that allow finding the packed keys of neighboring voxels by simple integer addition to a current voxel's packed key. It solves the problem of efficiently constructing adjacency graphs or performing neighborhood queries on sparse voxel clouds without resorting to expensive tree traversals or dense grid lookups. It differs from dense grid approaches by scaling with the number of voxels rather than the bounding box volume, and from tree-based methods by using sorted-array binary searches on linearized coordinates [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Connectivity type (integer: 6, 18, or 26).
*   **Output:** A list of integer deltas corresponding to the valid neighbor offsets.

**Data Structures:**
*   The packed key is a 64-bit integer. Based on the parent method [Sparse Voxel Key Packing and Unpacking](20132-sparse-voxel-key-packing-and-unpacking.md), coordinates are packed into 21-bit fields per axis.
*   Let `_X_SHIFT`, `_Y_SHIFT`, and `_Z_SHIFT` be the bit shifts for the X, Y, and Z axes respectively. Typically, if Z is the least significant, `_Z_SHIFT = 0`, `_Y_SHIFT = 21`, `_X_SHIFT = 42`. The remaining bits are unused or reserved.

**Steps:**
1.  Generate all 27 combinations of offset vectors $(dx, dy, dz)$ where $dx, dy, dz \in \{-1, 0, 1\}$.
2.  Exclude the zero vector $(0, 0, 0)$, as it represents the voxel itself.
3.  Calculate the L1 norm (Manhattan distance) for each offset: $L1 = |dx| + |dy| + |dz|$.
4.  Filter offsets based on the requested connectivity:
    *   **6-connectivity:** Keep offsets where $L1 == 1$ (face neighbors).
    *   **18-connectivity:** Keep offsets where $L1 \le 2$ (face and edge neighbors).
    *   **26-connectivity:** Keep all offsets (face, edge, and corner neighbors).
5.  Convert each valid offset $(dx, dy, dz)$ into a key delta using the packing shift constants.

**Formula:**
The key delta $\Delta$ for an offset $(dx, dy, dz)$ is calculated as:
$$ \Delta = (dx \ll \_X\_SHIFT) + (dy \ll \_Y\_SHIFT) + dz $$

**Note on Addition vs. Bitwise OR:**
Addition is used instead of bitwise OR because the offset components $dx, dy, dz$ are signed integers (specifically -1, 0, or 1). The packing scheme guarantees that the fields do not overlap in a way that causes carry propagation between axes for these specific small values, but addition correctly handles the two's complement representation of negative offsets when shifted into their respective fields. Specifically, $-1$ in a 21-bit field is `0xFFFFF800` (if viewed as a larger integer) or simply the bit pattern for -1 shifted. Adding the shifted signed values correctly constructs the delta key that, when added to the original key, yields the neighbor's key.

**Parameters:**
*   **connectivity:** Integer, 6, 18, or 26. Defines the neighborhood structure. 6 is face-connected, 18 adds edge-connected, 26 adds corner-connected.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_neighbor_deltas(connectivity: int):
    """
    Calculate key deltas for neighbor lookup.
    
    Args:
        connectivity: 6, 18, or 26.
        
    Returns:
        List of integer deltas.
    """
    # Shifts based on 21-bit packing per axis
    # Z is LSB, Y is middle, X is MSB
    _Z_SHIFT = 0
    _Y_SHIFT = 21
    _X_SHIFT = 42
    
    deltas = []
    
    # Step 1 & 2: Generate all non-zero offsets in {-1, 0, 1}^3
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                
                # Step 3: Calculate L1 norm
                l1 = abs(dx) + abs(dy) + abs(dz)
                
                # Step 4-6: Filter by connectivity
                keep = False
                if connectivity == 6:
                    if l1 == 1:
                        keep = True
                elif connectivity == 18:
                    if l1 <= 2:
                        keep = True
                elif connectivity == 26:
                    keep = True  # All non-zero offsets
                
                if keep:
                    # Step 7: Convert to key delta
                    # Using addition to handle signed shifts correctly
                    delta = (dx << _X_SHIFT) + (dy << _Y_SHIFT) + (dz << _Z_SHIFT)
                    deltas.append(delta)
    
    return deltas
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on the fact that adding the delta to the packed key does not cause unintended carries between the 21-bit fields. Since offsets are only -1, 0, or 1, and the fields are separated by 21 bits, this holds true for standard 64-bit integers.
*   **Boundaries:** This method calculates *relative* deltas. It does not handle boundary checks (e.g., if a neighbor is outside the valid coordinate space). Boundary checks must be performed after unpacking the neighbor key or by validating the resulting key against the valid range.
*   **Signedness:** Ensure the language/runtime handles bitwise shifts on negative integers correctly (arithmetic vs. logical shift). In Python, shifts on negative integers behave as expected for two's complement logic in this context because the result is added to the key.

## Subtopics

### [Sparse Voxel Adjacency via Packed Keys](23017-sparse-voxel-adjacency-via-packed-keys.md)
Constructs the undirected adjacency graph of a sparse voxel cloud by leveraging these packed keys and constant additive deltas. It differs from dense grid approaches by scaling with the number of voxels rather than the bounding box volume, and from tree-based methods by using sorted-array binary searches on linearized coordinates.

## Operating characteristics
*   **Time Complexity:** $O(1)$ to generate the deltas (constant number of offsets: 6, 18, or 26). $O(N)$ to find neighbors for $N$ voxels if using a hash set or sorted array lookup.
*   **Memory Complexity:** $O(1)$ for storing the delta table (max 26 integers).
*   **Strengths:** Extremely fast neighbor lookup; no tree traversal overhead; simple implementation.
*   **Limitations:** Requires the voxel keys to be sorted or stored in a hash set for efficient lookup. Does not inherently handle boundary conditions.
*   **When to use:** When constructing adjacency graphs or performing neighborhood operations on sparse voxel data where the voxel cloud is not dense.
*   **When not to use:** For dense grids where direct array indexing is faster, or when boundary handling is complex and requires explicit geometric checks.

## Background
This method is part of the `sparsecubes` library ecosystem, specifically within the `sparsecubes._keys` module, which provides shared packed-key machinery for sparse set-algebra and measurement primitives [S1, unofficial]. It enables efficient neighbor lookups by leveraging the linearized nature of packed voxel keys.

## Sources
[S1] navis-org/sparse-cubes, https://github.com/navis-org/sparse-cubes, fetched 2026-10-26 (unofficial)
