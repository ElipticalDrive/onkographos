# Budgeted-GS

Also called: Factoring LOD

## Overview
Budgeted-GS is a post-hoc method that converts any trained 3D Gaussian Splatting (3DGS) model into a multi-resolution hierarchy of moment-matched aggregates, enabling real-time rendering of city-scale scenes on consumer GPUs by selecting the level of detail that fits the target device's memory [S1, unofficial]. It solves the problem of excessive memory consumption in large-scale 3DGS by introducing a "capacity floor" derived from optimal transport in phase space, which determines the minimum number of primitives required for a given error budget, allowing the same model to serve GPUs with widely different memory capacities [S1, unofficial]. Unlike standard 3DGS which optimizes a fixed set of primitives, Budgeted-GS separates the representation into a hierarchy where a single quality parameter selects the appropriate level of detail per view, ensuring that redundant primitives are merged or dropped based on a pixel-footprint scheduler [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained 3DGS model consisting of a flat set of raw planks (Gaussian primitives) defined by their spatial cells, opacity weights, and value models [S1, unofficial].
*   **Output:** A rendered image stream emitted to any standard splatting rasterizer, where the stream consists of selected primitives from the factoring tree based on the current view and memory budget [S1, unofficial].

**Data Structures:**
The core data structure is the **Factoring Tree**, an octree of scale bands [S1, unofficial].
*   **Nodes:** Each node corresponds to a leaf in the spatial octree and a specific scale band [S1, unofficial].
*   **Content:** Nodes contain "moment-matched equivalent planks," which are pre-computed aggregates of raw planks that preserve the first and second moments (mean and covariance) of the original primitives within that spatial and scale region [S1, unofficial].
*   **Layout:** The tree is structured as `leaf × scale-band`. Raw planks are stored at the finest scale bands, while coarser bands store aggregated planks [S1, unofficial].

**Steps:**
1.  **Construction:** Convert the trained model (flat set of raw planks) into the factoring tree of leaf × scale-band nodes containing moment-matched equivalent planks [S1, unofficial]. This construction pass takes a few seconds [S1, unofficial].
2.  **Scheduling:** For each view, determine the level of detail using a one-parameter pixel-footprint scheduler [S1, unofficial].
3.  **Selection:** Select raw, fine, or aggregate content based on the threshold $t$. If the pixel footprint of a primitive is smaller than $t$, use aggregated content; otherwise, use raw or fine content [S1, unofficial].
4.  **Rendering:** Emit the selected stream of primitives to a standard splatting rasterizer [S1, unofficial].

**Formulas:**
The selection threshold $t$ is calculated as:
$$ t = c \cdot \frac{d}{f} $$
Where:
*   $t$: The pixel-footprint threshold [S1, unofficial].
*   $c$: A constant parameter [S1, unofficial].
*   $d$: Distance from the camera to the primitive [S1, unofficial].
*   $f$: Focal length of the camera [S1, unofficial].

The capacity floor, which dictates the minimum number of primitives $N^*$ per tile $j$, is given by:
$$ N^*_j \propto \frac{K_j s_j}{d_j} $$
Where:
*   $K_j$: A constant related to the tile [S1, unofficial].
*   $s_j$: Scale of the primitives in tile $j$ [S1, unofficial].
*   $d_j$: Distance to tile $j$ [S1, unofficial].

**Parameters:**
*   $c$: Constant in the pixel-footprint threshold formula. Typical value not given in the sources.
*   $t$: Threshold for LOD selection.

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on moment-matching, which may introduce approximation errors when aggregating primitives. The sources mention "bounded volumetric loss" but do not specify the error bounds [S1, unofficial].
*   **Boundaries:** The octree structure handles spatial boundaries, but the scale bands must be carefully defined to ensure smooth transitions between LOD levels. Not given in the sources.
*   **Degenerate Input:** The method assumes a trained 3DGS model [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def distance(p1, p2):
    """Euclidean distance between two 3D points."""
    return math.sqrt(sum((a - b) ** 2 for a, b in zip(p1, p2)))

def compute_footprint(plank, camera_position, camera_focal_length):
    """Compute pixel footprint by projecting Gaussian covariance to screen space [G]."""
    # Standard implementation: transform covariance to camera space, 
    # project to 2D Jacobian, compute bounding box area in pixels.
    # For this article, we assume the primitive provides its projected radius.
    return plank.get_screen_radius() * 2.0

def budgeted_gs_select_primitives(camera_position, camera_focal_length, tree, constant_c):
    """
    Selects primitives from the factoring tree based on the pixel-footprint threshold.
    Args:
        camera_position: 3D coordinates of the camera.
        camera_focal_length: Focal length f.
        tree: List of nodes, each containing 'center', 'scale_band', 'raw_planks', 'aggregate_planks'.
        constant_c: Constant c for threshold calculation.
    Returns:
        List of selected primitive objects to be passed to a rasterizer.
    """
    selected = []
    for node in tree:
        d = distance(camera_position, node.center)
        t = constant_c * (d / camera_focal_length)
        
        # Select raw/fine or aggregate based on footprint threshold
        for plank in node.raw_planks:
            footprint = compute_footprint(plank, camera_position, camera_focal_length)
            if footprint < t:
                # Use aggregate if available, else raw
                if node.aggregate_planks:
                    selected.extend(node.aggregate_planks)
                else:
                    selected.append(plank)
            else:
                selected.append(plank)
                
        # Add aggregate planks for regions where they are selected
        if node.aggregate_planks:
            for agg_plank in node.aggregate_planks:
                footprint = compute_footprint(agg_plank, camera_position, camera_focal_length)
                if footprint < t:
                    selected.append(agg_plank)
                    
    return selected

def render_splats(primitives, camera):
    """Standard splatting rasterization step [G]."""
    # Implementation details for rasterizing Gaussian splats are standard and external to this method.
    # This function accumulates screen-space contributions per pixel.
    return []
```

## Subtopics

### Differentiated Quality Modeling via Gaussian Masking
A quality modeling approach that classifies 3DGS videos into four tiers of models with varying qualities using a binary Gaussian mask based on significance distribution, considering both opacity and ellipsoid size to filter out less significant primitives [S2]. It identifies and filters out Gaussian primitives with lower transparency and smaller ellipsoid sizes, as they have minimal impact on rendering quality [S2]. Three distinct masking thresholds are established to classify the 3DGS videos within each Group of Frames (GoF) into four tiers of models with varying qualities [S2]. Thresholds are determined by referencing previously obtained saliency distribution maps and adjusting thresholds based on saliency weights within different tiles; for high-saliency regions, as many Gaussian primitives as possible are retained, while for less visually significant regions, thresholds are increased to exclude unnecessary primitives [S2]. All tiles of different quality levels are encoded to facilitate subsequent delivery from server to client [S2].

## Operating characteristics
*   **Time Complexity:** The construction pass takes a few seconds [S1, unofficial]. Rendering time depends on the number of selected primitives, which is dynamically adjusted by the scheduler [S1, unofficial].
*   **Memory Complexity:** The method reduces memory usage by aggregating primitives, allowing city-scale scenes to fit on consumer GPUs [S1, unofficial]. The exact memory savings depend on the scene and the chosen quality parameter [S1, unofficial].
*   **Measured Performance:** Validated on 13 public scenes under a preregistered protocol, demonstrating real-time rendering of city-scale scenes [S1, unofficial].
*   **Scaling:** The method scales to large scenes by using a multi-resolution hierarchy [S1, unofficial]. The capacity floor provides a theoretical limit on the number of primitives needed [S1, unofficial].
*   **Strengths:** Enables real-time rendering of large-scale scenes on consumer hardware; flexible quality adjustment via a single parameter; avoids wasted optimization effort in training [S1, unofficial].
*   **Limitations:** Requires a post-hoc construction pass; relies on the accuracy of the moment-matching approximation; performance depends on the scene's complexity and the chosen quality parameter [S1, unofficial].
*   **When to use:** When rendering large-scale, city-scale scenes on hardware with limited memory [S1, unofficial].
*   **When not to use:** For small-scale scenes where standard 3DGS fits in memory; when training time is a critical constraint and budget-centered training is not used [S1, unofficial].

## Background
Budgeted-GS was introduced in the paper "Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD" [S1, unofficial]. It is grounded in a measurable capacity floor derived from optimal transport in phase space, which answers how many primitives a scene actually needs and how many can safely be given up [S1, unofficial]. The method builds on the principles of 3D Gaussian Splatting [Parent](7342-hierarchical-3d-gaussian-representations.md) but extends it to handle large-scale scenes by introducing a multi-resolution hierarchy and a budget-centered training approach [S1, unofficial].

## Sources
[S1] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
[S2] Adaptive 3-D Gaussian Splatting Video Streaming, https://doi.org/10.1109/MNET.2025.3638049, fetched 2026-11-03
