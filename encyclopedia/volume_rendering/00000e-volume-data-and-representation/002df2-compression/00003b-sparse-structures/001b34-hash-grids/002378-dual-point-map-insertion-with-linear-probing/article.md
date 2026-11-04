# Dual Point Map Insertion with Linear Probing

Also called: Dual Point Map, Linear Probing Hash Map Insertion

## Overview
This method manages a sparse collection of "dual points" (typically edge midpoints or specific geometric features) by mapping unique point identifiers to their 3D positions using a hash table with linear probing [S1, unofficial]. It solves the problem of efficiently storing and retrieving geometric data associated with specific IDs in a compact, cache-friendly structure, building on the hashing strategy of [Hash Grids](936-hash-grids.md). It is distinguished from sibling structures like [Voxel World Data Structure](1101-voxel-world-data-structure.md) by its explicit handling of point IDs rather than spatial coordinates, and from [SpatialHash3D](5113-spatialhash3d.md) by its fixed-size 64-bit key-value packing and specific 18-bit ID masking.

## How it works
**Inputs:** A unique point identifier `id` (integer), a 3D position vector `p` (float3), and a global counter `CachedPointCounter` (integer) tracking the next available slot in the position array.
**Outputs:** The `DualPointMap` hash table is updated with the new entry, and `PrecalculatedDualPoints` array is extended with the position `p`.

**Data Structures:**
1.  `DualPointMap`: An array of 64-bit unsigned integers (`uint64`). Each entry stores a key-value pair.
    *   **Layout:** Upper 32 bits store the point `id` (masked to 18 bits). Lower 32 bits store the `pointIdx` (index into `PrecalculatedDualPoints`).
    *   **Sentinel:** The value `~0ull` (all bits set, i.e., `0xFFFFFFFFFFFFFFFF`) indicates an empty bucket.
2.  `PrecalculatedDualPoints`: An array of 3D vectors (`float3`). Index `i` corresponds to the `pointIdx` stored in the hash map.
3.  `CachedPointCounter`: A global integer variable, initialized to 0, incremented upon each successful insertion.

**Constants:**
*   `MASK = 0x3ffff` (binary `111111111111111111`, 18 bits). Used to constrain `id` and hash indices.
*   `SENTINEL = ~0ull` (all 1s).

**Algorithm Steps:**

1.  **Hash Initialization:** Compute the initial hash index `id` by masking the input identifier:
    $$ id \leftarrow id \ \& \ \text{MASK} $$
2.  **Search Loop:** Enter an infinite loop to find a suitable bucket.
3.  **Bucket Read:** Read the value `kv` from `DualPointMap[id]`.
4.  **Empty Check:** If `kv == SENTINEL`, the bucket is empty. Proceed to **Insertion**.
5.  **Collision Handling:** If `kv != SENTINEL`, the bucket is occupied.
    *   Increment `id` by 1.
    *   Mask `id` with `MASK`: `id \leftarrow (id + 1) \ \& \ \text{MASK}`.
    *   Continue to Step 3.
6.  **Insertion:**
    *   Assign the current value of `CachedPointCounter` to `pointIdx`.
    *   Increment `CachedPointCounter` by 1.
    *   Construct the 64-bit key-value pair `kv`:
        *   Upper 32 bits: `id` (masked to 32 bits, though only 18 are significant).
        *   Lower 32 bits: `pointIdx` (masked to 32 bits).
        *   Formula: `kv = (id << 32) | pointIdx`
    *   Write `kv` to `DualPointMap[id]`.
    *   Write the position `p` to `PrecalculatedDualPoints[pointIdx]`.
    *   Exit loop.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state
DualPointMap = array of uint64, size 2^18 (or larger), initialized to ~0ull
PrecalculatedDualPoints = array of float3, dynamic size
CachedPointCounter = 0
MASK = 0x3ffff
SENTINEL = 0xFFFFFFFFFFFFFFFF

def insert_dual_point(id: int, p: float3):
    # Mask ID to 18 bits
    id = id & MASK
    
    while True:
        kv = DualPointMap[id]
        
        # Check if bucket is empty
        if kv == SENTINEL:
            # Assign current counter to point index
            pointIdx = CachedPointCounter
            CachedPointCounter += 1
            
            # Construct 64-bit value: upper 32 bits = id, lower 32 bits = pointIdx
            # Note: id is already masked to 18 bits, fits in 32 bits
            kv = (id << 32) | pointIdx
            
            # Store in hash map
            DualPointMap[id] = kv
            
            # Store position in separate array
            PrecalculatedDualPoints.append(p)
            
            return
        
        # Collision: linear probe
        id = (id + 1) & MASK
```

**Edge Cases and Pitfalls:**
*   **Hash Table Full:** If the table becomes full, the loop will cycle indefinitely. The implementation assumes the table has sufficient capacity or handles fullness externally (not specified in sources) [S1, unofficial].
*   **ID Masking:** The 18-bit mask limits the number of unique IDs to $2^{18} = 262,144$. IDs outside this range are colliding before hashing [S1, unofficial].
*   **Sentinel Value:** The sentinel `~0ull` must not be a valid key-value pair. Since valid `id` values are masked to 18 bits, the upper 14 bits of the upper 32 bits are always 0. However, `~0ull` has all bits set, so it is distinct from any valid entry where the upper bits are not all 1s. Care must be taken if `id` could theoretically be all 1s in the upper 32 bits, but the mask prevents this [S1, unofficial].

## Subtopics

### Dual Point Map Lookup with Linear Probing
Retrieves a dual point position given its ID using linear probing. It follows the same hashing and probing logic as insertion but checks for key equality rather than emptiness. It returns a boolean indicating success and the position via an output parameter. If the bucket is empty (`~0ull`), it returns false. If occupied, it checks if the stored key (upper 18 bits of the 64-bit value) matches the target ID. If it matches, it extracts the index from the lower 32 bits and returns the position from `PrecalculatedDualPoints`. If not, it probes the next bucket. This method is used when the existence of a point is uncertain or when retrieving existing points [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Average case $O(1)$ for insert and lookup. Worst case $O(N)$ where $N$ is the table size, due to linear probing collisions. The 18-bit mask limits the maximum number of unique IDs to 262,144, but the table size itself is not explicitly defined in the sources, though the mask suggests a power-of-two size of at least $2^{18}$ [S1, unofficial].
*   **Memory Complexity:** $O(M)$ for the hash table, where $M$ is the number of buckets. Each bucket uses 64 bits. The position array uses 96 bits (3 floats) per point.
*   **Strengths:** Simple implementation, cache-friendly due to linear probing and contiguous memory access for positions. Compact storage with 64-bit packing.
*   **Limitations:** Performance degrades significantly as the table fills up (high load factor). No support for deletion (sentinel-based empty check only). Limited to 18-bit IDs.
*   **When to use:** When the set of dual points is static or grows monotonically, and fast lookup/insertion is required without complex deletion logic. Suitable for real-time rendering where simplicity and speed are prioritized over dynamic updates.
*   **When not to use:** When frequent deletions are needed, or when the number of unique IDs exceeds 262,144.

## Background
This method is derived from the `dualmc-compute` project, an unofficial implementation for dual marching cubes [S1, unofficial]. It is part of a broader family of sparse data structures used in volume rendering and geometry processing to manage geometric features efficiently. The use of linear probing and 64-bit packing is a common technique in high-performance graphics programming to minimize memory overhead and maximize cache efficiency.

## Sources
[S1] Markyparky56/dualmc-compute, https://github.com/Markyparky56/dualmc-compute, fetched 2026-10-27 (unofficial)
