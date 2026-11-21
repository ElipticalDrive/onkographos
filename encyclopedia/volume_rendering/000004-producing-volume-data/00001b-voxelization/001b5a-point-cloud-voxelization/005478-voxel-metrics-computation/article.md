# Voxel Metrics Computation

Also called: Voxel Feature Aggregation via Formula

## Overview
Voxel Metrics Computation is a specialized voxelization method implemented in the `lidR` R package that aggregates point cloud attributes into a regular 3D grid based on user-defined formulas, rather than performing simple spatial binning. It solves the problem of extracting complex statistical features (such as point counts, mean heights, or canopy cover) from LiDAR data for downstream analysis, returning a structured `lasmetrics3d` object. It differs from sibling methods like [Hard Voxelization](5345-hard-voxelization.md) or [Grid Partitioning by Cell Size](15111-grid-partitioning-by-cell-size.md) by decoupling the geometric grid definition from the attribute aggregation logic, allowing arbitrary R expressions to define voxel content.

## How it works
**Inputs and Outputs:**
- **Input:** A `LAS` object (containing point cloud data with X, Y, Z coordinates and optional attributes) and a resolution parameter `res`.
- **Output:** A `lasmetrics3d` object, which is a specialized R6 class inheriting from `lasmetrics` and `spatial` classes, containing the voxel grid and computed metrics.

**Data Structures:**
The `lasmetrics3d` object stores the voxel grid as a 3D array. Each voxel cell contains a list of computed metrics defined by the input formula. A critical structural requirement is the presence of a column named `N` (representing the number of points in the voxel) if the output is to be used by certain downstream functions like `lacunr::bounding_box()` [S1].

**Parameters:**
- `res`: The voxel resolution. This parameter accepts a numeric value or a vector of length 1 or 2. If a single value is provided, it applies to both X and Y dimensions. If two values are provided, they apply to X and Y respectively. The Z dimension resolution is not explicitly set by this parameter in the same manner, often implying a fixed or derived vertical resolution or requiring specific handling [S1, S2].
- `formula`: An R formula specifying the metrics to compute. The right-hand side must evaluate to a list of named vectors or scalars. For example, `~list(N = length(Z))` computes the point count [S1, S2].

**Algorithm Steps:**
1. **Grid Initialization:** Determine the bounding box of the input point cloud. Divide the X and Y extents into bins based on the `res` parameter. The number of bins in each dimension is determined by the alignment algorithm specific to `lidR`, which may differ from standard floor/ceiling binning [S1, S2].
2. **Point Assignment:** Assign each point in the `LAS` object to a voxel bin based on its X and Y coordinates. Points are grouped by their assigned voxel index.
3. **Metric Computation:** For each voxel containing points, evaluate the `formula`. The formula is executed in the context of the subset of points belonging to that voxel. The variables available in the formula context include the point attributes (e.g., `Z` for elevation, `X`, `Y`).
4. **Result Construction:** Assemble the results into a `lasmetrics3d` object. The object contains the grid dimensions and the computed metrics for each voxel. Empty voxels may be represented as `NA` or zero depending on the formula output [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode representing the logical flow of lidR::voxel_metrics
# Note: This is a conceptual translation of the R implementation logic.

def voxel_metrics(las_object, formula, res):
    """
    Compute voxel metrics for a point cloud.
    
    Args:
        las_object: LAS object with fields X, Y, Z, and other attributes.
        formula: A callable or expression that takes a subset of points and returns a dict of metrics.
        res: Resolution for X and Y dimensions. Can be scalar or 2-tuple.
        
    Returns:
        lasmetrics3d: A structured object containing the voxel grid and metrics.
    """
    
    # 1. Determine grid dimensions
    # Extract X and Y bounds from las_object
    x_min, x_max = min(las_object.X), max(las_object.X)
    y_min, y_max = min(las_object.Y), max(las_object.Y)
    
    # Handle resolution input
    if isinstance(res, (int, float)):
        res_x, res_y = res, res
    elif isinstance(res, tuple) and len(res) == 2:
        res_x, res_y = res
    else:
        raise ValueError("Resolution must be a scalar or a 2-tuple")
        
    # Calculate number of bins. 
    # Note: lidR's alignment may differ from simple floor((max-min)/res).
    # The exact bin count depends on the internal alignment algorithm.
    # For example, a 12m span with 1m res might yield 13 bins in lidR vs 12 in other tools.
    num_bins_x = calculate_bins(x_min, x_max, res_x) 
    num_bins_y = calculate_bins(y_min, y_max, res_y)
    
    # 2. Initialize grid
    # Grid is a 3D array or hash map. 
    # Each cell stores the list of point indices or the computed metrics directly.
    grid = initialize_grid(num_bins_x, num_bins_y)
    
    # 3. Assign points to voxels
    for point in las_object.points:
        # Compute voxel coordinates
        # The alignment logic here is specific to lidR
        vx = assign_to_bin(point.X, x_min, res_x)
        vy = assign_to_bin(point.Y, y_min, res_y)
        
        # Add point to the corresponding voxel
        grid[vx, vy].append(point)
        
    # 4. Compute metrics
    metrics_result = {}
    for vx in range(num_bins_x):
        for vy in range(num_bins_y):
            points_in_voxel = grid[vx, vy]
            
            if points_in_voxel:
                # Extract attributes for the subset
                subset_Z = [p.Z for p in points_in_voxel]
                subset_X = [p.X for p in points_in_voxel]
                subset_Y = [p.Y for p in points_in_voxel]
                
                # Evaluate formula
                # The formula is evaluated in a context where variables like Z, X, Y are available
                # Example formula: ~list(N = length(Z))
                computed_metrics = evaluate_formula(formula, {
                    'Z': subset_Z,
                    'X': subset_X,
                    'Y': subset_Y
                })
                
                metrics_result[(vx, vy)] = computed_metrics
            else:
                # Handle empty voxels
                metrics_result[(vx, vy)] = None # Or NA
                
    # 5. Return lasmetrics3d object
    return lasmetrics3d(grid=metrics_result, res=res)

def calculate_bins(min_val, max_val, res):
    """
    Calculate the number of bins. 
    lidR's behavior may result in ceil((max-min)/res) + 1 or similar alignment.
    """
    # This is a placeholder for the specific lidR alignment logic.
    # Standard binning might be floor((max-min)/res).
    # lidR often produces one more bin than expected for integer spans.
    span = max_val - min_val
    # Example: span=12, res=1 -> 13 bins in lidR
    return int(span / res) + 1

def assign_to_bin(value, min_val, res):
    """
    Assign a coordinate to a bin index.
    """
    return int((value - min_val) / res)

def evaluate_formula(formula, context):
    """
    Evaluate the R formula in the given context.
    """
    # In R, this would be eval(parse(text = formula))
    # Here we simulate the logic for the example formula ~list(N = length(Z))
    if 'N' in formula.variables:
        return {'N': len(context['Z'])}
    # Add other metric evaluations as needed
    return {}
```

**Edge Cases and Pitfalls:**
- **Missing 'N' Column:** If the output `lasmetrics3d` object is passed to `lacunr::bounding_box()`, it must contain a column named `N` recording the number of points in each voxel. Failure to include this column in the formula (e.g., by not computing `length(Z)`) will cause an error [S1].
- **Resolution Dimensions:** `voxel_metrics()` permits at most two dimensions for voxel resolution (X and Y). It does not allow independent specification of Z resolution in the same way `voxelize()` does [S1, S2].
- **Alignment Differences:** The alignment of the point cloud within the voxel grid differs from other tools like `lacunr::voxelize()`. This can result in different bin counts for the same input data and resolution. For instance, a 12-meter span with 1-meter resolution yields 13 voxels in `lidR` but 12 in `voxelize()` [S1, S2].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of points, as each point is assigned to a voxel and metrics are computed per voxel. The complexity of metric computation depends on the formula.
- **Memory Complexity:** $O(V)$, where $V$ is the number of voxels, to store the grid and metrics.
- **Strengths:** High flexibility in metric computation via R formulas. Integrated with the `lidR` ecosystem for LiDAR data processing.
- **Limitations:** Limited to 2D resolution specification. Different alignment behavior compared to other voxelization tools may cause inconsistencies in cross-tool workflows. Performance may be slower than C++-based voxelization methods due to R overhead.

## Background
The `voxel_metrics()` function is part of the `lidR` package, a comprehensive R package for LiDAR data processing. It was designed to provide a versatile alternative to simpler voxelization functions like `voxelize()` (adapted from `rTLS` package by J. Antonio Guzmán Q.). The function is particularly useful for ecological and forestry applications where complex metrics (e.g., canopy height percentiles, point density) are required per voxel [S1, S2].

## Sources
[S1] Interfacing with lidR, https://cran.rstudio.org/web/packages/lacunr/vignettes/working-with-lidR.html, fetched 2026-10-27
[S2] README, https://cran.rstudio.org/web/packages/lacunr/readme/README.html, fetched 2026-10-27
