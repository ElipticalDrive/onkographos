# Scene Geometry and Scalar Range Initialization

Also called: BuildSceneGeometry

## Overview
Constructs the scene geometry and scalar ranges for volume rendering by aggregating Adaptive Mesh Refinement (AMR) box data, computing global bounding boxes via MPI, applying normalization scaling, and determining scalar value ranges (original and processed) including optional logarithmic scaling. It differs from sibling methods by focusing on the pre-processing phase of establishing the spatial and data domain boundaries required for subsequent ray-casting, rather than the traversal or shading itself.

## How it works
**Inputs:**
- `ownedLevels`: A vector of AMR level data (e.g., `MultiFab` objects in AMReX).
- `options`: Configuration struct containing `minLevel`, `maxLevel`, `component` (scalar component index), `logScaleInput` (boolean), `noDataError` (string), and `invalidScalarError` (string).

**Outputs:**
- `scene`: A structure populated with:
  - `ownedLevels`: Moved from input.
  - `localBoxes`: Vector of `AmrBox` structures.
  - `globalMinOriginal`, `globalMaxOriginal`: Global bounding box in original coordinates.
  - `globalMin`, `globalMax`: Global bounding box after scaling.
  - `explicitBounds`: Bounding box with padding.
  - `originalScalarRange`: Min/max of original scalar values.
  - `processedScalarRange`: Min/max of processed (e.g., log-scaled) scalar values.
  - `scalarTransform`: Transformation parameters including `processedMin`, `processedMax`, `inverseProcessedSpan`, and `positiveFloor`.

**Data Structures:**

1.  **`VolumeRenderer::AmrBox`**:
    -   `cellDimensions`: `amrex::IntVect` (integer vector of cell counts).
    -   `validBox`: `amrex::Box` (logical box definition).
    -   `values`: `const array reference` from `MultiFab`.
    -   `component`: `int` (scalar component index).
    -   `minCorner`: `Vec3` (physical minimum corner).
    -   `maxCorner`: `Vec3` (physical maximum corner).

2.  **`LocalScalarStats`**:
    -   `minValue`: `amrex::Real` (initialized to $+\infty$).
    -   `maxValue`: `amrex::Real` (initialized to $-\infty$).
    -   `minPositive`: `amrex::Real` (initialized to $+\infty$).
    -   `finiteCount`: `amrex::Long` (initialized to 0).

**Steps:**

1.  **Validate Inputs**:
    -   If `ownedLevels` is null or `ownedLevels->size() != levelGeometry.size()`, throw `std::invalid_argument`.

2.  **Initialize Scene**:
    -   Move `ownedLevels` into `scene.ownedLevels`.
    -   Reserve 64 slots for `scene.localBoxes`.
    -   Initialize `localMinOriginal` and `localMaxOriginal` to $\pm \infty$.
    -   Set `hasLocalBoxes = false`.
    -   Initialize `localScalarStats`.

3.  **Iterate Levels**:
    -   Loop `level` from `options.minLevel` to `options.maxLevel`.
    -   If `level < 0` or `level >= scene.ownedLevels->size()`, skip.
    -   Get `MultiFab mf` from `ownedLevels[level]`.
    -   If `mf.size() == 0`, skip.

4.  **Compute Local Scalar Statistics**:
    -   Call `reduceLocalScalarStats(mf, options.component)` to compute `localScalarStats` for the current level.
    -   Merge results into the global `localScalarStats` accumulator.

5.  **Iterate Valid Boxes**:
    -   Iterate valid boxes in `mf` using `amrex::MFIter`.
    -   Get `box`, `lo` (smallEnd), `hi` (bigEnd), and dimensions `nx, ny, nz`.
    -   If any dimension $\le 0$, skip.
    -   Assign `minCorner` and `maxCorner` to `amrBox` using the formula:
        $$ \text{minCorner}[x] = \text{geom.probLo}[x] + \text{lo}[x] \times \text{geom.cellSize}[x] $$
        $$ \text{maxCorner}[x] = \text{geom.probLo}[x] + (\text{hi}[x] + 1) \times \text{geom.cellSize}[x] $$
        for $x \in \{0, 1, 2\}$.
    -   Update `localMinOriginal` and `localMaxOriginal` using `componentMin` and `componentMax` from the box values.
    -   Set `hasLocalBoxes = true`.
    -   Push `amrBox` to `scene.localBoxes`.

6.  **Compute Global Bounding Box (Original)**:
    -   Gather `localMinOriginal` and `localMaxOriginal` arrays to all ranks using `MPI_Allreduce` with `MPI_MIN` and `MPI_MAX` to obtain `globalMinOriginal` and `globalMaxOriginal`.

7.  **Compute Global Scale Factor**:
    -   Calculate `globalExtentOriginal` as `globalMaxOriginal - globalMinOriginal`.
    -   Find `minExtent` as the minimum non-zero finite axis length of `globalExtentOriginal`.
    -   Set `globalScale = 1.0 / minExtent`.
    -   If `minExtent <= 0` or is not finite, set `globalScale = 1.0`.
    -   If `globalScale` is not finite or $\le 0$, reset `globalScale = 1.0`.

8.  **Apply Global Scale**:
    -   If `globalScale != 1.0`, multiply all box `minCorner`/`maxCorner`, `globalMinOriginal`, and `globalMaxOriginal` by `globalScale`.

9.  **Compute Final Global Bounding Box**:
    -   Compute `globalMin` and `globalMax` from the scaled boxes using `MPI_Allreduce` (`MPI_MIN`/`MPI_MAX`).

10. **Compute Explicit Bounds**:
    -   Calculate `extent = globalMax - globalMin`.
    -   Find `maxExtent` as the maximum of the axis extents.
    -   Calculate `paddingAmount`:
        $$ \text{paddingAmount} = \begin{cases} \text{maxExtent} \times 0.05 & \text{if } \text{maxExtent} > 0 \\ 1.0 & \text{otherwise} \end{cases} $$
    -   Set `scene.explicitBounds.minCorner = globalMin - paddingAmount`.
    -   Set `scene.explicitBounds.maxCorner = globalMax + paddingAmount`.
    -   Set `scene.hasExplicitBounds = true`.

11. **Gather Scalar Statistics**:
    -   Send `localOriginalMinSend` and `localOriginalMaxSend` (derived from `localScalarStats` if `finiteCount > 0`) via `MPI_Allreduce` (`MPI_MIN`/`MPI_MAX`) to get `globalOriginalScalarMin` and `globalOriginalScalarMax`.
    -   Sum `finiteCount` via `MPI_SUM` to get `globalOriginalFiniteCount`.

12. **Set Original Scalar Range**:
    -   If `globalOriginalFiniteCount > 0` and both min/max are finite, set `scene.originalScalarRange = makeScalarRange(globalOriginalScalarMin, globalOriginalScalarMax)` and `scene.hasOriginalScalarRange = true`.

13. **Handle Logarithmic Scaling (if `options.logScaleInput` is true)**:
    -   Gather `globalPositiveMin` (minimum of positive values) via `MPI_MIN`.
    -   Count `globalPositiveCount` via `MPI_SUM`.
    -   If `globalPositiveCount <= 0` or `globalPositiveMin` is not finite or $\le 0$, throw `std::runtime_error` "Log scaling requested but no positive scalar values were found."
    -   Calculate processed range:
        $$ \text{processedMin} = \log(\text{globalPositiveMin}) $$
        $$ \text{processedMax} = \log(\max(\text{globalOriginalScalarMax}, \text{globalPositiveMin})) $$
    -   Set `scene.processedScalarRange`, `scene.hasProcessedScalarRange`, `scene.scalarTransform.processedMin`, `scene.scalarTransform.processedMax`, and `scene.scalarTransform.inverseProcessedSpan = 1.0 / (\text{processedMax} - \text{processedMin})`.
    -   Set `scene.scalarTransform.positiveFloor = globalPositiveMin`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_scene_geometry(owned_levels, level_geometry, options):
    # 1. Validate Inputs
    if owned_levels is None or len(owned_levels) != len(level_geometry):
        raise std.invalid_argument("Invalid level geometry")

    # 2. Initialize Scene
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

        # 4. Compute Local Scalar Statistics
        level_stats = reduce_local_scalar_stats(mf, options.component)
        local_scalar_stats.merge(level_stats)

        # 5. Iterate Valid Boxes
        for box in mf.valid_boxes():
            lo = box.small_end
            hi = box.big_end
            nx, ny, nz = box.dimensions()
            
            if nx <= 0 or ny <= 0 or nz <= 0:
                continue

            # Calculate physical corners
            min_corner = Vec3()
            max_corner = Vec3()
            for x in [0, 1, 2]:
                min_corner[x] = geom.prob_lo[x] + lo[x] * geom.cell_size[x]
                max_corner[x] = geom.prob_lo[x] + (hi[x] + 1) * geom.cell_size[x]

            # Update scalar stats
            comp_min, comp_max = get_component_min_max(mf, box, options.component)
            local_min_original = min(local_min_original, comp_min)
            local_max_original = max(local_max_original, comp_max)
            
            has_local_boxes = True
            amr_box = AmrBox(
                cell_dimensions=box.dimensions(),
                valid_box=box,
                values=mf.values(),
                component=options.component,
                min_corner=min_corner,
                max_corner=max_corner
            )
            scene.local_boxes.push_back(amr_box)

    # Pitfall: Handle empty ranges
    if not has_local_boxes:
        local_min_original = float('max')
        local_max_original = float('-max')

    # 6. Compute Global Bounding Box (Original)
    global_min_original = mpi_allreduce(local_min_original, op=mpi_min)
    global_max_original = mpi_allreduce(local_max_original, op=mpi_max)

    # Pitfall: Check for invalid global bounds
    if any(global_min_original[i] > global_max_original[i] for i in [0, 1, 2]):
        raise std.runtime_error(options.no_data_error)

    # 7. Compute Global Scale Factor
    global_extent_original = global_max_original - global_min_original
    min_extent = min([e for e in global_extent_original if e > 0 and isfinite(e)])
    
    if min_extent <= 0 or not isfinite(min_extent):
        global_scale = 1.0
    else:
        global_scale = 1.0 / min_extent

    if not isfinite(global_scale) or global_scale <= 0:
        global_scale = 1.0

    # 8. Apply Global Scale
    if global_scale != 1.0:
        for box in scene.local_boxes:
            box.min_corner *= global_scale
            box.max_corner *= global_scale
        global_min_original *= global_scale
        global_max_original *= global_scale

    # 9. Compute Final Global Bounding Box
    # Re-gather scaled bounds if necessary, or use the scaled global_min/max directly
    # Assuming we re-gather to ensure consistency across ranks for the final box
    global_min = mpi_allreduce(scene.local_boxes[0].min_corner if scene.local_boxes else Vec3(), op=mpi_min)
    global_max = mpi_allreduce(scene.local_boxes[0].max_corner if scene.local_boxes else Vec3(), op=mpi_max)
    
    # Note: The source implies computing final global min/max from scaled boxes.
    # In a distributed setting, this requires gathering all scaled corners or using the 
    # aggregated local mins/maxs scaled.
    if scene.local_boxes:
        global_min = mpi_allreduce(local_min_original * global_scale, op=mpi_min)
        global_max = mpi_allreduce(local_max_original * global_scale, op=mpi_max)

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

    global_original_scalar_min = mpi_allreduce(send_min, op=mpi_min)
    global_original_scalar_max = mpi_allreduce(send_max, op=mpi_max)
    global_original_finite_count = mpi_allreduce(local_scalar_stats.finite_count, op=mpi_sum)

    # 12. Set Original Scalar Range
    if global_original_finite_count > 0 and isfinite(global_original_scalar_min) and isfinite(global_original_scalar_max):
        scene.original_scalar_range = make_scalar_range(global_original_scalar_min, global_original_scalar_max)
        scene.has_original_scalar_range = True

    # 13. Handle Logarithmic Scaling
    if options.log_scale_input:
        # Gather positive min and count
        global_positive_min = mpi_allreduce(local_scalar_stats.min_positive, op=mpi_min)
        global_positive_count = mpi_allreduce(1 if local_scalar_stats.min_positive < float('inf') else 0, op=mpi_sum)
        
        if global_positive_count <= 0 or not isfinite(global_positive_min) or global_positive_min <= 0:
            raise std.runtime_error("Log scaling requested but no positive scalar values were found.")

        processed_min = log(global_positive_min)
        processed_max = log(max(global_original_scalar_max, global_positive_min))

        # Pitfall: Avoid division by zero
        if processed_min == processed_max:
            processed_max = processed_min + 1.0

        # Pitfall: Check finiteness
        if not isfinite(processed_min) or not isfinite(processed_max):
            raise std.runtime_error(options.invalid_scalar_error)

        scene.processed_scalar_range = make_scalar_range(processed_min, processed_max)
        scene.has_processed_scalar_range = True
        scene.scalar_transform.processed_min = processed_min
        scene.scalar_transform.processed_max = processed_max
        scene.scalar_transform.inverse_processed_span = 1.0 / (processed_max - processed_min)
        scene.scalar_transform.positive_floor = global_positive_min

    return scene
```

## Subtopics

### Data Bounds Determination
A procedure to obtain minimum and maximum data bounds for scaling colors, using one of three methods: manual user setting for consistency across file types, percentile bounds (default 1st and 99th) to handle outliers by scanning data for a histogram, or global bounds for the entire dataset [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of cells and boxes in the AMR hierarchy, plus $O(P \log P)$ or $O(P)$ for MPI reduction depending on the implementation of `MPI_Allreduce`, where $P$ is the number of processes.
- **Memory Complexity**: Proportional to the number of AMR boxes and the size of the scalar statistics arrays. The `localBoxes` vector stores one entry per valid box.
- **Strengths**: Handles distributed memory (MPI) environments efficiently. Provides robust handling of edge cases like empty data or non-positive values for log scaling.
- **Limitations**: Requires a full pass over the data to compute statistics. The global bounding box calculation assumes uniform scaling across all ranks.
- **When to use**: At the initialization phase of AMR volume rendering, before ray casting begins.
- **When not to use**: If dynamic scalar ranges are required per frame without recomputing global statistics, or if the dataset is too large to fit the box list in memory (though this is rare for AMR).

## Background
Originated in the `amrVolumeRenderer` project by Ben Wibking [S1, unofficial]. It is part of the broader effort to render Adaptive Mesh Refinement data, which requires handling unstructured topologies and adaptive resolution [G]. The method ensures that the rendering engine has a consistent coordinate system and scalar mapping before visualization.

## Sources
[S1] BenWibking/amrVolumeRenderer, https://github.com/BenWibking/amrVolumeRenderer, fetched 2026-10-26 (unofficial)
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
