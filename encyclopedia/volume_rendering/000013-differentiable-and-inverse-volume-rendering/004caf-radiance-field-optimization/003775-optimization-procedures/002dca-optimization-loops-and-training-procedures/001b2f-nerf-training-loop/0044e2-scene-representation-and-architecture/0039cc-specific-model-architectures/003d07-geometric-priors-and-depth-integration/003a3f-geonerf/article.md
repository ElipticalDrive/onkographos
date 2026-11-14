# GeoNeRF

Also called: GeoNeRF, A model with two parts: a geometric morphology reasoner and a renderer. The reasoner builds a cascaded cost volume for source view sequences. The renderer infers geometric and appearance representations using a Transformer-based attention mechanism and cascaded cost volumes, then renders fine images

## Overview
GeoNeRF is a multi-view stereo-based radiance field optimization method that replaces the standard per-ray MLP evaluation with a global geometric reasoning pipeline [S1, S2]. It constructs a cascaded 3D cost volume from 2D image features to infer geometric and appearance representations, enabling better generalization to new scenes compared to single-view or purely local methods [S1, S2]. It is distinguished from siblings like [SfMNeRF](8664-sfmnerf.md) and [DASNeRF](8668-dasnerf.md) by its explicit use of a cascaded cost volume and Transformer-based feature refinement rather than just depth-consistency loss terms or uniform sampling [S1, S2].

## How it works
**Inputs:** A sequence of $N$ source views (images).
**Outputs:** Rendered images for novel viewpoints.

**Data Structures:**
1.  **2D Image Features:** Extracted from each source view using a pretrained Feature Pyramid Network (FPN) [S1].
2.  **Cascaded 3D Cost Volume:** Constructed via plane sweeping from the 2D features [S1, S2]. This volume encodes geometric correspondence across views.
3.  **Feature Tokens:** For each query point along a ray, two types of tokens are extracted from the cost volume:
    *   One view-independent token.
    *   Multiple view-dependent tokens [S1].

**Algorithm Steps:**
1.  **Feature Extraction:** Extract 2D image features from every source view using a pretrained FPN [S1].
2.  **Cost Volume Construction:** Construct a cascaded 3D cost volume using plane sweeping on the extracted features [S1, S2].
3.  **Token Extraction:** For each of the $N$ query points along a ray, extract one view-independent feature token and multiple view-dependent feature tokens from the cascaded cost volume [S1].
4.  **Transformer Refinement:** Refine the extracted tokens using a Transformer-based attention mechanism [S1, S2].
5.  **Density Prediction:** Refine the $N$ view-independent tokens through an AutoEncoder network. The output is the $N$ density values ($\sigma$) along the ray [S1].
6.  **Color Prediction:** Feed each of the $N$ sets of view-dependent tokens into a separate Multi-Layer Perceptron (MLP) to extract color values ($\mathbf{c}$) [S1].
7.  **Volume Rendering:** Use standard volume rendering to composite the densities and colors into fine images [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def GeoNeRF(source_views):
    # Step 1: Extract 2D features
    # FPN is pretrained; returns feature maps for each view
    features_2d = [FPN(view) for view in source_views]
    
    # Step 2: Construct Cascaded 3D Cost Volume
    # Plane sweeping aggregates features across views and depth hypotheses
    cost_volume = PlaneSweeping(features_2d)
    
    # Step 3: Define Ray and Query Points
    # For a given camera pose, generate rays and sample N points
    rays = GenerateRays(camera_pose)
    query_points = SamplePointsAlongRays(rays, N)
    
    # Step 4 & 5: Extract and Refine Tokens
    # Extract view-independent and view-dependent tokens from cost volume
    view_independent_tokens = ExtractTokens(cost_volume, query_points, type='independent')
    view_dependent_tokens = ExtractTokens(cost_volume, query_points, type='dependent')
    
    # Refine tokens using Transformer
    view_independent_tokens = TransformerRefine(view_independent_tokens)
    view_dependent_tokens = TransformerRefine(view_dependent_tokens)
    
    # Step 6: Predict Densities
    # AutoEncoder processes view-independent tokens to return densities
    densities = AutoEncoder(view_independent_tokens)
    
    # Step 7: Predict Colors
    # MLP processes view-dependent tokens to return colors
    # Note: Each set of view-dependent tokens is fed into an MLP
    colors = [MLP(tokens) for tokens in view_dependent_tokens]
    
    # Step 8: Volume Rendering
    # Compute weighted sum of colors based on densities
    rendered_images = VolumeRender(densities, colors, rays)
    
    return rendered_images
```

**Parameters:**
*   $N$: Number of query points along a ray. Typical value: Not given in the sources.
*   Plane Sweeping Depth Range: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on the quality of the pretrained FPN and the accuracy of the plane sweeping cost volume. Degenerate views (e.g., severe occlusion or lack of texture) may result in poor cost volume construction [S1].
*   **Boundaries:** The visibility is computed as a cumulative density function written as a weighted sum of sigmoid functions [S1]. This formulation must be handled carefully to avoid numerical instability at extreme density values.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Outperforms methods such as pixelNeRF and MVSNeRF on the DTU, NeRF synthetic, and LLF Forward Facing datasets [S1].
*   **Generalization:** Can be pretrained and generalized well to new scenes, and can be further fine-tuned per scene [S1].
*   **Strengths:** Allows for complex occlusion reasoning and extends to RGB-D images [S2].
*   **Limitations:** Not given in the sources.

## Background
GeoNeRF was proposed by Johari et al. in November 2021 [S1, S2]. It is concurrent with LOLNeRF (November 2021) and RegNeRF (December 2021) [S1]. The method was evaluated on the DTU dataset (80 scenes, 49 or 64 calibrated cameras, 1200×1600 resolution images) [S2], the NeRF synthetic dataset, and the LLF Forward Facing dataset [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
