# Multi-Resolution Extraction

Also called: Multi-resolution isosurface extraction, Multi-resolution scalar field isosurface extraction

## Overview
Multi-resolution extraction is a technique within multi-isosurface rendering that optimizes computational cost by rendering selected isosurfaces at high resolution while rendering others at lower resolutions [S1, unofficial]. It achieves this by skipping voxels during the extraction process for lower-priority isosurfaces [S1, unofficial]. This approach differs from uniform multi-isosurface extraction by introducing variable grid densities per isovalue, trading geometric fidelity for performance on less critical surfaces [S1, unofficial][S2].

## How it works
The method modifies the standard Marching Cubes algorithm by introducing a stride parameter to control the sampling density of the input scalar field for each specific isovalue [S1, unofficial].

### Inputs and Outputs
*   **Input:** A 3D scalar field `dataset` of dimensions `maxX x maxY x maxZ`. A list of isovalue configurations, where each configuration specifies an `isoValue` and a resolution step `inc` [S1, unofficial].
*   **Output:** A set of triangle meshes (lists of vertices and indices) for each isovalue [S1, unofficial].

### Data Structures
The core data structure is the grid cell `v` processed by the Marching Cubes algorithm. For a cell starting at `(i, j, k)` with step `inc`, the 8 vertices are defined as:
*   `v.p[0] = (i, j, k)`
*   `v.p[1] = (i + inc, j, k)`
*   `v.p[2] = (i + inc, j + inc, k)`
*   `v.p[3] = (i, j + inc, k)`
*   `v.p[4] = (i, j, k + inc)`
*   `v.p[5] = (i + inc, j, k + inc)`
*   `v.p[6] = (i + inc, j + inc, k + inc)`
*   `v.p[7] = (i, j + inc, k + inc)`

The values `v.val[m]` are sampled from `dataset` at these coordinates [S1, unofficial].

### Algorithm Steps
1.  **Initialize:** For each isovalue configuration, set the iteration step `inc`. `inc = 1` for high resolution; `inc > 1` (e.g., 2, 4) for low resolution [S1, unofficial].
2.  **Iterate:** Loop through the volume grid with stride `inc`:
    ```python
    for i from 0 to maxX - inc step inc:
        for j from 0 to maxY - inc step inc:
            for k from 0 to maxZ - inc step inc:
                # Sample 8 corners of the current grid cell
                v.val[0] = dataset[i][j][k]
                v.val[1] = dataset[i + inc][j][k]
                v.val[2] = dataset[i + inc][j + inc][k]
                v.val[3] = dataset[i][j + inc][k]
                v.val[4] = dataset[i][j][k + inc]
                v.val[5] = dataset[i + inc][j][k + inc]
                v.val[6] = dataset[i + inc][j + inc][k + inc]
                v.val[7] = dataset[i][j + inc][k + inc]

                # Compute vertex positions (scaled by inc)
                v.p[0] = (i, j, k)
                v.p[1] = (i + inc, j, k)
                v.p[2] = (i + inc, j + inc, k)
                v.p[3] = (i, j + inc, k)
                v.p[4] = (i, j, k + inc)
                v.p[5] = (i + inc, j, k + inc)
                v.p[6] = (i + inc, j + inc, k + inc)
                v.p[7] = (i, j + inc, k + inc)

                # Perform Marching Cubes lookup and triangulation
                # using v.val and v.p relative to isoValue
                triangles = MarchingCubesStep(v, isoValue)
                add_to_mesh(triangles)
    ```
3.  **Refine (Variant):** In the Multi-resolution Scalar Field variant, construct a multi-resolution scalar field to handle ambiguities between different resolutions [S2]. Splice contours between high and low-resolution fields to remove gaps, keeping high-resolution contours uniform and using unique triangular patches for low-resolution areas [S2].

### Parameters
*   **inc:** Integer step size. `inc=1` is full resolution. `inc>1` reduces resolution by processing every `inc`-th voxel [S1, unofficial].

### Edge Cases and Pitfalls
*   **Boundary Conditions:** The loops must terminate at `maxX - inc`, `maxY - inc`, and `maxZ - inc` to ensure all 8 vertices of the grid cell exist within the dataset bounds [S1, unofficial].
*   **Gaps and Ambiguities:** Simple voxel skipping can create visual gaps or ambiguities where high and low-resolution regions meet. The variant in [S2] addresses this via contour splicing, but basic implementation may exhibit artifacts at resolution boundaries [S2].

## Subtopics

### Multi-resolution Scalar Field Isosurface Extraction
An isosurface extraction method based on multi-resolution scalar field construction and seamless intersecting surfaces [S2]. It improves efficiency by reducing extraction data while maintaining rendering quality [S2]. It constructs a multi-resolution scalar field to address ambiguity and gaps between different resolution fields [S2]. It splices contours between scalar fields of different resolutions to remove ambiguities, keeping high-resolution contours uniform and extracting low-resolution fields with unique triangular patches [S2].

## Operating characteristics
*   **Time Complexity:** For a volume of size $N^3$ and step `inc`, the complexity is $O((N/inc)^3)$ per isovalue [S1, unofficial].
*   **Memory Complexity:** Depends on the output mesh size. The input scalar field is typically accessed in place or cached [S1, unofficial].
*   **Strengths:** Significantly reduces rendering time for non-critical isosurfaces [S1, unofficial][S2].
*   **Limitations:** Reduced geometric fidelity for low-resolution isosurfaces; potential for visual artifacts (gaps) at resolution boundaries if not handled by splicing [S2].
*   **When to use:** When some isosurfaces require high detail while others are secondary or less sensitive to geometric precision [S1, unofficial].

## Background
The basic multi-resolution extraction scheme was implemented in an educational data visualization project by LSU-7433, providing interactive options for selecting isovalue resolutions [S1, S1, unofficial]. The concept of multi-resolution scalar field construction for seamless isosurface extraction was proposed by Ronghuan et al. to address efficiency and artifact issues in multi-resolution rendering [S2].

## Sources
[S1] LSU-7433-Data-Visiualization/multiple_isosurface, https://github.com/LSU-7433-Data-Visiualization/multiple_isosurface, fetched 2026-10-27 (unofficial)
[S2] Evaluation of Manifold Dual Contouring Algorithms Based on K-d Tree and Octree Data Structures, https://doi.org/10.31449/inf.v48i3.5420, fetched 2026-10-26
