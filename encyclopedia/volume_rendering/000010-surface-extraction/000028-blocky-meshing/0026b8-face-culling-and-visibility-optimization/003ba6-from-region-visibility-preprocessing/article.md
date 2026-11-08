# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes which parts of a 3D scene are visible from specific viewing regions to accelerate rendering. It solves the problem of expensive per-frame visibility determination for large datasets by shifting the computational load to an offline phase. Unlike sibling methods that rely on axis-aligned voxel occupancy checks [923-voxel-block-placement-with-face-culling.md] or simple backface culling [4569-voxel-backface-culling-and-internal-face-masking.md], this approach handles complex geometry and occluder fusion, though it incurs significant preprocessing overhead.

## How it works
The method involves precomputing visibility information for a set of predefined regions in the scene. The core algorithm, as presented by Wonka et al. [S1, unofficial], utilizes occluder fusion to reduce the number of primitives considered for visibility tests.

**Inputs and Outputs:**
*   **Input:** A 3D scene model (typically represented as triangles) and a set of viewing regions.
*   **Output:** A visibility map or precomputed data structure indicating which primitives are visible from each region.

**Data Structures:**
Not given in the sources.

**Steps:**
1.  **Region Definition:** Define the viewing regions for which visibility will be precomputed.
2.  **Occluder Fusion:** Identify and merge occluders to simplify the visibility computation. This reduces the number of primitives that need to be tested against the viewing regions [S1, unofficial].
3.  **Visibility Computation:** Determine which primitives are visible from each region. This step is computationally intensive. Wonka et al. used two processors to overlap visibility computation and rendering at runtime [S1, unofficial].
4.  **Storage:** Store the precomputed visibility data for runtime lookup.

**Formulas:**
Not given in the sources.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Preprocessing Time:** The algorithm requires long preprocessing times. For example, Wonka et al. reported 9 hours for a model with 8 million triangles [S1, unofficial]. Durand et al. reported 33 hours for a model with 6 million triangles [S1, unofficial].
*   **Dataset Limitations:** The initial version by Wonka et al. was limited to 2.5D datasets [S1, unofficial]. Later work by Wonka et al. [153] used a from-point approach needing little preprocessing but was only reported for in-core, 2.5D datasets [S1, unofficial]. Durand et al. extended this to 3D environments [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for From-Region Visibility Preprocessing
# Based on Wonka et al. [S1, unofficial]

def preprocess_visibility(scene, regions):
    """
    Precompute visibility for given regions in the scene.
    
    Args:
        scene: The 3D scene model (triangles).
        regions: List of viewing regions.
        
    Returns:
        visibility_map: Precomputed visibility data.
    """
    visibility_map = {}
    
    # Step 1: Occluder Fusion
    # Identify and merge occluders to simplify visibility tests
    fused_occluders = fuse_occluders(scene)
    
    # Step 2: Visibility Computation
    # Determine visible primitives for each region
    for region in regions:
        visible_primitives = []
        for primitive in scene.primitives:
            # Check if primitive is visible from region
            # This involves complex geometric tests not detailed in sources
            if is_visible(primitive, region, fused_occluders):
                visible_primitives.append(primitive)
        visibility_map[region] = visible_primitives
        
    return visibility_map

def fuse_occluders(scene):
    """
    Fuse occluders in the scene.
    Implementation details not given in the sources.
    """
    # Placeholder for occluder fusion logic
    pass

def is_visible(primitive, region, fused_occluders):
    """
    Check if a primitive is visible from a region, considering fused occluders.
    Implementation details not given in the sources.
    """
    # Placeholder for visibility test logic
    pass
```

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It is able to interactively render large datasets with less than one pixel of screen-space deviation and correct visibility [S1, unofficial]. However, the accuracy guarantee comes at a high cost: for a model with 13 million triangles, using a cluster of 16 PCs, it requires 128 hours of precomputation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time is very high. Reported times range from 9 hours (8 million triangles) to 128 hours (13 million triangles on 16 PCs) [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Wonka et al. used 2 processors to overlap visibility computation and rendering at runtime [S1, unofficial]. Chhugani et al. achieved interactive rendering with high image accuracy [S1, unofficial].
*   **Strengths:** Handles complex geometry and occluder fusion; provides high image accuracy with correct visibility.
*   **Limitations:** Extremely long preprocessing times; initial versions limited to 2.5D datasets; scalability issues for very large models without significant hardware resources.
*   **When to use:** When offline preprocessing time is acceptable and high-quality, accurate visibility is required for interactive rendering of large datasets.
*   **When not to use:** When real-time preprocessing is required or when the dataset is too large for the available preprocessing hardware/time.

## Background
The concept of from-region visibility preprocessing was introduced by Wonka et al. [152] and Durand et al. [41] [S1, unofficial]. Wonka et al. [152] presented a from-region visibility preprocessing algorithm with occluder fusion, limited to 2.5D datasets [S1, unofficial]. Durand et al. [41] extended this to handle 3D environments [S1, unofficial]. Later work by Wonka et al. [153] used a from-point approach needing little preprocessing [S1, unofficial]. Chhugani et al. [20] developed a system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
