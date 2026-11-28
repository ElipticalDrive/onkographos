# Dual Point Map Insertion with Linear Probing

Also called: Dual Point Map, Linear Probing Hash Map Insertion

## Overview
This method stores dual points in a hash table using linear probing to handle collisions, mapping unique point identifiers to their 3D positions via a combined 64-bit key-value pair [S1, unofficial]. It builds on the hashing strategy of [Hash Grids](936-hash-grids.md) by hashing vertex positions to bucket indices [G].

## How it works
**Inputs and Outputs:** A point identifier `id` (integer), a 3D position `p` (float3), and a global counter `CachedPointCounter` (integer) are provided as inputs [S1, unofficial]. The `DualPointMap` hash table is updated with the new entry, and the `PrecalculatedDualPoints` array is extended with `p` [S1, unofficial].

**Data Structures:** `DualPointMap` is an array of 64-bit unsigned integers where each entry stores a key-value pair [S1, unofficial]. The upper 32 bits store the point `id` (masked to 18 bits) and the lower 32 bits store the `pointIdx` (index into `PrecalculatedDualPoints`) [S1, unofficial]. The sentinel value `~0ull` (all bits set, `0xFFFFFFFFFFFFFFFF`) indicates an empty bucket [S1, unofficial]. `PrecalculatedDualPoints` is an array of 3D vectors (`float3`) where index `i` corresponds to the `pointIdx` stored in the hash map [S1, unofficial]. `CachedPointCounter` is a global integer variable initialized to 0, incremented upon each successful insertion [S1, unofficial].

**Constants:** `MASK = 0x3ffff` (binary `111111111111111111`, 18 bits) and `SENTINEL = 0xFFFFFFFFFFFFFFFF` [S1, unofficial].

**Numbered Steps:**
1. Mask the input `id` with `0x3ffff` to keep only the lower 18 bits [S1, unofficial].
2. Loop indefinitely to find an empty bucket in `DualPointMap` [S1, unofficial].
3. Read the value `kv` from `DualPointMap[id]` [S1, unofficial].
4. If `kv` equals `0xFFFFFFFFFFFFFFFF`, the bucket is empty. Proceed to insertion [S1, unofficial].
5. Assign the current `CachedPointCounter` value to `pointIdx` and increment `CachedPointCounter` by 1 [S1, unofficial].
6. Store the key-value pair in `DualPointMap[id]`: upper 32 bits are `id`, lower 32 bits are `pointIdx` (masked to 32 bits) [S1, unofficial].
7. Store the point position `p` in `PrecalculatedDualPoints` at index `pointIdx` [S1, unofficial].
8. If the bucket is not empty (`kv != 0xFFFFFFFFFFFFFFFF`), increment `id` by 1, masked with `0x3ffff`, and continue the loop from step 3 [S1, unofficial].

**Formulas:**
- Masking: `id = id & 0x3ffff` [S1, unofficial].
- Key-Value Packing: `kv = (id << 32) | pointIdx` [S1, unofficial].
- Linear Probing: `id = (id + 1) & 0x3ffff` [S1, unofficial].
Where `id` is the point identifier, `pointIdx` is the assigned slot index, and `kv` is the 64-bit packed value [S1, unofficial].

**Parameters:** Typical values are not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Hash Table Full:** If the table becomes full, the loop will cycle indefinitely [S1, unofficial]. The implementation assumes the table has sufficient capacity or handles fullness externally (not specified in sources) [G].
- **ID Masking:** The 18-bit mask limits the number of unique IDs to $2^{18} = 262,144$ [S1, unofficial]. IDs outside this range collide before hashing [G].
- **Sentinel Value:** The sentinel `~0ull` must not be a valid key-value pair [S1, unofficial]. Since valid `id` values are masked to 18 bits, the upper 14 bits of the upper 32 bits are always 0, ensuring `~0ull` is distinct from any valid entry [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state
DualPointMap = []  # Array of uint64, size not given in sources
PrecalculatedDualPoints = []  # Array of float3
CachedPointCounter = 0
MASK = 0x3FFFF
SENTINEL = 0xFFFFFFFFFFFFFFFF

def insert_dual_point(id: int, p: tuple) -> None:
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
            kv = (id << 32) | (pointIdx & 0xFFFFFFFF)
            
            # Store in hash map
            DualPointMap[id] = kv
            
            # Store position in separate array
            PrecalculatedDualPoints.append(p)
            
            return
        
        # Collision: linear probe
        id = (id + 1) & MASK
```

## Subtopics

### Dual Point Map Lookup with Linear Probing
Retrieves a dual point position given its ID using linear probing [S1, unofficial]. It follows the same hashing and probing logic as insertion but checks for key equality rather than emptiness [S1, unofficial]. It returns a boolean indicating success and the position via an output parameter [S1, unofficial]. This variant is chosen when retrieving existing points or verifying point existence is required [G].

## Operating characteristics
*   **Time Complexity:** Average case $O(1)$ for insert and lookup [S1, unofficial]. Worst case $O(N)$ due to linear probing collisions [G].
*   **Memory Complexity:** $O(M)$ for the hash table, where $M$ is the number of buckets [S1, unofficial]. Each bucket uses 64 bits and the position array uses 96 bits (3 floats) per point [S1, unofficial].
*   **Measured performance, scaling, strengths, limitations, and usage guidelines:** Not given in the sources [S1, unofficial].

## Background
Originates from the `dualmc-compute` repository [S1, unofficial]. History, development, and broader usage context are not given in the sources [S1, unofficial].

## Sources
[S1] Markyparky56/dualmc-compute, https://github.com/Markyparky56/dualmc-compute, fetched 2026-10-27 (unofficial)
