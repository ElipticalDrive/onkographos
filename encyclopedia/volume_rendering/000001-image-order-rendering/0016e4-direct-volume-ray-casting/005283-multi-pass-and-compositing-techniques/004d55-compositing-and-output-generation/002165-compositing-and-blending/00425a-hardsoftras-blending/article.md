# HardSoftRas Blending

Also called: HardSoftRas Shading and Blending

## Overview
HardSoftRas Blending is a pixel-wise compositing strategy for differentiable rendering that distinguishes between edge pixels (silhouettes and vertical surfaces) and non-edge pixels to balance anti-aliasing with texture sharpness [S1]. Unlike SoftRas, which blends all faces in non-edge regions, HardSoftRas retains the most frontal color for non-edges while applying weighted averaging for edges, thereby maintaining sharp textures inside silhouettes and generating far-range gradients around boundaries [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $K$ rasterized buffers. Each buffer $k$ contains a shaded color $C_k$ and a signed distance $dist_k$ from the pixel $p_i$ to the face in that buffer [S1]. A stencil mask $SH$ indicating the most frontal face in the frontal buffer [S1]. Parameters: temperature $\sigma > 0$, edge mask width $\delta$ (typically $\delta = r$, where $r$ is the pixel radius or a scaling factor), and number of buffers $K$ [S1].
- **Outputs:** Final color $I_c$ and silhouette intensity $I_s$ for each pixel [S1].

**Data Structures:**
- $C_k$: RGB color of the $k$-th buffer [S1].
- $dist_k$: Signed distance from pixel $p_i$ to the $k$-th face. The source explicitly removes the square of the distance used in SoftRas to avoid numerical instability in sigmoid computation [S1].
- $SH$: Binary mask (1 if face $k$ is most frontal, 0 otherwise) [S1].
- $E$: Binary edge mask derived from $SH$ [S1].

**Steps:**

1.  **Compute Edge Mask $E$:**
    Apply edge detection and dilation to the stencil mask $SH$ of hard faces in the frontal buffer. The resulting mask $E$ has a width $\delta$ (typically $\delta = r$) [S1].
    $$ E(p_i) = \text{Dilate}(\text{EdgeDetect}(SH), \delta) $$

2.  **Compute Face Probabilities $D_k$:**
    Calculate the probability $D_k$ for each buffer $k$ using a sigmoid function on the signed distance. This replaces the squared distance term found in SoftRas [S1].
    $$ D_k = \text{sigmoid}\left(\frac{dist_k}{\sigma}\right) $$
    where $\sigma$ is the temperature parameter (typically $\sigma = r/7$) [S1].

3.  **Compute Edge Color $I_{ce}$:**
    For pixels identified as edges ($E=1$), compute the blended color using weighted averaging of all buffers based on $D_k$. This provides transparency and far-range gradients [S1].
    $$ I_{ce} = \frac{\sum_{k=1}^{K} D_k C_k}{\sum_{k=1}^{K} D_k} $$

4.  **Compute Non-Edge Color $I_{cne}$:**
    For pixels identified as non-edges ($E=0$), retain the color of the most frontal face. This maintains sharp textures [S1].
    $$ I_{cne} = SH \cdot C_1 $$
    where $C_1$ is the color of the most frontal face (indicated by $SH$) [S1].

5.  **Compute Silhouette Edge Intensity $I_{se}$:**
    For silhouette rendering, compute the binary occupancy blending for edges. This represents the probability that at least one face is present [S1].
    $$ I_{se} = 1 - \prod_{k=1}^{K} (1 - D_k) $$

6.  **Compute Non-Edge Silhouette Intensity $I_{sne}$:**
    For non-edges, use flat intensity (binary occupancy of the frontal face) [S1].
    $$ I_{sne} = SH $$

7.  **Final Composition:**
    Combine edge and non-edge results using the binary mask $E$ [S1].
    $$ I_c = E \cdot I_{ce} + (1 - E) \cdot I_{cne} $$
    $$ I_s = E \cdot I_{se} + (1 - E) \cdot I_{sne} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def hardsoft_ras_blend(buffers, SH, sigma, delta):
    """
    HardSoftRas Blending for differentiable rendering.
    
    Args:
        buffers: List of K buffers. Each buffer is a dict with 'color' (H, W, 3) 
                 and 'dist' (H, W).
        SH: Stencil mask of hard faces in the frontal buffer (H, W). 
            1.0 for the most frontal face, 0.0 otherwise.
        sigma: Temperature parameter for sigmoid.
        delta: Width of the edge mask dilation.
        
    Returns:
        final_color: (H, W, 3)
        final_silhouette: (H, W)
    """
    H, W = SH.shape
    K = len(buffers)
    
    # 1. Compute Edge Mask E
    # Sources do not specify the exact edge detection/dilation algorithm; 
    # standard morphological operations are used [S1].
    E = compute_edge_mask(SH, delta) 
    
    # 2. Compute Probabilities D_k
    D = []
    for k in range(K):
        dist_k = buffers[k]['dist']
        D_k = sigmoid(dist_k / sigma)
        D.append(D_k)
    
    # 3. Compute Edge Color I_ce
    # Weighted average: sum(D_k * C_k) / sum(D_k)
    numerator = [[[0.0, 0.0, 0.0] for _ in range(W)] for _ in range(H)]
    denominator = [[0.0] * W for _ in range(H)]
    
    for k in range(K):
        D_k = D[k]
        C_k = buffers[k]['color']
        for i in range(H):
            for j in range(W):
                d = D_k[i][j]
                numerator[i][j][0] += d * C_k[i][j][0]
                numerator[i][j][1] += d * C_k[i][j][1]
                numerator[i][j][2] += d * C_k[i][j][2]
                denominator[i][j] += d
    
    I_ce = [[[0.0, 0.0, 0.0] for _ in range(W)] for _ in range(H)]
    for i in range(H):
        for j in range(W):
            denom = max(denominator[i][j], 1e-8) # Avoid division by zero
            I_ce[i][j][0] = numerator[i][j][0] / denom
            I_ce[i][j][1] = numerator[i][j][1] / denom
            I_ce[i][j][2] = numerator[i][j][2] / denom
    
    # 4. Compute Non-Edge Color I_cne
    # Keep most frontal color: SH * C_1
    frontal_color = [[[0.0, 0.0, 0.0] for _ in range(W)] for _ in range(H)]
    for k in range(K):
        C_k = buffers[k]['color']
        for i in range(H):
            for j in range(W):
                if SH[i][j] > 0.5:
                    frontal_color[i][j] = C_k[i][j]
    I_cne = frontal_color
    
    # 5. Final Color Composition
    final_color = [[[0.0, 0.0, 0.0] for _ in range(W)] for _ in range(H)]
    for i in range(H):
        for j in range(W):
            e = E[i][j]
            final_color[i][j][0] = e * I_ce[i][j][0] + (1.0 - e) * I_cne[i][j][0]
            final_color[i][j][1] = e * I_ce[i][j][1] + (1.0 - e) * I_cne[i][j][1]
            final_color[i][j][2] = e * I_ce[i][j][2] + (1.0 - e) * I_cne[i][j][2]
    
    # 6. Compute Silhouette Edge Intensity I_se
    # I_se = 1 - product(1 - D_k)
    prob_none = [[1.0] * W for _ in range(H)]
    for k in range(K):
        D_k = D[k]
        for i in range(H):
            for j in range(W):
                prob_none[i][j] *= (1.0 - D_k[i][j])
    I_se = [[1.0 - prob_none[i][j] for j in range(W)] for i in range(H)]
    
    # 7. Compute Non-Edge Silhouette Intensity I_sne
    I_sne = SH
    
    # 8. Final Silhouette Composition
    final_silhouette = [[0.0] * W for _ in range(H)]
    for i in range(H):
        for j in range(W):
            e = E[i][j]
            final_silhouette[i][j] = e * I_se[i][j] + (1.0 - e) * I_sne[i][j]
    
    return final_color, final_silhouette

def sigmoid(x):
    return 1.0 / (1.0 + math.exp(-x))

def compute_edge_mask(SH, delta):
    """
    Placeholder for image processing step: Edge detection and dilation on SH.
    Returns binary mask E with width delta.
    """
    # Sources do not specify the exact edge detection/dilation algorithm; 
    # standard morphological operations are used [S1].
    pass
```

**Parameters and Edge Cases:**
- $\sigma$: Temperature parameter. Default $\sigma = r/7$ [S1]. Smaller $\sigma$ makes the transition sharper [G].
- $\delta$: Edge mask width. Default $\delta = r$ [S1]. Controls the region around silhouettes that is blended [S1].
- Numerical Instability: The source explicitly states that the square of the distance is removed from the sigmoid argument compared to SoftRas to prevent numerical instability [S1].
- Pseudo-occlusions: Settings without `Enlarge()` or `Shift()` (concepts from the source's broader context) can suffer from pseudo-occlusions [S1]. The default setting uses `Enlarge()` and `Shift()` to produce smoother soft faces [S1].

## Operating characteristics
- **Strengths:** Maintains sharp textures inside silhouettes (unlike SoftRas) while providing anti-aliasing and far-range gradients around edges (unlike hard rasterization or Nvdiffrast's geometric edge blending) [S1].
- **Limitations:** Requires image processing steps (edge detection, dilation) which may be less differentiable or harder to integrate into pure GPU pipelines compared to purely geometric methods [S1].
- **Comparison:** Unlike Nvdiffrast, which geometrically detects edges and blends only pixels closer than one pixel to the edge, HardSoftRas blurs all edge pixels defined by the mask $E$ and uses image processing for silhouette edges [S1]. Unlike SoftRas, it does not blend non-edge regions, preserving sharpness [S1].

## Background
HardSoftRas Blending was introduced in the "Dressi" renderer, a hardware-agnostic differentiable renderer with reactive shader packing and soft rasterization [S1]. It generalizes and bridges anti-aliasing and SoftRas techniques [S1].

## Sources
[S1] Dressi: A Hardware‐Agnostic Differentiable Renderer with Reactive Shader Packing and Soft Rasterization, https://doi.org/10.1111/cgf.14455, fetched 2026-10-29
