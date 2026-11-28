# General Grid Resample

Also called: Grid Transformer, resampleToMatch

## Overview
Resamples a non-level-set grid (or a level-set grid that failed standard resampling) to match the spatial transform of a reference grid using a specified interpolation order [S1, unofficial]. It differs from [Cubic Upsampling via Tiling](2342-cubic-upsampling-via-tiling.md) by supporting arbitrary integer-order samplers (point, box, quadratic) rather than fixed cubic kernels, and from [3D Image Resize and Crop](2703-3d-image-resize-and-crop.md) by operating on structured tree-based volume data structures rather than raw image buffers.

## How it works
**Inputs:**
- `source_grid`: The input volume grid to be resampled.
- `reference_grid`: The target grid whose transform defines the output geometry.
- `order`: Integer specifying the sampler type (0, 1, or 2).

**Outputs:**
- `destination_grid`: A new grid instance containing the resampled data, aligned to `reference_grid`'s transform.

**Data Structures:**
The implementation relies on the OpenVDB grid structure. The `source_grid` contains a tree of voxels. The `destination_grid` is initialized with a new, empty tree. The transform is a 4x4 affine matrix (typically stored as a `Mat4d` or similar structure in C++ implementations) defining the coordinate mapping.

**Steps:**
1. **Initialize Destination:** Create a copy of the `source_grid` structure but allocate a new, empty tree for the `destination_grid` [S1, unofficial].
2. **Set Transform:** Copy the transform matrix from the `reference_grid` to the `destination_grid` [S1, unofficial]. This ensures the output voxels align with the reference coordinate system.
3. **Execute Resampling:** Call the internal tool `tools::resampleToMatch` (or equivalent function) passing the `source_grid`, `destination_grid`, and the `order` parameter [S1, unofficial].
4. **Sampler Selection:** The `order` parameter determines the interpolation kernel used during the voxel value transfer:
   - `order = 0`: Uses `PointSampler` (nearest-neighbor).
   - `order = 1`: Uses `BoxSampler` (trilinear/box filter).
   - `order = 2`: Uses `QuadraticSampler` (quadratic interpolation) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def general_grid_resample(source_grid, reference_grid, order):
    """
    Resamples source_grid to match reference_grid's transform.
    
    Args:
        source_grid: The input volume grid.
        reference_grid: The target grid defining the output transform.
        order: Integer sampler order (0=Point, 1=Box, 2=Quadratic).
        
    Returns:
        destination_grid: The resampled grid.
    """
    # Step 1: Create a copy of the source grid with a new tree
    # This initializes the destination grid structure
    destination_grid = source_grid.copy()
    destination_grid.tree = new_empty_tree()
    
    # Step 2: Set the destination grid's transform to a copy of the reference transform
    destination_grid.transform = reference_grid.transform.copy()
    
    # Step 3: Call resample tool with appropriate sampler based on order
    # The sampler handles the actual interpolation of values from source to destination
    if order == 0:
        sampler = PointSampler()
    elif order == 1:
        sampler = BoxSampler()
    elif order == 2:
        sampler = QuadraticSampler()
    else:
        raise ValueError("Order must be 0, 1, or 2")
        
    tools.resampleToMatch(source_grid, destination_grid, sampler)
    
    return destination_grid
```

**Parameters:**
- `order`: Integer. 0 for PointSampler, 1 for BoxSampler, 2 for QuadraticSampler [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Level Set Failure:** The method is explicitly designed for grids where standard level-set resampling fails or is not applicable [S1, unofficial].
- **Precision:** Interpolation accuracy depends on the sampler order. Point sampling (0) is fast but discontinuous; Box (1) is smoother; Quadratic (2) is smoother still but more expensive [S1, unofficial].
- **Boundaries:** The resampling tool must handle out-of-bounds queries from the source grid, typically by clamping or returning zero, depending on the sampler implementation [G].

## Subtopics

### Grid Normalization via Linear Interpolation
A preprocessing step used to normalize grid data from numerical simulations (e.g., MMIC) by dividing larger cells into smaller "basic cells" using linear interpolation, ensuring a uniform grid for visualization [S2, unofficial]. It is chosen when the source data has irregular cell sizes and a uniform output grid is required for rendering. Unlike the general resampler which maps between arbitrary transforms, this method specifically targets uniformization by subdividing based on the smallest existing cell [S2, unofficial].

**Definition:**
Find the smallest cell in the dataset and use it as the basic cell size. Redivide the whole region by this basic cell, meaning a big cell is divided into one or more basic cells. Calculate values of new grid points using linear interpolation from surrounding points [S2, unofficial].

**Steps:**
1. Identify the smallest cell dimension in the dataset; this defines the "basic cell" size [S2, unofficial].
2. Redivide the entire region such that every original cell is composed of an integer number of basic cells [S2, unofficial].
3. For a cell with corners $A, B, C, D$ divided into 4 smaller cells, add 5 new points: $T_{AB}, T_{BC}, T_{CD}, T_{AD}$ (edge midpoints) and $T_O$ (center) [S2, unofficial].
4. For cells divided into two, use similar linear interpolation methods [S2, unofficial].

**Formula:**
For a cell with corner values $A, B, C, D$:
$$T_{AB} = (A + B)/2$$
$$T_{BC} = (C + B)/2$$
$$T_{CD} = (C + D)/2$$
$$T_{AD} = (A + D)/2$$
$$T_O = (T_{AB} + T_{BC} + T_{CD} + T_{AD})/2$$
[S2, unofficial]

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources. The complexity depends on the number of voxels in the destination grid and the cost of the interpolation kernel (PointSampler is $O(1)$ per voxel, BoxSampler involves 8 neighbors, Quadratic involves more) [S1, unofficial].

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
- **Strengths:** Handles non-level-set grids and failed level-set resampling cases [S1, unofficial]. Provides flexible interpolation orders [S1, unofficial].
- **Limitations:** Requires a reference grid with a valid transform [S1, unofficial]. Not designed for level-set grids that can be resampled normally [S1, unofficial].

**When to Use:**
Use when you need to align a volume grid to a specific reference transform and the standard level-set resampling is not applicable or has failed [S1, unofficial]. Use [Grid Normalization via Linear Interpolation](#grid-normalization-via-linear-interpolation) if the goal is specifically to uniformize irregular simulation grids for visualization [S2, unofficial].

## Background
The `resampleToMatch` functionality is part of the OpenVDB library ecosystem, specifically referenced in NVIDIA-Omniverse's extension for OpenVDB [S1, unofficial]. The concept of normalizing irregular grids via linear interpolation for visualization purposes was described in the context of the MMIC numerical simulation program in 2006 [S2, unofficial].

## Sources
[S1] NVIDIA-Omniverse/ext-openvdb, https://github.com/NVIDIA-Omniverse/ext-openvdb, fetched 2026-10-27 (unofficial)
[S2] Visualization Technique of Explosion ..., http://paper.ijcsns.org/07_book/200611/200611A09.pdf, fetched 2026-10-29 (unofficial)
