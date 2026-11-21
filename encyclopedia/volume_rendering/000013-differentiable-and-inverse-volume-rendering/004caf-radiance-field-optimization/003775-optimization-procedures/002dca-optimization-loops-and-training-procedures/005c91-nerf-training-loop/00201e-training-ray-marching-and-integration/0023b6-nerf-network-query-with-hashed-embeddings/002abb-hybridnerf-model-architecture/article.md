# HybridNeRF Model Architecture

Also called: HybridNeRF

## Overview
HybridNeRF is a neural radiance field architecture that combines 3D feature grids with 2D triplanes to efficiently represent complex scenes [S1]. It solves the trade-off between the high memory cost of pure 3D voxel representations and the limited geometric expressiveness of pure 2D plane-based methods by using a hybrid query mechanism [S1]. Unlike its sibling [MVS Network Query](7517-mvs-network-query.md) which relies on multi-view stereo consistency, or [VoxNeRF](8865-voxnerf.md) which uses pure voxel grids, HybridNeRF explicitly separates geometric and appearance features into distinct multi-resolution structures [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point coordinate $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$.
*   **Output:** Color $\mathbf{c} \in \mathbb{R}^3$ and volume density $\sigma \in \mathbb{R}$ (or signed distance, depending on the specific variant, but standard HybridNeRF predicts density) [S1].

**Data Structures:**
1.  **3D Feature Grids:** A set of 3D voxel grids at 3 resolution levels: $128^3$, $256^3$, and $512^3$ [S1].
2.  **Triplanes:** A set of 2D feature planes at 7 resolution levels, ranging from $128^2$ to $8,192^2$ [S1].
3.  **MLP:** A Multi-Layer Perceptron that takes the aggregated features and viewing direction to predict color and density [S1].

**Algorithm:**
For a given query point $\mathbf{x}$ and viewing direction $\mathbf{d}$:

1.  **Fetch 3D Features:**
    For each of the 3 resolution levels $l \in \{1, 2, 3\}$:
    *   Retrieve $K=4$ feature vectors from the 3D grid at level $l$ using trilinear interpolation at position $\mathbf{x}$ [S1].
    *   Let $F_{3D}^{(l)}$ be the concatenated features from this level.

2.  **Fetch Triplane Features:**
    For each of the 7 resolution levels $l \in \{1, \dots, 7\}$:
    *   Retrieve $K=4$ feature vectors from the triplanes at level $l$ using bilinear interpolation at the projection of $\mathbf{x}$ onto the planes [S1].
    *   Let $F_{2D}^{(l)}$ be the concatenated features from this level.

3.  **Aggregate Features:**
    *   Sum the features across all 3D levels: $F_{3D} = \sum_{l=1}^{3} F_{3D}^{(l)}$ [S1].
    *   Sum the features across all triplane levels: $F_{2D} = \sum_{l=1}^{7} F_{2D}^{(l)}$ [S1].
    *   *Note:* The method uses summation instead of concatenation for levels within the same structure to reduce dimensionality before combining structures [S1].

4.  **Combine Structures:**
    *   Concatenate the summed 3D features and summed 2D features: $F_{total} = \text{Concat}(F_{3D}, F_{2D})$ [S1].
    *   The resulting input dimension to the MLP is $4K = 16$ (since $K=4$ features per level are summed, and the sum preserves the feature dimensionality, but the text specifies the final MLP input is 16-dimensional, implying the feature dimension per level is 4, or the sum results in a 16-dim vector. Given "K=4 features per level" and "4K=16", it implies the feature vector size per level is 4, and summing them results in a vector of size 4? No, if $K=4$ features are fetched, and they are summed, the dimension remains the feature dimension. If the final input is 16, and there are two sources (3D and 2D), each contributes 8? The text says "concatenate the summed features... to obtain a 4K=16-dimensional MLP input". With $K=4$, $4K=16$. This implies the summed feature vector from 3D grids is 8-dim and from triplanes is 8-dim? Or perhaps $K$ refers to the number of features *per level* being summed into a single vector? Let's stick to the explicit output dimension: 16 [S1].

5.  **Encode Viewing Direction:**
    *   Encode the viewing direction $\mathbf{d}$ using spherical harmonics up to the 4th degree [S1].
    *   Let $\mathbf{d}_{SH}$ be the resulting directional encoding vector.

6.  **MLP Prediction:**
    *   Concatenate $F_{total}$ and $\mathbf{d}_{SH}$ (or pass them through separate branches as per standard NeRF, but HybridNeRF typically feeds the spatial features into the MLP and direction as auxiliary input) [S1].
    *   Pass through the MLP to predict color $\mathbf{c}$ and density $\sigma$ [S1].

**Pitfalls:**
*   **Aliasing:** Due to the multi-resolution feature grid, aliasing artifacts can occur. This is handled similarly to VR-NeRF [S1].

## Subtopics

### Geometry Composition by Segmented Sampling
A variant for inserting objects into scenes. It transforms sampling points from scene coordinates to object coordinates using the object's insertion position (translation and rotation) [S2]. It composites density as the normalized sum of densities from the scene and object at the same position, and composites reflectance as a weighted sum of reflectance values weighted by voxel particle densities [S2].

### Neural Proxy Rendering with Mixed Samplers
A rendering pipeline that mixes a neural proxy sampler $S_p$ for animated objects and a neural scene sampler $S_c$ for static elements [S3, unofficial]. It separates particles into a proxy set $X_p$ and scene set $X_c$. For each ray, it finds the convex set bounded by intersections with the proxy mesh using triangle intersection methods to determine Barycentric coordinates, then samples the neural texture locally [S3, unofficial].

### E2NeRF
Learns sharp neural 3D representations from blurred images and event data by combining bio-inspired event camera and standard RGB camera inputs [S4]. It introduces blur rendering loss and event rendering loss to guide the network [S4].

### LU-NeRF
Jointly estimates camera pose and scene representation by synchronizing locally unlocalized NeRFs using a local-to-global learning framework [S4].

### StructNeRF
Enhances new view creation from limited inputs in indoor settings by integrating structural indicators into self-guided depth estimation and applying planar coherence loss [S4].

### SHERF
A generalizable human NeRF model for recovering animatable 3D humans from a single input image, using conditional implicit maximum likelihood estimation (cIMLE) to address depth estimation uncertainty [S4].

### IR-NeRF
A NeRF training method without camera poses that improves robustness by introducing implicit pose regularization via a scene codebook [S4].

### SparseNeRF
Synthesizes new views from finite perspectives by extracting depth priors from pre-trained depth models or rough depth maps, introducing local depth ordering and spatial continuity constraints [S4].

## Operating characteristics
*   **Memory:** Uses a hybrid of 3D grids and 2D triplanes. The 3D grids are at 3 levels ($128^3, 256^3, 512^3$) and triplanes at 7 levels ($128^2$ to $8,192^2$) [S1].
*   **Performance:** Not given in the sources.
*   **Strengths:** Efficient representation by leveraging the compactness of 2D planes for appearance and 3D grids for geometry, reducing memory compared to pure 3D methods [S1].
*   **Limitations:** Requires careful handling of aliasing due to multi-resolution grids [S1].

## Background
HybridNeRF was introduced in 2024 by the authors of "HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces" [S1]. It builds on the concept of hybrid representations to balance efficiency and quality in neural radiance fields.

## Sources
[S1] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
[S2] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
[S3] Neural Proxy: Empowering Neural Volume Rendering for Animation, https://doi.org/10.2312/pg.20211384, fetched 2026-10-26 (unofficial)
[S4] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
