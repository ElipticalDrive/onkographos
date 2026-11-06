# Hashed-Octree

Also called: Hash-Tables

## Overview
The Hashed-Octree is a sparse voxel representation that combines sparse voxel octrees (SVOs) with hash tables to provide constant-time data access for huge 3D environments [S1, unofficial]. It solves the problem of increasing data access complexity in standard SVOs, which scales as $O(d)$ with tree depth $d$ [S1, unofficial]. By using a hash table to access root nodes of small internal octrees (typically depth $d=1$), it reduces access complexity to $O(d_b)$, offering a significant speedup over standard SVOs [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Voxel index coordinates $(x, y, z)$.
- Output: Access to the corresponding node in the sparse voxel structure.

**Data Structures:**
- **Hash Table:** A dense hash map (specifically `google::dense_hash_map`) storing octree root nodes [S1, unofficial].
- **Internal Structure:** Each entry in the hash table points to a small octree (internal octree) of depth $d_b$ [S1, unofficial].

**Parameters:**
- `Log2x`, `Log2y`, `Log2z`: Compile-time constants defining the grid alignment for the `rootKey` calculation [S1, unofficial].
- `N`: The constant length of the hash being generated [S1, unofficial].
- Prime multipliers: `73856093`, `19349663`, `83492791` [S1, unofficial].

**Steps:**
1. **Compute `rootKey`:** Align the input coordinates $(x, y, z)$ to a grid defined by `Log2x`, `Log2y`, and `Log2z` using bitwise operations. This reduces to hardware AND instructions at compile time [S1, unofficial].
   $$
   \text{rootKey}[i] = \text{coord}_i \ \& \ \sim((1 \ll \text{Log2}_i) - 1) \quad \text{for } i \in \{x, y, z\}
   $$
2. **Compute `rootHash`:** Process the `rootKey` into a hash value using prime multipliers, XOR, and a mask.
   $$
   \text{rootHash} = (1 \ll (\text{Log2}N - 1)) \ \& \ (\text{rootKey}[x] \times 73856093 \ \oplus \ \text{rootKey}[y] \times 19349663 \ \oplus \ \text{rootKey}[z] \times 83492791)
   $$
   Where $\oplus$ is binary XOR, $\&$ is bitwise AND, and $\ll$ is bitwise left shift [S1, unofficial].
3. **Access Node:** Use `rootHash` to access the octree root node in the dense hash map [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants from source [S1, unofficial]
PRIME_X = 73856093
PRIME_Y = 19349663
PRIME_Z = 83492791

def compute_root_hash(x, y, z, Log2x, Log2y, Log2z, Log2N):
    # Step 1: Compute rootKey
    # ~((1 << Log2) - 1) creates a mask with 1s above the Log2 bits
    # In Python, ~ is infinite precision, so we mask to 32-bit unsigned if needed,
    # but the logic is bitwise alignment.
    mask_x = ~((1 << Log2x) - 1)
    mask_y = ~((1 << Log2y) - 1)
    mask_z = ~((1 << Log2z) - 1)
    
    # Note: In C++, & with negative numbers behaves differently than Python.
    # Assuming unsigned 32-bit integers for hardware alignment as per source context.
    rootKey_x = x & mask_x
    rootKey_y = y & mask_y
    rootKey_z = z & mask_z
    
    # Step 2: Compute rootHash
    # (1 << (Log2N - 1)) is the mask for the hash size.
    # Source formula: (1<<Log2N-1) & (...)
    # Assuming Log2N is the exponent for the hash table size power of 2.
    hash_mask = (1 << (Log2N - 1))
    
    # XOR and multiply
    hash_val = (rootKey_x * PRIME_X) ^ (rootKey_y * PRIME_Y) ^ (rootKey_z * PRIME_Z)
    
    # Apply mask
    rootHash = hash_val & hash_mask
    
    return rootHash

# Access logic (conceptual)
def access_voxel(x, y, z, hash_map, Log2x, Log2y, Log2z, Log2N):
    idx = compute_root_hash(x, y, z, Log2x, Log2y, Log2z, Log2N)
    # Retrieve root node from hash map
    root_node = hash_map.get(idx)
    # Navigate internal octree (depth d_b) from root_node
    # [G] Implementation details for internal octree traversal not provided in source
    return root_node
```

**Edge Cases and Pitfalls:**
- **Hash Collisions:** The source uses `google::dense_hash_map`, which handles collisions, but the specific collision resolution strategy is not detailed [S1, unofficial].
- **Precision:** The source implies 32-bit integer arithmetic for the hash calculation [S1, unofficial].

## Subtopics

### [Morton-coded Sparse Voxel Tree](5212-morton-coded-sparse-voxel-tree.md)
Uses Morton codes for spatial ordering instead of hash tables. Choose when spatial locality is critical for cache performance.

### [Spatio-Temporal Sparse Voxel Octree (STSVO)](7493-spatio-temporal-sparse-voxel-octree-stsvo.md)
Extends SVOs for temporal coherence. Choose for dynamic scenes with temporal redundancy.

## Operating characteristics
- **Time Complexity:** $O(d_b)$ where $d_b$ is the depth of internal octrees. Typically $d_b=1$, making it effectively constant time for root access, compared to $O(d)$ for standard SVOs [S1, unofficial].
- **Measured Performance:** Access time of 0.45μs for Hashed-Octree vs. 2.55μs for standard Octree (depth $d=16$) [S1, unofficial]. This represents a speedup factor of 16 [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Max Resolution:** Supports up to $32768^3$ voxels (327m³ @ 1cm) [S1, unofficial].
- **Strengths:** Constant data access complexity for huge 3D environments; significant speedup over standard SVOs [S1, unofficial].
- **Limitations:** Relies on hash table overhead; collision handling details not specified [S1, unofficial].
- **When to Use:** Large-scale 3D modeling from camera-based 3D sensors where fast access is critical [S1, unofficial].
- **When Not to Use:** When spatial locality for cache performance is more important than raw access speed (consider Morton-coded SVO).

## Background
The Hashed-Octree was proposed in "Sparse Volumes for Large Scale 3D Modelling" to address the $O(d)$ data access complexity of standard Sparse Voxel Octrees (SVOs) [S1, unofficial]. It was developed for incremental 3D modeling from camera-based 3D sensors, aiming to fuse large amounts of data into consistent 3D models of cities or large countryside areas [S1, unofficial]. The method applies to security-relevant modeling of dynamic environments, such as automated monitoring of road roughness and obstacles [S1, unofficial].

## Sources
[S1] Sparse Volumes for Large Scale 3D Modelling, https://elib.dlr.de/96355/1/HRSST_SIFO_Funk.pdf, fetched 2026-10-26 (unofficial)
