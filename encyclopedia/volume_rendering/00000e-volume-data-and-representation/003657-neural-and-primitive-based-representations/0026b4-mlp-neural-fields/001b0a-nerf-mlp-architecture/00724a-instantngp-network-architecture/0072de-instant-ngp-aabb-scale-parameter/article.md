# Instant-NGP AABB Scale Parameter

Also called: AABB scale, aabb_scale parameter, aabb_scale parameter tuning, axis-aligned bounding box scale, bounding box scale factor

## Overview
The `aabb_scale` parameter controls the spatial extent of the axis-aligned bounding box (AABB) used for spatial hashing and grid construction in Instant-NGP [S1]. It scales the scene such that camera positions average a specified distance from the origin, directly influencing the resolution of the hash grid relative to the scene geometry [S1]. Unlike standard SfM-MVS pipelines where mesh definition is highly detailed, Instant-NGP models show lower definition but remain comparable when `aabb_scale` is optimized (8 or 16) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The `aabb_scale` integer parameter, typically a power of 2.
*   **Output:** A scaled coordinate system for the neural field, affecting the spatial hashing grid resolution and the resulting visual quality (noise levels) of the rendered model [S1].

**Data Structures:**
The parameter modifies the implicit mapping from world coordinates to the hash grid indices. The AABB defines the bounds $[-S, S]^3$ where $S = \text{aabb\_scale} / 2$ (assuming centering at origin) or similar scaling relative to the camera distribution [S1]. The exact internal data structure layout is defined in [Unbounded Hash-Grid with Negative Integer Mapping](13889-unbounded-hash-grid-with-negative-integer-mapping.md).

**Steps:**
1.  Determine the appropriate `aabb_scale` value based on scene size. For small synthetic scenes (e.g., original NeRF dataset), set `aabb_scale` to 1 for fastest training [S1].
2.  For larger scenes, set `aabb_scale` to a power of 2 (up to 128) to extend the bounding box [S1].
3.  Upload the transformed JSON file (containing the adjusted scale) to Instant-NGP [S1].
4.  Visualize model construction to evaluate the resulting model definition and noise levels [S1].
5.  Select a value of 8 or 16 for optimal results in general photogrammetric applications [S1].
6.  Avoid values of 1 (for large scenes) and 128 due to high negative influence on quality [S1].
7.  Use the built-in tool to transform the voxel representation into a 3D mesh and select the region of interest if further refinement is needed [S1].

**Formulas:**
The scene is scaled such that camera positions average 1 unit from the origin when `aabb_scale` is 1 [S1]. For a general `aabb_scale` value $A$, the spatial extent is extended proportionally. The hash grid resolution is implicitly tied to this scale; a larger $A$ covers a larger volume with the same number of grid cells, effectively lowering the spatial resolution per unit volume [S1].

**Parameters:**
*   `aabb_scale`: Integer. Default is 1. Acceptable values are powers of 2 up to a maximum of 128 [S1].
*   Typical tested values: 1, 8, 16, 128 [S1].
*   Recommended optimal values: 8 or 16 [S1].

**Edge Cases and Pitfalls:**
*   **Low `aabb_scale` (e.g., 1):** Causes increased noise. A small base noise is detected on each object, which can be mitigated by choosing an appropriate `aabb_scale` value or by graphically selecting the object [S1]. Noise increases as `aabb_scale` decreases [S1].
*   **High `aabb_scale` (e.g., 128):** Degrades quality due to excessive spatial extent relative to the grid capacity, leading to under-sampling of the scene details [S1].
*   **Complex Acquisitions:** Datasets with 360-degree acquisition (e.g., Buddha dataset) exhibit higher noise, making elimination of artifacts complex; external editing software may be required after mesh export [S1].
*   **Training Speed:** Increasing `aabb_scale` slightly influences training speed [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def set_aabb_scale(scene_type: str, scene_size: str) -> int:
    """
    Determines the optimal aabb_scale parameter for Instant-NGP.
    
    Args:
        scene_type: 'small_synthetic' or 'large_real_world'
        scene_size: 'small' or 'large'
        
    Returns:
        int: The recommended aabb_scale value.
    """
    # For small synthetic scenes (e.g., original NeRF dataset), default is 1
    if scene_type == 'small_synthetic':
        return 1
    
    # For larger scenes, use powers of 2 up to 128
    # Optimal values found empirically are 8 or 16
    # Values 1 and 128 degrade quality
    if scene_size == 'large':
        return 8  # or 16, depending on specific scene characteristics
    
    # Default fallback
    return 1

def evaluate_model_quality(aabb_scale: int) -> str:
    """
    Evaluates the model quality based on aabb_scale.
    
    Args:
        aabb_scale: The integer scale parameter used.
        
    Returns:
        str: Quality assessment.
    """
    if aabb_scale == 1:
        return "High noise, fast training (only for small scenes)"
    elif aabb_scale == 128:
        return "Low quality, under-sampled"
    elif aabb_scale in [8, 16]:
        return "Optimal quality"
    else:
        return "Not given in the sources."
```

## Subtopics

### [iNGP Texture Storage](5826-ingp-texture-storage.md)
Stores the feature vectors associated with the hash grid. The `aabb_scale` determines the spatial density of these features.

### [Tiny MLP for Density and Color Estimation](8555-tiny-mlp-for-density-and-color-estimation.md)
Consumes the hashed features. The `aabb_scale` affects the input coordinate normalization, impacting MLP performance.

### [HyperINR](10461-hyperinr.md)
An alternative representation method. Comparison is limited to visual quality and noise characteristics.

### [Hybrid Fourier-Hash Encoding](13021-hybrid-fourier-hash-encoding.md)
Another encoding scheme. The `aabb_scale` parameter is specific to the hash-grid based approach of Instant-NGP.

### [Unbounded Hash-Grid with Negative Integer Mapping](13889-unbounded-hash-grid-with-negative-integer-mapping.md)
The underlying data structure for the hash grid. `aabb_scale` defines the bounding box for this grid.

### [INGeo](15132-ingeo.md)
A geometric variant. The `aabb_scale` parameter influences the geometric accuracy similarly.

### [Scene Scale Consistency Scoring](15618-scene-scale-consistency-scoring.md)
A metric for evaluating scale consistency. `aabb_scale` is a key input parameter for this evaluation.

### [Unrestricted Color Network (UCN) Distillation](16596-unrestricted-color-network-ucn-distillation.md)
A distillation method. The `aabb_scale` affects the source model's quality, which impacts distillation results.

## Operating characteristics
*   **Time Complexity:** Increasing `aabb_scale` slightly influences training speed [S1]. No computational problems were observed across the tested range [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the Buddha dataset (360-degree acquisition), noise was highest, requiring external editing for artifact removal [S1]. For general photogrammetric purposes, `aabb_scale` of 8 or 16 yields optimal results [S1].
*   **Scaling:** The parameter scales linearly with the spatial extent. A value of 128 extends the selection rectangle significantly compared to 1 [S1].
*   **Strengths:** Allows adaptation to various scene sizes. Simple integer parameter.
*   **Limitations:** Suboptimal values (1 for large scenes, 128) cause significant noise or quality loss. Low definition compared to SfM-MVS pipelines [S1].
*   **When to use:** Use for general 3D reconstruction with Instant-NGP. Use 1 for small synthetic scenes. Use 8 or 16 for larger, real-world scenes.
*   **When not to use:** Do not use 128 for standard scenes due to quality degradation. Do not use 1 for large scenes due to noise.

## Background
The `aabb_scale` parameter is part of the Instant-NGP architecture, which uses neural fields for fast rendering [S1]. It was introduced to handle the spatial hashing grid's bounding box efficiently. The parameter is particularly relevant in the context of integrating Instant-NGP with photogrammetric pipelines like Colmap, where the initial camera poses and scene scale are determined [S1]. The assessment of this parameter's impact on 3D model quality for photogrammetric purposes has been documented, highlighting the trade-offs between noise, resolution, and training speed [S1].

## Sources
[S1] Assessment of 3D Model for Photogrammetric Purposes Using AI Tools Based on NeRF Algorithm, https://doi.org/10.3390/heritage6080301, fetched 2026-10-28
