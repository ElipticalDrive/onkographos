# Sparse Voxel Morphology (Dilate/Erode)
Also called: Sparse Voxel Erosion, sparse-cubes binary operations, sparse-cubes erode

## Overview
Sparse Voxel Morphology performs topological dilation and erosion on voxel sets represented as sparse integer coordinate arrays, avoiding dense grid allocation. It solves the problem of efficiently modifying voxel geometry (expanding or shrinking boundaries) for applications like pathfinding, visibility culling, or noise reduction, where memory usage must scale with object size rather than bounding box volume. Unlike dense morphological operations which require allocating memory for the entire bounding box, this method uses sorted-array lookups on packed keys to operate only on occupied voxels [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `voxels`, an $(N, 3)$ integer array of voxel coordinates.
- Input: `connectivity`, integer defining neighbor definition (6, 18, or 26).
- Input: `iterations`, integer number of steps ($\ge 0$).
- Output: `result`, an $(M, 3)$ integer array of modified voxel coordinates.

**Data Structures:**
- **Packed Key:** A single `int64` value representing a 3D coordinate. The `to_keys` function packs $(x, y, z)$ into a sortable integer, typically by shifting coordinates into distinct bit ranges or using a space-filling curve order that preserves locality for sorted lookups. The `from_keys` function reverses this.
- **Sorted Keys Array:** A 1D array of packed keys, sorted in ascending order, used for efficient binary search (hit testing).

**Connectivity Deltas:**
The set of neighbor offsets $\Delta$ depends on `connectivity`:
- 6-connectivity (faces): $\{(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)\}$
- 18-connectivity (faces + edges): 6-face deltas + 12 edge deltas (e.g., $(1,1,0)$).
- 26-connectivity (faces + edges + corners): 18-delta + 8 corner deltas (e.g., $(1,1,1)$).

**Algorithm Steps:**

### Dilation
Dilation expands the voxel set by including all neighbors of existing voxels.

1.  **Key Conversion:** Convert input `voxels` to packed keys. To support multiple iterations, the margin for key generation must account for the maximum displacement.
    $$ \text{margin} = \max(1, \text{iterations}) $$
    $$ K_{orig} = \text{to\_keys}(\text{voxels}, \text{margin}) $$
2.  **Delta Generation:** Compute the set of connectivity deltas $\Delta$.
3.  **Iteration Loop:** For $i$ from 1 to `iterations`:
    a. Initialize $K_{new} = K_{orig}$.
    b. For each delta $d \in \Delta$:
       i. Compute shifted keys: $K_{shifted} = K_{orig} + \text{key\_offset}(d)$.
       ii. Concatenate: $K_{new} = \text{concat}(K_{new}, K_{shifted})$.
    c. Deduplicate and sort: $K_{orig} = \text{unique}(\text{sort}(K_{new}))$.
4.  **Result Conversion:** Convert final $K_{orig}$ back to coordinates.
    $$ \text{result} = \text{from\_keys}(K_{orig}, \text{shift}, \text{voxels.dtype}) $$

*Formula:*
$$ V_{new} = V_{old} \cup \bigcup_{d \in \Delta} (V_{old} + d) $$

### Erosion
Erosion shrinks the voxel set by removing voxels that have any neighbor in the background. A voxel survives only if all its connectivity neighbors are present in the set.

1.  **Key Conversion:** Convert input `voxels` to packed keys with a margin of 1 (since erosion checks immediate neighbors).
    $$ K_{orig} = \text{to\_keys}(\text{voxels}, \text{margin}=1) $$
2.  **Delta Generation:** Compute connectivity deltas $\Delta$.
3.  **Iteration Loop:** For $i$ from 1 to `iterations`:
    a. Initialize boolean mask `keep` of length $|K_{orig}|$ to all `True`.
    b. For each delta $d \in \Delta$:
       i. Compute shifted keys: $K_{shifted} = K_{orig} + \text{key\_offset}(d)$.
       ii. Check existence: For each key $k$ in $K_{orig}$, check if $k + \text{key\_offset}(d)$ exists in $K_{orig}$. This is done via binary search on the sorted $K_{orig}$ array.
       iii. Update mask: `keep` = `keep` AND `hit_mask`, where `hit_mask` is true if the neighbor exists.
    c. Filter keys: $K_{orig} = K_{orig}[\text{keep}]$.
    d. If $K_{orig}$ is empty, break loop.
4.  **Result Conversion:** Convert final $K_{orig}$ back to coordinates.
    $$ \text{result} = \text{from\_keys}(K_{orig}, \text{shift}, \text{voxels.dtype}) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def to_keys(voxels, margin=1):
    """
    Pack (N, 3) integer coordinates into sorted int64 keys.
    Assumes voxels are within bounds such that packing fits in int64.
    The margin parameter indicates the range of neighbors to consider
    for key generation validity, though the packing itself is coordinate-based.
    """
    # Implementation detail: Pack x, y, z into int64.
    # Example packing: (x << 42) | (y << 21) | z
    # This requires coordinates to be within specific ranges.
    # For a general implementation, we assume a helper exists that
    # maps (x,y,z) -> int64 uniquely and preserves sort order.
    # [G] Standard packing assumes coordinates are non-negative or offset.
    # Here we assume a generic pack function that handles the bit layout.
    keys = np.empty(len(voxels), dtype=np.int64)
    for i, (x, y, z) in enumerate(voxels):
        # Placeholder for actual bit-packing logic dependent on coordinate range
        # In sparse-cubes, this is optimized C/Fortran code.
        # We assume a valid packing scheme exists.
        keys[i] = pack_int64(x, y, z)
    return np.sort(keys)

def from_keys(keys, shift=0, dtype=np.int32):
    """
    Unpack int64 keys back to (M, 3) coordinates.
    """
    coords = np.empty((len(keys), 3), dtype=dtype)
    for i, k in enumerate(keys):
        x, y, z = unpack_int64(k)
        coords[i] = [x, y, z]
    return coords

def key_deltas(connectivity):
    """
    Return list of (dx, dy, dz) tuples for the given connectivity.
    """
    if connectivity == 6:
        return [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
    elif connectivity == 18:
        base = key_deltas(6)
        edges = [(1,1,0), (1,-1,0), (-1,1,0), (-1,-1,0),
                 (1,0,1), (1,0,-1), (-1,0,1), (-1,0,-1),
                 (0,1,1), (0,1,-1), (0,-1,1), (0,-1,-1)]
        return base + edges
    elif connectivity == 26:
        base = key_deltas(18)
        corners = [(1,1,1), (1,1,-1), (1,-1,1), (1,-1,-1),
                   (-1,1,1), (-1,1,-1), (-1,-1,1), (-1,-1,-1)]
        return base + corners
    else:
        raise ValueError("Connectivity must be 6, 18, or 26")

def key_offset(delta):
    """
    Convert a (dx, dy, dz) delta to an int64 offset.
    Assumes the same packing scheme as to_keys.
    """
    dx, dy, dz = delta
    return pack_int64(dx, dy, dz)

def sorted_hit(queries, sorted_array):
    """
    Check which elements in queries exist in sorted_array.
    Returns boolean array.
    """
    # Binary search implementation
    hits = np.zeros(len(queries), dtype=bool)
    for i, q in enumerate(queries):
        # np.searchsorted returns insertion point
        idx = np.searchsorted(sorted_array, q)
        if idx < len(sorted_array) and sorted_array[idx] == q:
            hits[i] = True
    return hits

def pack_int64(x, y, z):
    """
    [G] Simplified packing for demonstration. 
    Real implementation handles coordinate ranges and sign bits.
    Assumes coordinates are within 21-bit range for this example.
    """
    # Adjust for negative numbers if necessary
    x_off = x + 1048576
    y_off = y + 1048576
    z_off = z + 1048576
    return (x_off << 42) | (y_off << 21) | z_off

def unpack_int64(k):
    """
    [G] Simplified unpacking.
    """
    z = k & 0x1FFFFF
    y = (k >> 21) & 0x1FFFFF
    x = (k >> 42) & 0x1FFFFF
    return (x - 1048576, y - 1048576, z - 1048576)

def sparse_dilate(voxels, connectivity=6, iterations=1):
    if iterations == 0:
        return voxels
    
    margin = max(1, iterations)
    keys = to_keys(voxels, margin)
    deltas = key_deltas(connectivity)
    
    for _ in range(iterations):
        new_keys = keys.copy()
        for d in deltas:
            offset = key_offset(d)
            shifted = keys + offset
            new_keys = np.concatenate([new_keys, shifted])
        keys = np.unique(new_keys)
        
    return from_keys(keys)

def sparse_erode(voxels, connectivity=6, iterations=1):
    if iterations == 0:
        return voxels
    
    keys = to_keys(voxels, margin=1)
    deltas = key_deltas(connectivity)
    
    for _ in range(iterations):
        if len(keys) == 0:
            break
        keep = np.ones(len(keys), dtype=bool)
        for d in deltas:
            offset = key_offset(d)
            shifted = keys + offset
            hit = sorted_hit(shifted, keys)
            keep = keep & hit
        keys = keys[keep]
        
    return from_keys(keys)
```

**Parameters:**
- `connectivity`: 6 (default), 18, or 26.
- `iterations`: Default 1. Must be $\ge 0$.

**Pitfalls:**
- Iterating a 6-connected step $N$ times grows a diamond shape (Manhattan distance), not a Euclidean ball of radius $N$ [S1, unofficial].
- Erosion can consume a thin object entirely, resulting in an empty set. Use `thin` for topology-preserving medial curves [S1, unofficial].

## Subtopics

### [Sparse Filtering (Smooth, Correlate, Morphology)](20129-sparse-filtering-smooth-correlate-morphology.md)
Sparse filtering applies spatial filters—such as Gaussian smoothing, correlation, and morphological max/min operations—to sparse voxel sets with associated scalar values. It solves the problem of efficiently computing neighborhood aggregations on sparse data without allocating dense memory for the entire bounding box. Unlike [Sparse Voxel Morphology (Dilate/Erode)](20127-sparse-voxel-morphology-dilate-erode.md), which modifies the geometry of the voxel set, sparse filtering computes new values for voxels while potentially expanding the support set. It mimics the behavior of `scipy.ndimage` with `mode='constant'` and `cval=0`, treating absent voxels as zero.

### Sparse Voxel Opening/Closing
- **Opening:** Erode then dilate. Removes specks and thin spurs while keeping bulk shape. Inverse of closing [S1, unofficial].
- **Closing:** Dilate then erode. Bridges narrow gaps and fills small pits. Can fuse distinct structures within $2 \times \text{iterations}$ distance [S1, unofficial].
- **Comparison:** Closing can fuse distinct structures; use `fill_cavities(mode='exact')` for topology-safe enclosed void filling [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of voxels $N$ and connectivity $C$. Dilation involves concatenating $C+1$ arrays of size $N$ per iteration, followed by sorting and unique operations ($O(N \log N)$). Erosion involves $C$ binary searches per voxel per iteration ($O(C \cdot N \log N)$).
- **Memory Complexity:** Peak memory is $(C + 1) \cdot N$ int64 keys for one dilation step. 26-connectivity is the most expensive case [S1, unofficial].
- **Strengths:** Memory usage scales with object size, not bounding box. Handles large, sparse objects (e.g., neurons) that would be wasteful to densify.
- **Limitations:** 6-connected dilation does not produce Euclidean distance shapes. Erosion may destroy thin structures.

## Background
Sparse voxel morphology is part of the `sparse-cubes` library ecosystem, designed for efficient voxel operations on large, sparse datasets without dense grid allocation. It builds on the sparse voxel representation introduced in [VoxelGrid from Mesh (Fill Method)](52-name.md) and [Mathematical Morphology on Voxelization](4702-mathematical-morphology-on-voxelization.md). The library is maintained by the `navis-org` community [S1, unofficial].

## Sources
[S1] navis-org/sparse-cubes, https://github.com/navis-org/sparse-cubes, fetched 2026-10-26 (unofficial)
