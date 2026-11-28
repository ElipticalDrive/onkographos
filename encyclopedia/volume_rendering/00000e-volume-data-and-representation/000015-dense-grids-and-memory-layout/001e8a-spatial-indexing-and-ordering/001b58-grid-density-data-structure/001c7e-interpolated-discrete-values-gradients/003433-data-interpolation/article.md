# Data Interpolation

Also called: Grid Mapping with Filtering, Irregular to Regular Grid Conversion, Preprocessing Interpolation, Scattered Data to Rectangular Grid Conversion, Resampling to Regular Grid

## Overview
Data Interpolation is a preprocessing technique that maps arbitrarily shaped or scattered 3D data grids into rectangular grids for visualization software [S1, unofficial]. It solves the problem of uneven data structures by converting non-uniform or scattered inputs into regular grids, enabling volume texture generation and shader programming [S3, unofficial]. It is the simplest and most inefficient approach to render irregular grids by resampling them to a regular grid [S2, unofficial].

## How it works
**Inputs and Outputs**
- Input: Arbitrary grid data or scattered data points, potentially with non-uniform spacing (e.g., depth intervals, irregular latitude/longitude) [S1, unofficial].
- Output: A rectangular grid with constant intervals or user-defined spacing, ready for volume texture generation [S3, unofficial].

**Data Structures**
- Parameter File (`map.ini.c`): A text file where each line contains a numbered parameter, followed by at least one whitespace character, then the value [S1, unofficial]. Comments start with an asterisk (`*`) as the first non-whitespace character on a line [S1, unofficial]. Everything after a parameter on the same line is ignored [S1, unofficial]. Paths are acceptable for grid index file, grid data file, and output disk file [S1, unofficial].
- Rectangular Grid: Defined by planes spaced at constant intervals or concentrated where data points exist [S1, unofficial].

**Steps**
1. **Grid Construction**: Construct the target rectangular grid [S1, unofficial]. The method is optional:
   - Option A: Space planes at constant intervals along each axis direction between grid extremes [S1, unofficial].
   - Option B: Place planes where data points are concentrated [S1, unofficial].
   - Users may specify bounding $i, j, k$ values to carve out sections for zooming [S1, unofficial]. Carving can occur before or after mapping; pre-mapping carving offers more flexibility for memory management [S1, unofficial].
2. **Filtering**: Apply a filtering function to map the scattered data onto the rectangular grid [S1, unofficial]. Users must select from available filter types [S1, unofficial].
   - **Density Weighting Filter**: Based on distance [S1, unofficial]. Requires a user-specified multiplier for the search filter radius [S1, unofficial].
   - **Gaussian Filter**: Requires a user-specified number of standard deviations defining the filter radius [S1, unofficial].
   - **Z-buffer Type Filter**: Provided as an alternative [S1, unofficial].
3. **Interpolation Specifics**:
   - For vertical non-uniformity (e.g., depth), apply spline interpolation (e.g., over 0 m to -2000 m with 5 m intervals, resulting in 401 layers) [S3, unofficial].
   - For horizontal non-uniformity, apply cubic spline interpolation to generate regular latitude and longitude grids (e.g., 300 × 300 per depth layer, containing 90,000 points) [S3, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Point:
    def __init__(self, i, j, k, value, position):
        self.i = i
        self.j = j
        self.k = k
        self.value = value
        self.position = position

class Grid:
    def __init__(self, rows, cols, spacing):
        self.rows = rows
        self.cols = cols
        self.spacing = spacing
        self.cells = [[None] * cols for _ in range(rows)]

def compute_distance(p1, p2):
    return math.sqrt(sum((a - b) ** 2 for a, b in zip(p1, p2)))

def data_interpolation(scattered_data, params):
    grid_spacing_mode = params.get('grid_spacing', 'constant')
    filter_type = params.get('filter_type', 'gaussian')
    search_radius_multiplier = params.get('search_radius_multiplier', 1.0)
    gaussian_std_devs = params.get('gaussian_std_devs', 2.0)
    carve_bounds = params.get('carve_bounds', None)

    if carve_bounds:
        active_data = [p for p in scattered_data if carve_bounds.i_min <= p.i <= carve_bounds.i_max and
                       carve_bounds.j_min <= p.j <= carve_bounds.j_max and
                       carve_bounds.k_min <= p.k <= carve_bounds.k_max]
    else:
        active_data = scattered_data

    if grid_spacing_mode == 'constant':
        target_grid = create_constant_spacing_grid(active_data)
    else:
        target_grid = create_concentrated_spacing_grid(active_data)

    result_grid = [[0.0] * target_grid.cols for _ in range(target_grid.rows)]

    for r in range(target_grid.rows):
        for c in range(target_grid.cols):
            voxel_pos = (r * target_grid.spacing, c * target_grid.spacing, 0.0)
            if filter_type == 'density_weighting':
                radius = search_radius_multiplier * compute_base_spacing(active_data)
                weights = compute_density_weights(active_data, voxel_pos, radius)
                value = weighted_sum(active_data, weights)
            elif filter_type == 'gaussian':
                radius = gaussian_std_devs * compute_base_spacing(active_data)
                weights = compute_gaussian_weights(active_data, voxel_pos, radius, gaussian_std_devs)
                value = weighted_sum(active_data, weights)
            elif filter_type == 'z_buffer':
                # Z-buffer type filter: select point with minimum depth distance
                value = compute_z_buffer_filter(active_data, voxel_pos)
            else:
                raise ValueError("Unknown filter type")

            result_grid[r][c] = value

    return result_grid

def compute_base_spacing(data):
    if not data: return 1.0
    xs = [p.position[0] for p in data]
    ys = [p.position[1] for p in data]
    zs = [p.position[2] for p in data]
    dx = max(xs) - min(xs)
    dy = max(ys) - min(ys)
    dz = max(zs) - min(zs)
    return (dx + dy + dz) / 3.0

def create_constant_spacing_grid(data):
    # Space planes at constant intervals along each axis direction
    # Not given in the sources: exact algorithm for interval calculation
    xs = [p.position[0] for p in data]
    ys = [p.position[1] for p in data]
    return Grid(rows=10, cols=10, spacing=1.0)

def create_concentrated_spacing_grid(data):
    # Place planes where data points are concentrated
    # Not given in the sources: exact algorithm for concentration detection
    xs = [p.position[0] for p in data]
    ys = [p.position[1] for p in data]
    return Grid(rows=10, cols=10, spacing=1.0)

def compute_density_weights(data, target_pos, radius):
    weights = []
    for point in data:
        dist = compute_distance(point.position, target_pos)
        if dist <= radius:
            weights.append(1.0 / dist)
        else:
            weights.append(0.0)
    return weights

def compute_gaussian_weights(data, target_pos, radius, std_devs):
    sigma = radius / std_devs
    weights = []
    for point in data:
        dist = compute_distance(point.position, target_pos)
        w = math.exp(-0.5 * (dist / sigma) ** 2)
        weights.append(w)
    return weights

def weighted_sum(data, weights):
    total_weight = sum(weights)
    if total_weight == 0:
        return 0.0
    weighted_val = sum(w * p.value for w, p in zip(weights, data))
    return weighted_val / total_weight

def compute_z_buffer_filter(data, target_pos):
    # Select nearest point along viewing axis
    if not data: return 0.0
    return min(data, key=lambda p: compute_distance(p.position, target_pos)).value
```

**Parameters**
- Search filter radius multiplier: User-specified factor controlling the search radius for the density weighting filter [S1, unofficial].
- Gaussian filter standard deviations: User-specified number of standard deviations defining the filter radius for the Gaussian filter [S1, unofficial].
- Grid construction method: Optional; constant intervals or concentrated planes [S1, unofficial].
- Filter type: Density weighting, Gaussian, or Z-buffer [S1, unofficial].

**Edge Cases and Pitfalls**
- Carving before mapping provides more flexibility for memory management but requires careful handling of bounding indices [S1, unofficial].
- Error checking is difficult due to the global variable initialization structure in the reference implementation [S1, unofficial].
- Non-uniform depth intervals (e.g., ocean data) require specific vertical interpolation (spline) to avoid artifacts [S3, unofficial].
- High sampling rates are needed for accuracy, making the resulting regular grid too large for storage and rendering [S2, unofficial].

## Subtopics

### [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)
Finds the nearest voxel with periodic wrapping; use for toroidal or repeating domains where boundary artifacts must be avoided.

### [Dense Copy Extraction](3215-dense-copy-extraction.md)
Extracts dense subgrids directly without resampling; use when the source data is already regular and high-fidelity extraction is required.

### [Mipmap Grid Query](4154-mipmap-grid-query.md)
Queries multi-resolution grids; use for level-of-detail rendering where performance scales with viewing distance.

### [Cylindrical Coordinate Interpolation](4282-cylindrical-coordinate-interpolation.md)
Interpolates in cylindrical coordinates; use for axisymmetric data where Cartesian resampling introduces distortion.

### [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md)
Uses polynomial/spline bases; use when local smoothness is prioritized over global filtering and exact derivative continuity is needed.

### [Interpolated SDF Gradient Calculation](12389-interpolated-sdf-gradient-calculation.md)
Computes gradients from interpolated signed distance fields; use for ray marching or normal estimation where analytical derivatives are unavailable.

## Operating characteristics
- **Time Complexity**: Not given in the sources. [S1, unofficial]
- **Memory Complexity**: Not given in the sources. [S1, unofficial]
- **Measured Performance**: Not given in the sources. [S1, unofficial]
- **Scaling**: Not given in the sources. [S1, unofficial]
- **Strengths**: Flexibility in grid construction (constant vs. concentrated); ability to zoom into dense areas; multiple filter options for experimentation [S1, unofficial].
- **Limitations**: Requires preprocessing; parameter tuning (filter radius, std devs) is necessary; error checking is difficult in the reference implementation [S1, unofficial].
- **When to use**: When input data is scattered or non-uniform and must be converted to a regular grid for volume texture generation [S3, unofficial].
- **When not to use**: When data is already on a regular grid or when real-time interpolation is required without preprocessing [G].

## Background
Originating from requirements for visualization software in fluid flow research, this method addresses the need to map arbitrary grids into rectangular forms for volume visualization [S1, unofficial]. It was developed to handle data grids with broad ranges of densities, being dense in areas of high concern and sparse elsewhere [S1, unofficial]. The technique is used in ocean environmental parameter visualization and acoustic propagation loss modeling [S3, unofficial].

## Sources
[S1] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S3] The Implementation of a WebGPU-Based Volume Rendering Framework ... - MDPI, https://www.mdpi.com/2076-3417/15/5/2782, fetched 2026-10-29 (unofficial)
