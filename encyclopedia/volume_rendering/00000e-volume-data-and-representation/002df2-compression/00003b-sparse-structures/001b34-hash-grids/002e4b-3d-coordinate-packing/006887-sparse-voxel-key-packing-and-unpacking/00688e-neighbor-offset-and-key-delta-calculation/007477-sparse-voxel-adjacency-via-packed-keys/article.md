# Sparse Voxel Adjacency via Packed Keys

Also called: packed-key neighbor search, voxel graph construction

## Overview
This method constructs the undirected adjacency graph of a sparse voxel cloud by leveraging packed integer keys and constant additive deltas, thereby avoiding the overhead of KD-trees or dense neighbor block lookups [S1, unofficial]. It differs from dense grid approaches by scaling with the number of voxels rather than the bounding box volume, and from tree-based methods by using sorted-array binary searches on linearized coordinates [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** `voxels`, an $(N, 3)$ array of integer XYZ coordinates. The input must be deduplicated internally [S1, unofficial].
- **Output:**
  - `nodes`: An $(M, 3)$ array of deduplicated, sorted voxels with the same dtype as the input [S1, unofficial].
  - `edges`: An $(E, 2)$ int64 array of index pairs into `nodes`. Edges are canonical (index `lo < hi`), deduplicated, and sorted [S1, unofficial].

**Data Structures:**
- **Packed Keys:** 64-bit integers representing 3D coordinates. The packing scheme must ensure that adding a neighbor delta does not cause carry-over between coordinate fields [S1, unofficial].
- **Key Deltas:** Precomputed integer offsets corresponding to 6, 18, or 26-connectivity [S1, unofficial].

**Parameters:**
- `connectivity`: Integer specifying neighbor type: 6 (faces only), 18 (faces + edges), or 26 (faces + edges + corners). Default is 26 [S1, unofficial].
- `sort`: Boolean. If True, the output edge array is sorted lexicographically by the second column then the first column to ensure deterministic order [S1, unofficial].

**Steps:**
1. **Key Conversion:** Convert input voxel coordinates into packed integer keys using a function `to_keys`. A margin of 1 is applied to ensure that stepping a neighbor delta does not carry between packed fields, which would corrupt the key structure [S1, unofficial]. Record the shift used for later reconstruction [S1, unofficial].
2. **Delta Retrieval:** Retrieve the set of neighbor deltas corresponding to the specified `connectivity` using `key_deltas(connectivity)` [S1, unofficial].
3. **Positive Delta Selection:** Iterate only over positive deltas from the set (deltas $> 0$) to ensure each undirected edge is discovered exactly once in canonical order [S1, unofficial].
4. **Neighbor Candidate Generation:** For each positive delta $d$, compute potential neighbor keys by adding $d$ to all current keys: $nb = keys + d$ [S1, unofficial].
5. **Search:** Find the positions of these potential neighbors in the sorted keys array using `np.searchsorted`. Clip the result to the valid range $[0, m-1]$ to handle boundary cases where no neighbor exists [S1, unofficial].
6. **Verification:** Verify which potential neighbors actually exist by checking if $keys[pos] == nb$. If no hits exist for a delta, skip to the next delta [S1, unofficial].
7. **Edge Construction:** For each hit, create an edge pair consisting of the index of the original voxel (from `np.flatnonzero(hit)`) and the index of the neighbor (from `pos[hit]`) [S1, unofficial]. Stack these pairs and append to a list of parts [S1, unofficial].
8. **Concatenation:** Concatenate all parts into a single edge array $e$ and cast to int64 [S1, unofficial].
9. **Sorting:** If `sort` is True, sort the edge array lexicographically by the second column then the first column [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sparse_voxel_adjacency(voxels, connectivity=26, sort=True):
    """
    Compute adjacency graph for sparse voxels using packed keys.
    
    Args:
        voxels: (N, 3) int array of voxel coordinates.
        connectivity: 6, 18, or 26.
        sort: bool, sort output edges.
        
    Returns:
        nodes: (M, 3) sorted unique voxels.
        edges: (E, 2) int64 array of index pairs (lo < hi).
    """
    # Step 1: Deduplicate and sort input voxels
    # unique_sorted returns sorted unique rows
    nodes = unique_sorted(voxels)
    N = nodes.shape[0]
    
    if N == 0:
        return nodes, np.empty((0, 2), dtype=np.int64)

    # Convert to packed keys with margin=1 to prevent carry corruption
    keys = to_keys(nodes, margin=1)
    
    # Step 2: Get deltas for connectivity
    deltas = key_deltas(connectivity)
    
    # Step 3: Filter for positive deltas only (canonical undirected edges)
    # A delta is "positive" if it represents a unique direction in the 
    # canonical ordering (e.g., first non-zero component is positive).
    positive_deltas = [d for d in deltas if is_positive_delta(d)]
    
    edge_parts = []
    
    # Step 4-7: Iterate deltas and find neighbors
    for d in positive_deltas:
        # Compute potential neighbor keys
        nb_keys = keys + d
        
        # Step 5: Search for these keys in the sorted keys array
        # searchsorted returns indices where nb_keys would be inserted
        pos = np.searchsorted(keys, nb_keys)
        
        # Clip to valid range [0, N-1]
        pos = np.clip(pos, 0, N - 1)
        
        # Step 6: Verify existence
        # Check if the key at pos matches the expected neighbor key
        hit = (keys[pos] == nb_keys)
        
        if not np.any(hit):
            continue
            
        # Step 7: Construct edge pairs
        # Original voxel indices are 0..N-1 (indices into nodes)
        # Neighbor indices are pos[hit]
        lo_indices = np.flatnonzero(hit)
        hi_indices = pos[hit]
        
        # Ensure canonical order (lo < hi)
        # Since d is positive and keys are sorted, lo_indices < hi_indices 
        # should hold, but we enforce it for safety
        valid = lo_indices < hi_indices
        lo_indices = lo_indices[valid]
        hi_indices = hi_indices[valid]
        
        if len(lo_indices) > 0:
            edge_parts.append(np.column_stack((lo_indices, hi_indices)))
            
    if not edge_parts:
        return nodes, np.empty((0, 2), dtype=np.int64)
        
    # Step 8: Concatenate
    edges = np.concatenate(edge_parts, axis=0).astype(np.int64)
    
    # Step 9: Sort if requested
    if sort:
        # Lexicographic sort by second column, then first
        sort_idx = np.lexsort((edges[:, 0], edges[:, 1]))
        edges = edges[sort_idx]
        
    return nodes, edges

def is_positive_delta(delta):
    """Check if delta is in the canonical positive half-space."""
    # Find first non-zero component
    nz = np.nonzero(delta)[0]
    if len(nz) == 0:
        return False
    return delta[nz[0]] > 0
```

**Edge Cases and Pitfalls:**
- **Carry Corruption:** The input keys must have at least a one-voxel margin (achieved via `to_keys(..., margin=1)`) so that stepping a neighbor delta does not carry between packed fields [S1, unofficial].
- **Boundary Handling:** Clipping `searchsorted` results is critical to avoid index out-of-bounds errors when neighbors fall outside the voxel cloud [S1, unofficial].
- **Empty Input:** Handle $N=0$ explicitly to return empty arrays [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Scales with the voxel count $N$, not the bounding box volume. Each delta requires one $O(\log N)$ `searchsorted` operation [S1, unofficial].
- **Memory Complexity:** Transient memory never exceeds the voxel count. Peak memory tracks the voxel count, which is the whole point of the library [S1, unofficial].
- **Strengths:** Avoids dense grid allocation; efficient for sparse data; deterministic output when sorted [S1, unofficial].
- **Limitations:** Requires packed key conversion; limited to grid-aligned voxels [S1, unofficial].
- **When to Use:** When constructing adjacency graphs for sparse voxel clouds in volume rendering or mesh generation [S1, unofficial].
- **When Not to Use:** For dense grids where direct indexing is faster; for non-grid-aligned data [S1, unofficial].

## Background
This method is part of the `sparsecubes` library, which provides set algebra and morphology on sparse $(N, 3)$ voxel clouds [S1, unofficial]. It leverages packed int64 keys to perform membership and neighbor steps as sorted-array lookups rather than dense-grid indexing [S1, unofficial]. The approach was developed to address the memory inefficiency of dense grid operations in libraries like `scipy.ndimage` for sparse data [S1, unofficial].

## Sources
[S1] sparse-cubes, https://github.com/navis-org/sparse-cubes, fetched 2026-10-26 (unofficial)
