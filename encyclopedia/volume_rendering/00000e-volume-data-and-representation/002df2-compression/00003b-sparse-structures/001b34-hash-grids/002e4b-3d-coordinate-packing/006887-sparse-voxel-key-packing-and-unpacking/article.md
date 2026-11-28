# Sparse Voxel Key Packing and Unpacking

Also called: Packed Key Generation, getKey, unpack

## Overview
Sparse Voxel Key Packing and Unpacking converts 3D integer voxel coordinates into a single 64-bit integer key to enable efficient sorting, deduplication, and set operations on sparse voxel sets [S1, unofficial]. It builds on [3D Coordinate Packing](5755-3d-coordinate-packing.md) by enforcing a specific bit layout and shift strategy that guarantees neighbor offsets can be applied via simple integer addition without field carry corruption. Unlike [Grid Index Packing & Unpacking](12913-grid-index-packing-unpacking.md), which may use different stride calculations, this method explicitly reserves 21 bits per axis to isolate coordinate fields, ensuring that stepping one voxel along an axis corresponds to a constant addition on the key [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An $(N, 3)$ integer array of voxel coordinates $(x, y, z)$.
- Output: A sorted, unique array of 64-bit integer keys and a shift vector.

**Data Structures:**
- **Packed Key:** A 64-bit signed integer (`int64`).
- **Bit Layout:** The key contains three 21-bit fields for $x$, $y$, and $z$ coordinates.
    - $x$ field: bits 42–62 (shifted left by 42).
    - $y$ field: bits 21–41 (shifted left by 21).
    - $z$ field: bits 0–20 (shifted left by 0).
    - Bit 63 is the sign bit. Bits 63–63 are unused for coordinate storage but part of the sign. Total 63 bits used for coordinates.
- **Shift Vector:** A 3-element `int64` array representing the offset applied to coordinates before packing.

**Parameters:**
- `margin`: Integer slack added to the minimum coordinate before packing. Default is 1 [S1, unofficial]. It prevents field carry corruption when adding neighbor offsets. Must satisfy $\max(\text{coord}) + \text{margin} < 2^{21}$ [S1, unofficial].

**Steps:**
1. Validate input is an $(N, 3)$ integer array. If empty, return empty keys and zero shift [S1, unofficial].
2. Cast coordinates to `int64` [S1, unofficial].
3. Calculate the shift vector: $\text{shift} = \min(\text{voxels}, \text{axis}=0) - \text{margin}$ [S1, unofficial].
4. Subtract shift from voxels to get relative coordinates $v$: $v = \text{voxels} - \text{shift}$ [S1, unofficial].
5. Check range: Ensure $\max(v) + \text{margin} < 2^{21}$ (i.e., $\max(v) < 2^{21} - \text{margin}$). If not, raise `ValueError` [S1, unofficial].
6. Pack $v$ into keys using the formula below.
7. Return sorted unique keys and the shift vector [S1, unofficial].

**Formula:**
The key $K$ for a voxel $(x, y, z)$ is:
$$ K = (x \ll 42) + (y \ll 21) + z $$
where $x, y, z$ are the shifted coordinates $v_x, v_y, v_z$ [S1, unofficial].

**Neighbor Offsets (Key Deltas):**
To find neighbors, constant deltas are added to the key. These deltas are derived from 3D offset vectors $(dx, dy, dz)$:
$$ \Delta K = (dx \ll 42) + (dy \ll 21) + dz $$
Addition is used instead of bitwise OR because offsets are signed and the packing guarantees no carry between fields [S1, unofficial].
- Step +1 along X: $\Delta K = 1 \ll 42$
- Step +1 along Y: $\Delta K = 1 \ll 21$
- Step +1 along Z: $\Delta K = 1$ [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def pack_keys(voxels, margin=1):
    """
    Pack 3D voxel coordinates into 64-bit keys.
    
    Args:
        voxels: (N, 3) integer array of (x, y, z) coordinates.
        margin: Integer slack to prevent carry/borrow issues. Default 1.
        
    Returns:
        keys: Sorted, unique 64-bit integer keys.
        shift: (3,) int64 array, the shift applied to coordinates.
    """
    # 1. Validate input
    if not isinstance(voxels, np.ndarray):
        raise TypeError(f"Expected numpy array, got {type(voxels)}")
    if voxels.ndim != 2 or voxels.shape[1] != 3:
        raise ValueError(f"Expected shape (N, 3), got {voxels.shape}")
    if not np.issubdtype(voxels.dtype, np.integer):
        raise TypeError(f"Expected integer dtype, got {voxels.dtype}")
        
    # 2. Handle empty input
    if voxels.size == 0:
        return np.array([], dtype=np.int64), np.array([0, 0, 0], dtype=np.int64)
        
    # 3. Cast to int64
    voxels = voxels.astype(np.int64)
    
    # 4. Calculate shift
    min_coords = np.min(voxels, axis=0)
    shift = min_coords - margin
    
    # 5. Compute relative coordinates
    v = voxels - shift
    
    # 6. Check range
    # The maximum value in any dimension after shift must be < 2^21 - margin
    # Since v = voxels - (min - margin) = voxels - min + margin
    # max(v) = max(voxels) - min(voxels) + margin
    # We need max(v) < 2^21
    max_v = np.max(v)
    if max_v >= (1 << 21):
        raise ValueError(
            f"Voxel extent plus margin exceeds 21-bit range. "
            f"Max coordinate {max_v} >= {1 << 21}. "
            f"Consider downsampling or splitting the volume."
        )
        
    # 7. Pack
    # x << 42 | y << 21 | z
    keys = (v[:, 0] << 42) | (v[:, 1] << 21) | v[:, 2]
    
    # 8. Sort and deduplicate
    unique_keys = np.unique(keys)
    
    return unique_keys, shift

def unpack_keys(keys, shift):
    """
    Unpack 64-bit keys back to 3D voxel coordinates.
    
    Args:
        keys: 1D array of 64-bit integer keys.
        shift: (3,) int64 array, the shift used during packing.
        
    Returns:
        voxels: (M, 3) integer array of (x, y, z) coordinates.
    """
    # Extract fields
    z = keys & ((1 << 21) - 1)
    y = (keys >> 21) & ((1 << 21) - 1)
    x = (keys >> 42) & ((1 << 21) - 1)
    
    # Unshift
    voxels = np.column_stack([x, y, z]).astype(np.int64) + shift
    
    return voxels
```

**Edge Cases and Pitfalls:**
- **Range Exceedance:** If the voxel extent plus margin exceeds 21 bits per axis, packing fails with a `ValueError` [S1, unofficial]. The code checks `max_coord + margin >= 2^21` and suggests downsampling or splitting the volume [S1, unofficial].
- **Silent Aliasing:** Refuses inputs whose extent plus slack would exceed the 21-bit range to avoid silent aliasing due to carry/borrow [S1, unofficial].
- **Neighbor Steps:** Stepping one voxel along an axis is a constant addition on the key only if no field carries or borrows into its neighbor [S1, unofficial]. The margin ensures this invariant holds for standard neighborhood operations.

## Subtopics

### Sparse Voxel Set Algebra
Set operations on sparse voxel sets using packed int64 keys for efficient membership testing via sorted-array lookups [S1, unofficial].
- **Union:** Convert all input voxel sets to common keys using `to_common_keys(voxel_sets)`, concatenate all key arrays, deduplicate with `unique()`, and convert back [S1, unofficial].
- **Intersection:** Voxels present in every set. Implementation uses sorted-array intersection logic (implied by `sorted_hit` and module description) [S1, unofficial].
- **Parameters:** `voxel_sets`: one or more $(N, 3)$ integer arrays of XYZ voxel coordinates [S1, unofficial].
- **Pitfalls:** Union requires at least one voxel set; raises `TypeError` if empty [S1, unofficial].

### Neighbor Offset and Key Delta Calculation
Generates 3D offset vectors for 6, 18, or 26-connectivity and converts them into packed key deltas. These deltas allow finding neighbor keys by simple addition to the current key [S1, unofficial].
- **Steps:**
    1. Generate all 27 combinations of $dx, dy, dz \in \{-1, 0, 1\}$ [S1, unofficial].
    2. Exclude $(0,0,0)$ [S1, unofficial].
    3. Calculate L1 norm for each offset [S1, unofficial].
    4. For connectivity 6, keep offsets where L1==1 [S1, unofficial].
    5. For connectivity 18, keep offsets where L1<=2 [S1, unofficial].
    6. For connectivity 26, keep all [S1, unofficial].
    7. Convert offsets to key deltas using `key_deltas` [S1, unofficial].
- **Formula:** $\Delta K = (dx \ll 42) + (dy \ll 21) + dz$ [S1, unofficial].
- **Parameters:** `connectivity`: Integer, 6, 18, or 26. Defines the neighborhood structure. 6 is face-connected, 18 adds edge-connected, 26 adds corner-connected [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Packing is $O(N)$ where $N$ is the number of voxels. Sorting and deduplication are $O(N \log N)$. Set operations (union/intersection) on sorted keys are $O(N + M)$ for sets of size $N$ and $M$.
- **Memory Complexity:** $O(N)$ for storing keys. Each key is 8 bytes. No dense grid is allocated, so memory scales with voxel count, not bounding box volume [S1, unofficial].
- **Performance:** Efficient for large, sparse objects (e.g., neurons spanning a huge bounding box at low occupancy) that would be wasteful to densify [S1, unofficial].
- **Strengths:** Enables fast set algebra, neighbor lookups, and deduplication without dense grid allocation. Constant-time neighbor access via addition.
- **Limitations:** Limited to $2^{21}$ range per axis. Large volumes must be split or downsampled. Diagonal surfaces in meshing may appear as 90° steps if not smoothed [S1, unofficial].

## Background
Sparse Voxel Key Packing is part of the `sparse-cubes` library ecosystem, which provides tools for sparse voxel operations, meshing, and skeletonization [S1, unofficial]. It is designed to work directly on $(N, 3)$ coordinate arrays, avoiding the allocation of dense grids [S1, unofficial]. The method supports integration with `scipy.sparse` COO arrays, treating them as equivalent to index arrays [S1, unofficial].

## Sources
[S1] navis-org/sparse-cubes, https://github.com/navis-org/sparse-cubes, fetched 2026-10-26 (unofficial)
