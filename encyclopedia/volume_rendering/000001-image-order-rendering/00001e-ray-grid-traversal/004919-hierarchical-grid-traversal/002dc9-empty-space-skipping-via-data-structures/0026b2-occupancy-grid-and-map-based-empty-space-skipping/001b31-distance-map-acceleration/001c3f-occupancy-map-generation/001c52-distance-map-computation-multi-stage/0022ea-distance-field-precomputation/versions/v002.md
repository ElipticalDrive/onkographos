# Distance Field Precomputation

Also called: Distance Field Computation, Precomputed Distance Field Generation

## Overview
Distance Field Precomputation calculates distance fields for scalar fields defining fiber surfaces to enable fast intersection detection via the rule of signs [S1]. It builds on [Distance Map Computation (Multi-Stage)](1205-distance-map-computation-multi-stage.md) [G]. Unlike [Empty Space Map Generation](1771-empty-space-map-generation.md) which tracks occupancy, or [Distance Transformation (Proximity Clouds)](5720-distance-transformation-proximity-clouds.md) which uses proximity clouds, this method targets geometric boundaries of fiber surfaces for ray-tracing acceleration [S1].

## How it works
**Inputs and Outputs:**
*   Input: Scalar fields defining fiber surfaces (e.g., FSCP) [S1].
*   Output: A precomputed 2D distance field array [S1].

**Data Structures:**
*   Distance Field Array: A 2D grid of size $64 \times 64$ [S1].

**Steps:**
1.  Identify the relevant scalar fields defining the fiber surfaces [S1].
2.  Compute the distance field for these fields [S1]. The computation time is fixed and depends only on the distance field dimensions [S1].
3.  Store the result in the $64 \times 64$ array for use during ray traversal [S1].

**Parameters:**
*   Distance field array size: $64 \times 64$ [S1].
*   Typical performance: 1–2 milliseconds for 4 FSCP with 4 segments per polygon [S1].

**Edge Cases and Pitfalls:**
*   Precomputation time scales linearly with the total number of voxels, but the distance field update itself is fixed time [S1].
*   Smoother data and FSCPs with more empty space allow coarser macrocells, but the distance field size remains fixed at $64 \times 64$ for accuracy [S1].

## Subtopics

### [Fast Marching Method](17009-fast-marching-method.md)
Computes the distance field by solving the Eikonal equation to determine the arrival time of a front expanding in the normal direction from a given boundary condition. It respects causality by expanding outwards where the speed function $F > 0$, ensuring the solution at any grid point depends only on points with earlier arrival times.

## Operating characteristics
*   **Time Complexity:** Distance field updating time is fixed and depends only on the distance field dimensions [S1]. Total precomputation time (including macrocell updates) depends linearly on the total number of voxels [S1].
*   **Performance:** For 4 FSCP with 4 segments, computation takes 1–2 milliseconds [S1]. It is not a bottleneck [S1].
*   **Memory:** Requires storage for the $64 \times 64$ distance field array [S1].
*   **Strengths:** Facilitates fast intersection detection via the rule of signs [S1]. Preserves interactivity when exploring larger datasets with coarser macrocells [S1].
*   **Limitations:** The source does not specify memory complexity beyond the array size.

## Background
Originated in the context of Direct Multifield Volume Ray Casting of Fiber Surfaces [S1].

## Sources
[S1] Direct Multifield Volume Ray Casting of Fiber Surfaces, https://doi.org/10.1109/tvcg.2016.2599040, fetched 2026-10-26
