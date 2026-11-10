# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes which parts of a 3D model are visible from specific regions of the viewing space, allowing for aggressive culling of occluded geometry during rendering [S1, unofficial]. It differs from sibling methods like [Block Visibility Culling](5334-block-visibility-culling.md) or [Face Culling](8735-face-culling.md) by operating on large-scale geometric clusters or regions rather than individual voxels or faces, and by explicitly handling occluder fusion to reduce the number of visibility queries [S1, unofficial]. While it enables interactive frame rates for massive datasets, it is distinguished by its high preprocessing cost and historical limitation to 2.5D environments in early implementations [S1, unofficial].

## How it works
The method involves partitioning the scene into regions and precomputing visibility maps for each region against the scene geometry.

**Inputs:**
- A 3D model (typically represented as a mesh of triangles or axis-aligned cells) [S1, unofficial].
- A set of predefined viewing regions or viewpoints [S1, unofficial].

**Outputs:**
- A visibility map or culling structure that indicates which triangles or clusters are visible from each region [S1, unofficial].

**Data Structures:**
- The specific layout is not detailed in the sources, but the algorithm relies on storing visibility results for "from-region" queries [S1, unofficial].
- Occluder fusion structures are implied to reduce the number of occluders considered [S1, unofficial].

**Steps:**
1. **Region Definition:** Define the viewing regions for which visibility will be precomputed [S1, unofficial].
2. **Visibility Computation:** For each region, determine which triangles in the model are visible. This step may involve occluder fusion to simplify the set of occluding geometry [S1, unofficial].
3. **Storage:** Store the visibility results, mapping regions to visible triangles or clusters [S1, unofficial].
4. **Runtime Rendering:** During rendering, identify the current viewing region and use the precomputed visibility map to cull invisible triangles [S1, unofficial].
5. **Parallel Processing:** To mitigate long preprocessing times, the computation can be overlapped with rendering using multiple processors [S1, unofficial].

**Parameters:**
- Number of processors: Early implementations used 2 processors to overlap visibility computation and rendering [S1, unofficial].
- Cluster size: Later systems used clusters of PCs (e.g., 16 PCs) for precomputation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Preprocessing Time:** The method requires significant preprocessing time. For example, 9 hours for 8 million triangles (Wonka et al.) and 33 hours for 6 million triangles (Durand et al.) [S1, unofficial].
- **Dataset Limitations:** Early versions were limited to 2.5D datasets [S1, unofficial].
- **Model Assumptions:** Some variants assume models are made of axis-aligned cells, though later methods like PLP and cPLP removed this assumption [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for From-Region Visibility Preprocessing
# Note: Specific algorithms for visibility computation (e.g., ray casting, hierarchical culling) are not detailed in the sources.

def preprocess_visibility(model, regions):
    """
    Precomputes visibility for a set of regions.
    
    Args:
        model: The 3D model (mesh of triangles).
        regions: A list of viewing regions.
        
    Returns:
        visibility_map: A dictionary mapping region IDs to visible triangle indices.
    """
    visibility_map = {}
    
    # Parallel processing can be used here to overlap computation and rendering
    # [S1, unofficial]
    
    for region_id, region in regions:
        visible_triangles = compute_visibility_for_region(model, region)
        visibility_map[region_id] = visible_triangles
        
    return visibility_map

def compute_visibility_for_region(model, region):
    """
    Computes visible triangles for a specific region.
    This function's internal logic (e.g., occluder fusion) is not detailed in the sources.
    [S1, unofficial]
    """
    # Placeholder for the actual visibility computation algorithm
    # [S1, unofficial]
    pass
```

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It guarantees less than one pixel of screen-space deviation and correct visibility [S1, unofficial]. For a model with 13 million triangles, using a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Preprocessing time is high. Reported times include 9 hours for 8 million triangles, 33 hours for 6 million triangles, and 128 hours for 13 million triangles [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Enables interactive rendering of large datasets (e.g., 13 million triangles) after preprocessing [S1, unofficial].
- **Scaling:** Preprocessing time scales with the number of triangles and the complexity of the visibility computation [S1, unofficial].
- **Strengths:** Allows for real-time rendering of very large datasets by culling invisible geometry [S1, unofficial].
- **Limitations:** Long preprocessing times; early versions limited to 2.5D datasets [S1, unofficial].
- **When to use:** When rendering extremely large datasets where preprocessing time is acceptable and interactive frame rates are required [S1, unofficial].
- **When not to use:** When dynamic geometry changes frequently or when preprocessing time is prohibitive [S1, unofficial].

## Background
The concept was introduced by Wonka et al. [152] as a from-region visibility preprocessing algorithm with occluder fusion, limited to 2.5D datasets [S1, unofficial]. Durand et al. [41] extended this to 3D environments, though with even longer preprocessing times (33 hours for 6 million triangles) [S1, unofficial]. Chhugani et al. [20] developed a system that incorporated levels of detail and focused on image accuracy, using a cluster of 16 PCs for precomputation [S1, unofficial]. Later work by Wonka et al. [153] shifted to a from-point approach with less preprocessing but remained limited to 2.5D datasets [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
