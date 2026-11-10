# Semantic and Language Embeddings

Also called: Gaussian Grouping, SAM-guided Gaussian Grouping

## Overview
This family extends 3D Gaussian Splatting by attaching high-dimensional semantic or language feature vectors to each Gaussian, enabling open-vocabulary segmentation, phrase grounding, and visual question answering via real-time alpha-blended feature maps [S2, unofficial]. It differs from sibling methods by prioritizing semantic understanding over geometric fidelity or appearance; while [3D Gaussian Splatting MVP](3410-3d-gaussian-splatting-mvp.md) focuses on minimal viable rendering and [Analytic-Splatting](8084-analytic-splatting.md) on mathematical precision, this approach leverages the rasterizer as a feature distillation pipeline for vision-language models [S2, unofficial].

## Shared principles
The core mechanism involves storing a feature vector $\mathbf{f}_i$ (of dimension $D$, typically 512 or 768) alongside the standard Gaussian parameters (position $\mathbf{x}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, spherical harmonics coefficients) for each Gaussian $i$ [S2, unofficial]. During rendering, these features are accumulated per pixel using alpha-blending, producing a 2D feature map $\mathbf{F}(p)$ that can be compared against text embeddings or decoded by a segmentation head [S2, unofficial].

The feature blending uses the same alpha-compositing structure as standard color rendering [S2, unofficial]:

$$
\mathbf{F}(p) = \sum_{i \in \mathcal{O}} \mathbf{f}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
$$

where:
- $\mathcal{O}$ is the set of Gaussians sorted by depth (front-to-back) that overlap pixel $p$ [S2, unofficial].
- $\mathbf{f}_i$ is the feature vector of Gaussian $i$ [S2, unofficial].
- $\alpha_i$ is the opacity of Gaussian $i$ [S2, unofficial].
- The term $\prod_{j=1}^{i-1} (1 - \alpha_j)$ represents the accumulated transparency from all Gaussians $j$ rendered before $i$ in the sorted order [S2, unofficial].

The resulting $\mathbf{F}(p)$ is a vector in $\mathbb{R}^D$ [S2, unofficial]. To enable semantic tasks, $\mathbf{F}(p)$ is compared against text embeddings $\mathbf{t}_k$ (e.g., from CLIP) using cosine similarity or dot product to determine class labels or phrase grounding scores [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def render_feature_map(gaussians, camera, image_width, image_height):
    """
    Render a feature map from 3D Gaussians.
    gaussians: List of Gaussian objects with attributes:
        - x: (N, 3) position
        - cov2d: (N, 2, 2) 2D covariance matrix
        - alpha: (N,) opacity
        - f: (N, D) feature vector
    camera: Camera object with projection and view matrices
    """
    # 1. Project Gaussians to 2D screen space
    # Compute 2D mean m and covariance cov2d for each Gaussian
    # Projection and covariance computation handled by parent rasterizer [Parent]
    projected_gaussians = gaussians  # Placeholder for projected data structure

    # 2. Initialize output feature map
    D = gaussians.f.shape[1]
    feature_map = torch.zeros((image_height, image_width, D), device=gaussians.f.device)

    # 3. Rasterize and accumulate features
    # For each pixel, iterate over overlapping Gaussians sorted by depth
    for y in range(image_height):
        for x in range(image_width):
            # Get list of Gaussians overlapping pixel (x, y)
            # Sort them by depth (front-to-back)
            # Tile-based overlap retrieval handled by parent rasterizer [Parent]
            overlapping = get_overlapping_gaussians(x, y, projected_gaussians)
            overlapping.sort(key=lambda g: g.depth)

            accumulated_transparency = 1.0
            pixel_feature = torch.zeros(D, device=gaussians.f.device)

            for g in overlapping:
                # Compute 2D Gaussian weight at pixel (x, y)
                dist = torch.tensor([x - g.m[0], y - g.m[1]])
                inv_cov = torch.inverse(g.cov2d)
                exponent = -0.5 * dist @ inv_cov @ dist
                weight = torch.exp(exponent)

                # Alpha contribution
                alpha_contrib = g.alpha * weight

                # Blend feature: F_new = F_old + f_i * alpha_i * accumulated_transparency
                pixel_feature += g.f * alpha_contrib * accumulated_transparency

                # Update accumulated transparency
                accumulated_transparency *= (1 - alpha_contrib)

                # Early exit if fully opaque
                if accumulated_transparency < 1e-4:
                    break

            feature_map[y, x] = pixel_feature

    return feature_map
```

## Subtopics

### [3D Gaussian Splatting with Feature Vectors](8060-3d-gaussian-splatting-with-feature-vectors.md)
This method extends standard 3D Gaussian Splatting by associating a high-dimensional feature vector $\mathbf{f}_i$ with each Gaussian primitive, enabling the rendering of semantic, linguistic, or visual feature maps alongside color. It solves the problem of integrating 2D vision-language models (e.g., CLIP) or segmentation heads into a 3D representation, allowing for real-time open-vocabulary semantic segmentation and phrase grounding. Unlike standard rendering which outputs RGB, this approach outputs a vector field $\mathbf{F}(p)$ per pixel, preserving semantic consistency through alpha-blending.

### [Text Embedding Generation with Directional Views](8239-text-embedding-generation-with-directional-views.md)
Encodes text prompts into latent embeddings using a frozen CLIP text encoder, and generates directional view embeddings by appending view descriptors to prompts. Implementation details for text embedding generation are not given in the sources.

### [LSeg Feature Encoding](12078-lseg-feature-encoding.md)
Not given in the sources.

### Language Embedded 3D Gaussians
Expands feature blending to full scene understanding, encoding object-level embeddings for phrase grounding in addition to pixel-level features [S2, unofficial].

### LangSplat
Pioneers open-vocabulary semantic segmentation in real time by rendering feature maps that are compared against text embeddings from CLIP [S2, unofficial].

### 3D Vision-Language GS
Projects vision-language models into the 3DGS framework, training a joint embedding space where rendered feature maps can be queried with natural language [S2, unofficial]. Enables tasks like 3D visual question answering and zero-shot semantic reasoning [S2, unofficial].

## Background
The concept of attaching semantic features to 3D Gaussians emerged as a natural extension of 3D Gaussian Splatting's ability to render high-dimensional data [S2, unofficial]. Feature-3DGS and LangSplat were among the first to demonstrate that 2D vision-language models (like CLIP) could be distilled into 3D Gaussians for open-vocabulary segmentation [S2, unofficial]. Language Embedded 3D Gaussians and 3D Vision-Language GS further expanded this to support phrase grounding and visual question answering [S2, unofficial]. The "Gaussian Grouping" approach, supervised by SAM (Segment Anything Model) or video tracking, provides an alternative method for assigning per-Gaussian instance IDs, focusing on instance segmentation rather than open-vocabulary semantics [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
