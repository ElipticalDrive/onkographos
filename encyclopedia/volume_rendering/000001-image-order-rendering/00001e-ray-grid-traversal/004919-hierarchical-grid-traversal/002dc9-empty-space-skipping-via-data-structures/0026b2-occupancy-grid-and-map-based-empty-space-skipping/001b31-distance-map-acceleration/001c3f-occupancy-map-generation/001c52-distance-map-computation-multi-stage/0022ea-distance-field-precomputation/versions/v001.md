# Distance Field Precomputation

Also called: Distance Field Computation, Precomputed Distance Field Generation

## Overview
Distance Field Precomputation involves calculating distance fields for scalar fields involved in fiber surface definitions to facilitate fast intersection detection via the rule of signs [S1]. It builds on [Distance Map Computation (Multi-Stage)](1205-distance-map-computation-multi-stage.md). Unlike [Empty Space Map Generation](1771-empty-space-map-generation.md) which focuses on occupancy, or [Distance Transformation (Proximity Clouds)](5720-distance-transformation-proximity-clouds.md) which may use proximity clouds, this method specifically targets the geometric boundaries of fiber surfaces for ray-tracing acceleration [S1].

## How it works
**Inputs and Outputs:**
*   Input: Scalar fields defining fiber surfaces (e.g., FSCP - Fiber Surface Control Points).
*   Output: A precomputed 2D distance field array.

**Data Structures:**
*   Distance Field Array: A 2D grid. The source specifies a size of $64 \times 64$ [S1].

**Steps:**
1.  Identify the relevant scalar fields defining the fiber surfaces [S1].
2.  Compute the distance field for these fields. The computation time is fixed and depends only on the distance field dimensions [S1].
3.  Store the result in the $64 \times 64$ array for use during ray traversal [S1].

**Parameters:**
*   Distance field array size: $64 \times 64$ [S1].
*   Typical performance: For 4 FSCP with 4 segments per polygon, computation takes 1–2 milliseconds [S1].

**Edge Cases and Pitfalls:**
*   The source notes that smoother data and FSCPs with more empty space allow for coarser macrocells, but the distance field size remains fixed at $64 \times 64$ for accuracy [S1].
*   Precomputation time is linear with respect to the total number of voxels, but the distance field update itself is fixed time [S1].

## Subtopics

### Fast Marching Method
Computes the distance field by solving the Eikonal equation to determine the arrival time of a front expanding in the normal direction from a given boundary condition [S2]. Proposed first by Tsitsiklis, then by Sethian [S2].

## Operating characteristics
*   **Time Complexity:** The time for distance field updating is fixed and depends only on the size of the distance field dimensions [S1]. The total precomputation time (including macrocell updates) depends linearly on the total number of voxels [S1].
*   **Performance:** For 4 FSCP with 4 segments, it takes around 1 to 2 milliseconds [S1]. It is not a bottleneck [S1].
*   **Memory:** Requires storage for the $64 \times 64$ distance field array [S1].
*   **Strengths:** Facilitates fast intersection detection via the rule of signs [S1]. Preserves interactivity when exploring larger datasets with coarser macrocells [S1].
*   **Limitations:** The source does not specify memory complexity beyond the array size.

## Background
Originated in the context of Direct Multifield Volume Ray Casting of Fiber Surfaces [S1]. The Fast Marching Method variant was proposed first by Tsitsiklis and then by Sethian [S2].

## Sources
[S1] Direct Multifield Volume Ray Casting of Fiber Surfaces, https://doi.org/10.1109/tvcg.2016.2599040, fetched 2026-10-26
[S2] Distance Field based Haptic Rendering of Scattered Oriented Points, https://doi.org/10.5120/9937-4581, fetched 2026-10-26
