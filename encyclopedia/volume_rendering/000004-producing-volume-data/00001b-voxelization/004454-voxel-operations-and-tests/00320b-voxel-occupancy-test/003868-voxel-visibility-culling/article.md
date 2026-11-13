# Voxel Visibility Culling

Also called: Inside/Outside Clipping, Structure Visibility, Visibility Metric

## Overview
Voxel Visibility Culling determines whether a voxel or a collection of voxels (such as an atomic structure) is sufficiently revealed from a given viewpoint to warrant rendering, discarding occluded or irrelevant data to save computation. It differs from its parent [Voxel Occupancy Test](1281-voxel-occupancy-test.md) by incorporating view-dependent criteria (such as gradient magnitude or camera distance) rather than relying solely on static spatial occupancy. It also differs from sibling methods like [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by evaluating perceptual or geometric visibility thresholds rather than simple containment.

## How it works
The method operates in two primary modes: structural visibility metrics based on gradient modulation [S1] and geometric clipping based on isovalue signs [S2, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Voxel grid data containing scalar values and gradient magnitudes $g_v$ for each voxel $v$.
    *   User-defined thresholds: base voxel visibility threshold $t_{vv}$ and structure visibility guarantee threshold $t_{sv}$ [S1].
    *   Camera parameters (position, lookat vector, near plane) for geometric culling [S4, unofficial].
    *   Display geometry parameters (voxel dimensions $VOXELS_X, VOXELS_Y$) for cylindrical masking [S5, unofficial].
*   **Outputs:**
    *   Boolean mask indicating visible voxels.
    *   Structure visibility score $V(s)$ [S1].
    *   List of retained voxels after culling [S3, unofficial].

### Data Structures
*   **Voxel Data:** Array of voxels, each containing position $(x,y,z)$, scalar value, and gradient magnitude $g_v$.
*   **Occupied Set:** For interior culling, a hash set of packed coordinate keys [S3, unofficial].
*   **Visibility Mask:** Boolean array or list of indices for visible voxels.

### Formulas
1.  **Voxel-Level Visibility Threshold:**
    $$ t_{vv} \cdot G(g_v) $$
    Where $t_{vv}$ is the user-specified base threshold and $G(g_v)$ is the gradient modulation function [S1].

2.  **Structure Visibility:**
    $$ V(s) = \frac{N_{vv}(s)}{N_{tv}(s)} $$
    Where $V(s)$ is the structure visibility of atomic structure $s$, $N_{vv}(s)$ is the number of visible voxels in $s$, and $N_{tv}(s)$ is the total number of voxels in $s$ [S1].

3.  **Cylindrical Masking Distance:**
    $$ d^2 = x_{norm}^2 + y_{norm}^2 $$
    $$ \text{Keep if } d^2 \le \left(\frac{VOXELS_X + VOXELS_Y}{2}\right)^2 $$
    Where $x_{norm} = (x \cdot 2) - (VOXELS_X - 1)$ and $y_{norm} = (y \cdot 2) - (VOXELS_Y - 1)$ [S5, unofficial].

### Steps

#### Mode 1: Gradient-Modulated Structural Visibility [S1]
1.  For each voxel $v$ in structure $s$:
    a. Calculate the gradient magnitude $g_v$.
    b. Compute the visibility threshold $T_v = t_{vv} \cdot G(g_v)$.
    c. Determine if voxel $v$ is visible (implementation of $G(g_v)$ and comparison logic depends on specific renderer; typically if $g_v > T_v$ or similar metric [S1]).
2.  Count $N_{vv}(s)$, the number of visible voxels in $s$.
3.  Calculate $V(s) = N_{vv}(s) / N_{tv}(s)$.
4.  If $V(s) \ge t_{sv}$, mark structure $s$ as visible enough to render [S1].

#### Mode 2: Isosurface Clipping [S2, unofficial]
1.  For each voxel $v$ with center value $val_v$:
    a. If $val_v < 0.0$, discard the instance (Inside Clipping) [S2, unofficial].
    b. If $val_v > 0.0$, discard the instance (Outside Clipping) [S2, unofficial].
    *(Note: This implies culling based on the sign of the scalar field relative to the isosurface at 0.0)*.

#### Mode 3: Near-Plane Culling [S4, unofficial]
1.  Identify visible voxels where $n\_duplicates > 0$: $vis\_idx = \text{torch.where}(n\_duplicates > 0)[0]$.
2.  Calculate z-distance from voxel centers to camera position along the lookat vector:
    $$ zdist = \sum ((vox\_centers[vis\_idx] - cam.position) \cdot cam.lookat) $$
3.  Mark voxels as near if:
    $$ zdist \le near\_plane + vox\_lengths.squeeze(1)[vis\_idx] $$
4.  Update `is_near` mask: `is_near[vis_idx] |= (zdist <= near_plane + vox_lengths.squeeze(1)[vis_idx])` [S4, unofficial].

#### Mode 4: Cylindrical Volume Masking [S5, unofficial]
1.  Normalize x-coordinate: $x_{norm} = (x \cdot 2) - (VOXELS_X - 1)$.
2.  Normalize y-coordinate: $y_{norm} = (y \cdot 2) - (VOXELS_Y - 1)$.
3.  Compute squared distance: $d^2 = x_{norm}^2 + y_{norm}^2$.
4.  Compute squared radius: $R^2 = ((VOXELS_X + VOXELS_Y) / 2)^2$.
5.  Keep voxel if $d^2 \le R^2$ [S5, unofficial].

### Parameters
*   $t_{vv}$: User-specified voxel-level visibility threshold base value [S1].
*   $t_{sv}$: Structure visibility guarantee threshold. If $V(s) \ge t_{sv}$, the structure is considered visible enough [S1].
*   $near\_plane$: Default value 0.2, used as the threshold for near-plane culling [S4, unofficial].

### Edge Cases and Pitfalls
*   **Gradient Modulation:** The function $G(g_v)$ is referenced but not explicitly defined in the source [S1]. Implementers must define $G$ (e.g., linear, sigmoid) based on the specific rendering pipeline's requirements for feature outline importance [S1].
*   **Coordinate Packing:** Interior voxel culling assumes coordinates fit in 20 bits each (up to ~1M per axis). Negative coordinates require an offset (e.g., +1,000,000) before packing [S3, unofficial].
*   **Cylindrical Approximation:** Cylindrical masking discards corner voxels. If the display is not a spinning drum or the aspect ratio is extreme, this may remove visible content [S5, unofficial].

## Subtopics

### [Visible Bounds Calculation](12016-visible-bounds-calculation.md)
Visible Bounds Calculation determines the physical-space bounding box of voxels with composited opacity above a specific threshold, allowing the camera to frame only the visible content rather than the full volume extent. It differs from sibling methods by using a downsampled, peak-based Gaussian approximation of opacity to achieve significant performance gains (from ~1-2s to ~10ms) while maintaining sufficient accuracy for camera framing.

### [Obstacle-Flag](14294-obstacle-flag.md)
The Obstacle-Flag method is a dynamic job-scheduling mechanism used in distributed volume rendering to determine when a sub-volume is "combinable" (i.e., ready to be rendered and merged into the final image). It solves the inefficiency of sequential job-scheduling, where agents remain idle waiting for rigid, pre-computed visibility orders, by tracking real-time obscuration relationships between neighboring sub-volumes. Unlike static visibility culling, it updates its state as rendering progresses, allowing agents to immediately pick up the next available sub-volume as soon as its obscuring neighbors are completed.

### Interior Voxel Culling
Removes voxels completely surrounded by other voxels on all 6 faces to reduce instance count without visual impact [S3, unofficial]. It builds a hash set of occupied positions by packing each voxel's $(x,y,z)$ into a unique long key. For each voxel, it checks if all 6 neighbors exist in the hash set. If all 6 neighbors are present, the voxel is marked as fully enclosed and excluded from the visible list. The coordinate packing assumes coordinates fit in 20 bits each (up to ~1M per axis), with negative coordinates handled by adding a 1,000,000 offset before packing. Complexity is $O(N)$ time for building the hash set and $O(N)$ for culling, where $N$ is the number of voxels [S3, unofficial].

### Near Plane Visibility Culling
Determines if voxels are too close to the camera (near plane) for visibility, returning a boolean mask [S4, unofficial]. It identifies visible voxels where $n\_duplicates > 0$, calculates the z-distance from voxel centers to the camera position along the lookat vector, and marks voxels as near if $zdist \le near\_plane + vox\_lengths$. The default `near_plane` parameter is 0.2 [S4, unofficial].

### Cylindrical Volume Masking
Filters voxels to only those within the physical cylindrical display area, discarding corner voxels that would not be visible on a spinning drum [S5, unofficial]. It normalizes x and y coordinates, computes the squared distance from the center, and compares it against the squared radius defined by the average of the voxel dimensions [S5, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   Gradient-modulated structural visibility: $O(N)$ per structure, where $N$ is the number of voxels in the structure [S1].
    *   Interior Voxel Culling: $O(N)$ for building the hash set and $O(N)$ for culling [S3, unofficial].
    *   Near-Plane Culling: $O(N)$ for distance calculations [S4, unofficial].
*   **Memory Complexity:**
    *   Interior Voxel Culling requires $O(N)$ memory for the hash set of packed keys [S3, unofficial].
    *   Other methods typically require $O(N)$ for masks or temporary buffers.
*   **Performance:**
    *   Visible Bounds Calculation using Gaussian approximation reduces computation time from ~1-2s to ~10ms [S1].
*   **Strengths:**
    *   Reduces rendering load by discarding occluded or irrelevant voxels [S1].
    *   Dynamic scheduling (Obstacle-Flag) improves utilization in distributed systems [unofficial].
    *   Geometric culling (Near Plane, Cylindrical) handles specific hardware or camera constraints efficiently [S4, S5, unofficial].
*   **Limitations:**
    *   Gradient modulation function $G(g_v)$ requires implementation [S1].
    *   Interior culling may remove voxels that are technically "inside" but contribute to subtle surface details if the enclosure is not perfect [S3, unofficial].
    *   Cylindrical masking is specific to cylindrical displays [S5, unofficial].

## Background
The concept of visibility-aware rendering in volumes was formalized in "Visibility-Aware Direct Volume Rendering" [S1], which introduced the structure visibility metric $V(s)$ and gradient modulation to ensure important features are revealed. The method builds on [Voxel Occupancy Test](1281-voxel-occupancy-test.md) by adding view-dependent criteria. Variants like Interior Voxel Culling [S3, unofficial] and Near-Plane Culling [S4, unofficial] are found in modern voxel rendering implementations for games and scientific visualization.

## Sources
[S1] Visibility-Aware Direct Volume Rendering, https://doi.org/10.1007/s11390-011-9428-3, fetched 2026-10-26
[S2] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
[S3] Il-castor/VoxelRenderer, https://github.com/Il-castor/VoxelRenderer, fetched 2026-10-26 (unofficial)
[S4] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S5] Multivox: Volumetric Display, https://github.com/AncientJames/multivox, fetched 2026-10-26 (unofficial)
