# Scene Geometry and Scalar Range Initialization

Also called: BuildSceneGeometry

## Overview
Constructs the scene geometry and scalar ranges for volume rendering by aggregating Adaptive Mesh Refinement (AMR) box data, computing global bounding boxes via MPI, applying normalization scaling, and determining scalar value ranges (original and processed) including optional logarithmic scaling. It differs from sibling methods by explicitly handling the aggregation of hierarchical box structures into a unified scene representation and managing the statistical reduction of scalar values across distributed ranks, whereas siblings like [TET-AMR Data Structure](1094-tet-amr-data-structure.md) focus on element topology or [AMR Basis Reconstruction Filter](5581-amr-basis-reconstruction-filter.md) on value interpolation.

## How it works
**Inputs and Outputs:**
*   **Inputs:** `ownedLevels` (array of AMR level data, e.g., `MultiFab`), `levelGeometry` (geometry information for each level), `options` (configuration struct containing `minLevel`, `maxLevel`, `component`, `logScaleInput`, `noDataError`, `invalidScalarError`).
*   **Outputs:** `scene` object containing `ownedLevels`, `localBoxes`, `globalMin`, `globalMax`, `explicitBounds`, `originalScalarRange`, `processedScalarRange`, and `scalarTransform`.

**Data Structures:**
*   `VolumeRenderer::AmrBox`:
    *   `cellDimensions`: `amrex::IntVect` (3 integers).
    *   `validBox`: `amrex::Box` (defines the grid indices).
    *   `values`: `const array reference` from `MultiFab`.
    *   `component`: `int` (index of the scalar component).
    *   `minCorner`: `Vec3` (3 floats, physical coordinates).
    *   `maxCorner`: `Vec3` (3 floats, physical coordinates).
*   `LocalScalarStats`:
    *   `minValue`: `amrex::Real` (initialized to $+\infty$).
    *   `maxValue`: `amrex::Real` (initialized to $-\infty$).
    *   `minPositive`: `amrex::Real` (initialized to $+\infty$).
    *   `finiteCount`: `amrex::Long` (initialized to 0).

**Steps:**

1.  **Validate Inputs:** If `ownedLevels` is null or `ownedLevels->size() != levelGeometry.size()`, throw `std::invalid_argument`.
2.  **Initialize Scene:** Move `ownedLevels` into `scene.ownedLevels`. Reserve 64 slots for `scene.localBoxes`. Initialize `localMinOriginal` and `localMaxOriginal` to $+\infty$ and $-\infty$ respectively. Set `hasLocalBoxes = false`. Initialize `localScalarStats` with default values.
3.  **Iterate Levels:** Loop `level` from `options.minLevel` to `options.maxLevel`.
    *   Skip if `level < 0` or `level >= scene.ownedLevels->size()`.
    *   Get `MultiFab mf` from `ownedLevels[level]`.
    *   Skip if `mf.size() == 0`.
4.  **Compute Local Scalar Statistics:** Call `reduceLocalScalarStats(mf, options.component)` to compute `localScalarStats` for the current level's component. Merge these into the global `localScalarStats` accumulator.
5.  **Iterate Boxes:** Use `amrex::MFIter` to iterate over valid boxes in `mf`.
    *   Get `box`, `lo` (smallEnd), `hi` (bigEnd).
    *   Compute dimensions `nx, ny, nz`. Skip if any dimension $\le 0$.
    *   Create `AmrBox` instance.
    *   Compute physical corners using the formula:
        $$ \text{minCorner}[x] = \text{geom.probLo}[x] + \text{lo}[x] \times \text{geom.cellSize}[x] $$
        $$ \text{maxCorner}[x] = \text{geom.probLo}[x] + (\text{hi}[x] + 1) \times \text{geom.cellSize}[x] $$
        for $x \in \{0, 1, 2\}$.
    *   Update `localMinOriginal` and `localMaxOriginal` using the component's min/max values from the box.
    *   Set `hasLocalBoxes = true`.
    *   Push `AmrBox` to `scene.localBoxes`.
6.  **Compute Global Bounding Box (Original):** Gather `localMinOriginal` and `localMaxOriginal` arrays to all ranks using `MPI_Allreduce` with `MPI_MIN` and `MPI_MAX` operations to obtain `globalMinOriginal` and `globalMaxOriginal`.
7.  **Compute Global Scale Factor:**
    *   Calculate `globalExtentOriginal` as `globalMaxOriginal - globalMinOriginal`.
    *   Find `minExtent` as the minimum non-zero finite axis length of `globalExtentOriginal`.
    *   Set `globalScale = 1.0 / minExtent`.
    *   If `minExtent <= 0` or is not finite, set `globalScale = 1.0`.
    *   If `globalScale` is not finite or $\le 0$, reset `globalScale = 1.0`.
8.  **Apply Global Scale:** If `globalScale != 1.0`, multiply all `AmrBox` `minCorner`/`maxCorner`, `globalMinOriginal`, and `globalMaxOriginal` by `globalScale`.
9.  **Compute Final Global Bounding Box:** Compute `globalMin` and `globalMax` from the scaled boxes using `MPI_Allreduce` (`MPI_MIN`/`MPI_MAX`).
10. **Compute Explicit Bounds:**
    *   Calculate `extent = globalMax - globalMin`.
    *   Calculate `maxExtent = max(axis extents)`.
    *   Calculate `paddingAmount = maxExtent * 0.05` if `maxExtent > 0`, else `1.0`.
    *   Set `scene.explicitBounds.minCorner = globalMin - paddingAmount`.
    *   Set `scene.explicitBounds.maxCorner = globalMax + paddingAmount`.
    *   Set `scene.hasExplicitBounds = true`.
11. **Gather Scalar Statistics:**
    *   Send `localOriginalMinSend` and `localOriginalMaxSend` (derived from `localScalarStats` if `finiteCount > 0`) via `MPI_Allreduce` (`MPI_MIN`/`MPI_MAX`) to get `globalOriginalScalarMin` and `globalOriginalScalarMax`.
    *   Sum `finiteCount` via `MPI_SUM` to get `globalOriginalFiniteCount`.
12. **Set Original Scalar Range:** If `globalOriginalFiniteCount > 0` and both min/max are finite, set `scene.originalScalarRange = makeScalarRange(globalOriginalScalarMin, globalOriginalScalarMax)` and `scene.hasOriginalScalarRange = true`.
13. **Logarithmic Scaling (Optional):** If `options.logScaleInput` is true:
    *   Gather `globalPositiveMin` (minimum of positive values) via `MPI_MIN`.
    *   Count `globalPositiveCount` via `MPI_SUM`.
    *   If `globalPositiveCount <= 0` or `globalPositiveMin` is not finite or $\le 0$, throw `std::runtime_error` "Log scaling requested but no positive scalar values were found."
    *   Compute `processedMin = log(globalPositiveMin)`.
    *   Compute `processedMax = log(max(globalOriginalScalarMax, globalPositiveMin))`.
    *   Set `scene.processedScalarRange = makeScalarRange(processedMin, processedMax)`.
    *   Set `scene.hasProcessedScalarRange = true`.
    *   Set `scene.scalarTransform.processedMin = processedMin`.
    *   Set `scene.scalarTransform.processedMax = processedMax`.
    *   Set `scene.scalarTransform.inverseProcessedSpan = 1.0 / (processedMax - processedMin)`.
    *   Set `scene.scalarTransform.positiveFloor = globalPositiveMin`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_scene_geometry(owned_levels, level_geometry, options):
    # 1. Validate Inputs
    if owned_levels is None or len(owned_levels) != len(level_geometry):
        raise std.invalid_argument("Invalid input levels")

    scene = Scene()
    scene.owned_levels = owned_levels
    scene.local_boxes.reserve(64)
    
    local_min_original = float('inf')
    local_max_original = float('-inf')
    has_local_boxes = False
    local_scalar_stats = LocalScalarStats() # min=inf, max=-inf, min_pos=inf, count=0

    # 3. Iterate Levels
    for level in range(options.min_level, options.max_level + 1):
        if level < 0 or level >= len(scene.owned_levels):
            continue
        
        mf = owned_levels[level]
        if mf.size() == 0:
            continue

        # 4. Compute Local Scalar Statistics
        level_stats = reduce_local_scalar_stats(mf, options.component)
        local_scalar_stats.merge(level_stats)

        # 5. Iterate Boxes
        for box in mf.iter_boxes():
            lo = box.small_end
            hi = box.big_end
            nx, ny, nz = box.dimensions()
            
            if nx <= 0 or ny <= 0 or nz <= 0:
                continue

            geom = level_geometry[level]
            min_corner = Vec3()
            max_corner = Vec3()
            
            for x in [0, 1, 2]:
                min_corner[x] = geom.prob_lo[x] + lo[x] * geom.cell_size[x]
                max_corner[x] = geom.prob_lo[x] + (hi[x] + 1) * geom.cell_size[x]

            # Update local bounds
            comp_min, comp_max = get_component_min_max(mf, box, options.component)
            local_min_original = min(local_min_original, comp_min)
            local_max_original = max(local_max_original, comp_max)
            
            has_local_boxes = True
            scene.local_boxes.append(AmrBox(
                cell_dimensions=box.dimensions(),
                valid_box=box,
                values=mf.get_ref(),
                component=options.component,
                min_corner=min_corner,
                max_corner=max_corner
            ))

    # Pitfall: Handle empty levels
    if not has_local_boxes:
        local_min_original = float('max') # Implementation specific max
        local_max_original = float('-max')

    # 6. Compute Global Bounding Box
    global_min_original = mpi_allreduce(local_min_original, op=mpi_min)
    global_max_original = mpi_allreduce(local_max_original, op=mpi_max)

    # 7. Compute Global Scale Factor
    global_extent = global_max_original - global_min_original
    min_extent = min([e for e in global_extent if e > 0 and is_finite(e)])
    
    if min_extent <= 0 or not is_finite(min_extent):
        global_scale = 1.0
    else:
        global_scale = 1.0 / min_extent

    if not is_finite(global_scale) or global_scale <= 0:
        global_scale = 1.0

    # 8. Apply Global Scale
    if global_scale != 1.0:
        for box in scene.local_boxes:
            box.min_corner *= global_scale
            box.max_corner *= global_scale
        global_min_original *= global_scale
        global_max_original *= global_scale

    # 9. Compute Final Global Bounding Box
    # Re-gather from scaled boxes if necessary, or use scaled globals
    # Assuming we use the scaled globals for the main bounds
    global_min = global_min_original
    global_max = global_max_original

    # 10. Compute Explicit Bounds
    extent = global_max - global_min
    max_extent = max(extent)
    padding_amount = max_extent * 0.05 if max_extent > 0 else 1.0
    
    scene.explicit_bounds.min_corner = global_min - padding_amount
    scene.explicit_bounds.max_corner = global_max + padding_amount
    scene.has_explicit_bounds = True

    # 11. Gather Scalar Statistics
    if local_scalar_stats.finite_count > 0:
        send_min = local_scalar_stats.min_value
        send_max = local_scalar_stats.max_value
    else:
        send_min = float('inf')
        send_max = float('-inf')

    global_orig_scalar_min = mpi_allreduce(send_min, op=mpi_min)
    global_orig_scalar_max = mpi_allreduce(send_max, op=mpi_max)
    global_orig_finite_count = mpi_allreduce(local_scalar_stats.finite_count, op=mpi_sum)

    # 12. Set Original Scalar Range
    if global_orig_finite_count > 0 and is_finite(global_orig_scalar_min) and is_finite(global_orig_scalar_max):
        scene.original_scalar_range = (global_orig_scalar_min, global_orig_scalar_max)
        scene.has_original_scalar_range = True

    # 13. Logarithmic Scaling
    if options.log_scale_input:
        # Gather positive min and count
        global_pos_min = mpi_allreduce(local_scalar_stats.min_positive, op=mpi_min)
        global_pos_count = mpi_allreduce(1 if local_scalar_stats.min_positive < float('inf') else 0, op=mpi_sum)
        
        # Note: The source logic implies gathering min of positive values. 
        # If no positive values, min_positive remains inf.
        
        if global_pos_count <= 0 or not is_finite(global_pos_min) or global_pos_min <= 0:
            raise std.runtime_error("Log scaling requested but no positive scalar values were found.")

        processed_min = math.log(global_pos_min)
        processed_max = math.log(max(global_orig_scalar_max, global_pos_min))

        scene.processed_scalar_range = (processed_min, processed_max)
        scene.has_processed_scalar_range = True
        scene.scalar_transform.processed_min = processed_min
        scene.scalar_transform.processed_max = processed_max
        
        span = processed_max - processed_min
        if span == 0:
            span = 1.0 # Avoid division by zero
        scene.scalar_transform.inverse_processed_span = 1.0 / span
        scene.scalar_transform.positive_floor = global_pos_min

    # Pitfall: Check for invalid ranges
    if not is_finite(scene.scalar_transform.processed_min) or not is_finite(scene.scalar_transform.processed_max):
        raise std.runtime_error(options.invalid_scalar_error)
        
    if scene.scalar_transform.processed_min == scene.scalar_transform.processed_max:
        scene.scalar_transform.processed_max = scene.scalar_transform.processed_min + 1.0

    # Pitfall: Check for invalid geometry
    if any(global_min[i] > global_max[i] for i in range(3)):
        raise std.runtime_error(options.no_data_error)

    return scene
```

## Subtopics

### [Data Bounds Determination](3997-data-bounds-determination.md)
Data Bounds Determination is the procedure of computing the minimum and maximum scalar values ($v_{min}, v_{max}$) for a volume dataset to enable consistent color mapping and transfer function application. It solves the problem of outlier sensitivity and cross-dataset consistency by selecting a representative scalar range from the raw data. It differs from sibling methods by offering distinct strategies for handling statistical outliers (percentiles) versus global extremes, and by supporting manual overrides for multi-file consistency.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the total number of cells and boxes across all AMR levels, plus $O(P)$ for MPI communication where $P$ is the number of processes.
*   **Memory Complexity:** Linear with respect to the number of boxes stored in `scene.local_boxes`.
*   **Strengths:** Handles distributed data efficiently via MPI; normalizes geometry to prevent numerical issues during ray intersection; provides robust handling of scalar ranges including logarithmic scaling.
*   **Limitations:** Requires all ranks to participate in MPI reductions; sensitive to degenerate boxes (zero dimensions) if not filtered; logarithmic scaling fails if no positive values exist.
*   **When to use:** At the initialization phase of AMR volume rendering, before ray casting begins.
*   **When not to use:** When scalar ranges are known a priori and do not require statistical aggregation, or when running on a single rank without MPI overhead concerns (though the logic still applies).

## Background
This method is part of the `amrVolumeRenderer` project, specifically the scene construction pipeline. It aggregates hierarchical AMR data into a renderable scene representation, handling the geometric and statistical normalization required for subsequent ray marching.

## Sources
[S1] BenWibking/amrVolumeRenderer, https://github.com/BenWibking/amrVolumeRenderer, fetched 2026-10-26 (unofficial)
