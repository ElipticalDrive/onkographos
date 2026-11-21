# Voxel Center Computation from Indices

Also called: Voxel Center Coordinate Generation, Voxel-to-World Coordinate Mapping for Crosshair

## Overview
Computes the 3D world-space coordinates of voxel centers given their discrete indices, bounding box origins, and voxel sizes [S3, unofficial]. It solves the problem of mapping discrete grid positions to continuous physical space for ray casting, shading, or crosshair placement. It is distinguished from [Voxel Grid Coordinate Mapping](15198-voxel-grid-coordinate-mapping.md) by focusing specifically on center point calculation rather than handling linear buffer indexing schemes or geometry-based vertex generation [S2, unofficial].

## How it works
**Inputs:**
- `voxel_indices`: Array of shape $[N, 3]$ or $[N, 4]$ containing discrete voxel indices $(i, j, k)$ [S3, unofficial]. If `cat_batch_ids` is true, the first column is a batch ID, and indices are in columns 1-3 [S3, unofficial].
- `voxel_size`: Voxel dimensions $(s_x, s_y, s_z)$ as a scalar or vector [S3, unofficial].
- `bb_mins`: Minimum bounding box coordinates for each voxel/batch, shape $[N, 3]$ or broadcastable [S3, unofficial].
- `shift`: Boolean flag determining if an offset of half-voxel size is applied [S3, unofficial].
- `cat_batch_ids`: Boolean flag indicating if batch IDs are present in the input indices [S3, unofficial].

**Outputs:**
- `centers`: Array of shape $[N, 3]$ containing world-space coordinates $(x, y, z)$ for each voxel center [S3, unofficial].

**Data Structures:**
- `voxel_indices`: Integer array.
- `voxel_size`: Float scalar or vector.
- `bb_mins`: Float vector or array.
- `centers`: Float array $[N, 3]$.

**Steps:**
1. **Extract Indices:** If `cat_batch_ids` is true, extract indices from `voxel_indices[:, 1:]`. Otherwise, use `voxel_indices` directly [S3, unofficial].
2. **Retrieve Parameters:** Get `voxel_size` and `bb_mins`. If `voxel_size` is a scalar, treat it as $(s, s, s)$ [S3, unofficial].
3. **Compute Centers:** Apply the formula based on `shift` and `cat_batch_ids` [S3, unofficial]:
   - If `shift` is true and `cat_batch_ids` is true:
     $$ \text{centers} = \text{indices} \cdot \text{voxel\_size} + \text{bb\_mins} $$
   - If `shift` is false and `cat_batch_ids` is true:
     $$ \text{centers} = \text{indices} \cdot \text{voxel\_size} + \text{bb\_mins} + 0.5 \cdot \text{voxel\_size} $$
   - [G] If `cat_batch_ids` is false, the implementation raises `NotImplementedError` as only batch-concatenated cases are supported [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_voxel_centers(voxel_indices, voxel_size, bb_mins, shift, cat_batch_ids):
    """
    Compute world-space centers of voxels from discrete indices.
    
    Args:
        voxel_indices: Array of shape [N, 3] or [N, 4]. If cat_batch_ids is True, 
                       column 0 is batch ID, columns 1-3 are indices.
        voxel_size: Scalar or array-like of shape [3] representing (sx, sy, sz).
        bb_mins: Array of shape [N, 3] or broadcastable minimum bounding box coordinates.
        shift: Boolean. If True, no half-voxel offset is added. If False, 0.5 * voxel_size is added.
        cat_batch_ids: Boolean. If True, batch IDs are present in voxel_indices.
        
    Returns:
        centers: Array of shape [N, 3] with world-space coordinates.
        
    Raises:
        NotImplementedError: If cat_batch_ids is False.
    """
    if not cat_batch_ids:
        raise NotImplementedError("Only cases with batch IDs concatenated are supported.")
    
    # Extract indices (skip batch ID column)
    indices = voxel_indices[:, 1:]
    
    # Ensure voxel_size is broadcastable to [3]
    if np.isscalar(voxel_size):
        voxel_size = np.array([voxel_size, voxel_size, voxel_size])
    else:
        voxel_size = np.asarray(voxel_size)
        
    # Compute centers based on shift flag
    # Formula: centers = indices * voxel_size + bb_mins + (0 if shift else 0.5 * voxel_size)
    offset = 0.0 if shift else 0.5 * voxel_size
    
    centers = indices * voxel_size + bb_mins + offset
    
    return centers
```

**Edge Cases and Pitfalls:**
- **Batch ID Handling:** The function strictly requires `cat_batch_ids` to be true. Passing indices without batch IDs raises `NotImplementedError` [S3, unofficial].
- **Shift Logic:** The `shift` parameter inverts the typical "center" logic. `shift=True` implies indices map directly to the start of the voxel (or a specific alignment), while `shift=False` adds the standard half-voxel offset to reach the center [S3, unofficial].
- **Precision:** Inputs should be cast to `float32` or `float64` before arithmetic to avoid integer division issues [S2, S3, unofficial].

## Subtopics

### [Voxel Grid Coordinate Mapping](15198-voxel-grid-coordinate-mapping.md)
Handles bijective transformations between discrete indices, linear buffer positions, and world coordinates, including row-major flattening and vertex generation for geometry-based rendering. Choose this when you need to map linear buffer indices or generate geometry vertices rather than just center points.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels. The computation involves element-wise multiplication and addition.
- **Memory Complexity:** $O(N)$ for the output array.
- **Strengths:** Simple, efficient, and directly maps indices to world space. Suitable for real-time applications like crosshair placement or ray casting origins.
- **Limitations:** Limited to regular grids. Does not handle irregular voxel sizes or non-axis-aligned bounding boxes. The requirement for batch IDs restricts its use in single-batch scenarios without modification.
- **When to Use:** When you have discrete voxel indices and need their corresponding world-space centers for shading or ray casting.
- **When Not to Use:** When you need to map linear buffer indices to world space (use [Voxel Grid Coordinate Mapping](15198-voxel-grid-coordinate-mapping.md)) or when dealing with irregular voxel structures.

## Background
The method is derived from implementations in volumetric rendering and 3D Gaussian Splatting frameworks, specifically those involving depth estimation and scene reconstruction [S3, unofficial]. It is used in systems like VGVGGT for computing voxel centers for Gaussian scene decoding [S3, unofficial]. The approach is also found in medical imaging viewers for label map visualization [S1, unofficial] and in autonomous driving datasets for voxel-based occupancy prediction [S2, unofficial].

## Sources
[S1] caitlin-leonard/medviewer, https://github.com/caitlin-leonard/medviewer, fetched 2026-10-27 (unofficial)
[S2] ZZY816/VoxelSplat, https://github.com/ZZY816/VoxelSplat, fetched 2026-10-27 (unofficial)
[S3] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
