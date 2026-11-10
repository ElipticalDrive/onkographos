# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes the set of visible geometry for specific viewing regions or viewpoints to accelerate rendering. It solves the problem of expensive per-frame visibility testing for large datasets by shifting the computational burden to an offline phase. Unlike [Face Culling](8735-face-culling.md) which operates on local voxel adjacency, this method considers global occlusion relationships, often involving complex occluder fusion, but historically suffers from significant preprocessing overhead.

## How it works
The method involves generating visibility data structures (such as visibility polygons or occlusion culling maps) for predefined regions or points in the scene.

**Inputs:**
*   3D scene geometry (triangles or meshes).
*   Predefined viewing regions or points.

**Outputs:**
*   Visibility maps or culling data structures indicating which triangles are visible from each region/point.

**Data Structures:**
*   Not specified in the sources. Typically involves spatial partitioning or occluder lists.

**Steps:**
1.  **Region/Point Definition:** Define the set of viewing regions or points for which visibility will be precomputed [S1, unofficial].
2.  **Visibility Computation:** For each region/point, determine the set of visible triangles. This may involve occluder fusion to simplify the set of occluding geometry [S1, unofficial].
3.  **Occluder Fusion:** Combine occluders to reduce the complexity of the visibility representation [S1, unofficial].
4.  **Storage:** Store the resulting visibility data for runtime lookup.

**Parameters:**
*   Number of regions/points: Not specified.
*   Preprocessing time: Varies significantly by implementation and dataset size.

**Edge Cases and Pitfalls:**
*   **Preprocessing Time:** The method can require extremely long preprocessing times. For example, Wonka et al. required 9 hours for 8 million triangles [S1, unofficial]. Durand et al. required 33 hours for 6 million triangles [S1, unofficial].
*   **Dataset Limitations:** Early implementations were limited to 2.5D datasets [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode based on general description of from-region visibility preprocessing
# Specific algorithms (e.g., Wonka et al., Durand et al.) are not detailed in the sources.

def preprocess_visibility(scene_geometry, viewing_regions):
    """
    Precompute visibility for given viewing regions.
    
    Args:
        scene_geometry: List of triangles in the scene.
        viewing_regions: List of viewing regions or points.
        
    Returns:
        visibility_data: Dictionary mapping regions to visible triangles.
    """
    visibility_data = {}
    
    for region in viewing_regions:
        # Determine visible triangles for this region
        # This step involves complex visibility computation and occluder fusion
        # which is not detailed in the provided sources.
        visible_triangles = compute_visibility_for_region(scene_geometry, region)
        
        # Store the result
        visibility_data[region] = visible_triangles
        
    return visibility_data

def compute_visibility_for_region(triangles, region):
    """
    Compute visible triangles for a specific region.
    
    Note: The specific algorithm (e.g., ray casting, z-buffering, occluder fusion)
    is not detailed in the sources. This is a placeholder for the complex computation.
    """
    # Implementation details are not provided in the sources.
    # [G] Typically involves projecting occluders and determining visibility.
    pass
```

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It is able to interactively render large datasets with less than one pixel of screen-space deviation and correct visibility [S1, unofficial]. However, it has high preprocessing costs; for a model with 13 million triangles using a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S1, unofficial].

## Operating characteristics
*   **Preprocessing Time:** Very high. Reported times include 9 hours for 8 million triangles (Wonka et al.) [S1, unofficial], 33 hours for 6 million triangles (Durand et al.) [S1, unofficial], and 128 hours for 13 million triangles (Chhugani et al.) [S1, unofficial].
*   **Runtime Performance:** Can enable interactive rendering if preprocessing is complete [S1, unofficial].
*   **Memory Usage:** Not specified in the sources.
*   **Strengths:** Can provide accurate visibility and support Level of Detail (LOD) [S1, unofficial].
*   **Limitations:** Long preprocessing times, potential limitation to 2.5D datasets in early versions [S1, unofficial].

## Background
The concept of from-region visibility preprocessing has been explored by several researchers. Wonka et al. [152] presented a from-region visibility preprocessing algorithm with occluder fusion, which used 2 processors to overlap visibility computation and rendering at runtime [S1, unofficial]. This work was limited to 2.5D datasets. Later, Wonka et al. [153] used a from-point approach needing little preprocessing, but only reported results for in-core, 2.5D datasets [S1, unofficial]. Durand et al. [41] presented a from-region visibility preprocessing algorithm that could handle 3D environments, but required long preprocessing times (33 hours for 6 million triangles) [S1, unofficial]. Schaufler et al. [120] also presented a from-region 3D visibility preprocessing algorithm, but their largest test model had only 0.6 million triangles [S1, unofficial]. Chhugani et al. [20] developed a system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
