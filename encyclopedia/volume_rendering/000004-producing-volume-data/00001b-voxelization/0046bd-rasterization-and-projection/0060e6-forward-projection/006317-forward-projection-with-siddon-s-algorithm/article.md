# Forward Projection with Siddon's Algorithm

Also called: ForwardProjectorByBinUsingRayTracing, Ray Tracing (in context of forward projection), Siddon's algorithm for ray tracing

## Overview
Forward Projection with Siddon's Algorithm computes the length of intersection between rays and voxels to project volume data into sinogram bins [S1]. It distinguishes itself from sibling methods like [Robust SIRT](18352-robust-sirt.md) by explicitly calculating geometric ray-voxel intersections rather than relying on iterative reconstruction or simplified binning approximations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume represented as `DiscretisedDensity<3, float>` with voxel data on a Cartesian grid (`VoxelsOnCartesianGrid`) [S1].
*   **Input:** Projection data information of type `ProjDataInfoCylindrical` defining the bin geometry [S1].
*   **Output:** Projection values accumulated into bins based on the computed intersection lengths [S1].

**Data Structures:**
*   **Bin:** Stores coordinates and value of a single projection bin [S1].
*   **Line of Interest (LOR):** A ray segment used for tracing. The implementation traces a maximum of 3 LORs per bin [S1].

**Algorithm Steps:**
1.  **Initialization:** Store geometric information from the projection data info and the density image info [S1].
2.  **Ray Tracing:** For each projection bin, determine the corresponding ray path.
3.  **Intersection Calculation:** Use Siddon's algorithm to compute the length of intersection between the ray and each voxel it traverses [S1].
4.  **LOI Handling:**
    *   If the z-voxel size is exactly twice the sampling in the axial direction, multiple Lines of Interest (LORs) are used to avoid missing voxels [S1].
    *   The LOI values are divided by `voxel_size.x()` by default. If the preprocessor macro `NEWSCALE` is defined during compilation, this division is skipped [S1].
5.  **Accumulation:** Add the computed intersection lengths to the corresponding bin values [S1].

**Formulas and Parameters:**
*   **Antisymmetry Constraint:** The implementation assumes the s-coordinate is antisymmetric with respect to `tang_pos_num`:
    $$ s(\text{Bin}(\dots, \text{tang\_pos\_num})) = -s(\text{Bin}(\dots, -\text{tang\_pos\_num})) $$
    [S1]
*   **FOV Radius:** The implementation uses a circular Field of View (FOV) with a radius approximately 1 voxel smaller than the maximum possible [S1].

**Pitfalls and Edge Cases:**
*   **Voxel Size Assumptions:** The implementation assumes x and y voxel sizes are at least as large as the sampling in the tangential direction [S1].
*   **Axial Sampling:** The z-voxel size must be either equal to or exactly twice the sampling in the axial direction of the segments [S1].
*   **Data Type Requirements:** The input image must be of type `VoxelsOnCartesianGrid`, and the projection data info must be of type `ProjDataInfoCylindrical` [S1].
*   **FOV Boundary:** Rays near the edge of the circular FOV may be clipped or handled with reduced precision due to the ~1 voxel margin [S1].

## Subtopics

### [3D Ray Tracing (3DRT)](19579-3d-ray-tracing-3drt.md)
Accounts for non-telecentricity and finite etendue by simulating optical system geometry, unlike the idealized parallel projection paths of Siddon's algorithm.

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Provides accurate geometric intersection lengths for cylindrical projection data [S1].
*   **Limitations:** Restricted to cylindrical projection data info and Cartesian grid voxels; requires specific voxel size ratios for axial sampling [S1].

## Background
This method is implemented in the STIR library as the class `stir::ForwardProjectorByBinUsingRayTracing` [S1]. It is part of the reconstruction building blocks for projection building blocks [S1].

## Sources
[S1] ForwardProjectorByBinUsingRay..., https://stir.sourceforge.net/documentation/doxy/html/classstir_1_1ForwardProjectorByBinUsingRayTracing.html, fetched 2026-10-25
