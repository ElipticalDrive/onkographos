# Block ID Layer Lookup Table

Also called: Block ID to Layer Lookup Table, Lookup Table Optimization for Meshing, MAX_BLOCK_ID Lookup

## Overview
The Block ID Layer Lookup Table replaces hash-based set lookups with direct array indexing to classify block IDs into mesh layers during the inner meshing loop [S1, unofficial]. It solves the performance bottleneck of iterating over `Set` or `HashSet` structures when processing hundreds of thousands of cells per chunk by pre-computing a fixed-size typed array indexed by block ID [S1, unofficial].

## How it works
**Inputs:**
- `config`: An object containing sets of block IDs for different material types [S1, unofficial].
  - `config.waterBlockIds`: Set of block IDs classified as water [S1, unofficial].
  - `config.transparentSolidBlockIds`: Set of block IDs classified as transparent solids [S1, unofficial].
- `MAX_BLOCK_ID`: The maximum valid block ID value, inclusive [S1, unofficial].

**Outputs:**
- `lookup`: A `Uint8Array` of size `MAX_BLOCK_ID + 1` [S1, unofficial]. Each index corresponds to a block ID, and the value at that index is an integer representing the mesh layer [S1, unofficial].

**Data Structures:**
- `MESH_LAYERS`: An ordered array of strings defining the layers: `['opaque', 'water', 'transparentSolid']` [S1, unofficial].
- `lookup`: A `Uint8Array` where `lookup[blockId]` stores the index of the layer in `MESH_LAYERS` [S1, unofficial].

**Numbered Steps:**
1. Initialize a `lookup` array of size `MAX_BLOCK_ID + 1` with default values of `0` (representing `opaque`) [S1, unofficial].
2. Iterate over `config.waterBlockIds` and set `lookup[blockId]` to `MESH_LAYERS.indexOf('water')` (which evaluates to `1`) [S1, unofficial].
3. Iterate over `config.transparentSolidBlockIds` and set `lookup[blockId]` to `MESH_LAYERS.indexOf('transparentSolid')` (which evaluates to `2`) [S1, unofficial].
4. Validate that all `blockId` values in the configuration sets are $\le MAX\_BLOCK\_ID$ before population to prevent out-of-bounds access [S1, unofficial].

**Formulas:**
- Table size: $N = MAX\_BLOCK\_ID + 1$ [S1, unofficial].
- Cells visited per chunk: $C = FACES \times CHUNK\_SIZE \times CHUNK\_SIZE \times CHUNK\_HEIGHT = 6 \times 16 \times 16 \times 256 = 393,216$ [S1, unofficial].

**Parameters:**
- `MAX_BLOCK_ID`: The maximum valid block ID, inclusive [S1, unofficial]. The table size is exactly `MAX_BLOCK_ID + 1` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Out-of-Bounds Access:** If a block ID in the configuration exceeds `MAX_BLOCK_ID`, direct array access will throw an error or corrupt memory [S1, unofficial].
- **Sparse IDs:** If `MAX_BLOCK_ID` is very large but the number of unique block IDs is small, the table consumes significant memory [S1, unofficial].
- **Default Behavior:** IDs not explicitly listed in `waterBlockIds` or `transparentSolidBlockIds` default to `0` (opaque) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_layer_lookup(config, max_block_id):
    # Initialize lookup array of size max_block_id + 1 with default value 0 (opaque) [S1, unofficial]
    lookup = [0] * (max_block_id + 1)
    
    # Iterate over config.waterBlockIds and set lookup[blockId] = MESH_LAYERS.indexOf('water') [S1, unofficial]
    for block_id in config.waterBlockIds:
        lookup[block_id] = 1
        
    # Iterate over config.transparentSolidBlockIds and set lookup[blockId] = MESH_LAYERS.indexOf('transparentSolid') [S1, unofficial]
    for block_id in config.transparentSolidBlockIds:
        lookup[block_id] = 2
        
    return lookup
```

## Operating characteristics
**Time and Memory Complexity:**
- **Build Time:** $O(W + T)$, where $W$ is the number of water block IDs and $T$ is the number of transparent solid block IDs [S1, unofficial].
- **Lookup Time:** $O(1)$ per cell [S1, unofficial].
- **Memory:** $O(MAX\_BLOCK\_ID)$ bytes [S1, unofficial].

**Measured Performance:**
- The inner loop performs approximately 393,216 calls per chunk (6 face passes over a $16 \times 16 \times 256$ cell grid) [S1, unofficial].
- The lookup table method is faster than iterating a `Set` in the inner meshing loop because it uses array indexing instead of hash lookup [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Direct array indexing provides minimal constant factors for $O(1)$ access [S1, unofficial].
- **Limitations:** Memory usage scales linearly with `MAX_BLOCK_ID` [S1, unofficial].

**When to Use:**
- Use when the inner meshing loop requires high-frequency lookups and block IDs are bounded [S1, unofficial].

**When Not to Use:**
- Use hash sets or other sparse data structures if block IDs are sparse or the maximum ID is extremely large relative to the number of active block types [S1, unofficial].

## Background
The Block ID Layer Lookup Table is an optimization for the inner meshing loop in voxel chunk processing [S1, unofficial]. It addresses the performance bottleneck identified in earlier implementations that used `Set` or `HashSet` for block ID classification [S1, unofficial].

## Sources
[S1] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
