# Normal Regularization

Also called: spatial interpolation of normals

## Overview
Normal Regularization is a technique to alleviate numerical instability (jitter) of surface normals near the Unsigned Distance Function (UDF) zero-level set, specifically at cusps where the function is not first-order differentiable [S1]. It solves the problem of unreliable gradients leading to inaccurate surface reconstruction in differentiable volume rendering by replacing naively sampled normals with interpolated normals from the neighborhood [S1]. It builds on [SDF Normal Estimation via Finite Differences](4619-sdf-normal-estimation-via-finite-differences.md) and is distinguished from siblings like [Lambertian Shading with Gradient Normals](1229-lambertian-shading-with-gradient-normals.md) and [Gradient-Based Surface Normal Estimation](4261-gradient-based-surface-normal-estimation.md) by its specific focus on stabilizing gradients for UDF-based open surface reconstruction during volume rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of points $p(t_i)$ along a ray, the UDF $\Psi(\cdot)$, its derivative $\Psi'(\cdot)$ which returns gradients, and a neighborhood size parameter $K$ [S1].
*   **Output:** A regularized normal vector $\mathbf{n}(p(t_i))$ at the query point [S1].

**Data Structures:**
*   **Point Sequence:** An ordered list of 3D points $p(t_i)$ sampled along the ray, where $t_i$ is the distance along the ray [S1].
*   **Gradient Buffer:** Stores $\Psi'(p(t_i-k))$ for $k \in [1, K]$ [S1].

**Parameters:**
*   **$K$:** Number of neighboring points used for interpolation [S1]. Typical values are not given in the sources.

**Algorithm:**
1.  Identify the query point $p(t_i)$ on the ray where the normal is needed [S1].
2.  Select $K$ preceding points $p(t_{i-k})$ for $k=1, \dots, K$ along the ray [S1].
3.  Compute the squared Euclidean distance between the query point and each neighbor: $w_{i-k} = \|p(t_i) - p(t_{i-k})\|_2^2$ [S1].
4.  Evaluate the UDF gradient at each neighbor: $\mathbf{g}_{i-k} = \Psi'(p(t_{i-k}))$ [S1].
5.  Compute the weighted sum of the gradients: $\mathbf{N}_{\text{sum}} = \sum_{k=1}^K w_{i-k} \mathbf{g}_{i-k}$ [S1].
6.  Compute the sum of the weights: $W_{\text{sum}} = \sum_{k=1}^K w_{i-k}$ [S1].
7.  Calculate the regularized normal: $\mathbf{n}(p(t_i)) = \frac{\mathbf{N}_{\text{sum}}}{W_{\text{sum}}}$ [S1].
8.  Renormalize $\mathbf{n}(p(t_i))$ to unit length if required by the shading model [G].

**Pitfalls:**
*   **Cusps:** Points in the UDF's zero-level set are cusps that are not first-order differentiable, causing gradients of sampled points in the vicinity of the learned surface to be numerically unstable (jittered) [S1].
*   **Boundary Conditions:** The method assumes preceding points are available. If $K$ exceeds the number of available previous samples, the window must be truncated or padded [G].
*   **Weighting:** Using squared distance emphasizes closer points significantly more than linear distance [S1].

## Subtopics

### [Robust Structure Rendering](10494-robust-structure-rendering.md)
Robust Structure Rendering is a methodology for visualizing structures with consistently well-defined boundaries and rapid density changes normal to the boundary, such as bone in CT scans. It addresses the problem of digitization effects and numerical instability in normal estimation by retaining the original scene for scene-space normal estimation while preparing the structure itself via combinations of interpolation, grey transforms, and filtering. It is distinguished from frail structure rendering by its reliance on the original scene's density gradients for normals, assuming the structure satisfies specific robustness conditions.

### Shepard Normalization for Normal Maps
Shepard Normalization for Normal Maps is a post-processing step to produce normal images for evaluation and visualization by Shepard-normalizing per-surfel unit normals within each surface layer and renormalizing the result [S2, unofficial]. The steps are: 1) Shepard-normalize the per-surfel unit normals within each surface layer; 2) Renormalize the resulting vector to unit length [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(K)$ per query point, where $K$ is the number of neighbors [G].
*   **Memory Complexity:** $O(K)$ to store neighbor gradients and weights [G].
*   **Strengths:** Prevents highly bright and dark regions in 2D images that occur with high-quality reconstruction without it, leading to smoother open surface reconstruction [S1].
*   **Limitations:** Adjusting $K$ allows trade-offs between smoothness and geometric detail [S1].
*   **When to use:** When rendering UDF-based surfaces where gradient jitter at the zero-level set causes visual artifacts [S1].
*   **When not to use:** Not given in the sources.

## Background
Normal Regularization was introduced in the context of learning Neural Unsigned Distance Fields with Volume Rendering [S1]. It addresses the inherent instability of gradients at the zero-level set of UDFs, which are non-differentiable at cusps [S1]. The technique leverages the fact that unstable normals only exist near the surface, using offset points to approximate the unstable normal [S1].

## Sources
[S1] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
[S2] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
