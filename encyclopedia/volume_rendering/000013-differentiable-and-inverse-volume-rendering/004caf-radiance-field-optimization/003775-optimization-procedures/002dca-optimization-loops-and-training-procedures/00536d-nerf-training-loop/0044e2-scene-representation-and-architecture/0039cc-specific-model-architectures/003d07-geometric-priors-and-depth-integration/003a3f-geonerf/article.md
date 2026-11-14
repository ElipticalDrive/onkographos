# GeoNeRF

Also called: GeoNeRF, A model with two parts: a geometric morphology reasoner and a renderer. The reasoner builds a cascaded cost volume for source view sequences. The renderer infers geometric and appearance representations using a Transformer-based attention mechanism and cascaded cost volumes, then renders fine images

## Overview
GeoNeRF is a multi-view stereo-based neural rendering method that extracts 2D image features from every view using a pretrained Feature Pyramid Network (FPN), constructs a cascading 3D cost volume using plane sweeping, and refines features using a Transformer and AutoEncoder [S1, S2]. It differs from coordinate-based methods like NeRF by relying on explicit 3D cost volumes and multi-view feature aggregation rather than implicit MLPs mapping 6D coordinates to radiance [S1, S2]. It is designed to generalize well to new scenes and outperform methods such as pixelNeRF and MVSNeRF on DTU, NeRF synthetic, and LLF Forward Facing datasets [S1].

## How it works
**Inputs:** A sequence of $N$ calibrated source views (images and camera poses).
**Outputs:** Rendered fine images via volume rendering.

**Data Structures:**
1.  **2D Image Features:** Extracted from each source view using a pretrained Feature Pyramid Network (FPN).
2.  **Cascaded 3D Cost Volume:** Constructed via plane sweeping from the 2D features. This volume encodes geometric and appearance information across multiple scales [S1, S2].
3.  **Feature Tokens:** For each of the $N$ query points along a ray, one view-independent token and multiple view-dependent tokens are extracted from the cost volume [S1].

**Steps:**
1.  **Feature Extraction:** Extract 2D image features from every source view using a pretrained FPN [S1].
2.  **Cost Volume Construction:** Construct a cascading 3D cost volume using plane sweeping on the extracted features [S1, S2].
3.  **Token Extraction:** For each of the $N$ query points along a ray, extract one view-independent feature token and multiple view-dependent feature tokens from the cost volume [S1].
4.  **Transformer Refinement:** Refine the extracted tokens using a Transformer-based attention mechanism [S1, S2].
5.  **Density Prediction:** Refine the $N$ view-independent tokens through an AutoEncoder, which returns the $N$ densities along the ray [S1].
6.  **Color Prediction:** Feed the $N$ sets of view-dependent tokens into separate MLPs to extract color [S1].
7.  **Volume Rendering:** Use volume rendering technology to render fine images from the predicted densities and colors [S1, S2].

**Formulas:**
The visibility is computed as a cumulative density function written as a weighted sum of sigmoid functions [S1].
$$
V = \sum w_i \cdot \text{sigmoid}(\dots)
$$
where $w_i$ are weights. Not given in the sources.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def GeoNeRF(source_views, camera_poses):
    # Step 1: Extract 2D image features using pretrained FPN
    # source_views: List of images
    features_2d = []
    for view in source_views:
        features_2d.append(FPN(view))
    
    # Step 2: Construct cascading 3D cost volume using plane sweeping
    cost_volume = PlaneSweep(features_2d, camera_poses)
    
    # Step 3: Extract tokens for N query points along a ray
    # query_points: Array of N 3D points along the ray
    view_independent_tokens = []
    view_dependent_tokens = []
    
    for point in query_points:
        # Extract from cost volume at point
        vi_token = ExtractToken(cost_volume, point, type='view_independent')
        vd_tokens = ExtractToken(cost_volume, point, type='view_dependent')
        view_independent_tokens.append(vi_token)
        view_dependent_tokens.append(vd_tokens)
        
    # Step 4: Refine tokens using Transformer
    # Assuming Transformer processes the sequence of tokens
    refined_vi_tokens = Transformer(view_independent_tokens)
    refined_vd_tokens = [Transformer(vd) for vd in view_dependent_tokens]
    
    # Step 5: Predict densities using AutoEncoder
    # AutoEncoder takes view-independent tokens and outputs densities
    densities = AutoEncoder(refined_vi_tokens)
    
    # Step 6: Predict colors using MLPs
    # Each set of view-dependent tokens goes to an MLP
    colors = []
    for vd_token in refined_vd_tokens:
        colors.append(MLP(vd_token))
        
    # Step 7: Volume Rendering
    # Render fine images using densities and colors
    rendered_image = VolumeRendering(densities, colors)
    
    return rendered_image
```

## Subtopics
Not given in the sources.

## Operating characteristics
GeoNeRF generalizes well to new scenes and can be further fine-tuned to exceed the performance of the baseline NeRF model [S1]. It achieves great results on the DTU, NeRF synthetic, and LLF Forward Facing datasets, outperforming methods such as pixelNeRF and MVSNeRF [S1]. The networks can all be pretrained [S1].

## Background
GeoNeRF was proposed in November 2021 [S1, S2]. It was developed by Johari et al. [S2]. It is used for novel view synthesis and 3D reconstruction [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
