# Scene Geometry and Scalar Range Initialization

Also called: BuildSceneGeometry

## Overview
Constructs the scene geometry and scalar ranges for volume rendering by aggregating Adaptive Mesh Refinement (AMR) box data, computing global bounding boxes via MPI, applying normalization scaling, and determining scalar value ranges (original and processed) including optional logarithmic scaling. It differs from sibling methods by focusing on the initialization phase—establishing the coordinate space and data normalization parameters—rather than the topological data structure layout ([TET-AMR Data Structure](1094-tet-amr-data-structure.md)), the ray-casting intersection logic ([Exact AMR Rendering](4399-exact-amr-rendering.md)), or the signal processing of basis functions ([AMR Basis Reconstruction Filter](5581-amr-basis-reconstruction-filter.md)).

## How it works
**Inputs and Outputs**
*   **Inputs**: `ownedLevels` (array of AMR level data, e.g., `MultiFab`), `levelGeometry` (geometry information for each level), `options` (configuration struct containing `minLevel`, `maxLevel`, `component`, `logScaleInput`, `noDataError`, `invalidScalarError`).
*   **Outputs**: A populated `scene` object containing `ownedLevels`, `localBoxes`, `globalMin`/`globalMax`, `explicitBounds`, `originalScalarRange`, `processedScalarRange`, and `scalarTransform`.

**Data Structures**
*   `VolumeRenderer::AmrBox`:
    *   `cellDimensions`: `amrex::IntVect` (dimensions of the box in cells).
    *   `validBox`: `amrex::Box` (valid index range).
    *   `values`: `const array reference` from `MultiFab`.
    *   `component`: `int` (component index).
    *   `minCorner`: `Vec3` (physical minimum corner).
    *   `maxCorner`: `Vec3` (physical maximum corner).
*   `LocalScalarStats`:
    *   `minValue`: `amrex::Real` (initialized to $+\infty$).
    *   `maxValue`: `amrex::Real` (initialized to $-\infty$).
    *   `minPositive`: `amrex::Real` (initialized to $+\infty$).
    *   `finiteCount`: `amrex::Long` (initialized to 0).

**Numbered Steps**

1.  **Validate Inputs**: Throw `std::invalid_argument` if `ownedLevels` is null or if `ownedLevels->size() != levelGeometry.size()` [S1, unofficial].
2.  **Initialize Scene**: Move `ownedLevels` into `scene.ownedLevels`. Reserve 64 slots for `scene.localBoxes`. Initialize `localMinOriginal` and `localMaxOriginal` to $+\infty$ and $-\infty$ respectively. Set `hasLocalBoxes = false`. Initialize `localScalarStats` [S1, unofficial].
3.  **Iterate Levels**: Loop `level` from `options.minLevel` to `options.maxLevel`. Skip if `level < 0` or `level >= scene.ownedLevels->size()` [S1, unofficial].
4.  **Get MultiFab**: Retrieve `MultiFab mf` from `ownedLevels[level]`. Skip if `mf.size() == 0` [S1, unofficial].
5.  **Compute Local Scalar Stats**: Call `reduceLocalScalarStats(mf, options.component)` to compute statistics for the current component and merge results into `localScalarStats` [S1, unofficial].
6.  **Iterate Boxes**: Iterate valid boxes in `mf` using `amrex::MFIter`. For each box:
    *   Get `box`, `lo` (smallEnd), `hi` (bigEnd). Calculate dimensions `nx`, `ny`, `nz`. Skip if any dimension $\le 0$ [S1, unofficial].
    *   Assign `minCorner` and `maxCorner` to `amrBox` using the formula:
        $$ \text{minCorner}[x] = \text{geom.probLo}[x] + \text{lo}[x] \times \text{geom.cellSize}[x] $$
        $$ \text{maxCorner}[x] = \text{geom.probLo}[x] + (\text{hi}[x] + 1) \times \text{geom.cellSize}[x] $$
        for $x \in \{0, 1, 2\}$ [S1, unofficial].
    *   Update `localMinOriginal` and `localMaxOriginal` using `componentMin` and `componentMax` from the box data [S1, unofficial].
    *   Set `hasLocalBoxes = true`. Push `amrBox` to `scene.localBoxes` [S1, unofficial].
7.  **Compute Global Bounding Box**: Gather `localMinOriginal` and `localMaxOriginal` arrays to all ranks using `MPI_Allreduce` with `MPI_MIN` and `MPI_MAX` operations to obtain `globalMinOriginal` and `globalMaxOriginal` [S1, unofficial].
8.  **Compute Global Scale Factor**:
    *   Calculate `globalExtentOriginal` as `globalMaxOriginal - globalMinOriginal`.
    *   Find `minExtent` as the minimum non-zero finite axis length of `globalExtentOriginal` [S1, unofficial].
    *   Set `globalScale = 1.0 / minExtent`. If `minExtent <= 0` or is not finite, set `globalScale = 1.0`. If `globalScale` is not finite or $\le 0$, reset to `1.0` [S1, unofficial].
9.  **Apply Global Scale**: If `globalScale != 1.0`, multiply all box `minCorner`/`maxCorner`, `globalMinOriginal`, and `globalMaxOriginal` by `globalScale` [S1, unofficial].
10. **Compute Final Global Bounding Box**: Compute `globalMin` and `globalMax` from the scaled boxes using `MPI_Allreduce` (`MPI_MIN`/`MPI_MAX`) [S1, unofficial].
11. **Compute Explicit Bounds**:
    *   Calculate `extent = globalMax - globalMin`.
    *   Calculate `maxExtent = max(axis extents)`.
    *   Calculate `paddingAmount = maxExtent * 0.05` if `maxExtent > 0`, else `1.0`.
    *   Set `scene.explicitBounds.minCorner = globalMin - padding` and `scene.explicitBounds.maxCorner = globalMax + padding`.
    *   Set `scene.hasExplicitBounds = true` [S1, unofficial].
12. **Gather Scalar Statistics**:
    *   Send `localOriginalMinSend` and `localOriginalMaxSend` (derived from `localScalarStats` if `finiteCount > 0`) via `MPI_Allreduce` (`MPI_MIN`/`MPI_MAX`) to get `globalOriginalScalarMin` and `globalOriginalScalarMax`.
    *   Sum `finiteCount` via `MPI_SUM` to get `globalOriginalFiniteCount` [S1, unofficial].
13. **Set Original Scalar Range**: If `globalOriginalFiniteCount > 0` and both min/max are finite, set `scene.originalScalarRange = makeScalarRange(globalOriginalScalarMin, globalOriginalScalarMax)` and `scene.hasOriginalScalarRange = true` [S1, unofficial].
14. **Logarithmic Scaling (Optional)**:
    *   If `options.logScaleInput` is true:
        *   Gather `globalPositiveMin` (minimum of positive values) via `MPI_MIN`.
        *   Count `globalPositiveCount` via `MPI_SUM`.
        *   If `globalPositiveCount <= 0` or `globalPositiveMin` is not finite or $\le 0$, throw `std::runtime_error` "Log scaling requested but no positive scalar values were found." [S1, unofficial].
        *   Calculate `processedMin = log(globalPositiveMin)`.
        *   Calculate `processedMax = log(max(globalOriginalScalarMax, globalPositiveMin))`.
        *   Set `scene.processedScalarRange = makeScalarRange(processedMin, processedMax)`.
        *   Set `scene.hasProcessedScalarRange = true`.
        *   Set `scene.scalarTransform.processedMin = processedMin`, `scene.scalarTransform.processedMax = processedMax`, `scene.scalarTransform.inverseProcessedSpan = 1.0 / (processedMax - processedMin)`.
        *   Set `scene.scalarTransform.positiveFloor = globalPositiveMin` [S1, unofficial].

**Pseudocode**

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
    local_scalar_stats = LocalScalarStats()

    # 3. Iterate Levels
    for level in range(options.min_level, options.max_level + 1):
        if level < 0 or level >= len(scene.owned_levels):
            continue
            
        mf = scene.owned_levels[level]
        if mf.size() == 0:
            continue

        # 5. Compute Local Scalar Stats
        reduce_local_scalar_stats(mf, options.component, local_scalar_stats)

        # 6. Iterate Boxes
        for box in mf.valid_boxes():
            lo = box.small_end
            hi = box.big_end
            nx, ny, nz = box.dimensions
            
            if nx <= 0 or ny <= 0 or nz <= 0:
                continue

            geom = level_geometry[level]
            
            # Formula for corners
            min_corner = Vec3(
                geom.prob_lo[0] + lo[0] * geom.cell_size[0],
                geom.prob_lo[1] + lo[1] * geom.cell_size[1],
                geom.prob_lo[2] + lo[2] * geom.cell_size[2]
            )
            max_corner = Vec3(
                geom.prob_lo[0] + (hi[0] + 1) * geom.cell_size[0],
                geom.prob_lo[1] + (hi[1] + 1) * geom.cell_size[1],
                geom.prob_lo[2] + (hi[2] + 1) * geom.cell_size[2]
            )

            # Update local stats
            comp_min = box.component_min(options.component)
            comp_max = box.component_max(options.component)
            local_min_original = min(local_min_original, comp_min)
            local_max_original = max(local_max_original, comp_max)
            
            has_local_boxes = True
            scene.local_boxes.append(AmrBox(
                cell_dimensions=box.dimensions,
                valid_box=box,
                values=box.values,
                component=options.component,
                min_corner=min_corner,
                max_corner=max_corner
            ))

    # 7. Compute Global Bounding Box
    global_min_original = mpi_all_reduce(local_min_original, op=mpi_min)
    global_max_original = mpi_all_reduce(local_max_original, op=mpi_max)

    # 8. Compute Global Scale Factor
    global_extent = global_max_original - global_min_original
    min_extent = min([e for e in global_extent if e > 0 and isfinite(e)])
    
    if min_extent <= 0 or not isfinite(min_extent):
        global_scale = 1.0
    else:
        global_scale = 1.0 / min_extent
        
    if not isfinite(global_scale) or global_scale <= 0:
        global_scale = 1.0

    # 9. Apply Global Scale
    if global_scale != 1.0:
        for box in scene.local_boxes:
            box.min_corner *= global_scale
            box.max_corner *= global_scale
        global_min_original *= global_scale
        global_max_original *= global_scale

    # 10. Compute Final Global Bounding Box
    # Re-gather min/max from scaled boxes
    global_min = mpi_all_reduce_min([b.min_corner for b in scene.local_boxes])
    global_max = mpi_all_reduce_max([b.max_corner for b in scene.local_boxes])

    # 11. Compute Explicit Bounds
    extent = global_max - global_min
    max_extent = max(extent)
    padding_amount = max_extent * 0.05 if max_extent > 0 else 1.0
    
    scene.explicit_bounds.min_corner = global_min - padding_amount
    scene.explicit_bounds.max_corner = global_max + padding_amount
    scene.has_explicit_bounds = True

    # 12. Gather Scalar Statistics
    if local_scalar_stats.finite_count > 0:
        send_min = local_scalar_stats.min_value
        send_max = local_scalar_stats.max_value
    else:
        send_min = float('inf')
        send_max = float('-inf')
        
    global_orig_scalar_min = mpi_all_reduce(send_min, op=mpi_min)
    global_orig_scalar_max = mpi_all_reduce(send_max, op=mpi_max)
    global_orig_finite_count = mpi_all_reduce(local_scalar_stats.finite_count, op=mpi_sum)

    # 13. Set Original Scalar Range
    if global_orig_finite_count > 0 and isfinite(global_orig_scalar_min) and isfinite(global_orig_scalar_max):
        scene.original_scalar_range = make_scalar_range(global_orig_scalar_min, global_orig_scalar_max)
        scene.has_original_scalar_range = True

    # 14. Logarithmic Scaling
    if options.log_scale_input:
        # Gather positive min and count
        global_positive_min = mpi_all_reduce(local_scalar_stats.min_positive, op=mpi_min)
        global_positive_count = mpi_all_reduce(1 if local_scalar_stats.min_positive < float('inf') else 0, op=mpi_sum) # Simplified logic based on source description
        
        # Note: Source implies gathering global positive min specifically. 
        # Assuming a helper or specific reduction for positive values exists.
        # If no positive values found:
        if global_positive_count <= 0 or not isfinite(global_positive_min) or global_positive_min <= 0:
            raise std.runtime_error("Log scaling requested but no positive scalar values were found.")
            
        processed_min = log(global_positive_min)
        processed_max = log(max(global_orig_scalar_max, global_positive_min))
        
        scene.processed_scalar_range = make_scalar_range(processed_min, processed_max)
        scene.has_processed_scalar_range = True
        scene.scalar_transform.processed_min = processed_min
        scene.scalar_transform.processed_max = processed_max
        scene.scalar_transform.inverse_processed_span = 1.0 / (processed_max - processed_min)
        scene.scalar_transform.positive_floor = global_positive_min

    # Pitfalls / Edge Cases
    if not has_local_boxes:
        local_min_original = float('max') # Indicate empty
        local_max_original = float('-max') # Indicate empty
        
    if any(global_min[i] > global_max[i] for i in range(3)):
        raise std.runtime_error(options.no_data_error)
        
    if not isfinite(processed_min) or not isfinite(processed_max):
        raise std.runtime_error(options.invalid_scalar_error)
        
    if processed_min == processed_max:
        processed_max = processed_min + 1.0 # Avoid division by zero

    return scene
```

## Subtopics

### [Data Bounds Determination](3997-data-bounds-determination.md)
A procedure to obtain minimum and maximum data bounds for scaling colors, using one of three methods.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of AMR boxes and levels, $O(N_{boxes} + N_{levels})$, plus MPI communication overhead proportional to the number of ranks.
*   **Memory Complexity**: Linear with respect to the number of boxes stored in `scene.localBoxes`.
*   **Strengths**: Handles distributed memory (MPI) environments robustly. Provides automatic normalization scaling to prevent numerical issues in rendering.
*   **Limitations**: Requires all ranks to participate in reductions. If the domain is empty or degenerate, specific error handling is required.
*   **When to use**: At the start of the rendering pipeline, before ray casting or data transfer.
*   **When not to use**: If the scene geometry is static and pre-computed, or if custom scaling logic is required outside the provided options.

## Background
This method is part of the `amrVolumeRenderer` project by Ben Wibking [S1, unofficial]. It implements the initialization phase for rendering AMR data, ensuring that geometric and scalar data are correctly normalized and distributed across MPI ranks before rendering begins.

## Sources
[S1] BenWibking/amrVolumeRenderer, https://github.com/BenWibking/amrVolumeRenderer, fetched 2026-10-26 (unofficial)
