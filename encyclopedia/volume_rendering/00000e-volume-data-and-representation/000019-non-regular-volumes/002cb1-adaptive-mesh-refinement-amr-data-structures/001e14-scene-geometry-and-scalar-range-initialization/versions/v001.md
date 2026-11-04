# Scene Geometry and Scalar Range Initialization

Also called: BuildSceneGeometry

## Overview
Constructs the scene geometry and scalar ranges for volume rendering by aggregating Adaptive Mesh Refinement (AMR) box data, computing global bounding boxes via MPI, applying normalization scaling, and determining scalar value ranges including optional logarithmic scaling [S1, unofficial]. It differs from sibling methods by explicitly handling the aggregation of distributed AMR level data into a unified scene representation with normalized coordinates and robust scalar range detection, whereas [Exact AMR Rendering](4399-exact-amr-rendering.md) focuses on the ray-casting traversal and [TET-AMR Data Structure](1094-tet-amr-data-structure.md) focuses on tetrahedral mesh construction.

## How it works
**Inputs:**
- `ownedLevels`: A vector of `MultiFab` objects, one per AMR level.
- `levelGeometry`: A vector of `Geometry` objects corresponding to `ownedLevels`.
- `options`: Configuration struct containing:
  - `minLevel`, `maxLevel`: Integer bounds for levels to process.
  - `component`: Integer index of the scalar component in the `MultiFab`.
  - `logScaleInput`: Boolean flag to enable logarithmic scaling.
  - `noDataError`, `invalidScalarError`: Error messages for runtime exceptions.

**Outputs:**
- `scene`: A `VolumeRenderer::Scene` object populated with:
  - `ownedLevels`: Moved from input.
  - `localBoxes`: List of `AmrBox` structures.
  - `globalMin`, `globalMax`: Global bounding box corners.
  - `explicitBounds`: Bounding box with padding.
  - `originalScalarRange`, `processedScalarRange`: Scalar value ranges.
  - `scalarTransform`: Transformation parameters for scalar mapping.

**Data Structures:**

`VolumeRenderer::AmrBox`:
- `cellDimensions`: `amrex::IntVect` (nx, ny, nz).
- `validBox`: `amrex::Box` (logical grid indices).
- `values`: `const array reference` from `MultiFab`.
- `component`: `int`.
- `minCorner`: `Vec3` (physical coordinates).
- `maxCorner`: `Vec3` (physical coordinates).

`LocalScalarStats`:
- `minValue`: `amrex::Real` (initialized to $+\infty$).
- `maxValue`: `amrex::Real` (initialized to $-\infty$).
- `minPositive`: `amrex::Real` (initialized to $+\infty$).
- `finiteCount`: `amrex::Long` (initialized to 0).

**Steps:**

1.  **Validate Inputs:**
    - If `ownedLevels` is null or `ownedLevels->size() != levelGeometry.size()`, throw `std::invalid_argument` [S1, unofficial].

2.  **Initialize Scene:**
    - Move `ownedLevels` into `scene.ownedLevels`.
    - Reserve 64 slots for `scene.localBoxes`.
    - Initialize `localMinOriginal` and `localMaxOriginal` to $\pm \infty$.
    - Set `hasLocalBoxes = false`.
    - Initialize `localScalarStats` [S1, unofficial].

3.  **Iterate Levels:**
    - Loop `level` from `options.minLevel` to `options.maxLevel`.
    - If `level < 0` or `level >= scene.ownedLevels->size()`, skip.
    - Get `MultiFab mf` from `ownedLevels[level]`.
    - If `mf.size() == 0`, skip.

4.  **Compute Local Scalar Statistics:**
    - Call `reduceLocalScalarStats(mf, options.component)` to compute `LocalScalarStats` for the current `MultiFab`.
    - Merge results into `localScalarStats` (sum `finiteCount`, min/max `minValue`, `maxValue`, `minPositive`) [S1, unofficial].
    - *Note:* The GPU reduce kernel checks finiteness. For each value, if finite, it returns `{raw, raw, positive, 1}`; otherwise `{inf, -inf, inf, 0}` [S1, unofficial].

5.  **Iterate Valid Boxes:**
    - Iterate valid boxes in `mf` using `amrex::MFIter`.
    - Get `box`, `lo` (smallEnd), `hi` (bigEnd).
    - Compute dimensions `nx, ny, nz`.
    - If any dimension $\le 0$, skip.
    - Get `geom` from `levelGeometry[level]`.
    - Compute `minCorner` and `maxCorner` using:
      $$ \text{minCorner}[x] = \text{geom.probLo}[x] + \text{lo}[x] \times \text{geom.cellSize}[x] $$
      $$ \text{maxCorner}[x] = \text{geom.probLo}[x] + (\text{hi}[x] + 1) \times \text{geom.cellSize}[x] $$
      for $x \in \{0, 1, 2\}$ [S1, unofficial].
    - Update `localMinOriginal` and `localMaxOriginal` using `componentMin` and `componentMax` from the box values.
    - Set `hasLocalBoxes = true`.
    - Push `AmrBox` (with computed corners) to `scene.localBoxes` [S1, unofficial].

6.  **Compute Global Bounding Box (Original):**
    - Gather `localMinOriginal` and `localMaxOriginal` arrays to all ranks using `MPI_Allreduce` with `MPI_MIN` and `MPI_MAX` respectively.
    - Result: `globalMinOriginal`, `globalMaxOriginal` [S1, unofficial].

7.  **Compute Global Scale Factor:**
    - Compute `globalExtentOriginal` = `globalMaxOriginal` - `globalMinOriginal`.
    - Find `minExtent` as the minimum non-zero finite axis length of `globalExtentOriginal`.
    - If `minExtent <= 0` or not finite, set `globalScale = 1.0`.
    - Else, `globalScale = 1.0 / minExtent`.
    - If `globalScale` is not finite or $\le 0$, reset to `1.0` [S1, unofficial].

8.  **Apply Global Scale:**
    - If `globalScale != 1.0`:
      - Multiply all `box.minCorner` and `box.maxCorner` in `scene.localBoxes` by `globalScale`.
      - Multiply `globalMinOriginal` and `globalMaxOriginal` by `globalScale` [S1, unofficial].

9.  **Compute Final Global Bounding Box:**
    - Compute `globalMin` and `globalMax` from the scaled boxes in `scene.localBoxes` using `MPI_Allreduce` (`MPI_MIN`/`MPI_MAX`) [S1, unofficial].

10. **Compute Explicit Bounds:**
    - `extent` = `globalMax` - `globalMin`.
    - `maxExtent` = max(axis extents).
    - If `maxExtent > 0`, `paddingAmount` = `maxExtent * 0.05`. Else, `paddingAmount` = `1.0`.
    - `scene.explicitBounds.minCorner` = `globalMin` - `paddingAmount`.
    - `scene.explicitBounds.maxCorner` = `globalMax` + `paddingAmount`.
    - `scene.hasExplicitBounds = true` [S1, unofficial].

11. **Gather Scalar Statistics:**
    - Determine `localOriginalMinSend` and `localOriginalMaxSend` based on `localScalarStats.finiteCount > 0`. If `finiteCount == 0`, use neutral values (e.g., 0) or handle via MPI flags [G].
    - Send via `MPI_Allreduce` (`MPI_MIN`/`MPI_MAX`) to get `globalOriginalScalarMin` and `globalOriginalScalarMax`.
    - Sum `finiteCount` via `MPI_SUM` to get `globalOriginalFiniteCount` [S1, unofficial].

12. **Set Original Scalar Range:**
    - If `globalOriginalFiniteCount > 0` and both `globalOriginalScalarMin` and `globalOriginalScalarMax` are finite:
      - `scene.originalScalarRange` = `makeScalarRange(globalOriginalScalarMin, globalOriginalScalarMax)`.
      - `scene.hasOriginalScalarRange = true` [S1, unofficial].

13. **Logarithmic Scaling (Optional):**
    - If `options.logScaleInput` is true:
      - Gather `globalPositiveMin` (min of positive values) via `MPI_MIN`.
      - Count `globalPositiveCount` via `MPI_SUM`.
      - If `globalPositiveCount <= 0` or `globalPositiveMin` not finite or $\le 0$, throw `std::runtime_error` "Log scaling requested but no positive scalar values were found." [S1, unofficial].
      - Compute:
        $$ \text{processedMin} = \log(\text{globalPositiveMin}) $$
        $$ \text{processedMax} = \log(\max(\text{globalOriginalScalarMax}, \text{globalPositiveMin})) $$
      - Set `scene.processedScalarRange` = `makeScalarRange(processedMin, processedMax)`.
      - Set `scene.hasProcessedScalarRange = true`.
      - Set `scene.scalarTransform.processedMin` = `processedMin`.
      - Set `scene.scalarTransform.processedMax` = `processedMax`.
      - Set `scene.scalarTransform.inverseProcessedSpan` = $1.0 / (\text{processedMax} - \text{processedMin})$.
      - Set `scene.scalarTransform.positiveFloor` = `globalPositiveMin` [S1, unofficial].

14. **Edge Cases and Pitfalls:**
    - If `hasLocalBoxes` is false (no boxes processed), reset `localMinOriginal` to `max()` and `localMaxOriginal` to `-max()` to indicate empty range [S1, unofficial].
    - If `globalMin > globalMax` on any axis, throw `std::runtime_error` with `options.noDataError` [S1, unofficial].
    - If `processedMin` or `processedMax` is not finite, throw `std::runtime_error` with `options.invalidScalarError` [S1, unofficial].
    - If `processedMin == processedMax`, set `processedMax = processedMin + 1.0` to avoid division by zero [S1, unofficial].

## Subtopics

### Data Bounds Determination
A procedure to obtain minimum and maximum data bounds for scaling colors, using one of three methods [S2, unofficial]. Method 1: Bounds are set manually by the user to ensure consistent coloring across multiple file types. Method 2: Percentile bounds are used (default 1st and 99th) to handle outliers; Smokeview scans data to compute a histogram and picks bounds at specified percentile levels. Method 3: Global bounds are selected for the entire dataset [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the total number of grid points and boxes across all levels, plus $O(P \log P)$ or $O(P)$ for MPI reductions depending on implementation, where $P$ is the number of processes.
- **Memory Complexity:** Linear with respect to the number of boxes and levels. The `localBoxes` vector stores one `AmrBox` per valid box.
- **Strengths:** Handles distributed AMR data robustly via MPI; normalizes geometry to unit cube; supports logarithmic scaling for data with wide dynamic ranges.
- **Limitations:** Requires all ranks to participate in MPI reductions; logarithmic scaling fails if no positive values exist.
- **When to use:** When preparing AMR volume data for rendering, especially in distributed memory environments.
- **When not to use:** For single-processor, non-AMR data where simpler bounding box computation suffices.

## Background
Originated in the `amrVolumeRenderer` project by Ben Wibking [S1, unofficial]. The method is part of the initialization pipeline for rendering Adaptive Mesh Refinement data, ensuring that geometric and scalar data are properly aggregated and normalized before ray casting. The concept of data bounds determination is also discussed in the context of Smokeview, which uses percentile or global bounds for color mapping [S2, unofficial].

## Sources
[S1] BenWibking/amrVolumeRenderer, https://github.com/BenWibking/amrVolumeRenderer, fetched 2026-10-26 (unofficial)
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
