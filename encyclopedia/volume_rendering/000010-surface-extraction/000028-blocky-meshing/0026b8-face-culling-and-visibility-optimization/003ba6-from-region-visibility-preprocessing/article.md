# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes the visibility of geometric primitives (typically triangles) from specific regions of the viewing space to accelerate rendering [S1, unofficial]. It solves the problem of expensive per-frame visibility culling by shifting the computational load to an offline or semi-offline phase [S1, unofficial]. Unlike sibling methods that rely on voxel adjacency or block-level occupancy (e.g., [Face Culling](8735-face-culling.md)), this approach operates on continuous geometry or 2.5D/3D scene representations, focusing on occluder fusion and region-based culling rather than voxel grid traversal [S1, unofficial].

## How it works
The method involves generating a visibility map or culling structure that associates regions of the camera space with subsets of the scene geometry.

**Inputs:**
*   Scene geometry (triangular meshes).
*   Definition of viewing regions (e.g., grid cells, frusta, or specific viewpoints).

**Outputs:**
*   A data structure mapping each region to a list of visible primitives or a visibility mask.

**Data Structures:**
*   **Region Grid:** A partitioning of the 3D space into cells or volumes.
*   **Visibility List:** For each region, a list of triangle indices or a bitmask indicating which triangles are potentially visible from within that region.
*   **Occluder Data:** Information about triangles that occlude others, often used for occluder fusion.

**Steps:**
1.  **Region Partitioning:** Divide the space around the scene into discrete regions. The granularity depends on the desired accuracy and preprocessing time [S1, unofficial].
2.  **Visibility Computation:** For each region, determine which triangles are visible. This often involves ray casting or projection-based techniques to identify occluders and occludees [S1, unofficial].
    *   *Occluder Fusion:* Combine overlapping occluders to reduce the number of primitives considered for culling in subsequent steps [S1, unofficial].
3.  **Data Structure Assembly:** Store the resulting visibility information in a format suitable for runtime lookup. This may involve sorting triangles by region or creating hierarchical structures [S1, unofficial].
4.  **Runtime Integration:** During rendering, identify the region(s) the camera currently occupies and retrieve the precomputed visibility list to cull invisible triangles [S1, unofficial].

**Parameters:**
*   **Region Size:** Smaller regions increase accuracy but increase preprocessing time and memory usage.
*   **Preprocessing Time:** Can be significant, ranging from hours to days for large models [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Preprocessing Time:** The method can require extremely long preprocessing times (e.g., 9–33 hours for millions of triangles) [S1, unofficial].
*   **Dataset Limitations:** Early implementations were limited to 2.5D datasets [S1, unofficial].
*   **Memory Usage:** Storing visibility data for large scenes can consume significant memory.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for From-Region Visibility Preprocessing
# Note: Specific algorithms vary (e.g., Wonka et al. vs. Durand et al.)

def preprocess_visibility(scene_triangles, region_grid):
    """
    Precompute visibility for each region in the grid.
    """
    visibility_data = {}

    for region in region_grid:
        # Determine which triangles are visible from this region
        # This step is computationally expensive and varies by implementation
        visible_triangles = compute_region_visibility(region, scene_triangles)
        
        # Store the result
        visibility_data[region.id] = visible_triangles

    return visibility_data

def compute_region_visibility(region, triangles):
    """
    Compute visible triangles for a given region.
    Implementation details depend on the specific variant (2.5D vs 3D).
    """
    # Example: Ray casting or projection-based culling
    # This is a placeholder for the complex geometric computation
    visible = []
    for tri in triangles:
        if is_visible_from_region(tri, region):
            visible.append(tri.id)
    return visible

def is_visible_from_region(triangle, region):
    """
    Check if a triangle is potentially visible from the region.
    """
    # Geometric visibility test
    # Returns True if the triangle is not fully occluded by other geometry
    # within the region's view volume
    return True # Placeholder for actual geometric logic
```

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It guarantees less than one pixel of screen-space deviation and correct visibility [S1, unofficial]. However, it requires significant computational resources; for a model with 13 million triangles using a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time is high, often measured in hours for large models (e.g., 9–33 hours for 6–8 million triangles) [S1, unofficial].
*   **Memory Complexity:** Depends on the number of regions and the size of the visibility lists. Can be substantial for large scenes.
*   **Performance:** Runtime performance is improved due to reduced per-frame culling overhead [S1, unofficial].
*   **Strengths:** High accuracy in visibility culling; suitable for interactive walkthroughs when preprocessing is acceptable [S1, unofficial].
*   **Limitations:** Long preprocessing times; limited to specific dataset types (e.g., 2.5D in early versions) [S1, unofficial].
*   **When to Use:** When interactive frame rates are critical and preprocessing time is acceptable, or for static scenes where visibility does not change.
*   **When Not to Use:** For dynamic scenes with frequent geometry changes, or when preprocessing time is a bottleneck.

## Background
The concept of from-region visibility preprocessing was introduced by Wonka et al. [152] and Durand et al. [41] [S1, unofficial]. Wonka et al. presented a 2.5D algorithm with occluder fusion, while Durand et al. extended it to 3D environments [S1, unofficial]. Both approaches required significant preprocessing time [S1, unofficial]. Later work by Wonka et al. [153] explored a from-point approach with less preprocessing but limited to 2.5D datasets [S1, unofficial]. Chhugani et al. [20] further developed the concept by integrating level-of-detail (LOD) with from-region visibility [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
