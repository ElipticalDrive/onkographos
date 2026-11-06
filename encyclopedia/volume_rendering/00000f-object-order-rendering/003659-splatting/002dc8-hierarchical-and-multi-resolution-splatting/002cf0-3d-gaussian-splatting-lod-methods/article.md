# 3D Gaussian Splatting LOD methods

Also called: Hierarchical 3D Gaussian Splatting, Octree-GS, LODGE, MIP-Splatting, CityGaussian, VastGaussian, DoGaussian, K-wise moment-matched merging

## Overview
3D Gaussian Splatting Level-of-Detail (LOD) methods address the scalability and aliasing limitations of monolithic 3D Gaussian Splatting (3DGS) [S1, S2, unofficial]. Standard 3DGS stores all Gaussians in a flat list, causing excessive memory usage for large scenes and visual artifacts (dilation/erosion) when camera distance or resolution changes [S1, S2, unofficial]. These methods introduce hierarchical structures, spatial partitioning, or frequency-aware filtering to reduce rendering load and improve visual fidelity at varying scales [S1, S2, unofficial]. They differ from [Early Splat Termination](811-early-splat-termination.md) by modifying the primitive representation or structure rather than just culling during rasterization, and from [EWA Splatting](1069-ewa-splatting.md) by operating on the learned 3D Gaussian parameters rather than just the projection kernel [S1, S2, unofficial].

## How it works
The core challenge is managing the trade-off between scene complexity (number of Gaussians) and rendering performance/quality. Methods generally fall into three categories: hierarchical clustering, spatial tiling, and frequency-aware filtering.

### Data Structures
1.  **Hierarchical Trees (Octree/Cluster):** Nodes in an octree or cluster tree store a "merged" Gaussian that summarizes the radiance, opacity, and covariance of its children [S1, S2, unofficial]. Each node typically contains:
    *   Position $\mathbf{p}$ (3 floats)
    *   Covariance $\Sigma$ (6 floats, symmetric positive definite)
    *   Opacity $\alpha$ (1 float)
    *   Spherical Harmonics coefficients $\mathbf{c}$ (variable, e.g., 9 for SH band 1)
    *   Children pointers or indices [S1, S2, unofficial].
2.  **Spatial Tiles:** The scene is divided into disjoint or overlapping 3D bounding boxes (tiles) [S1, unofficial]. Each tile contains a subset of Gaussians. Overlap regions use feathered blending to handle boundary artifacts [S1, unofficial].
3.  **MIP-Filtered Gaussians:** Standard 3DGS parameters are augmented with 2D and 3D Gaussian filters to control frequency content [S1, unofficial].

### Algorithm: Hierarchical Rendering (e.g., Octree-GS, Hierarchical 3DGS)
1.  **Build:** Construct a multi-resolution pyramid or octree over the scene. For each node, compute a representative Gaussian by merging children [S1, S2, unofficial].
    *   *Merging Strategy:* Methods vary. [K-wise moment-matched merging](5440-k-wise-moment-matched-merging-with-l2-optimal-amplitude.md) matches statistical moments and optimizes amplitude [S1, unofficial]. Others use simple averaging or learned decoders [S2, unofficial].
2.  **Traverse:** For each frame, traverse the tree from the root.
3.  **Select LOD:** Determine the appropriate LOD for each node based on camera distance and screen-space footprint size [S1, S2, unofficial].
    *   If the node's footprint is smaller than a threshold, render the node's representative Gaussian.
    *   If larger, recurse to children.
4.  **Render:** Rasterize the selected Gaussians using standard 3DGS splatting [S1, S2, unofficial].

### Algorithm: Spatial Tiling (e.g., CityGaussian, VastGaussian)
1.  **Partition:** Divide the scene into spatial tiles [S1, unofficial].
2.  **Train:** Train independent 3DGS models for each tile [S1, unofficial].
3.  **Merge:** At render time, merge tiles with overlapping boundaries using feathered blending to ensure continuity [S1, unofficial].
4.  **Stream:** Use a LoD pyramid to stream only visible tiles to the GPU [S1, unofficial].

### Algorithm: Frequency-Aware Filtering (MIP-Splatting)
1.  **Filter:** Replace the standard dilation filter with explicit 2D and 3D Gaussian filters [S1, unofficial].
2.  **Render:** Apply these filters during projection to prevent aliasing artifacts when zooming [S1, unofficial].

### Pseudocode: Hierarchical LOD Selection
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_hierarchical_gaussians(camera, root_node, screen_width, screen_height):
    """
    Renders a hierarchical 3DGS structure.
    camera: Object with position, orientation, fov, near, far.
    root_node: The root of the Gaussian octree/cluster tree.
    """
    active_splats = []
    queue = [(root_node, 0)]  # (node, depth)
    
    while queue:
        node, depth = queue.pop(0)
        
        # Project node center to screen
        screen_pos = project_to_screen(node.position, camera)
        
        # Estimate screen-space radius of the node's Gaussian
        # This depends on the node's covariance and camera distance
        screen_radius = estimate_screen_radius(node.covariance, camera)
        
        # LOD Threshold: e.g., if radius < 1 pixel, use this node
        if screen_radius < 1.0:
            active_splats.append(node.representative_gaussian)
        else:
            # Recurse to children if available
            if node.has_children:
                for child in node.children:
                    queue.append((child, depth + 1))
            else:
                # Leaf node: render if visible
                if is_visible(node, camera):
                    active_splats.append(node.representative_gaussian)
                    
    # Rasterize active_splats using standard 3DGS splatting
    rasterize(active_splats, camera, screen_width, screen_height)
```

### Parameters and Edge Cases
*   **LOD Threshold:** Typically 1–2 pixels. Too low causes popping; too high reduces detail [S1, S2, unofficial].
*   **Merge Error:** Merging Gaussians introduces approximation error. [K-wise merging](5440-k-wise-moment-matched-merging-with-l2-optimal-amplitude.md) minimizes L2 error [S1, unofficial].
*   **Boundary Artifacts:** In tiled methods, overlap must be handled carefully to avoid double-counting or seams [S1, unofficial].
*   **Precision:** Quantization of positions (16-bit) and SH coefficients (INT6/FP16) is common for compression [S1, unofficial].

## Subtopics

### [K-wise moment-matched merging with L2-optimal amplitude](5440-k-wise-moment-matched-merging-with-l2-optimal-amplitude.md)
A LOD technique that reduces splat count by grouping $K$ splats into one representative splat. It matches statistical moments and optimizes amplitude to minimize L2 error. Unlike [MIP-Splatting](6063-mip-splatting.md) which handles aliasing, or [CityGaussian](6143-citygaussian-vastgaussian-dogaussian.md) which uses spatial clustering, this method focuses on substitutive merging.

### [MIP-Splatting](6063-mip-splatting.md)
An extension of 3DGS that introduces 2D and 3D Gaussian filters to improve robustness to aliasing from resolution or distance changes. It eliminates dilation/erosion artifacts by replacing simple dilation with explicit frequency control.

### [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md)
Block-based partitioning methods for large-scale urban scenes. They divide the environment into tiles, train independent 3DGS models per tile, and merge them at render time. They solve memory limits by decoupling scene complexity from GPU memory, using explicit spatial tiling with overlap handling.

### Hierarchical 3DGS
Uses an octree-like tree of clusters; renders only the relevant LoD; achieves 99% reduction at far distances [S1, unofficial]. A cluster-based LoD method developed by Inria, capable of rendering large scenes like Mont Saint-Michel interactively [S1, unofficial].

### Octree-GS
A hierarchical LoD method that builds an octree over the scene, associating each node with a merged Gaussian that summarizes the radiance and opacity of its children [S2, unofficial].

### LODGE
A level-of-detail framework for large-scale 3D Gaussian Splatting that enables efficient rendering by dynamically selecting appropriate detail levels [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Hierarchical methods reduce the number of splats rendered at a distance, improving frame rate. Traversal overhead is minimal compared to the savings in rasterization [S1, S2, unofficial].
*   **Memory Complexity:** Hierarchical and tiled methods significantly reduce memory usage by storing only relevant LODs or partitioning data [S1, S2, unofficial]. Compression (quantization) further reduces size by 8–20x [S1, unofficial].
*   **Performance:** Enables real-time rendering of city-scale scenes (square kilometers) [S1, S2, unofficial].
*   **Strengths:** Scalability to large scenes, reduced memory footprint, improved visual quality at varying distances (MIP-Splatting) [S1, S2, unofficial].
*   **Limitations:** Increased complexity in training and data structure management; potential for popping artifacts if LOD transitions are not smooth [S1, S2, unofficial].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1, S2, unofficial]. The scalability and aliasing issues of the original method led to the development of LOD techniques. Hierarchical 3DGS was developed by Inria (SIGGRAPH'24) [S1, unofficial]. Octree-GS and Scaffold-GS were introduced to handle large scenes using hierarchical structures [S2, unofficial]. MIP-Splatting addresses frequency aliasing [S1, unofficial]. CityGaussian and VastGaussian target urban-scale rendering [S1, S2, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
