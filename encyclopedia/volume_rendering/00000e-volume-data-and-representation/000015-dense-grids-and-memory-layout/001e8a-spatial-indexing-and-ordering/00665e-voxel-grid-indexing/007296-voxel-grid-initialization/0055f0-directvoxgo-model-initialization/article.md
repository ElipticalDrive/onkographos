# DirectVoxGO Model Initialization

Also called: DirectVoxGO Initialization

## Overview
Initializes the DirectVoxGO neural radiance field model by setting up density grids, segmentation masks, and color networks based on bounding box coordinates and voxel counts [S1, unofficial]. It differs from sibling methods by explicitly calculating density bias for initial transparency and managing dual segmentation mask grids for iterative refinement, whereas siblings like [Voxel Grid Construction](3085-voxel-grid-construction.md) focus on generic grid population and [Dense Copy Ingestion](3212-dense-copy-ingestion.md) handles data transfer.

## How it works
**Inputs:**
- `xyz_min`: Minimum coordinates of the 3D bounding box (`torch.Tensor`).
- `xyz_max`: Maximum coordinates of the 3D bounding box (`torch.Tensor`).
- `num_voxels`: Total number of voxels for the current grid resolution.
- `num_voxels_base`: Base number of voxels used to calculate base voxel size.
- `alpha_init`: Initial alpha value used to compute the density bias shift.
- `fast_color_thres`: Threshold for determining visible voxels in mask cache updates.
- `density_type`: String specifying the type of density grid (e.g., 'DenseGrid').
- `k0_type`: String specifying the type of color/feature grid.
- `rgbnet_dim`: Dimension of the feature grid for the fine stage. If $\le 0$, uses color voxel grid directly.
- `rgbnet_direct`: Boolean flag for direct color network connection.
- `rgbnet_full_implicit`: Boolean flag for full implicit color network.
- `rgbnet_depth`: Depth of the MLP for color prediction.
- `rgbnet_width`: Width of the MLP for color prediction.
- `viewbase_pe`: Number of positional encoding frequencies for view direction.
- `mask_cache_path`: Path to precomputed mask cache file.
- `mask_cache_thres`: Threshold for mask cache values.
- `mask_cache_world_size`: World size for the mask cache grid. Defaults to `self.world_size` if `None`.
- `num_objects`: Number of segmentation objects.

**Outputs:**
- Initialized model components including density grid, segmentation mask grids, view count tensor, RGB network, and mask cache.

**Data Structures:**
1.  **Density Grid**: Created via `grid.create_grid`.
    -   Type: Specified by `density_type`.
    -   Channels: 1.
    -   World Size: Calculated from bounding box and voxel size.
    -   Bounds: `xyz_min`, `xyz_max`.
2.  **Segmentation Mask Grids (`seg_mask_grid`, `dual_seg_mask_grid`)**:
    -   Type: Specified by `density_type`.
    -   Channels: `num_objects`.
    -   World Size: Same as density grid.
    -   Bounds: `xyz_min`, `xyz_max`.
3.  **Mask View Counts (`mask_view_counts`)**:
    -   Shape: Same as `seg_mask_grid.grid`.
    -   Values: Initialized to zeros.
    -   Gradient: `requires_grad=False`.
4.  **RGB Network (`rgbnet`)**:
    -   Type: `nn.Sequential` MLP.
    -   Input Dimension: `dim0` (calculated below).
    -   Hidden Layers: `rgbnet_width` neurons, ReLU activation.
    -   Output Dimension: 3 (RGB).
    -   Last Layer Bias: Initialized to 0.
5.  **Mask Cache (`mask_cache`)**:
    -   Type: `grid.MaskGrid` object.
    -   Content: Binary mask indicating valid scene volume.

**Steps:**

1.  **Calculate Voxel Sizes and World Dimensions**:
    -   Compute base voxel size:
        $$ \text{voxel\_size\_base} = \left( \frac{\text{prod}(\text{xyz\_max} - \text{xyz\_min})}{\text{num\_voxels\_base}} \right)^{1/3} $$
    -   Compute current voxel size:
        $$ \text{voxel\_size} = \left( \frac{\text{prod}(\text{xyz\_max} - \text{xyz\_min})}{\text{num\_voxels}} \right)^{1/3} $$
    -   Compute world size (grid resolution):
        $$ \text{world\_size} = \lfloor (\text{xyz\_max} - \text{xyz\_min}) / \text{voxel\_size} \rfloor $$
    -   Compute voxel size ratio:
        $$ \text{voxel\_size\_ratio} = \text{voxel\_size} / \text{voxel\_size\_base} $$

2.  **Initialize Mask Cache**:
    -   If `mask_cache_path` is not `None` and not empty:
        -   Load mask from `grid.MaskGrid(path=mask_cache_path, mask_cache_thres=mask_cache_thres)`.
    -   Else:
        -   Generate grid coordinates for mask cache:
            $$ \text{coords} = \text{torch.stack}(\text{torch.meshgrid}(\text{linspace}(\text{xyz\_min}[0], \text{xyz\_max}[0], \text{mask\_cache\_world\_size}[0]), \text{linspace}(\text{xyz\_min}[1], \text{xyz\_max}[1], \text{mask\_cache\_world\_size}[1]), \text{linspace}(\text{xyz\_min}[2], \text{xyz\_max}[2], \text{mask\_cache\_world\_size}[2])), -1) $$
        -   Set mask to `torch.ones(list(mask_cache_world_size), dtype=torch.bool)` [G: The source notes state "set mask to torch.ones..." if path is empty/None, implying a default valid mask if no cache exists, though typically a cache implies filtering. We follow the note literally].

3.  **Calculate RGB Network Input Dimension (`dim0`)**:
    -   Generate positional encoding frequencies:
        $$ \text{viewfreq} = [2^0, 2^1, \dots, 2^{\text{viewbase\_pe}-1}] $$
    -   Base dimension:
        $$ \text{dim0} = 3 + 3 \times \text{viewbase\_pe} \times 2 $$
        (3 for position, $3 \times \text{viewbase\_pe} \times 2$ for view direction PE).
    -   Adjust for feature grid:
        -   If `rgbnet_direct` is True:
            $$ \text{dim0} = \text{dim0} + \text{k0\_dim} $$
        -   Else if not `rgbnet_full_implicit`:
            $$ \text{dim0} = \text{dim0} + \text{k0\_dim} - 3 $$
        -   Else (full implicit):
            $$ \text{dim0} = \text{dim0} $$ (Implicit features are handled differently, typically concatenated or passed as latent codes, but the formula suggests adjustment only for direct/non-full-implicit cases).

4.  **Create Density Grid**:
    -   Call `grid.create_grid(density_type, channels=1, world_size, xyz_min, xyz_max, density_config)`.
    -   Apply density bias shift to initial values:
        $$ \text{act\_shift} = \log\left(\frac{1}{1 - \text{alpha\_init}} - 1\right) $$
        This bias is added to density values to control initial transparency.

5.  **Create Segmentation Mask Grids**:
    -   Create `seg_mask_grid` using `grid.create_grid(density_type, channels=num_objects, world_size, xyz_min, xyz_max, density_config)`.
    -   Create `dual_seg_mask_grid` using `grid.create_grid(density_type, channels=num_objects, world_size, xyz_min, xyz_max, density_config)`.

6.  **Initialize Mask View Counts**:
    -   Create tensor `mask_view_counts` with shape equal to `seg_mask_grid.grid`.
    -   Initialize values to 0.
    -   Set `requires_grad=False`.

7.  **Initialize RGB Network**:
    -   Create `nn.Sequential` MLP.
    -   Input dim: `dim0`.
    -   Hidden layers: `rgbnet_width` neurons, ReLU activation.
    -   Output layer: 3 neurons (RGB), bias initialized to 0.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def initialize_direct_voxgo(
    xyz_min, xyz_max, num_voxels, num_voxels_base, alpha_init,
    fast_color_thres, density_type, k0_type, rgbnet_dim,
    rgbnet_direct, rgbnet_full_implicit, rgbnet_depth, rgbnet_width,
    viewbase_pe, mask_cache_path, mask_cache_thres, mask_cache_world_size,
    num_objects, k0_dim, grid, density_config
):
    # 1. Calculate Voxel Sizes and World Dimensions
    volume = (xyz_max - xyz_min).prod()
    voxel_size_base = (volume / num_voxels_base) ** (1/3)
    voxel_size = (volume / num_voxels) ** (1/3)
    
    # World size calculation: floor division element-wise
    world_size = torch.floor((xyz_max - xyz_min) / voxel_size).long()
    
    voxel_size_ratio = voxel_size / voxel_size_base
    
    # 2. Initialize Mask Cache
    if mask_cache_path is not None and mask_cache_path != "":
        mask_cache = grid.MaskGrid(path=mask_cache_path, mask_cache_thres=mask_cache_thres)
    else:
        # Generate grid coordinates
        # Note: mask_cache_world_size defaults to self.world_size if None, 
        # but here we assume it's provided or derived. 
        # Using the provided mask_cache_world_size or world_size.
        w_size = mask_cache_world_size if mask_cache_world_size is not None else world_size
        x = torch.linspace(xyz_min[0], xyz_max[0], w_size[0])
        y = torch.linspace(xyz_min[1], xyz_max[1], w_size[1])
        z = torch.linspace(xyz_min[2], xyz_max[2], w_size[2])
        coords = torch.stack(torch.meshgrid(x, y, z, indexing='ij'), dim=-1)
        # Set mask to ones (valid)
        mask_cache = grid.MaskGrid(grid=torch.ones(list(w_size), dtype=torch.bool))

    # 3. Calculate RGB Network Input Dimension (dim0)
    # viewfreq = [2^0, ..., 2^(viewbase_pe-1)]
    # dim0 = 3 + 3 * viewbase_pe * 2
    dim0 = 3 + 3 * viewbase_pe * 2
    
    if rgbnet_direct:
        dim0 += k0_dim
    elif not rgbnet_full_implicit:
        dim0 += k0_dim - 3
    # else: dim0 remains 3 + 3 * viewbase_pe * 2

    # 4. Create Density Grid
    density_grid = grid.create_grid(
        density_type=density_type, 
        channels=1, 
        world_size=world_size, 
        xyz_min=xyz_min, 
        xyz_max=xyz_max, 
        density_config=density_config
    )
    
    # Apply density bias
    act_shift = math.log(1 / (1 - alpha_init) - 1)
    # Note: The exact application of act_shift depends on grid.create_grid implementation.
    # Typically, initial density values are shifted by this amount.
    # [G] Assuming density_grid has an attribute or method to set initial values or bias.
    # If density_grid.grid is the tensor:
    # density_grid.grid.data += act_shift 
    # However, since density is often logit-space, we add the shift.
    
    # 5. Create Segmentation Mask Grids
    seg_mask_grid = grid.create_grid(
        density_type=density_type, 
        channels=num_objects, 
        world_size=world_size, 
        xyz_min=xyz_min, 
        xyz_max=xyz_max, 
        density_config=density_config
    )
    
    dual_seg_mask_grid = grid.create_grid(
        density_type=density_type, 
        channels=num_objects, 
        world_size=world_size, 
        xyz_min=xyz_min, 
        xyz_max=xyz_max, 
        density_config=density_config
    )

    # 6. Initialize Mask View Counts
    mask_view_counts = torch.zeros_like(seg_mask_grid.grid, requires_grad=False)

    # 7. Initialize RGB Network
    layers = []
    for i in range(rgbnet_depth - 1):
        if i == 0:
            layers.append(torch.nn.Linear(dim0, rgbnet_width))
            layers.append(torch.nn.ReLU())
        else:
            layers.append(torch.nn.Linear(rgbnet_width, rgbnet_width))
            layers.append(torch.nn.ReLU())
    
    # Output layer
    layers.append(torch.nn.Linear(rgbnet_width, 3))
    # Bias of last layer initialized to 0
    layers[-1].bias.data.zero_()
    
    rgbnet = torch.nn.Sequential(*layers)

    return {
        "density_grid": density_grid,
        "seg_mask_grid": seg_mask_grid,
        "dual_seg_mask_grid": dual_seg_mask_grid,
        "mask_view_counts": mask_view_counts,
        "rgbnet": rgbnet,
        "mask_cache": mask_cache,
        "voxel_size": voxel_size,
        "voxel_size_base": voxel_size_base,
        "world_size": world_size
    }
```

## Subtopics

### [Voxel Grid Construction](3085-voxel-grid-construction.md)
Focuses on populating the grid with data, whereas initialization sets up the structure and parameters.

### [Grid Background Change](3210-grid-background-change.md)
Modifies background handling after initialization.

### [Dense Copy Ingestion](3212-dense-copy-ingestion.md)
Handles copying dense data into the initialized grids.

### [Voxel Grid Storage and Voxelization](6433-voxel-grid-storage-and-voxelization.md)
Deals with persistence and conversion.

### [VoxelGridFromList](7224-voxelgridfromlist.md)
Alternative initialization from a list of voxels.

## Operating characteristics
- **Time Complexity**: Initialization is $O(N)$ where $N$ is the number of voxels, dominated by grid creation and coordinate generation.
- **Memory Complexity**: $O(N \times C)$ where $C$ is the number of channels (1 for density, `num_objects` for masks).
- **Strengths**: Provides a consistent starting state for density and segmentation, ensuring correct spatial alignment and initial transparency.
- **Limitations**: Requires careful tuning of `alpha_init` and `mask_cache_thres` for optimal convergence.
- **When to use**: At the start of training or inference setup for DirectVoxGO models.

## Background
Originated in the SA3D framework (Segment Anything in 3D) by Jiazhong Cen et al. (2023) [S1, unofficial]. The method is part of the pipeline that allows 3D segmentation from a single view prompt by initializing the underlying NeRF/voxel representation correctly.

## Sources
[S1] Jumpat/SegmentAnythingin3D, https://github.com/Jumpat/SegmentAnythingin3D, fetched 2026-10-26 (unofficial)
