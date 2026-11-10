# Data Interpolation

Also called: Grid Mapping with Filtering, Irregular to Regular Grid Conversion, Preprocessing Interpolation, Scattered Data to Rectangular Grid Conversion

## Overview
Data Interpolation is a preprocessing technique that maps arbitrarily shaped or scattered 3D data grids into rectangular grids suitable for visualization software and volume texture generation [S1, S2, unofficial]. It solves the problem of uneven data structures by converting non-uniform or scattered inputs into regular grids, enabling shader programming and standard volume rendering pipelines [S2, unofficial]. It differs from sibling methods like [Nearest Voxel Lookup](1573-nearest-voxel-lookup-with-periodic-boundary.md) by actively reconstructing values via filtering rather than simple lookup, and from [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md) by focusing on the global grid construction and filtering strategy rather than local basis functions alone.

## How it works
**Inputs and Outputs**
- Input: Arbitrary grid data or scattered data points, potentially with non-uniform spacing (e.g., depth intervals, irregular latitude/longitude) [S1, S2, unofficial].
- Output: A rectangular grid with constant intervals or user-defined spacing, ready for volume texture generation [S1, S2, unofficial].

**Data Structures**
- Parameter File (`map.ini.c`): A text file where each line contains a numbered parameter, followed by whitespace, then the value. Comments start with an asterisk (`*`) as the first non-whitespace character. Everything after a parameter on the same line is ignored. Paths are acceptable for grid index file, grid data file, and output disk file [S1, unofficial].
- Rectangular Grid: Defined by planes spaced at constant intervals or concentrated where data points exist [S1, unofficial].

**Steps**
1. **Grid Construction**: Construct the target rectangular grid. The method is optional:
   - Option A: Space planes at constant intervals along each axis direction between grid extremes [S1, unofficial].
   - Option B: Place planes where data points are concentrated [S1, unofficial].
   - Users may specify bounding $i, j, k$ values to carve out sections for zooming. Carving can occur before or after mapping; pre-mapping carving offers more flexibility for memory management [S1, unofficial].
2. **Filtering**: Apply a filtering function to map the scattered data onto the rectangular grid. Users must select from available filter types [S1, unofficial].
   - **Density Weighting Filter**: Based on distance. Requires a user-specified multiplier for the search filter radius [S1, unofficial].
   - **Gaussian Filter**: Requires a user-specified number of standard deviations defining the filter radius [S1, unofficial].
   - **Z-buffer Type Filter**: Provided as an alternative [S1, unofficial].
3. **Interpolation Specifics**:
   - For vertical non-uniformity (e.g., depth), apply spline interpolation (e.g., over 0 m to -2000 m with 5 m intervals, resulting in 401 layers) [S2, unofficial].
   - For horizontal non-uniformity, apply cubic spline interpolation to generate regular latitude and longitude grids (e.g., 300 × 300 per depth layer, containing 90,000 points) [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def data_interpolation(scattered_data, params):
    # 1. Parse parameters from map.ini.c structure
    # Format: "number value # comment"
    # Ignore lines starting with '*'
    grid_spacing = params.get('grid_spacing', 'constant') # 'constant' or 'concentrated'
    filter_type = params.get('filter_type', 'gaussian')
    search_radius_multiplier = params.get('search_radius_multiplier', 1.0)
    gaussian_std_devs = params.get('gaussian_std_devs', 2.0)
    carve_bounds = params.get('carve_bounds', None) # (i_min, i_max, j_min, j_max, k_min, k_max)

    # 2. Construct Rectangular Grid
    if carve_bounds:
        # Carve section before mapping for flexibility
        active_data = filter_scattered_data(scattered_data, carve_bounds)
    else:
        active_data = scattered_data

    if grid_spacing == 'constant':
        target_grid = create_constant_spacing_grid(active_data)
    else:
        target_grid = create_concentrated_spacing_grid(active_data)

    # 3. Map Data using Filtering
    result_grid = create_empty_grid(target_grid.shape)

    for voxel in target_grid:
        # Determine search radius based on filter type
        if filter_type == 'density_weighting':
            radius = search_radius_multiplier * get_average_spacing(active_data)
            weights = compute_density_weights(active_data, voxel.position, radius)
            value = weighted_sum(active_data, weights)
        elif filter_type == 'gaussian':
            radius = gaussian_std_devs * get_average_spacing(active_data)
            weights = compute_gaussian_weights(active_data, voxel.position, radius)
            value = weighted_sum(active_data, weights)
        elif filter_type == 'z_buffer':
            # Z-buffer type filter implementation details not specified in sources [G]
            value = compute_z_buffer_filter(active_data, voxel.position)
        else:
            raise ValueError("Unknown filter type")

        result_grid[voxel.index] = value

    return result_grid

def filter_scattered_data(data, bounds):
    # Filter data points within bounding i, j, k values [G]
    return [p for p in data if bounds.i_min <= p.i <= bounds.i_max and
            bounds.j_min <= p.j <= bounds.j_max and
            bounds.k_min <= p.k <= bounds.k_max]

def create_constant_spacing_grid(data):
    # Space planes at constant intervals along each axis direction [S1, unofficial]
    # Not given in the sources: exact algorithm for interval calculation [G]
    pass

def create_concentrated_spacing_grid(data):
    # Place planes where data points are concentrated [S1, unofficial]
    # Not given in the sources: exact algorithm for concentration detection [G]
    pass

def compute_density_weights(data, target_pos, radius):
    # Density weighting filter based on distance [S1, unofficial]
    weights = []
    for point in data:
        dist = distance(point.position, target_pos)
        if dist <= radius:
            weights.append(1.0 / dist) # Standard inverse distance weighting [G]
        else:
            weights.append(0.0)
    return weights

def compute_gaussian_weights(data, target_pos, radius):
    # Gaussian filter with specified standard deviations [S1, unofficial]
    # Not given in the sources: exact sigma calculation from std devs [G]
    sigma = radius / gaussian_std_devs
    weights = []
    for point in data:
        dist = distance(point.position, target_pos)
        w = exp(-0.5 * (dist / sigma) ** 2)
        weights.append(w)
    return weights

def weighted_sum(data, weights):
    # Compute weighted average
    total_weight = sum(weights)
    if total_weight == 0:
        return 0.0
    weighted_val = sum(w * p.value for w, p in zip(weights, data))
    return weighted_val / total_weight
```

**Parameters**
- Search filter radius multiplier: User-specified factor controlling the search radius for the density weighting filter [S1, unofficial].
- Gaussian filter standard deviations: User-specified number of standard deviations defining the filter radius for the Gaussian filter [S1, unofficial].
- Grid construction method: Optional; constant intervals or concentrated planes [S1, unofficial].
- Filter type: Density weighting, Gaussian, or Z-buffer [S1, unofficial].

**Edge Cases and Pitfalls**
- Carving before mapping provides more flexibility for memory management but requires careful handling of bounding indices [S1, unofficial].
- Error checking is difficult due to the global variable initialization structure in the reference implementation [S1, unofficial].
- Non-uniform depth intervals (e.g., ocean data) require specific vertical interpolation (spline) to avoid artifacts [S2, unofficial].

## Subtopics

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
Collects cells using corner indices; use when precise cell topology is needed for irregular grids.

### [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)
Finds nearest voxel with periodic wrapping; use for toroidal or repeating domains.

### [Mipmap Grid Query](4154-mipmap-grid-query.md)
Queries multi-resolution grids; use for LOD rendering.

### [Cylindrical Coordinate Interpolation](4282-cylindrical-coordinate-interpolation.md)
Interpolates in cylindrical coordinates; use for axisymmetric data.

### [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md)
Uses polynomial/spline bases; use when local smoothness is prioritized over global filtering.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Not given in the sources.
- **Strengths**: Flexibility in grid construction (constant vs. concentrated); ability to zoom into dense areas; multiple filter options for experimentation [S1, unofficial].
- **Limitations**: Requires preprocessing; parameter tuning (filter radius, std devs) is necessary; error checking is difficult in the reference implementation [S1, unofficial].
- **When to use**: When input data is scattered or non-uniform and must be converted to a regular grid for volume texture generation [S2, unofficial].
- **When not to use**: When data is already on a regular grid or when real-time interpolation is required without preprocessing [G].

## Background
Originating from requirements for visualization software in fluid flow research, this method addresses the need to map arbitrary grids into rectangular forms for volume visualization [S1, unofficial]. It was developed to handle data grids with broad ranges of densities, being dense in areas of high concern and sparse elsewhere [S1, unofficial]. The technique is used in ocean environmental parameter visualization and acoustic propagation loss modeling [S2, unofficial].

## Sources
[S1] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
[S2] The Implementation of a WebGPU-Based Volume Rendering Framework ... - MDPI, https://www.mdpi.com/2076-3417/15/5/2782, fetched 2026-10-29 (unofficial)
