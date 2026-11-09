# Multi-Layer Surface Compositing

Also called: Coverage from Accumulated Weight, Coverage-Based Compositing, Front-to-back residual compositing, Layer-based compositing, Multi-layer compositing with coverage alpha

## Overview
Multi-Layer Surface Compositing converts accumulated unnormalized Gaussian weights into per-layer coverage opacities to composite surface layers front-to-back, replacing threshold-based clamping with a smooth, monotonically increasing mapping [S1, unofficial]. It solves the problem of jagged, aliased edges and incorrect blending in surface splatting by treating each layer as a partially covering opaque surface, allowing residual transmittance to reveal deeper layers [S1, unofficial]. Unlike the single-layer ternary depth test which produces hard transitions, this method yields smooth opacity falloff against the background and gradual transitions in overlap regions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $K$ surface layers detected per pixel. Each layer $k$ has an accumulated unnormalized EWA weight $W_k$ and a normalized color $\hat{C}_k$ [S1, unofficial].
*   **Outputs:** Final pixel color $C_{\text{out}}$ and residual transmittance $\tilde{T}_{K+1}$ [S1, unofficial].

**Data Structures:**
*   **Layer List:** A list of $K$ layers sorted front-to-back (closest to camera to furthest) [S1, unofficial].
*   **Layer Fields:**
    *   $W_k$: Accumulated unnormalized Gaussian weight for layer $k$ [S1, unofficial].
    *   $\hat{C}_k$: Normalized color vector for layer $k$ [S1, unofficial].

**Parameters:**
*   $\gamma$: Gain parameter for the opacity mapping. Typical value: $\gamma = 2\pi$ [S1, unofficial].

**Steps:**
1.  **Compute Coverage Opacity:** For each layer $k=1 \dots K$, convert the accumulated weight $W_k$ into coverage opacity $\alpha_k$ using the mapping derived from the partition-of-unity assumption:
    $$ \alpha_k = 1 - \exp\left(-\gamma \cdot \frac{W_k}{2\pi}\right) $$
    [S1, unofficial]
    *   Note: When $W_k \approx 2\pi$ (full coverage), $\alpha_k \approx 0.998$. When $W_k$ is low (partial coverage), $\alpha_k$ decreases smoothly [S1, unofficial].
2.  **Composite Front-to-Back:** Initialize the residual transmittance $\tilde{T}_1 = 1.0$ [G]. Initialize output color $C_{\text{out}} = \mathbf{0}$ [G].
    For each layer $k=1 \dots K$:
    *   Compute the contribution of layer $k$:
        $$ \text{Contribution}_k = \tilde{T}_k \cdot \alpha_k \cdot \hat{C}_k $$
    *   Add to output:
        $$ C_{\text{out}} \leftarrow C_{\text{out}} + \text{Contribution}_k $$
    *   Update residual transmittance for the next layer:
        $$ \tilde{T}_{k+1} = \tilde{T}_k \cdot (1 - \alpha_k) $$
    [S1, unofficial]
3.  **Final Output:** The final pixel color is $C_{\text{out}}$ [S1, unofficial]. The value $\tilde{T}_{K+1} = \prod_{k=1}^K (1 - \alpha_k)$ represents the fraction of the pixel not covered by any surface, acting as a differentiable soft silhouette mask [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def multi_layer_compositing(layers, gamma=2 * math.pi):
    """
    Composite K layers front-to-back using coverage-based opacity.
    
    Args:
        layers: List of dicts, each containing:
            - 'W': accumulated unnormalized Gaussian weight (float)
            - 'C': normalized color (3-tuple of floats)
        gamma: Gain parameter (float). Default 2*pi.
        
    Returns:
        C_out: Final pixel color (3-tuple of floats)
        T_residual: Residual transmittance (float)
    """
    K = len(layers)
    C_out = (0.0, 0.0, 0.0)
    T_residual = 1.0
    
    for k in range(K):
        W_k = layers[k]['W']
        C_k = layers[k]['C']
        
        # Step 1: Convert weight to coverage opacity
        alpha_k = 1.0 - math.exp(-gamma * W_k / (2 * math.pi))
        
        # Step 2: Composite layer
        contribution = (T_residual * alpha_k * C_k[0],
                        T_residual * alpha_k * C_k[1],
                        T_residual * alpha_k * C_k[2])
        C_out = (C_out[0] + contribution[0],
                 C_out[1] + contribution[1],
                 C_out[2] + contribution[2])
        
        # Step 3: Update residual transmittance
        T_residual = T_residual * (1.0 - alpha_k)
        
    return C_out, T_residual
```

**Edge Cases and Pitfalls:**
*   **Partition of Unity Assumption:** The method assumes surfel distribution approximates a partition of unity, which adaptive density control actively enforces [S1, unofficial].
*   **Single Surfel Edge Case:** In single-layer splatting, a pixel at a surface edge with only one surfel has normalized color constant w.r.t position, nullifying gradients [S1, unofficial]. Multi-layer compositing avoids this because low accumulated weight $W_k$ leads to low $\alpha_k$, which modulates the final pixel value through the residual transmittance chain, providing informative gradients [S1, unofficial].
*   **Isolated Surfels:** An isolated surfel detaching from a surface layer forms its own group with low $W_k$, resulting in low $\alpha_k$ [S1, unofficial]. This reduces its contribution, encouraging it to rejoin a coherent surface (self-correction) [S1, unofficial].

## Subtopics

### Silhouette Gradient Flow via Multi-layer Compositing
A mechanism to provide informative gradients to geometric parameters at sparsely covered edges, circumventing the gradient nullification problem of single-layer splatting with Shepard normalization [S1, unofficial]. In single-layer splatting, a pixel at a surface edge with only one surfel produces a normalized color constant w.r.t position/scale, nullifying gradients [S1, unofficial]. Multi-layer compositing ensures that even if normalization flattens the color within a layer, the sub-unity coverage $\alpha_k$ produced by low accumulated weight modulates the final pixel value through the residual transmittance chain, providing gradients to all geometric parameters [S1, unofficial]. It also provides a self-correcting signal: an isolated surfel forming its own group with low accumulated weight has low $\alpha$, reducing its contribution and encouraging it to rejoin a coherent surface [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Compositing is performed entirely within the rasterization kernel, requiring no post-processing pass or multi-buffer readback [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Produces anti-aliased silhouettes with smooth opacity falloff against the background. Correctly blends overlapping surfaces with gradual transitions rather than hard steps [S1, unofficial]. Provides differentiable soft silhouette masks for supervision [S1, unofficial]. Circumvents gradient nullification at edges [S1, unofficial].
*   **Limitations:** Relies on the assumption that surfel distribution approximates a partition of unity, requiring adaptive density control to maintain [S1, unofficial].
*   **When to Use:** When smooth, anti-aliased surface rendering is required, especially at edges and overlaps, and when gradient flow for optimization is critical [S1, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
Originated in the 3DSS (3D Surface Splatting for Inverse Rendering) paper [S1, unofficial]. It builds on the concept of coverage-based opacity mapping [4519-coverage-based-opacity-mapping.md], replacing fixed scaling threshold clamping used in classical A-buffer approaches with a smooth mapping [S1, unofficial]. The method is designed to work with EWA (Elliptical Weighted Average) splatting and adaptive density control mechanisms similar to those in 3DGS [S1, unofficial].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
