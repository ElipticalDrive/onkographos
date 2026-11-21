# Basic GPU D2VR Framework
Also called: D2VR Framework

## Overview
The Basic GPU D2VR Framework is a method for GPU-accelerated Direct Volume Rendering (D2VR) that reconstructs a 3D volume from 2D projections distributed around a circular orbit [S1, unofficial]. It specifically enables the reconstruction of image-aligned volume slices in front-to-back order, a structural requirement that facilitates occlusion culling and empty space skipping during rendering [S1, unofficial]. It differs from general GPU ray casting by explicitly managing the geometric relationship between projection sets and viewing vectors to maintain slice alignment for acceleration techniques [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $M$ 2D projection images.
*   **Output:** A rendered 3D volume visualization.
*   **Data Structures:** The framework organizes projections into two sets: a Viewing set (V-set) and a Support set (S-set). These sets are used to reconstruct two corresponding texture stacks: the Viewing stack (V-stack) and the Support stack (S-stack) [S1, unofficial].

**Parameters:**
*   **$N$:** The desired volume resolution dimension ($N \times N \times N$).
*   **$M$:** The number of projections. For maximal volume resolution $N^3$ without aliasing using parallel beam data, $M = \frac{\pi}{2} \cdot N$ projections are required [S1, unofficial].
*   **Viewing Angle Constraint:** The viewing vector must be tilted within $\pm 45^\circ$ perpendicular to the reference plane (e.g., the x-z plane) when using the projective texture method [S1, unofficial].

**Method Steps:**
1.  **Projection Organization:** Distribute $M$ projections around a circular orbit. If the projections are perpendicular to the x-z plane, classify them based on their alignment with the viewing vector [S1, unofficial].
2.  **Viewing Vector Check:** Determine if the current viewing vector is within the $\pm 45^\circ$ range perpendicular to the x-z plane [S1, unofficial].
    *   **If within range:** Proceed with the Projective Texture Method.
    *   **If outside range:** Switch to the Texture Spreading Method or provide an orthogonal set of projections (e.g., around the x-y plane) [S1, unofficial].
3.  **Stack Assignment (Projective Texture Method):**
    *   Rotate the volume proxy polygons so that one slice set is perpendicular and the other is orthogonal to the viewing vector [S1, unofficial].
    *   Define the perpendicular slices as the Viewing (V) stack and the orthogonal slices as the Support (S) stack [S1, unofficial].
    *   Divide the projections into the V-set (projections more perpendicular to the V-stack) and the S-set (projections more perpendicular to the S-stack) [S1, unofficial].
4.  **Reconstruction:**
    *   Reconstruct the S-stack from the S-set projections.
    *   Reconstruct the V-stack from the V-set projections.
    *   The reconstruction aims to produce image-aligned volume slices [S1, unofficial].
5.  **Rendering Preparation:** The reconstructed slices are arranged in front-to-back order relative to the viewing direction to enable occlusion culling [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Viewing Angle Exceedance:** If the viewing vector exceeds the $\pm 45^\circ$ limit, the projective texture method fails to maintain alignment. The implementer must either switch methods or provide additional orthogonal projection sets [S1, unofficial].
*   **Tuy’s Condition:** If a single method is to be used for all viewpoints, two orthogonal half-orbits of projections, equally spaced in angle, must be provided to satisfy Tuy’s condition for exact 3D reconstruction [S1, unofficial].

## Subtopics

### [GPU-Accelerated D2VR](16638-gpu-accelerated-d2vr.md)
Extends the basic framework with occlusion culling and empty space skipping for interactive speeds.

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** The framework is designed to enable occlusion culling, which optimizes rendering performance by skipping empty space [S1, unofficial].
*   **Limitations:** The method is constrained by the viewing angle ($\pm 45^\circ$) when using the projective texture approach, requiring specific projection arrangements or method switches for other angles [S1, unofficial].
*   **Memory:** The gradient-from-samples (GFS) approach, mentioned as a related technique in the source, reduces memory bandwidth in the accumulator by reconstructing only density values and estimating gradients from volume samples, though it slows down the rendering procedure [S1, unofficial].

## Background
The framework was introduced in the context of GPU-accelerated volume rendering from 2D projections. It relies on Tuy’s condition for exact 3D reconstruction from projection data [S1, unofficial].

## Sources
[S1] GPU-Accelerated D2VR, https://doi.org/10.2312/vg/vg06/023-030, fetched 2026-10-26 (unofficial)
