# Differentiable Rasterization

Also called: DiffSoup, Direct Differentiable Rasterization of Triangle Soup

## Overview
DiffSoup is a radiance field representation and differentiable rasterization method that reconstructs 3D scenes from multi-view RGB images using a small, unstructured soup of opaque triangles equipped with compact neural textures and binary opacity [S1, unofficial]. It solves the problem of extreme model simplification in radiance field reconstruction, enabling high-fidelity rendering with fewer than 20,000 primitives compared to the millions required by existing methods [S1, unofficial]. It sets itself apart from siblings like [Ray Marching with Differential Diffusion](3547-ray-marching-with-differential-diffusion.md) and [Differentiable Monte Carlo Ray Tracing](8822-differentiable-monte-carlo-ray-tracing.md) by using standard depth-tested rasterization of opaque triangles rather than ray sampling, and from [Soft Rasterizer](6779-soft-rasterizer.md) by avoiding mollifiers (smoothed rasterization) and instead using stochastic opacity masking to differentiate opaque rasterization directly [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view RGB images; initial triangle soup geometry initialized around Structure-from-Motion (SfM) keypoints [S1, unofficial].
*   **Outputs:** Reconstructed 3D scene represented as a triangle soup with learned neural textures (color and opacity functions) and optimized vertex positions [S1, unofficial].

**Data Structures:**
The scene is represented as a triangle soup $T = \{T_t\}_{t=1}^{|T|}$ [S1, unofficial].
*   Each triangle $T_t$ consists of three vertices $v_1, v_2, v_3 \in \mathbb{R}^3$ [S1, unofficial].
*   Per-triangle texture functions:
    *   Color texture $C = \{C_t\}_{t=1}^{|T|}$, where $C_t: \mathbb{R}^2 \to \mathbb{R}^3$ maps barycentric coordinates $b$ to color [S1, unofficial].
    *   Opacity texture $A = \{\alpha_t\}_{t=1}^{|T|}$, where $\alpha_t: \mathbb{R}^2 \to [0, 1]$ maps barycentric coordinates to scalar opacity [S1, unofficial].
*   Rasterization produces an array of fragments $F = \{f_f\}_{f=1}^{|F|}$ per pixel [S1, unofficial].
*   Each fragment $f_f$ encodes the pixel-center sample as a tuple $f_f = (b_f, D_f, t_f)$, where:
    *   $b_f \in \mathbb{R}^2$ are barycentric coordinates [S1, unofficial].
    *   $D_f$ is the fragment depth [S1, unofficial].
    *   $t_f$ is the triangle index [S1, unofficial].

**Parameters:**
*   **Opacity Threshold:** Binary opacity threshold is 0.5. Fragments with opacity $\alpha(b_f) > 0.5$ are kept; others are discarded [S1, unofficial].
*   **Triangle Count:** Typically fewer than 20,000 triangles [S1, unofficial].

**Steps:**
1.  **Initialization:** Initialize the triangle soup vertices around Structure-from-Motion (SfM) keypoints [S1, unofficial]. Initialize color and opacity textures (e.g., neural network parameters for $C_t$ and $\alpha_t$) [S1, unofficial].
2.  **Camera Space Transformation:** Transform triangle vertices into camera space [S1, unofficial].
3.  **Depth-Tested Rasterization with Binary Opacity:**
    *   Perform standard rasterization to generate fragments $F$ for all triangles intersecting each pixel [S1, unofficial].
    *   For each pixel, identify the foremost fragment $f^*$ by minimizing depth $D_f$ among fragments satisfying the binary opacity condition:
        $$ f^* = \arg \min_{f \in \{f ; \alpha(b_f) > 0.5\}} D_f $$
        [S1, unofficial]
4.  **Color Computation:** Compute the output pixel color $\hat{C}$ using the color texture of the selected fragment:
    $$ \hat{C} = C_{t_{f^*}}(b_{f^*}) $$
    [S1, unofficial]
    *Note: Shorthands $C_f$ and $\alpha_f$ are used for $C_{t_f}$ and $\alpha_{t_f}$ respectively [S1, unofficial].*
5.  **Neural Texture Mapping:** The rasterization produces a high-dimensional color-feature image, which is mapped to final RGB values using a small shared Multi-Layer Perceptron (MLP) trained to minimize error against ground-truth images [S1, unofficial].
6.  **Stochastic Opacity Masking for Training:** To handle discontinuities in opaque rasterization and enable gradient flow:
    *   Interpret every point in space as an infinitesimal opaque surface occurring with probability equal to its opacity [S1, unofficial].
    *   Compute the stochastic surface loss $L_{sum}$:
        $$ L_{sum} = \sum_i w_i L_1(C_i) $$
        where $w_i = \bar{\alpha}_i \alpha_i$, $C_i$ and $\alpha_i$ are the color and alpha values at the $i$-th sample, and $\bar{\alpha}_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the cumulative transmittance [S1, unofficial].
    *   This loss yields robust gradient signals under severe occlusion while producing the exact pixel color [S1, unofficial].
7.  **Optimization:**
    *   Optimize opacity parameters through stochastic opacity masking [S1, unofficial].
    *   Update vertex positions using edge gradients for opaque triangles [S1, unofficial].
    *   Update neural texture parameters (color/opacity functions) via backpropagation of the loss [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class DiffSoupRenderer:
    def __init__(self, triangles, color_net, opacity_net, threshold=0.5):
        """
        triangles: Tensor of shape (N, 3, 3), N triangles, 3 vertices, 3 coords.
        color_net: nn.Module mapping barycentric coords to high-dim features.
        opacity_net: nn.Module mapping barycentric coords to scalar opacity.
        threshold: Binary opacity threshold (default 0.5).
        """
        self.triangles = triangles
        self.color_net = color_net
        self.opacity_net = opacity_net
        self.threshold = threshold
        self.feature_to_rgb = nn.Linear(color_net.out_features, 3)

    def transform_to_camera_space(self, triangles, camera_matrix):
        """Transform vertices to camera space using a 4x4 projection matrix."""
        verts = triangles.view(-1, 3)
        cam_verts = (verts @ camera_matrix[:3, :3].T) + camera_matrix[:3, 3]
        return cam_verts.view(-1, 3, 3)

    def rasterize(self, cam_triangles, image_size):
        """
        Standard triangle rasterization. Returns fragment buffers for all triangles.
        all_bary: (H, W, N, 2) barycentric coords
        all_depth: (H, W, N) fragment depths
        all_tri_idx: (H, W, N) triangle indices
        """
        H, W = image_size
        N = cam_triangles.shape[0]
        all_bary = torch.zeros(H, W, N, 2, device=cam_triangles.device)
        all_depth = torch.full((H, W, N), float('inf'), device=cam_triangles.device)
        all_tri_idx = torch.arange(N, device=cam_triangles.device).unsqueeze(0).unsqueeze(0).expand(H, W, -1)
        
        # Standard rasterization loop: for each triangle, compute screen-space coverage,
        # barycentric coordinates, and depth at each pixel center.
        for t in range(N):
            v0, v1, v2 = cam_triangles[t]
            # Compute bounding box, rasterize triangle, fill all_bary and all_depth
            # where the triangle covers the pixel. (Standard geometric rasterization)
            pass
        return all_bary, all_tri_idx, all_depth

    def forward(self, camera_matrix, image_size, gt_images):
        # 1. Transform to camera space
        cam_triangles = self.transform_to_camera_space(self.triangles, camera_matrix)
        
        # 2. Rasterize to get all fragments
        all_bary, all_tri_idx, all_depth = self.rasterize(cam_triangles, image_size)
        
        # 3. Evaluate opacity for all fragments
        all_opacity = self.opacity_net(all_bary)  # (H, W, N)
        
        # 4. Apply binary opacity threshold for rendering
        valid_mask = all_opacity > self.threshold
        depths_for_selection = all_depth.clone()
        depths_for_selection[~valid_mask] = float('inf')
        best_depth, best_k_idx = torch.min(depths_for_selection, dim=2)  # (H, W)
        
        # Gather best fragment attributes
        best_k_idx_expanded = best_k_idx.unsqueeze(-1).unsqueeze(-1).expand(-1, -1, -1, 2)
        best_bary = torch.gather(all_bary, 2, best_k_idx_expanded).squeeze(2)  # (H, W, 2)
        best_tri_idx = torch.gather(all_tri_idx, 2, best_k_idx.unsqueeze(-1)).squeeze(2)  # (H, W)
        
        # 5. Compute rendered color
        color_features = self.color_net(best_bary)  # (H, W, C_feat)
        rendered_rgb = self.feature_to_rgb(color_features)  # (H, W, 3)
        
        # 6. Compute stochastic surface loss for training
        # Sort fragments by depth for each pixel
        sorted_depth, sorted_indices = torch.sort(all_depth, dim=2, ascending=True)
        sorted_opacity = torch.gather(all_opacity, 2, sorted_indices).squeeze(2)  # (H, W, N)
        sorted_features = torch.gather(color_features.unsqueeze(2), 2, sorted_indices.unsqueeze(-1).expand(-1,-1,-1,color_features.shape[-1])).squeeze(2)
        sorted_rgb = self.feature_to_rgb(sorted_features)  # (H, W, N, 3)
        
        # Compute cumulative transmittance: alpha_bar_i = prod_{j=1}^{i-1} (1 - alpha_j)
        one_minus_alpha = 1.0 - sorted_opacity
        cum_transmittance = torch.cumprod(one_minus_alpha, dim=2)
        alpha_bar = torch.cat([torch.ones_like(sorted_opacity[:, :, :1]), cum_transmittance[:, :, :-1]], dim=2)
        
        # Weights: w_i = alpha_bar_i * alpha_i
        weights = alpha_bar * sorted_opacity  # (H, W, N)
        
        # Ground truth expansion
        gt_expanded = gt_images.unsqueeze(2).expand_as(sorted_rgb)
        l1_per_sample = torch.abs(sorted_rgb - gt_expanded).mean(dim=3)  # (H, W, N)
        
        # Stochastic loss: L_sum = sum_i w_i L1(C_i)
        loss = (weights * l1_per_sample).sum()
        
        return loss, rendered_rgb
```

**Edge Cases and Pitfalls:**
*   **Opaque Triangles:** Opaque triangles fully occlude surfaces behind them and produce discontinuous color transitions at silhouettes, impeding smooth gradient propagation [S1, unofficial]. DiffSoup handles this via stochastic opacity masking and extended silhouette differentiation [S1, unofficial].
*   **Discontinuities:** Directly optimizing opaque rasterization is difficult due to discontinuities in occlusion and visibility transitions [S1, unofficial].
*   **Mollifier Avoidance:** Unlike previous methods, DiffSoup does not rely on a mollifier (smoothed rasterization) during optimization, which can suppress high-frequency appearance details or require careful scheduling of blur strength [S1, unofficial].

## Subtopics

### [Soft Rasterizer](6779-soft-rasterizer.md)
The Soft Rasterizer is a differentiable rendering method that replaces the deterministic, binary visibility test of traditional rasterization with a probabilistic aggregation of all projected triangles, including occluded ones. By treating triangles as probabilistic clouds and blending their contributions based on screen-space distance and depth, it ensures that pixel color is a smooth, differentiable function of vertex positions, allowing gradients to flow through occlusions and dis-occlusions. It is distinguished from siblings like [Ray Marching with Differential Diffusion](3547-ray-marching-with-differential-diffusion.md) by its rasterization-based approach rather than ray sampling, and from [HardSoftRas](10744-hardsoftras.md) by its purely soft, non-hybrid depth blending which avoids pseudo-occlusion artifacts at the cost of potential transparency artifacts.

### [Generalized Differentiable Renderer (GenDR)](11684-generalized-differentiable-renderer-gendr.md)
GenDR is a generalized differentiable mesh renderer that models the forward rendering process under a probabilistic perturbation defined by a probability distribution with density $f$ and a temperature/scale parameter $\tau > 0$, making the renderer continuous everywhere and differentiable almost everywhere. It computes pixel occupancy/coverage by aggregating face contributions using a sigmoid function (CDF of a probability distribution) and a T-conorm for aggregation. Unlike hard renderers which are discrete and non-differentiable, GenDR provides meaningful gradients for gradient-based optimization tasks such as shape optimization. It generalizes existing differentiable renderers like SoftRas and DIB-R by allowing an array of different smoothing distributions.

### [Differentiable Mesh Rendering](18490-differentiable-mesh-rendering.md)
Differentiable mesh rendering is a family of techniques that make the traditional triangle rasterization or ray-tracing pipeline differentiable, enabling gradient-based optimization of 3D scene parameters (geometry, textures, camera poses) directly from 2D image supervision. It solves the problem of non-differentiable visibility changes and discrete rasterization steps by introducing soft approximations or analytic gradient estimators. What sets it apart from its siblings is the specific handling of mesh primitives: [Soft Rasterizer](6779-soft-rasterizer.md) and [Generalized Differentiable Renderer (GenDR)](11684-generalized-differentiable-renderer-gendr.md) focus on probabilistic or generalized soft rasterization, while [Stochastic Opacity Masking](20635-stochastic-opacity-masking.md) addresses stochastic visibility; differentiable mesh rendering encompasses both rasterization-based approaches (like OpenDR, NMR, PyTorch3D) and physically-based ray-tracing approaches (like Mitsuba 2, nvdiffrast) that operate on explicit mesh geometry.

### [Stochastic Opacity Masking](20635-stochastic-opacity-masking.md)
Stochastic Opacity Masking is a differentiable rasterization technique that enables the evaluation of stochastic surface losses within a conventional, non-sorting depth-tested rasterization pipeline. It solves the problem that standard object-order rasterizers do not produce depth-sorted fragments per pixel, which makes direct evaluation of volumetric blending losses impossible. By simulating a discrete stochastic process during fragment generation, it selects a single visible surface point per pixel, allowing gradients to be computed for that fragment and others based on their depth relationship to the selected fragment. It differs from [Soft Rasterizer](6779-soft-rasterizer.md) by selecting a single hard fragment rather than blending all contributions, and from [Generalized Differentiable Renderer (GenDR)](11684-generalized-differentiable-renderer-gendr.md) by relying on a probabilistic thresholding mechanism rather than continuous relaxation of the depth test.

## Operating characteristics
*   **Time and Memory Complexity:** DiffSoup uses standard depth-tested rasterization, which is highly optimized in traditional graphics pipelines, enabling interactive rendering on consumer-grade laptops and mobile devices [S1, unofficial]. It requires significantly less memory than methods using millions of primitives, as it typically uses fewer than 20,000 triangles [S1, unofficial].
*   **Measured Performance:** DiffSoup reconstructs high-fidelity scenes using typically fewer than 20,000 triangles, whereas existing methods rely on millions of primitives [S1, unofficial]. It produces more detail-preserving novel views than baselines [S1, unofficial].
*   **Scaling:** The method scales well to diverse platforms due to its reliance on standard rasterization and small number of primitives [S1, unofficial].
*   **Strengths:** Enables extreme model simplification; stable training without a mollifier; robust gradient signals under severe occlusion; seamless integration into traditional graphics pipelines; interactive rendering on consumer hardware [S1, unofficial].
*   **Limitations:** Opaque triangles introduce difficulties in training due to discontinuities at silhouettes and full occlusion, requiring specialized handling (stochastic opacity masking) [S1, unofficial].
*   **When to Use:** When extreme model simplification is required, and interactive rendering on consumer devices is necessary [S1, unofficial].
*   **When Not to Use:** When high-frequency appearance details are critical and might be suppressed by the binary opacity representation, or when the scene requires explicit primitive sorting for complex transparency effects not handled by the stochastic loss [S1, unofficial].

## Background
DiffSoup was introduced in the paper "DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification" [S1, unofficial]. It addresses the challenge of extreme model simplification for radiance field-based view synthesis, aiming to represent a 3D scene using an unstructured soup of a small number of textured triangles [S1, unofficial]. The method is inspired by recent work on implicit surface reconstruction, particularly the stochastic surface interpretation where every point in space is an infinitesimal opaque surface occurring with probability equal to that point's opacity [S1, unofficial]. It extends the silhouette-differentiation method of [41] to handle implicit discontinuities introduced by opacity masking [S1, unofficial].

## Sources
[S1] DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification, https://doi.org/10.48550/arXiv.2603.27151, fetched 2026-10-30 (unofficial)
