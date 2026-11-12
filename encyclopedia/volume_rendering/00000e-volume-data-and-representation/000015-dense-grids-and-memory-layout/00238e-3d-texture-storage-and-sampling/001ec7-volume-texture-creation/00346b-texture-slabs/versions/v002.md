# Texture Slabs
## Overview
Texture Slabs is a volume rendering technique that loads 3D density data into graphics card texture map memory, treating the volume as a series of slabs between slices [S1, S2, unofficial]. It differs from standard ray casting by enabling pre-integration of the volume rendering integral, which allows efficient accumulation of color and opacity along rays [S1, S2, unofficial].

## How it works
**Inputs:**
*   3D density data (e.g., from a CAT scan) [S1, unofficial].
*   3D color field $C(x,y,z)$ [S1, unofficial].
*   3D opacity field $\alpha(x,y,z)$ [S1, unofficial].

**Outputs:**
*   Rendered image with color $C$ and opacity $\alpha$ [S1, unofficial].

**Data Structures:**
*   The volume is stored in texture map memory [S1, S2, unofficial].
*   The volume is conceptually divided into slabs between each pair of volume rendering slices [S1, S2, unofficial].

**Method:**
1.  Load the 3D volume data into the graphics card's texture map memory [S1, S2, unofficial].
2.  Treat the volume as a series of slabs located between adjacent slices [S1, S2, unofficial].
3.  Perform rendering by accumulating information along a 1-dimensional line through the volume [S1, unofficial].
4.  Utilize the slab structure to enable pre-integration of the volume rendering integral [S1, S2, unofficial].

**Formulas:**
The rendering process accumulates attenuated information along the ray. The cumulative attenuated info $B$ along a ray is defined as [S1, S2, unofficial]:

$$ B = \int_{I}^{D} D (\cos \theta) e^{-\tau \int_{D} ds} dt $$

Where:
*   $B$: Cumulative attenuated information along the ray [S1, S2, unofficial].
*   $D$: Density function [S1, S2, unofficial].
*   $\theta$: Angle between $I$ and $E$ at each voxel on the ray [S1, S2, unofficial].
*   $\tau$: Decay constant [S1, S2, unofficial].
*   $\int_{D} ds$: Accumulated densities between the voxel and the light source [S1, S2, unofficial].
*   $I$: Light source direction [S1, S2, unofficial].
*   $E$: Direction vector [S1, S2, unofficial].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   The method is view-dependent [S1, S2, unofficial].
*   It uses all data instead of just some [S1, S2, unofficial].
*   The appearance can range from fuzzy to sharp [S1, S2, unofficial].
*   No thresholding is needed [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires texture map memory on the graphics card [S1, S2, unofficial].
*   **Strengths:** Allows pre-integration of the volume rendering integral [S1, S2, unofficial]; uses all data [S1, S2, unofficial]; no thresholding needed [S1, S2, unofficial].
*   **Limitations:** View-dependent [S1, S2, unofficial].
*   **When to use:** When pre-integration is desired and texture memory is available [S1, S2, unofficial].
*   **When not:** When memory is constrained or non-view-dependent methods are preferred [G].

## Background
Texture Slabs is categorized as an "other method" in early direct volume rendering literature, distinct from the two general methods of Image Order (ray casting) and Object Order (splatting) [S1, S2, unofficial]. It was described in the context of utilizing graphics card hardware for volume rendering [S1, S2, unofficial].

## Sources
[S1] Direct Volume Rendering Direct Volume Rendering. What is volume rendering?. Accumulate information along 1 dimension line through volume. Volume rendering vs. isosurfaces. No intermediate geometry No , https://www.slideserve.com/lenci/direct-volume-rendering, fetched 2026-10-25 (unofficial)
[S2] Indirect volume rendering, https://slidetodoc.com/direct-volume-rendering-what-is-volume-rendering-accumulate/, fetched 2026-10-29 (unofficial)
