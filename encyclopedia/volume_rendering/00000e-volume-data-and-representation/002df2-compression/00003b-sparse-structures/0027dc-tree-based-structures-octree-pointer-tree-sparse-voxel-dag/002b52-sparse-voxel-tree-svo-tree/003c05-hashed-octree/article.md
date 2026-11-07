# Hashed-Octree

Also called: Hash-Tables, Linear World Octree with 53-bit Hash Key, Sparse Linear Octree

## Overview
The Hashed-Octree is a sparse voxel representation that combines octrees with hash tables to provide constant-time data access complexity for huge 3D environments, addressing the $O(d)$ access time of standard Sparse Voxel Octrees (SVOs) [S1, unofficial]. It eliminates the need for explicit pointers by storing octants in a hash map using a unique 53-bit locational code key, allowing intermediate octants to be omitted [S2]. Unlike its siblings which may rely on Morton codes or pointer trees, this method uses a direct hash of coordinate-aligned root keys for top-level access, reducing data access complexity from $O(d)$ to $O(d_b)$ where $d_b$ is the depth of internal octrees (typically $d=1$) [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   3D voxel index coordinates $(x, y, z)$.
*   Target Level of Detail (LOD).
*   Base cell size $s_0$.
*   Compile-time constants: `Log2x`, `Log2y`, `Log2z` (grid alignment), `Log2N` (hash length).

**Outputs:**
*   A reference to the octant node (or null if empty) stored in the hash map.
*   A unique 53-bit key for the octant at the specified LOD.

### Data Structures
**Hash Map:**
*   Uses a dense hash map (e.g., `google::dense_hash_map`) to store octree root nodes [S1, unofficial].
*   Keys are 53-bit unsigned integers.
*   Values are pointers or handles to octant nodes.

**Octant Node:**
*   **Intermediate Octants:** Store an 8-bit mask indicating the existence of their 8 immediate children in the next lower LOD grid [S2].
*   **Leaf Octants:** Store voxel data (e.g., density, color).

**Key Layout (53-bit):**
*   The key encodes X, Y, and Z coordinates. LOD is not part of the key but selects a distinct octant grid [S2].
*   **Bit Allocation:**
    *   X coordinate: 40% of available bits (21 bits).
    *   Y coordinate: 20% of available bits (11 bits).
    *   Z coordinate: 40% of available bits (21 bits).
    *   Unused/Sign bits: Remaining bits (10 bits) are unusable due to JavaScript 32-bit bitwise operation constraints [S2].
*   **Coordinate Range:**
    *   X and Z: $2^{21}$ (2,097,152) distinct integers.
    *   Y: $2^{11}$ (2,048) distinct integers [S2].
*   **World Extent:** Assuming a base cell size $s_0 = 20$ meters, the managed world space is $41,943.04 \text{ km} \times 40.96 \text{ km} \times 41,943.04 \text{ km}$ [S2].

### Steps

1.  **Coordinate Translation:**
    Translate the input 3D position to origin $(0,0,0)$ to get zero-based unsigned coordinates. Divide by the cell size of the target LOD grid: $s_{lod} = s_0 \times 2^{lod}$. Truncate fractional parts to obtain integer key coordinates $(k_x, k_y, k_z)$ [S2].

2.  **Root Key Alignment (S1 Method):**
    Compute the signed `rootKey` for voxel index coordinates $(x, y, z)$ using bit-wise operations to align coordinates to a grid defined by `Log2x`, `Log2y`, `Log2z`. This reduces to hardware AND instructions at compile time [S1, unofficial].
    $$
    \begin{aligned}
    \text{rootKey}[0] &= x \ \& \ \sim((1 \ll \text{Log2x}) - 1) \\
    \text{rootKey}[1] &= y \ \& \ \sim((1 \ll \text{Log2y}) - 1) \\
    \text{rootKey}[2] &= z \ \& \ \sim((1 \ll \text{Log2z}) - 1)
    \end{aligned}
    $$
    *Note: The S1 method uses aligned indices directly. The S2 method uses translated coordinates. Implementers should choose based on whether the input is already grid-aligned (S1) or a continuous world position (S2).*

3.  **Hash Generation (S1 Method):**
    Process the `rootKey` into a hash value using prime number multipliers, XOR operations, and a bit-wise AND mask [S1, unofficial].
    $$
    \text{rootHash} = (1 \ll (\text{Log2N} - 1)) \ \& \ (\text{rootKey}[0] \times 73856093 \oplus \text{rootKey}[1] \times 19349663 \oplus \text{rootKey}[2] \times 83492791)
    $$
    Where:
    *   $73856093, 19349663, 83492791$ are large prime numbers [S1, unofficial].
    *   $\oplus$ is binary XOR.
    *   $N$ is the constant length of the hash being generated [S1, unofficial].

4.  **Key Packing (S2 Method):**
    Pack the integer key coordinates $(k_x, k_y, k_z)$ into a unique 53-bit key. Due to JavaScript 32-bit bitwise limitations, keys must be split into high and low parts for coordinate retrieval [S2]. The bit allocation is 21 bits for X, 11 for Y, 21 for Z [S2].

5.  **Hash Map Access:**
    Use the computed `rootHash` (S1) or the packed key (S2) to access and store octree root nodes in the dense hash map [S1, unofficial]. If the key exists, return the node; otherwise, the space is empty.

6.  **LOD Translation (S2 Method):**
    To translate key coordinates from current LOD to target LOD:
    *   If $\text{target} > \text{current}$: Right shift by $\text{target} - \text{current}$ bits.
    *   If $\text{target} < \text{current}$: Left shift by $\text{current} - \text{target}$ bits [S2].

7.  **Child Index Calculation:**
    To find the index of a child octant relative to its parent, calculate the positional offset remainders $(r_x, r_y, r_z)$ using bitwise modulo:
    $$
    \begin{aligned}
    r_x &= i \ \& \ 1 \\
    r_y &= (i \gg 1) \ \& \ 1 \\
    r_z &= (i \gg 2) \ \& \ 1
    \end{aligned}
    $$
    The index into the offset table is:
    $$
    \text{Index} = (r_x \ll 2) + (r_y \ll 1) + r_z
    $$
    This index corresponds to the 8-bit mask stored in intermediate octants [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants from S1
PRIME_X = 73856093
PRIME_Y = 19349663
PRIME_Z = 83492791
LOG2N = 16  # Example hash length constant
LOG2X = 4   # Example grid alignment constants
LOG2Y = 4
LOG2Z = 4

def compute_root_hash_s1(x, y, z):
    """
    S1 Method: Compute hash for aligned voxel indices.
    """
    # Step 1: Align coordinates
    # ~((1 << Log2) - 1) creates a mask that clears the lower Log2 bits
    mask_x = ~((1 << LOG2X) - 1)
    mask_y = ~((1 << LOG2Y) - 1)
    mask_z = ~((1 << LOG2Z) - 1)
    
    root_key_x = x & mask_x
    root_key_y = y & mask_y
    root_key_z = z & mask_z
    
    # Step 2: Hash generation
    # (1 << (Log2N - 1)) creates the mask for the hash size
    hash_mask = (1 << (LOG2N - 1))
    
    # XOR and multiply
    hash_val = (root_key_x * PRIME_X) ^ (root_key_y * PRIME_Y) ^ (root_key_z * PRIME_Z)
    
    # Apply mask
    root_hash = hash_val & hash_mask
    
    return root_hash

def compute_key_s2(world_x, world_y, world_z, lod, s0):
    """
    S2 Method: Compute 53-bit key for world coordinates.
    """
    # Step 1: Translate to origin and divide by cell size
    # Assuming world_x, world_y, world_z are relative to origin (0,0,0)
    # Cell size at LOD
    cell_size = s0 * (2 ** lod)
    
    # Truncate fractional parts
    k_x = int(world_x / cell_size)
    k_y = int(world_y / cell_size)
    k_z = int(world_z / cell_size)
    
    # Step 2: Pack into 53-bit key
    # Allocation: 21 bits X, 11 bits Y, 21 bits Z
    # Note: In JavaScript, bitwise ops are 32-bit. 
    # Here we simulate the logical packing.
    
    # Mask for 21 bits
    mask_21 = (1 << 21) - 1
    # Mask for 11 bits
    mask_11 = (1 << 11) - 1
    
    # Ensure coordinates are within range (21 bits for X/Z, 11 for Y)
    # In practice, negative coordinates must be handled via offset or two's complement
    # The source notes imply unsigned zero-based coordinates after translation.
    
    packed_key = (k_x & mask_21) | ((k_y & mask_11) << 21) | ((k_z & mask_21) << 32)
    
    return packed_key

def translate_lod(key, current_lod, target_lod):
    """
    S2 Method: Translate key between LODs.
    """
    if target_lod > current_lod:
        # Right shift for higher LOD (coarser)
        shift = target_lod - current_lod
        # Zero-fill right shift simulation
        # In languages without >>>, use: (key >> shift) & ((1 << (53-shift)) - 1) if needed,
        # but standard right shift on unsigned is usually sufficient if key is positive.
        return key >> shift
    else:
        # Left shift for lower LOD (finer)
        shift = current_lod - target_lod
        return key << shift

def get_child_index(parent_key):
    """
    Calculate child index from parent key's lowest bits.
    """
    rx = parent_key & 1
    ry = (parent_key >> 1) & 1
    rz = (parent_key >> 2) & 1
    return (rx << 2) + (ry << 1) + rz
```

### Parameters
*   **Log2x, Log2y, Log2z:** Compile-time constants defining the grid alignment for the rootKey calculation [S1, unofficial].
*   **N:** The constant length of the hash being generated, used in the mask $(1 \ll \text{Log2N}-1)$ [S1, unofficial].
*   **Key Size:** 53 bits, constrained by JavaScript's IEEE 754 binary64 Double safe integer limit (ES2017) [S2].
*   **Base Cell Size ($s_0$):** Default 20 meters [S2].

### Edge Cases and Pitfalls
*   **JavaScript Bitwise Operations:** JavaScript interprets the result of 32-bit operations as signed integers. To prevent sign bit preservation, a zero-fill right shift (`>>>`) must follow operations [S2]. Example: `((2**32 - 1) | 0) >>> 0 = 4294967295`, whereas without `>>>`, it remains `-1` [S2].
*   **Key Design Limitations:** The key design disallows bit allotments > 32 bits because JS bitwise ops apply to DWords (32-bit). Keys must be split into high and low parts for coordinate retrieval [S2].
*   **Hash Collisions:** The S2 method claims no collisions in the underlying hash maps due to the unique 53-bit locational code [S2]. The S1 method uses a hash function which may have collisions, mitigated by the dense hash map's collision resolution [S1, unofficial].
*   **Root Node Cost:** Creating and deleting nodes at the top of hashed Octrees is very costly because the locational code of all nodes below the new root node gets 3 bits longer and must be updated. Consequently, the hash map must be updated as well [S2].

## Operating characteristics
*   **Time Complexity:** Data access complexity is reduced from $O(d)$ in standard octrees to $O(d_b)$ where $d_b$ is the depth of internal octrees (typically $d=1$) [S1, unofficial]. This provides a speedup factor of 16 for a standard octree depth of $d=16$ [S1, unofficial].
*   **Measured Performance:** Access time for Hashed-Octree is $0.45\mu s$ compared to $2.55\mu s$ for standard Octree (d=16) [S1, unofficial]. Both support a max resolution of $32768^3$ ($327 m^3$ @ 1cm) [S1, unofficial].
*   **Memory Complexity:** Requires less memory than traditional pointer-based octrees because no pointers need to be stored; the tree hierarchy is encoded in the locational codes [S2].
*   **Strengths:** Constant data access for huge 3D environments; efficient storage; fast data access [S1, unofficial].
*   **Limitations:** High cost for root node updates; constrained by 53-bit integer limits in JavaScript environments; hash collisions possible in S1 implementation [S1, unofficial][S2].
*   **When to Use:** Suitable for incremental 3D modeling from camera-based 3D sensors for large-scale environments like cities or countryside [S1, unofficial].
*   **When Not to Use:** When frequent top-level node creation/deletion is required, or when working in environments without 64-bit integer support and 53-bit precision is insufficient [S2].

## Background
The concept of linear octrees stems from linear quadtrees first proposed by [26] (cited in S2). Linear octrees store all their octants in a hash map, identified by locational codes. The Hashed-Octree specifically was proposed to address the $O(d)$ access time of Sparse Voxel Octrees (SVOs) [S1, unofficial]. The approach was inspired by the Voxel Farm engine [28] (cited in S2). It is applied for incremental 3D modeling from camera-based 3D sensors, enabling the fusion of tremendous amounts of data into consistent large 3D models [S1, unofficial].

## Sources
[S1] Sparse Volumes for Large Scale 3D Modelling, https://elib.dlr.de/96355/1/HRSST_SIFO_Funk.pdf, fetched 2026-10-26 (unofficial)
[S2] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
