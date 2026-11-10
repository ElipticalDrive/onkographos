# 3D Gaussian Splatting with Feature Vectors

Also called: Feature-3DGS, LangSplat, Semantic 3DGS, Vision-Language 3DGS

## Overview
This method extends standard 3D Gaussian Splatting by associating a high-dimensional feature vector $\mathbf{f}_i$ with each Gaussian primitive, enabling the rendering of semantic, linguistic, or visual feature maps alongside color. It solves the problem of integrating 2D vision-language models (e.g., CLIP) or segmentation heads into a 3D representation, allowing for real-time open-vocabulary semantic segmentation and phrase grounding [S1, unofficial]. Unlike standard rendering which outputs RGB, this approach outputs a vector field $\mathbf{F}(p)$ per pixel, preserving semantic consistency through alpha-blending.

## How it works
**Inputs and Outputs:**
- **Input:** A set of $N$ 3D Gaussians, each defined by position $\mathbf{x}_i$, covariance $\Sigma_i$ (or rotation/scale), opacity $\alpha_i$, spherical harmonics coefficients $\mathbf{c}_i$, and a feature vector $\mathbf{f}_i \in \mathbb{R}^D$.
- **Output:** A 2D feature map $\mathbf{F}(p) \in \mathbb{R}^D$ for each pixel $p$ in the image plane.

**Data Structures:**
Each Gaussian $i$ stores:
- $\mathbf{x}_i \in \mathbb{R}^3$: Center position.
- $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (often parameterized by rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$).
- $\alpha_i \in [0, 1]$: Opacity.
- $\mathbf{c}_i$: Spherical harmonics coefficients for color.
- $\mathbf{f}_i \in \mathbb{R}^D$: Feature vector. Typical dimensions $D$ are 512 or 768 [S1, unofficial].

**Rendering Steps:**
1. **Rasterization:** Project each Gaussian to 2D screen space. Compute the 2D covariance $\Sigma'$ and mean $\mathbf{m}$ for each Gaussian.
2. **Sorting:** Sort the overlapping Gaussians $\mathcal{O}$ for each pixel $p$ by depth (front-to-back).
3. **Feature Accumulation:** Compute the accumulated feature vector $\mathbf{F}(p)$ using alpha-blending:

$$
\mathbf{F}(p) = \sum_{i \in \mathcal{O}} \mathbf{f}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
$$

where:
- $\mathcal{O}$ is the set of Gaussians sorted by depth that overlap pixel $p$.
- $\mathbf{f}_i$ is the feature vector of Gaussian $i$.
- $\alpha_i$ is the opacity of Gaussian $i$.
- The term $\prod_{j=1}^{i-1} (1 - \alpha_j)$ represents the accumulated transparency from all Gaussians $j$ rendered before $i$ in the sorted order.

4. **Semantic Decoding:** The resulting $\mathbf{F}(p)$ is compared against text embeddings $\mathbf{t}_k$ (e.g., from CLIP) using cosine similarity or dot product to determine class labels or phrase grounding scores [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
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
    # 1. Project Gaussians to 2D
    # Compute 2D mean m and covariance cov2d for each Gaussian
    # This step depends on the specific 3DGS rasterizer implementation
    projected_gaussians = project_to_screen(gaussians, camera)
    
    # 2. Initialize output feature map
    D = gaussians.f.shape[1]
    feature_map = torch.zeros((image_height, image_width, D), device=gaussians.f.device)
    
    # 3. Rasterize and accumulate features
    # For each pixel, iterate over overlapping Gaussians sorted by depth
    # Note: In practice, this is done via a tile-based rasterizer loop
    for y in range(image_height):
        for x in range(image_width):
            # Get list of Gaussians overlapping pixel (x, y)
            # Sort them by depth (front-to-back)
            overlapping = get_overlapping_gaussians(x, y, projected_gaussians)
            overlapping.sort(key=lambda g: g.depth)
            
            accumulated_transparency = 1.0
            pixel_feature = torch.zeros(D, device=gaussians.f.device)
            
            for g in overlapping:
                # Compute 2D Gaussian weight at pixel (x, y)
                # w = exp(-0.5 * (p - m)^T Sigma^-1 (p - m))
                dist = torch.tensor([x - g.m[0], y - g.m[1]])
                inv_cov = torch.inverse(g.cov2d)
                exponent = -0.5 * dist @ inv_cov @ dist
                weight = torch.exp(exponent)
                
                # Alpha contribution
                alpha_contrib = g.alpha * weight
                
                # Blend feature
                # F_new = F_old + f_i * alpha_i * (1 - accumulated_alpha)
                # Note: The formula uses accumulated transparency from previous steps
                # accumulated_transparency here is prod(1 - alpha_j) for j < i
                pixel_feature += g.f * alpha_contrib * accumulated_transparency
                
                # Update accumulated transparency
                accumulated_transparency *= (1 - alpha_contrib)
                
                # Early exit if fully opaque
                if accumulated_transparency < 1e-4:
                    break
            
            feature_map[y, x] = pixel_feature
            
    return feature_map
```

**Edge Cases and Pitfalls:**
- **Precision:** Feature vectors are typically high-dimensional (512-768). Accumulating many small contributions may lead to precision loss if not using float32 or float16 with care [S1, unofficial].
- **Boundary Conditions:** Gaussians partially outside the image plane must be handled correctly during projection to avoid artifacts.
- **Degenerate Input:** If a Gaussian has zero opacity or zero feature vector, it contributes nothing and can be skipped.

## Subtopics

### Scale-Aware Semantic Feature Gating
A mechanism to modulate rendered semantic features based on scale, using learned or fixed gate vectors derived from sampled scales [S2, unofficial]. It allows the semantic features to adapt to the spatial extent of the Gaussian.
- **Steps:**
  1. Map the distribution of sampled scales to a Gaussian distribution using `self.q_trans(sampled_scales)` [S2, unofficial].
  2. If `scale_aware_dim <= 0` or `scale_aware_dim >= 32`, compute gates by passing sampled scales through `self.scale_gate`: `gates = self.scale_gate(sampled_scales.unsqueeze(-1))` [S2, unofficial].
  3. Otherwise, discretize scales to integers: `int_sampled_scales = ((1 - sampled_scales.squeeze()) * scale_aware_dim).long()`, then lookup pre-computed gates: `gates = self.fixed_scale_gate[int_sampled_scales].detach()` [S2, unofficial].
  4. Repeat rendered features for each sampled scale: `feature_with_scale = rendered_features.unsqueeze(0).repeat([sampled_scales.shape[0], 1, 1, 1])` [S2, unofficial].
  5. Multiply features by gate vectors: `feature_with_scale = feature_with_scale * gates.unsqueeze(-1).unsqueeze(-1)` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for rasterization, where $N$ is the number of Gaussians. The feature accumulation is vectorized and runs at similar speed to color rendering, as the blending logic is identical [S1, unofficial].
- **Memory Complexity:** $O(N \cdot D)$ to store feature vectors, where $D$ is the feature dimension. For $D=512$, this adds significant memory overhead compared to standard 3DGS.
- **Performance:** Real-time performance is maintained due to the efficiency of the rasterizer. However, high-dimensional feature vectors increase memory bandwidth requirements [S1, unofficial].
- **Strengths:** Enables open-vocabulary segmentation, phrase grounding, and semantic understanding without retraining the 3DGS structure.
- **Limitations:** Requires pre-computed or distilled features (e.g., from CLIP). The quality of semantic segmentation depends on the quality of the input features and the accuracy of the 3DGS reconstruction.

## Background
The concept of adding feature vectors to 3D Gaussians was pioneered by Feature-3DGS [102] and LangSplat [103], which demonstrated that 2D vision-model features could be distilled into a 3D representation using the speed of the 3DGS rasterizer [S1, unofficial]. Language Embedded 3D Gaussians [104] expanded this to full scene understanding, encoding object-level embeddings for phrase grounding. Subsequent works like SplatTalk [105] and 3D Vision-Language GS [106] further explored these capabilities [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] yzslab/gaussian-splatting-lightning, https://github.com/yzslab/gaussian-splatting-lightning, fetched 2026-10-27 (unofficial)
