# Masked-Rasterization
## Overview
Masked-Rasterization is a technique that allows probabilistically existing Gaussians to receive gradients during the blending process, even when they are masked out of the final image [S1]. It solves the problem of "stale" Gaussians in pruning pipelines by ensuring that unsampled entities can still update their existence probability based on their virtual contribution, rather than being permanently discarded due to zero alpha filtering [S1]. It differs from sibling methods like [Optimal Gaussian Splatting](3293-optimal-gaussian-splatting.md) by applying masks during the blending stage rather than modifying Gaussian attributes (opacity/scale) prior to rasterization, which prevents the loss of gradient flow [S1].

## How it works
**Inputs:**
- A set of 3D Gaussians $G_i$, each defined by center, scale, rotation, and an existence probability $p_i$ [S1].
- A differentiable mask $m_i \in [0, 1]$ sampled for each Gaussian based on $p_i$ [S1].
- Camera parameters for projection [G].

**Outputs:**
- Rendered color image [G].
- Gradients for Gaussian attributes (for sampled Gaussians) and mask values (for all Gaussians) [S1].

**Data Structures:**
- **Gaussian Attributes:** Standard 3DGS attributes (position $\mathbf{t}$, scale $\mathbf{s}$, rotation $\mathbf{q}$, opacity $\alpha$, color $\mathbf{c}$) [G].
- **Mask Array:** A 1D array of floats $m_i$ aligned with the Gaussian list [S1].

**Steps:**
1. **Standard Splatting:** For each Gaussian $G_i$, compute the projected 2D covariance and alpha value $\alpha_i$ using standard 3DGS rasterization formulas based on center, scale, and rotation [S1].
2. **Alpha Filtering:** Filter out splats where $\alpha_i = 0$ [S1]. All remaining splats (both sampled and masked) proceed to the blending stage [S1].
3. **Masked Blending:** For each pixel, iterate through the sorted splats. Apply the mask $m_i$ to the transmittance and color accumulation as follows:
   - Update transmittance: $T_{i} = T_{i-1} \cdot (1 - m_i \cdot \alpha_i)$
   - Accumulate color: $C = \sum_{i} m_i \cdot \alpha_i \cdot \mathbf{c}_i \cdot \prod_{j < i} (1 - m_j \cdot \alpha_j)$
   *(Note: The sources cite Eq 3 and Eq 4 for the exact transmittance and color evolution, but the text describes the application points as transmittance attenuation and color accumulation [S1]. The standard 3DGS blending is modified by scaling the effective alpha by the mask $m_i$ [S1].)*
4. **Gradient Propagation:**
   - For sampled Gaussians ($m_i \approx 1$): Gradients flow to standard attributes ($\mathbf{t}, \mathbf{s}, \mathbf{q}, \alpha, \mathbf{c}$) and the mask [S1].
   - For masked Gaussians ($m_i \approx 0$): Gradients do **not** flow to standard attributes (position, scale, rotation, opacity) because the effective alpha contribution is zero [S1]. However, gradients **do** flow to the mask $m_i$ [S1]. This allows the existence probability $p_i$ to be updated based on the error signal, even if the Gaussian did not visually contribute to the pixel [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def masked_rasterization(gaussians, masks, image_width, image_height):
    splats = []
    for i, g in enumerate(gaussians):
        # Compute projected 2D covariance and alpha_i from center, scale, rotation [G]
        alpha_i = 0.5  # Standard 3DGS projection result
        if alpha_i == 0:
            continue
        splats.append({
            'index': i,
            'alpha': alpha_i,
            'color': g.color,
            'mask': masks[i],
            'depth': g.depth,
            'screen_coords': g.screen_coords
        })
    
    splats.sort(key=lambda s: s['depth'])
    
    color_buffer = [[0.0, 0.0, 0.0] for _ in range(image_height)]
    transmittance_buffer = [[1.0] for _ in range(image_height)]
    
    for splat in splats:
        px, py = splat['screen_coords']
        alpha = splat['alpha']
        mask = splat['mask']
        color = splat['color']
        
        current_T = transmittance_buffer[py][px]
        effective_alpha = mask * alpha
        
        new_T = current_T * (1 - effective_alpha)
        transmittance_buffer[py][px] = new_T
        
        contribution = mask * effective_alpha * current_T * color
        color_buffer[py][px][0] += contribution[0]
        color_buffer[py][px][1] += contribution[1]
        color_buffer[py][px][2] += contribution[2]
        
    return color_buffer, splats
```

**Parameters:**
- **Mask Sampling:** Masks are sampled from a Bernoulli distribution defined by the existence probability $p_i$. In practice, this is often implemented using the Gumbel-Max trick or straight-through estimator for differentiability [G].
- **Typical Values:** Pruning ratios of 62.4% (Mip-NeRF360), 67.7% (Tanks & Temples), and 75.3% (Deep Blending) are reported with speedups of 2.05×, 2.19×, and 3.16× respectively, with only a 0.02 PSNR decline on the first two datasets [S1].

**Edge Cases and Pitfalls:**
- **Zero Alpha Filtering:** If a Gaussian has zero alpha (due to zero opacity or scale), it is filtered out *before* masked-rasterization. It receives no gradients at all, hindering the update of its existence probability. This is a critical pitfall of previous methods that multiply masks with attributes. Masked-Rasterization avoids this by applying the mask *during* blending, allowing non-zero alpha Gaussians to participate even if masked [S1].
- **Gradient Isolation:** Masked Gaussians must not update their geometric attributes (position, scale, rotation) to avoid instability. The gradient path to these attributes must be blocked for masked splats [S1].

## Subtopics

### MaskGaussian Pipeline
The broader framework where Masked-Rasterization is used. It formulates pruning based on existence probability and uses the rasterization technique to dynamically assess Gaussian importance [S1]. [MaskGaussian](https://doi.org/10.1109/cvpr52734.2025.00072)

### Probabilistic Pruning
The method of sampling masks based on existence probability. This allows for diverse selection of pruned Gaussians over time [S1]. [MaskGaussian](https://doi.org/10.1109/cvpr52734.2025.00072)

## Operating characteristics
- **Time Complexity:** Similar to standard 3DGS rasterization, as the masking operation is a simple multiplication per splat per pixel. The overhead is negligible [G].
- **Memory Complexity:** Requires storing an additional mask value per Gaussian [S1].
- **Performance:** Achieves pruning ratios of 62.4% (Mip-NeRF360), 67.7% (Tanks & Temples), and 75.3% (Deep Blending) with speedups of 2.05×, 2.19×, and 3.16× respectively, with only a 0.02 PSNR decline on the first two datasets [S1].
- **Strengths:** Prevents "stale" Gaussians from becoming outdated by allowing them to receive gradients for their existence probability. Superior to multiplying masks with opacity/scale [S1].
- **Limitations:** Requires careful implementation to ensure gradients do not flow to geometric attributes for masked Gaussians [S1].

## Background
Masked-Rasterization was introduced in the **MaskGaussian** paper by researchers at Shanghai AI Laboratory, presented at CVPR 2025 [S1]. It addresses the limitations of previous pruning methods (e.g., [4, 17, 34]) that multiplied masks with Gaussian attributes, which caused zero alpha filtering and blocked gradient flow. The method models Gaussians as probabilistic entities, allowing for dynamic assessment of their contribution to the evolving scene [S1].

## Sources
[S1] MaskGaussian: Adaptive 3D Gaussian Representation from Probabilistic Masks, https://doi.org/10.1109/cvpr52734.2025.00072, fetched 2026-10-30
