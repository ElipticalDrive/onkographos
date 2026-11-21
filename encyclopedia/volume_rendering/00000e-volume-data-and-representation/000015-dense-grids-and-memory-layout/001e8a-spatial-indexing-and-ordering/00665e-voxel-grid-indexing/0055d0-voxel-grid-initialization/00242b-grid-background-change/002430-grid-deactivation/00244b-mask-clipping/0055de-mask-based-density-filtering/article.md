# Mask-based Density Filtering

Also called: Density Disk Function, Segmentation to Density Conversion

## Overview
Mask-based Density Filtering modifies a volumetric density distribution by applying a spatial mask to remove or zero out density in specific regions, such as creating a disk shape from a sphere by chopping off the top and bottom [S1, unofficial]. It serves as a post-processing step to refine density grids derived from segmentation or other spatial constraints, distinct from [Mask Clipping](3229-mask-clipping.md) which filters active voxels based on mask presence, and [Bounding Box Clipping](4643-bounding-box-clipping.md) which restricts volume to axis-aligned bounds.

## How it works
**Inputs:**
- `density_grid`: A 3D grid of density values.
- `mask_grid`: A 3D grid of binary or categorical values defining the active regions.

**Outputs:**
- `filtered_density_grid`: A 3D grid with density values modified by the mask.

**Data Structures:**
- `density_grid`: Array of shape $(D, H, W)$, where $D$ is depth, $H$ is height, $W$ is width. Values are floating-point densities.
- `mask_grid`: Array of shape $(D, H, W)$. Values are integers or floats representing segmentation labels or binary flags.

**Steps:**
1. Initialize `mask_grid` with zeros, matching the shape of `density_grid`.
2. Populate `mask_grid` by setting values to 1 where `seg_mask_grid.grid > 0`. This converts a segmentation label grid into a binary active mask [S2, unofficial].
3. Multiply `density_grid` element-wise by `mask_grid`. This retains density only where the mask is active (value 1) and sets it to 0 elsewhere [S2, unofficial].
4. Clamp or set `density_grid` values to a minimum threshold (e.g., $-1e7$) to handle numerical precision or invalid states [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mask_based_density_filtering(density_grid, seg_mask_grid):
    """
    Applies a spatial mask to a density grid, filtering out density
    where the segmentation mask is inactive.
    
    Args:
        density_grid: 3D numpy array of density values.
        seg_mask_grid: 3D numpy array of segmentation labels.
        
    Returns:
        filtered_density_grid: 3D numpy array with masked density.
    """
    # Step 1: Create binary mask from segmentation labels
    # Assert that the mask grid has the expected shape (e.g., single object label dimension)
    assert seg_mask_grid.shape[1] == 1, "Multi-object seg labels cannot be applied directly to the density grid."
    
    # Initialize mask grid with zeros
    mask_grid = np.zeros_like(density_grid)
    
    # Set mask values to 1 where segmentation label is > 0
    mask_grid[seg_mask_grid > 0] = 1.0
    
    # Step 2: Apply mask to density
    filtered_density_grid = density_grid * mask_grid
    
    # Step 3: Clamp density values to a minimum threshold
    # This handles potential numerical artifacts or invalid states
    filtered_density_grid = np.maximum(filtered_density_grid, -1e7)
    
    return filtered_density_grid
```

**Parameters:**
- `threshold`: Minimum density value after filtering. Default: $-1e7$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Multi-object Segmentation:** The method assumes a single-object segmentation label grid where `shape[1] == 1`. Multi-object segmentation labels cannot be applied directly without prior aggregation or selection [S2, unofficial].
- **Precision:** Floating-point multiplication may introduce small errors; clamping helps mitigate this [S2, unofficial].
- **Boundary Conditions:** The mask must align exactly with the density grid dimensions; misalignment leads to incorrect filtering.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the grid, due to element-wise operations.
- **Memory Complexity:** $O(N)$ for the intermediate `mask_grid`.
- **Strengths:** Simple, efficient, and effective for refining density distributions based on spatial masks.
- **Limitations:** Limited to binary or single-label masks; not suitable for complex multi-object scenes without preprocessing.
- **When to Use:** When refining density grids derived from segmentation masks or when specific spatial regions need to be zeroed out.
- **When Not to Use:** For multi-object segmentation or when complex spatial relationships beyond binary masking are required.

## Background
The technique is demonstrated in the context of volume rendering projects, where it is used to transform spherical volumes into disk shapes by removing density from specific regions [S1, unofficial]. It is also employed in 3D segmentation pipelines, such as SegmentAnythingin3D, to convert segmentation labels into density masks for rendering [S2, unofficial].

## Sources
[S1] Volume Rendering Final Project (Longer), https://www.youtube.com/watch?v=j4tECzsy0CE, fetched 2026-10-27 (unofficial)
[S2] Jumpat/SegmentAnythingin3D, https://github.com/Jumpat/SegmentAnythingin3D, fetched 2026-10-26 (unofficial)
