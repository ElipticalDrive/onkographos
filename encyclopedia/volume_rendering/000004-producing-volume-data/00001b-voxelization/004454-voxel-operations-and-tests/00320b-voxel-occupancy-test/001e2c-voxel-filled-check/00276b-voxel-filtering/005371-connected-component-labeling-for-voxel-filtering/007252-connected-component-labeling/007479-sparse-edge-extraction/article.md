# Sparse Edge Extraction

Also called: sc.edges

## Overview
Extracts the adjacency graph of a voxel set, returning sorted unique nodes and an edge list of index pairs into the node list [S1, unofficial]. It walks positive packed-key deltas to find neighbors, ensuring each undirected edge is found exactly once [S1, unofficial]. It uses sorted-array membership (`searchsorted`) instead of KD-trees or dense neighbor blocks, making it efficient for sparse data [S1, unofficial].

## How it works
**Inputs:** A set of occupied voxel coordinates (typically as packed integer keys or `(N, 3)` integer arrays).
**Outputs:** A sorted array of unique nodes (packed keys) and an edge list of shape `(E, 2)` containing index pairs into the node array.
**Parameters:** `connectivity`: Integer, 6, 18, or 26. Defines the adjacency graph structure [S1, unofficial].

**Data Structures:**
*   `keys`: Sorted, unique packed integer keys representing voxel positions.
*   `deltas`: Positive half of the symmetric connectivity set (e.g., for 6-connectivity, deltas are `(1,0,0), (0,1,0), (0,0,1)` in packed form).

**Steps:**
1.  Convert input voxels to sorted unique packed keys.
2.  For each connectivity delta (positive half of symmetric set), search for neighbors using `searchsorted` on the sorted keys.
3.  For each found neighbor, create an edge `(min_index, max_index)` where indices refer to positions in the sorted unique node list.
4.  Deduplicate and sort edges.
5.  Return nodes and edges.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from itertools import combinations

def sparse_edge_extraction(keys, connectivity=6):
    """
    Extracts adjacency graph from sorted unique packed voxel keys.
    
    Args:
        keys: Sorted, unique 1D array of packed voxel keys.
        connectivity: 6, 18, or 26.
        
    Returns:
        nodes: Sorted unique keys (same as input if already sorted/unique).
        edges: (E, 2) array of index pairs into nodes.
    """
    # 1. Ensure keys are sorted and unique (input assumption per notes)
    # If raw coordinates are provided, they must be packed and sorted first.
    # Packing function assumed to be available: pack((x,y,z)) -> int
    # Unpacking function assumed to be available: unpack(int) -> (x,y,z)
    
    # 2. Define positive deltas based on connectivity
    # Note: The specific deltas depend on the packing scheme. 
    # Assuming a standard 3D packing where neighbors are offset by specific integers.
    # For 6-connectivity: neighbors differ by 1 in one axis.
    # For 18/26, more complex offsets apply.
    # The notes state: "walks positive packed-key deltas".
    
    # Generate deltas for the chosen connectivity.
    # This requires knowing the exact packing scheme (e.g., bit-interleaving or linear).
    # Since the source doesn't specify the packing formula, we assume a helper `get_deltas` exists.
    deltas = get_positive_deltas(connectivity) 
    
    edges = []
    
    # 3. For each delta, find neighbors
    for delta in deltas:
        # Search for keys that are exactly 'delta' away
        # We look for key + delta in the sorted keys array
        target_keys = keys + delta
        
        # Use searchsorted to find positions of target_keys in keys
        # This identifies which voxels have a neighbor in the direction of 'delta'
        indices = np.searchsorted(keys, target_keys)
        
        # Check if the found index actually matches the target key
        # Handle boundary conditions for searchsorted
        valid = (indices < len(keys)) & (keys[indices] == target_keys)
        
        # Get the indices of the current voxel and its neighbor
        # The neighbor's index is indices[valid]
        # The current voxel's index is the index in 'keys' where target_keys came from?
        # Actually, we iterate through 'keys' and check if keys[i] + delta exists.
        
        # Correct approach per notes: "For each connectivity delta... search for neighbors"
        # Let's iterate through each key and check for its neighbor.
        
        # Re-evaluating step 2/3 based on "walks positive packed-key deltas"
        # It's more efficient to iterate deltas and use searchsorted on the whole array.
        
        # Find positions of (keys + delta) in keys
        pos = np.searchsorted(keys, keys + delta)
        
        # Clip to valid range
        pos = np.clip(pos, 0, len(keys) - 1)
        
        # Check if the value at pos is actually keys + delta
        found = (keys[pos] == keys + delta)
        
        # Get the indices of the pairs
        # i is the index of the first voxel
        # j is the index of the neighbor
        i = np.arange(len(keys))
        j = pos
        
        # Filter to only valid edges
        valid_edges = found
        i_valid = i[valid_edges]
        j_valid = j[valid_edges]
        
        # Create edge as (min_index, max_index)
        edge_indices = np.column_stack((np.minimum(i_valid, j_valid), np.maximum(i_valid, j_valid)))
        edges.append(edge_indices)
        
    # 4. Deduplicate and sort edges
    if not edges:
        return keys, np.empty((0, 2), dtype=np.int64)
        
    all_edges = np.vstack(edges)
    unique_edges = np.unique(all_edges, axis=0)
    
    # 5. Return nodes and edges
    return keys, unique_edges

def get_positive_deltas(connectivity):
    """
    Returns positive packed deltas for 6, 18, or 26 connectivity.
    Assumes a standard 3D integer grid packing.
    Note: The exact packing values depend on the specific implementation of 'pack'.
    For example, if pack(x,y,z) = x | (y << 10) | (z << 20), then delta for +x is 1.
    This function is a placeholder for the specific packing logic not detailed in the source.
    [G] The source mentions 'positive packed-key deltas' but does not define the packing scheme.
    A generic implementation would require the packing function.
    """
    # This is a gap-filler [G] as the source does not provide the packing formula.
    # In a real implementation, this would return the integer offsets corresponding to
    # the positive neighbors in the chosen connectivity.
    raise NotImplementedError("Packing scheme not defined in source.")
```

**Edge Cases and Pitfalls:**
*   **Packing Scheme:** The method relies on "packed-key deltas." The exact integer offsets for neighbors depend on the voxel packing scheme (e.g., bit-interleaving vs. linear). If the packing is not consistent with the deltas, neighbor detection will fail [S1, unofficial].
*   **Boundary Conditions:** `searchsorted` must be handled carefully at the boundaries of the sorted array to avoid index out-of-bounds errors [S1, unofficial].
*   **Duplicate Edges:** Since the graph is undirected, each edge must be normalized to `(min_index, max_index)` and deduplicated to ensure each edge is found exactly once [S1, unofficial].

## Subtopics

### Edge/Corner Contact Table
Precomputed table of voxel pairs that touch along an edge or at a corner but share no face. Used for specialized adjacency operations [S1, unofficial].
*   **Definition:** Stores tuple `(sum(steps), steps[0], between)` in table. Fixing first axis step to +1 ensures every unordered pair is visited exactly once (6 edges, 4 corners) [S1, unofficial].
*   **Steps:**
    1.  Generate axis steps: `axis = [s * AXIS_STEPS[a] for a in range(3) for s in (1, -1)]` [S1, unofficial].
    2.  Create combinations: +x with +-y, +x with +-z, +y with +-z, and all combinations of +x, +-y, +-z for corners [S1, unofficial].
    3.  For each combination, compute 'between' cells as all proper non-empty subset sums of the axis steps using `itertools.combinations` [S1, unofficial].
    4.  Store tuple `(sum(steps), steps[0], between)` in table [S1, unofficial].
*   **When to choose:** Use when specialized adjacency operations involving edge or corner contacts (but not face contacts) are required [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by sorting the keys and the `searchsorted` operations for each delta. Sorting is $O(N \log N)$. For each of the $D$ deltas, `searchsorted` is $O(N \log N)$. Total time is $O(D \cdot N \log N)$.
*   **Memory Complexity:** $O(N)$ for storing keys and edges.
*   **Performance:** Efficient for sparse data due to the use of sorted-array membership instead of KD-trees or dense neighbor blocks [S1, unofficial].
*   **Strengths:** Simple implementation, efficient for sparse voxel sets, avoids complex data structures like KD-trees.
*   **Limitations:** Requires sorted unique keys. Performance degrades if the voxel set is dense (approaching $O(N^2)$ edges). Relies on a consistent packing scheme.

## Background
Originated in the `sparse-cubes` library by BorisTheBrave [S1, unofficial]. The method is part of a suite of voxel operations including `sc.voxelize`, `sc.mesh`, and `sc.dual_contour` [S1, unofficial]. The library provides a sparse representation of voxel data, enabling efficient operations on large, sparse volumes [S1, unofficial].

## Sources
[S1] navis-org/sparse-cubes, https://github.com/navis-org/sparse-cubes, fetched 2026-10-26 (unofficial)
