# Square-based voxel rendering optimizations

Also called: Voxel rendering acceleration

## Overview
Square-based voxel rendering optimizations are techniques designed to reduce the computational load and rasterization overhead when rendering voxel models as geometric primitives (squares or cubes) [S1]. The primary problem addressed is the excessive number of pixels overwritten due to redundant rasterization of non-visible or distant voxels [S1]. These methods distinguish themselves from general geometry rendering by exploiting the structured, grid-based nature of voxel data to skip calculations for occluded or insignificant voxels, often transitioning from square primitives to point primitives at distance [S1].

## How it works
The optimization pipeline modifies the standard voxel rendering loop by introducing visibility culling, distance-based primitive simplification, and rendering order management [S1].

**Inputs and Outputs**
*   **Input**: A voxel grid (scalar field), camera parameters (position, orientation, projection), and a threshold distance for primitive simplification.
*   **Output**: A rasterized image with reduced overdraw and improved frame rate.

**Data Structures**
To support coherent subset handling, an enclosing data structure is required that identifies adjacent voxels [S1]. This structure allows the renderer to treat a group of adjacent voxels as a single unit for transformation calculations, rather than processing each voxel individually [S1]. The specific layout of this structure is not detailed in the sources, but it must support efficient iteration over coherent subsets [S1].

**Steps**
1.  **Visibility Determination**: Determine which voxels are visible based on the camera angle and model orientation [S1]. Non-visible voxels are skipped to reduce renderer load [S1].
2.  **Distance-Based Simplification**: For voxels beyond a certain z-distance, render them as simple points (pixels) instead of squares [S1]. This is justified because boundary points of mapped voxels slip together at distance, making square rendering redundant [S1].
3.  **Sorting**: Sort the visible voxels along the z-direction (depth) [S1].
4.  **Rendering Order**: Draw voxels from nearest to farthest [S1]. This minimizes overdraw because the z-buffer can reject pixels that are already covered by closer voxels [S1].
5.  **Coherent Subset Processing**: If an enclosing data structure is used, perform transformations on coherent subsets of adjacent voxels as a unit [S1]. Use precalculated structure-level parameters for drawing individual voxels within the subset [S1].

**Pseudocode**
The following pseudocode illustrates the optimized rendering loop, incorporating distance-based simplification and z-sorting. Note that the specific implementation of the "enclosing data structure" is abstracted as a helper function `get_coherent_subset`, as its internal layout is not specified in the sources [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_optimized_voxels(voxel_grid, camera, threshold_distance):
    # 1. Determine visible voxels and sort by z-depth
    # Sources indicate sorting along z-direction minimizes overdraw [S1]
    visible_voxels = []
    for voxel in voxel_grid:
        if is_visible(voxel, camera):
            visible_voxels.append(voxel)
    
    # Sort by z-distance (nearest first) [S1]
    visible_voxels.sort(key=lambda v: v.z_distance)

    # 2. Render voxels
    for voxel in visible_voxels:
        # 3. Distance-based simplification: use points for distant voxels [S1]
        if voxel.z_distance > threshold_distance:
            render_point(voxel)
        else:
            # 4. Render squares for closer voxels
            # If using coherent subsets, this might be handled by a subset iterator
            render_square(voxel)

def is_visible(voxel, camera):
    # Determine visible voxels based on camera angle and model [S1]
    # Implementation details for this specific culling method are not provided in sources [S1]
    return True # Placeholder for actual visibility test

def render_point(voxel):
    # Assign a simple pixel to a voxel [S1]
    pass

def render_square(voxel):
    # Rasterize the voxel as a square [S1]
    pass
```

**Parameters**
*   **Threshold Distance**: The z-distance at which voxels are rendered as points instead of squares. The specific value is not given in the sources [S1].
*   **Voxel Count**: In the cited example, 1,548,288 voxels were rendered [S1].

**Edge Cases and Pitfalls**
*   **Precision**: Not given in the sources.
*   **Boundaries**: Not given in the sources.
*   **Degenerate Input**: Not given in the sources.

## Subtopics

### Dreamcast Voxel Engine Loop Optimizations
This variant focuses on low-level loop optimizations for the Sega Dreamcast hardware (SH4 architecture) [S2, unofficial]. It replaces inner `for` loops with `while` loops that terminate early when the projected height falls below zero or above the screen height [S2, unofficial]. It reorders `heightMap` and `pixelmap` arrays to align adjacent memory elements with adjacent points in the world, improving cache locality and reducing cache misses [S2, unofficial]. Loop-invariant values (`sh4FSCARadianSine`, `sh4FSCARadianCosine`, `plx`, `ply`, `prx`, `pry`) are precomputed before iteration [S2, unofficial]. The loop iterating over the z-variable is unrolled by incrementing z by 2 each iteration, executing the loop body twice per z value [S2, unofficial]. Variables `rx` and `ry` are computed outside the z-loop and incremented inside, eliminating one multiplication and one addition per iteration [S2, unofficial]. This variant is chosen when targeting specific embedded hardware with limited computational resources and cache constraints [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Rendering 1,548,288 voxels as points at distance shows a significant performance jump compared to rendering as true squares [S1]. On Dreamcast hardware, optimizations increased performance from 21fps to 35fps [S2, unofficial].
*   **Scaling**: Not given in the sources.
*   **Strengths**: Reduces overdraw via z-sorting and culling; reduces primitive count via distance-based simplification; improves cache locality via array reordering [S1][S2, unofficial].
*   **Limitations**: Requires additional data structures for coherent subsets; visibility determination requires substantial changes to basic structure [S1].
*   **When to use**: When rendering large voxel models where overdraw is significant and distant voxels can be approximated as points [S1].
*   **When not**: When high-fidelity square rendering is required at all distances [S1].

## Background
The concept of optimizing square-based voxel rendering was introduced by Péter Mileff and Judit Dudra in their paper "Simplified voxel based visualization" [S1]. They identified that redundant rasterization due to the order of rendering leads to unnecessary pixel overwrites [S1]. Their work proposed skipping non-visible voxels, using point primitives for distant voxels, and sorting voxels by depth to minimize overdraw [S1]. The Dreamcast-specific optimizations were developed by Ian Micheal for the "Voxel Space Engine" project, focusing on hardware-specific loop unrolling and cache optimization [S2, unofficial].

## Sources
[S1] Simplified voxel based visualization, https://ojs.uni-miskolc.hu/index.php/psaie/article/view/3824, fetched 2026-10-31
[S2] Dreamcast Voxel engine - Page 2 - Dreamcast-Talk.com, https://www.dreamcast-talk.com/forum/viewtopic.php?start=10&t=16230, fetched 2026-10-30 (unofficial)
