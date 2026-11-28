# Sum Compositing


## Overview
Sum Compositing creates a new volume grid by summing the values of two or more input maps at corresponding grid points. It solves the problem of combining multiple scalar fields (e.g., density maps) into a single unified representation. It is distinguished from [Max Compositing](8121-max-compositing.md) by preserving additive intensity rather than peak values, and from [Multiply Compositing](8122-multiply-compositing.md) by avoiding multiplicative attenuation. It builds on [Multi-Grid Merging](7905-multi-grid-merging.md) by defining the specific arithmetic operation for the merge.

## How it works
**Inputs:** Two or more volume maps (grids).
**Outputs:** A new volume map.

**Data Structures:**
The implementation relies on a target grid defined by `gridmap`. The target grid has dimensions $D_x \times D_y \times D_z$, spacing $S_x, S_y, S_z$, and origin $O$. Each voxel stores a value of type `valueType`.

**Steps:**
1.  **Initialize Target Grid:** Determine the output grid geometry. If `boundingGrid` is true and no specific `gridmap` is provided, the grid bounds are extended to encompass all input maps. Otherwise, use the specified `gridmap` or the first input map. Apply `gridStep` to subsample if necessary. Define the active region using `gridSubregion`.
2.  **Resample Inputs:** For each input map $M_i$:
    *   Map the input coordinates to the target grid coordinates.
    *   Sample the value $v_i$ at each target grid point using trilinear interpolation from the input map $M_i$.
    *   Apply a multiplier $k_i$ to the sampled value. The multiplier is specified by `scaleFactors`. If `fromlevels` is used, $k_i = 1 / L_i$, where $L_i$ is the highest surface threshold of map $M_i$. Otherwise, $k_i$ is the user-provided scalar.
3.  **Accumulate:** Initialize the target grid values to zero. For each target grid point $(x, y, z)$, compute the sum:
    $$ V_{out}(x, y, z) = \sum_{i} k_i \cdot \text{trilinear}(M_i, x, y, z) $$
4.  **Type Conversion:** Cast the resulting floating-point sums to `valueType` (e.g., `float32`, `int32`).

**Parameters:**
*   `scaleFactors`: Multipliers for each map. Comma-separated values (no spaces) or the keyword `fromlevels` [S2].
*   `onGrid`: Model number preceded by `#` of the map whose grid is used for the output. Defaults to the first map in `volume-spec` [S2].
*   `spacing`: Grid spacing for the output. Single value for all axes; three comma-separated values for X, Y, Z [S2].
*   `boundingGrid`: Boolean to adjust the grid of `gridmap` to bound the input maps. Default is `true` when adding maps without specifying a `gridmap`, otherwise `false` [S2].
*   `gridStep`: Integer subsample step size (default 1). Single value for all axes; three comma-separated values for X, Y, Z [S2].
*   `gridSubregion`: Subregion of `gridmap` to use. `all` (default) or grid indices $i1–i2$ (X), $j1–j2$ (Y), $k1–k2$ (Z). Indices are integers starting at zero [S2].
*   `valueType`: Output data type. Defaults to current type of `onGrid` map or first map. Options: `int8`, `int16`, `int32`, `uint8`, `uint16`, `uint32`, `float32`, `float64` [S2].

**Pitfalls:**
*   **Memory Limit:** The whole new map must fit in memory. The command may fail for lack of memory if the map is large (e.g., a whole tomogram) [S2].
*   **Resolution Loss:** Resampling causes some loss in resolution [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sum_compositing(input_maps, scale_factors, on_grid_map=None, 
                    spacing=None, bounding_grid=True, grid_step=1,
                    grid_subregion='all', value_type='float32'):
    """
    Combines multiple volume maps by summing their resampled values.
    
    Args:
        input_maps: List of volume map objects. Each has a grid and a method to sample values.
        scale_factors: List of multipliers (one per map) or 'fromlevels'.
        on_grid_map: The map defining the output grid geometry. Defaults to input_maps[0].
        spacing: Output grid spacing (scalar or tuple of 3).
        bounding_grid: If True, expand output grid to cover all inputs.
        grid_step: Subsample step size (1, 2, ...).
        grid_subregion: 'all' or tuple of (x_start, x_end, y_start, y_end, z_start, z_end).
        value_type: Output data type string.
    """
    # 1. Determine Output Grid Geometry
    if on_grid_map is None:
        base_map = input_maps[0]
    else:
        base_map = on_grid_map
    
    # Get base grid dimensions and spacing
    base_dims = base_map.grid.dimensions # (Dx, Dy, Dz)
    base_spacing = base_map.grid.spacing # (Sx, Sy, Sz)
    base_origin = base_map.grid.origin   # (Ox, Oy, Oz)
    
    # Apply spacing override if provided
    if spacing is not None:
        if isinstance(spacing, (int, float)):
            out_spacing = (spacing, spacing, spacing)
        else:
            out_spacing = tuple(spacing)
    else:
        out_spacing = base_spacing
        
    # Apply bounding grid logic
    if bounding_grid:
        # Extend bounds to cover all input maps
        min_coords = [float('inf')] * 3
        max_coords = [float('-inf')] * 3
        for m in input_maps:
            b = m.grid.bounds
            min_coords = [min(min_coords[i], b.min[i]) for i in range(3)]
            max_coords = [max(max_coords[i], b.max[i]) for i in range(3)]
        
        # Recalculate dimensions based on new bounds and spacing
        # Note: Exact dimension calculation depends on grid implementation details [G]
        # Assuming grid covers [origin, origin + dims * spacing]
        # We need to find a new origin and dims that cover min_coords and max_coords
        # For simplicity, we assume the base_map's origin is the anchor, 
        # and we expand dims if necessary.
        # A robust implementation would snap origin to grid lines.
        pass 
    else:
        min_coords = base_origin
        max_coords = [base_origin[i] + base_dims[i] * base_spacing[i] for i in range(3)]

    # Apply grid step (subsample)
    # If grid_step > 1, the effective resolution is lower
    # The output grid points are sampled at intervals of grid_step
    # This effectively reduces the number of output voxels
    if grid_step > 1:
        # Adjust dimensions to reflect subsampling
        # The physical extent remains the same, but fewer voxels
        # Or, the grid_step applies to the input sampling? 
        # Source says "subsample step size". Usually means output stride.
        # We will assume it reduces the output grid density.
        out_dims = tuple(d // grid_step if isinstance(grid_step, int) else d // gs 
                         for d, gs in zip(base_dims, (grid_step, grid_step, grid_step) if isinstance(grid_step, int) else grid_step))
    else:
        out_dims = base_dims

    # Apply grid subregion
    if grid_subregion != 'all':
        # grid_subregion is i1-i2, j1-j2, k1-k2
        # This restricts the output to a subset of the calculated out_dims
        # We need to map these indices to the output array
        pass

    # Initialize output buffer
    # Use a temporary float buffer for accumulation to avoid overflow during sum
    out_buffer = zeros(out_dims, dtype=float)
    
    # 2. Resample and Sum
    # Determine scale factors
    if scale_factors == 'fromlevels':
        factors = []
        for m in input_maps:
            level = m.highest_surface_threshold # Assumed property [G]
            if level == 0:
                raise ValueError("Map has no surface with nonzero highest threshold")
            factors.append(1.0 / level)
    else:
        factors = scale_factors
        
    # Ensure factors list matches input_maps length
    if len(factors) != len(input_maps):
        raise ValueError("Number of scale factors must match number of input maps")

    # For each output voxel, sum contributions from all input maps
    # To optimize, iterate over input maps and scatter-add to output
    
    # Coordinate transformation: Input (x_in, y_in, z_in) -> Output (x_out, y_out, z_out)
    # We need to map output grid coordinates to input grid coordinates for interpolation
    
    for i, m in enumerate(input_maps):
        k = factors[i]
        
        # Iterate over output grid points
        # Note: In practice, iterating over input active voxels and scattering 
        # might be faster if inputs are sparse, but Sum Compositing typically 
        # operates on dense grids or requires dense output [G]
        
        for x_out in range(out_dims[0]):
            for y_out in range(out_dims[1]):
                for z_out in range(out_dims[2]):
                    # Calculate world coordinate of output voxel
                    # Assuming regular grid
                    wx = base_origin[0] + x_out * out_spacing[0]
                    wy = base_origin[1] + y_out * out_spacing[1]
                    wz = base_origin[2] + z_out * out_spacing[2]
                    
                    # Sample input map at this world coordinate
                    # Trilinear interpolation
                    val = trilinear_sample(m, wx, wy, wz)
                    
                    # Accumulate
                    out_buffer[x_out, y_out, z_out] += k * val

    # 3. Type Conversion
    # Cast to valueType
    # Note: Casting from float to int involves truncation or rounding. 
    # Standard behavior is usually truncation or nearest. [G]
    if value_type == 'float32':
        result_data = out_buffer.astype(np.float32)
    elif value_type == 'float64':
        result_data = out_buffer.astype(np.float64)
    elif value_type == 'int8':
        result_data = np.clip(out_buffer, -128, 127).astype(np.int8)
    elif value_type == 'int16':
        result_data = np.clip(out_buffer, -32768, 32767).astype(np.int16)
    elif value_type == 'int32':
        result_data = np.clip(out_buffer, -2147483648, 2147483647).astype(np.int32)
    elif value_type == 'uint8':
        result_data = np.clip(out_buffer, 0, 255).astype(np.uint8)
    elif value_type == 'uint16':
        result_data = np.clip(out_buffer, 0, 65535).astype(np.uint16)
    elif value_type == 'uint32':
        result_data = np.clip(out_buffer, 0, 4294967295).astype(np.uint32)
    else:
        result_data = out_buffer # Default to float if type not recognized
        
    return VolumeMap(data=result_data, grid=base_map.grid, spacing=out_spacing)

def trilinear_sample(volume_map, x, y, z):
    """
    Performs trilinear interpolation on a volume map.
    Returns 0.0 if coordinates are outside the map bounds.
    """
    # Convert world coordinates to grid coordinates
    # gx = (x - origin.x) / spacing.x
    # ...
    # Check bounds
    # Interpolate 8 neighbors
    # Return weighted sum
    pass # Implementation details of trilinear interpolation are standard [G]
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of voxels in the output grid and $M$ is the number of input maps. Each voxel requires $M$ trilinear interpolations.
*   **Memory Complexity:** $O(N)$ for the output grid, plus $O(N)$ temporary buffer for accumulation if not done in-place. Input maps must be loaded into memory.
*   **Performance:** Performance is limited by memory bandwidth due to the random access pattern of trilinear interpolation from multiple source grids. Resampling is the dominant cost.
*   **Strengths:** Simple, intuitive addition of densities. Preserves total mass/intensity.
*   **Limitations:** Resolution loss due to resampling. Memory intensive for large volumes. No handling of non-overlapping regions other than zero-filling.
*   **When to use:** When combining multiple density fields where additive intensity is physically meaningful (e.g., combining multiple particle clouds).
*   **When not to use:** When preserving sharp boundaries is critical (use Max Compositing). When dealing with multiplicative effects like light attenuation (use Multiply Compositing).

## Background
Sum Compositing is a standard operation in volume data processing, particularly in scientific visualization and medical imaging. It is implemented in software such as UCSF ChimeraX [S2] and referenced in Julia-based volume rendering libraries like Lyr.jl [S1, unofficial]. The method is fundamental to multi-grid merging strategies where multiple resolution levels or modalities are combined.

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] Command: volume, https://www.rbvi.ucsf.edu/chimerax/docs/user/commands/volume.html, fetched 2026-10-27
