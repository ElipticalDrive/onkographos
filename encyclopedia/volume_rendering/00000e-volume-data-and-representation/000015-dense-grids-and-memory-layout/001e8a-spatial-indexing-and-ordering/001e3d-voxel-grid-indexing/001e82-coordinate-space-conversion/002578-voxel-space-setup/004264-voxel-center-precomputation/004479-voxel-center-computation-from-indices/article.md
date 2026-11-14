# Voxel Center Computation from Indices

Also called: Voxel Center Coordinate Generation

## Overview
Computes the 3D world coordinates of voxel centers given their discrete integer indices, bounding box origins, and voxel sizes [S2, unofficial]. It transforms discrete grid indices into continuous spatial coordinates, handling both single-grid and batched scenarios [S1, S2, unofficial]. It differs from sibling methods by explicitly supporting batched inputs with concatenated batch IDs and enforcing specific axis-swapping conventions required by certain rendering pipelines [S1, S2, unofficial].

## How it works
**Inputs:**
- `voxel_indices`: Array of shape `[N, 3]` or `[N, 4]` containing discrete voxel indices. If `cat_batch_ids` is true, the first column is the batch ID, and indices are columns 1-3 [S2, unofficial].
- `bounding_boxes`: Array of shape `[B, 3]` or `[B, 1]` containing the minimum coordinates (origin) for each batch [S2, unofficial].
- `voxel_size`: Scalar or array of shape `[B]` defining the voxel dimension [S2, unofficial].
- `shift`: Boolean flag determining if a 0.5 offset is applied [S2, unofficial].
- `cat_batch_ids`: Boolean flag indicating if batch IDs are concatenated in `voxel_indices` [S2, unofficial].

**Outputs:**
- `centers`: Array of shape `[N, 3]` containing the 3D world coordinates of the voxel centers [S2, unofficial].

**Data Structures:**
- `coords_grid`: Temporary array of shape `[N, 3]`, dtype `float32` [S1, unofficial].

**Steps:**
1. **Validate Batch Mode:** If `cat_batch_ids` is false, raise `NotImplementedError` [S2, unofficial].
2. **Extract Batch Information:**
   - If `cat_batch_ids` is true, extract batch IDs from `voxel_indices[:, 0]` [S2, unofficial].
   - Select the minimum bounding box coordinate for each voxel: `bb_mins = bounding_boxes[batch_ids][:, 0]` [S2, unofficial].
   - Retrieve the voxel size for each voxel, handling scalar or tensor inputs [S2, unofficial].
3. **Extract Indices:**
   - If `cat_batch_ids` is true, extract indices from `voxel_indices[:, 1:]` [S2, unofficial].
   - Otherwise, use `voxel_indices` directly [S2, unofficial].
4. **Compute Centers:**
   - If `shift` is true: `centers = indices * voxel_size_b + bb_mins` [S2, unofficial].
   - If `shift` is false: `centers = indices * voxel_size_b + bb_mins + 0.5 * voxel_size_b` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_voxel_centers(voxel_indices, bounding_boxes, voxel_size, shift, cat_batch_ids):
    if not cat_batch_ids:
        raise NotImplementedError("Only cases with batch IDs concatenated are supported.")
    
    batch_ids = voxel_indices[:, 0].astype(int)
    indices = voxel_indices[:, 1:].astype(float32)
    
    bb_mins = bounding_boxes[batch_ids][:, 0]
    
    if isinstance(voxel_size, (int, float)):
        voxel_size_b = np.full_like(bb_mins, voxel_size, dtype=float32)
    else:
        voxel_size_b = voxel_size[batch_ids]
    
    if shift:
        centers = indices * voxel_size_b + bb_mins
    else:
        centers = indices * voxel_size_b + bb_mins + 0.5 * voxel_size_b
        
    return centers
```

**Note on Single-Grid Generation:**
For generating centers from scratch without pre-existing indices, the following steps are used [S1, unofficial]:
1. Create ranges `g_xx = np.arange(0, dims[0] + 1)`, similarly for y, z.
2. Generate meshgrid of voxel indices: `xx, yy, zz = np.meshgrid(g_xx[:-1], g_yy[:-1], g_zz[:-1])`.
3. Flatten and transpose to shape `[N, 3]`.
4. Cast to `float32`.
5. Scale by resolution: `coords_grid = (coords_grid * resolution) + resolution / 2`.
6. Swap x and y axes: `temp[:, 0] = coords_grid[:, 1]; temp[:, 1] = coords_grid[:, 0]`.

**Parameters:**
- `dims`: List/tuple of 3 ints, grid dimensions `[x, y, z]` [S1, unofficial].
- `resolution`: Float, voxel size [S1, unofficial].

**Edge Cases and Pitfalls:**
- The function raises `NotImplementedError` if `cat_batch_ids` is false [S2, unofficial].
- Axis swapping is required in some implementations to match coordinate system conventions [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This method is used in voxel-based rendering and reconstruction pipelines, such as VoxelSplat and VGVGGT, to map discrete grid indices to continuous world coordinates [S1, S2, unofficial].

## Sources
[S1] ZZY816/VoxelSplat, https://github.com/ZZY816/VoxelSplat, fetched 2026-10-27 (unofficial)
[S2] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
