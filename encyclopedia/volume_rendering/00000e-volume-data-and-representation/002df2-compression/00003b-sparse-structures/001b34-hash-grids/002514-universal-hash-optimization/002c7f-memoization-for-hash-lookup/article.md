# Memoization for Hash Lookup

Also called: Chunk Hashing for Synchronization, External Hash Table Caching for Hash Grids, Hash Cache

## Overview
Memoization for Hash Lookup is an optimization applied to sparse voxel structures (specifically Sparse Voxel Octrees) to avoid redundant recursive hash computations during node searches [S1, unofficial]. It caches the computed hash values of nodes in an external hash table keyed by memory address, returning the cached value if the node's version matches [S1, unofficial][S2, unofficial]. This reduces the computational overhead of traversing the tree structure repeatedly, complementing the uniform distribution provided by Universal Hash Optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A target node (or chunk) to hash, an external hash table (`hashCache`), and the current node version.
- Output: The computed hash value for the node, or the cached value if valid.

**Data Structures:**
- `hashCache`: A map (dictionary) where keys are memory addresses (pointers) of nodes/chunks, and values are objects containing:
  - `v`: The version number of the node at the time of caching.
  - `h`: The computed hash value.

**Steps:**
1. **Initialization**: Initialize or retrieve the `hashCache` map. If null, initialize an empty map [S2, unofficial].
2. **Cache Lookup**: For the target chunk/node, check if an entry exists in `hashCache` using its memory address as the key [S1, unofficial][S2, unofficial].
3. **Version Check**: If an entry exists, compare the stored version `v` with the current node's version [S2, unofficial].
   - If versions match, return the cached hash `h` [S1, unofficial].
   - If versions mismatch or entry is missing, proceed to compute the hash [S1, unofficial][S2, unofficial].
4. **Hash Computation**: Compute the hash based on the node type:
   - **If the chunk has an SDF (Signed Distance Field)**:
     $$ h = \text{FNV-1a}(c.\text{mat}, \text{FNV-1a}(\text{Uint8Array}(c.\text{sdf.buffer}, c.\text{sdf.byteOffset}, c.\text{sdf.byteLength}))) $$
     Where `c.mat` is the material, and the SDF data is read as a byte array [S2, unofficial].
   - **If the chunk is uniform (no SDF)**:
     $$ h = \text{FNV-1a}(\text{Uint8Array}([c.\text{fill} \ \& \ 255, c.\text{fillMat}])) $$
     Where `c.fill` is the fill value and `c.fillMat` is the fill material [S2, unofficial].
5. **Cache Update**: Store the result in `hashCache` with the key as the memory address and value as `{ v: c.version, h }` [S2, unofficial].
6. **Return**: Return the computed hash value [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def memoized_hash_lookup(node, hash_cache):
    """
    Computes or retrieves the hash for a node using memoization.
    
    Args:
        node: The chunk/node object to hash.
        hash_cache: Dictionary mapping memory addresses to {v: version, h: hash}.
        
    Returns:
        The hash value for the node.
    """
    # Get memory address (simulated via id() in Python)
    addr = id(node)
    
    # Step 2: Check cache
    if addr in hash_cache:
        entry = hash_cache[addr]
        # Step 3: Version check
        if entry['v'] == node.version:
            return entry['h']
    
    # Step 4: Compute hash
    if node.has_sdf:
        # Extract SDF bytes
        sdf_bytes = node.sdf.buffer[node.sdf.byteOffset : node.sdf.byteOffset + node.sdf.byteLength]
        # FNV-1a of SDF bytes, then FNV-1a of material
        h_sdf = fnv1a(sdf_bytes)
        h = fnv1a(node.mat, h_sdf)
    else:
        # Uniform chunk: FNV-1a of fill (masked) and fillMat
        fill_byte = node.fill & 255
        h = fnv1a([fill_byte, node.fillMat])
    
    # Step 5: Update cache
    hash_cache[addr] = {'v': node.version, 'h': h}
    
    # Step 6: Return result
    return h
```

**Parameters:**
- `keys`: Optional array of chunk keys to hash. If null, hashes all chunks [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Memory Address Stability**: The cache key relies on memory addresses. If nodes are moved or reallocated in memory, the cache becomes invalid [S1, unofficial].
- **Version Synchronization**: If a node's structure is modified, its entry must be removed from the external hash table to prevent stale results [S1, unofficial].
- **Precision**: Hash collisions are possible but mitigated by the Universal Hash function used in the broader context [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ average case for cache hits; $O(k)$ for cache misses where $k$ is the cost of computing the hash (proportional to SDF size or constant for uniform chunks).
- **Memory Complexity**: $O(N)$ where $N$ is the number of unique nodes/chunks cached.
- **Performance**: Reduces recursive search overhead significantly for static or rarely modified structures [S1, unofficial].
- **Strengths**: Avoids redundant computation; simple to implement.
- **Limitations**: Requires memory address stability; cache invalidation must be handled manually upon modification [S1, unofficial].
- **When to Use**: When hash computation is expensive (e.g., large SDFs) and nodes are frequently queried but rarely modified.
- **When Not to Use**: When nodes are frequently modified and cache invalidation overhead outweighs computation savings.

## Background
Originated in the context of Sparse Voxel Octree (SVO) compression and global illumination calculations [S1, unofficial]. The technique was identified as a primary optimization to improve the efficiency of hash-based node searches, which otherwise suffered from recursive traversal overhead [S1, unofficial]. It is often paired with Universal Hash Optimization to further reduce collision rates [S1, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] sh1ftmaker/voxel-wars, https://github.com/sh1ftmaker/voxel-wars, fetched 2026-10-28 (unofficial)
