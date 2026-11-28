# Vectorized Front-to-Back Compositing

Also called: NumPy Front-to-Back Compositing, Per-tile sorting, Tiled Rasterization, Vectorized Alpha Compositing

## Overview
A Python/NumPy implementation of front-to-back alpha compositing that processes all pixels in an image simultaneously using vectorized operations, rather than per-pixel loops [S1, unofficial]. Unlike GPU implementations that can perform per-pixel early termination, this NumPy version processes all splats for all pixels regardless of opacity, trading computational efficiency for code simplicity and gradient stability [S1, unofficial].

## How it works
**Inputs:**
- A list of splats, each defined by center $(u, v)$ in pixel coordinates, a $2 \times 2$ covariance matrix $\Sigma'$ (or its inverse $\Sigma'^{-1}$), RGB color $\mathbf{c}$, stored opacity $\alpha_{\text{stored}}$, and depth $Z_c$ [S1, unofficial].
- Image dimensions $H \times W$ [S1, unofficial].
- Background color $\mathbf{c}_{\text{bg}}$ [S1, unofficial].

**Outputs:**
- Final image $\mathbf{C}$ of shape $(H, W, 3)$ [S1, unofficial].

**Data Structures:**
- `pix`: Array of shape $(H, W, 2)$ containing pixel coordinates $(u, v)$ [S1, unofficial].
- `C`: Accumulated color array of shape $(H, W, 3)$, initialized to zeros [S1, unofficial].
- `T`: Transmittance array of shape $(H, W)$, initialized to ones [S1, unofficial].

**Steps:**
1. **Sort Splats:** Sort the list of splats by depth $Z_c$ in ascending order (near to far) [S1, unofficial].
2. **Initialize:** Set $\mathbf{C} = \mathbf{0}$ and $T = \mathbf{1}$ for all pixels [S1, unofficial].
3. **Iterate:** For each splat $i$ in sorted order:
   a. Compute pixel offsets: $\mathbf{d} = \text{pix} - \text{center}_i$ [S1, unofficial].
   b. Compute quadratic form: $q = \mathbf{d}^T \Sigma'^{-1}_i \mathbf{d}$ [S1, unofficial].
   c. Compute Gaussian falloff: $G = \exp(-0.5 \cdot q)$ [S1, unofficial]. The normalization constant $\frac{1}{2\pi|\Sigma'|}$ is omitted or absorbed [S1, unofficial].
   d. Compute effective alpha: $\alpha_i = \alpha_{\text{stored}, i} \cdot G$ [S1, unofficial].
   e. Clip alpha: $\alpha_i = \min(\alpha_i, 0.999)$ to prevent $\alpha=1.0$ [S1, unofficial].
   f. Compute weight: $w = \alpha_i \cdot T$ [S1, unofficial].
   g. Update color: $\mathbf{C} \leftarrow \mathbf{C} + w \cdot \mathbf{c}_i$ [S1, unofficial].
   h. Update transmittance: $T \leftarrow T \cdot (1 - \alpha_i)$ [S1, unofficial].
4. **Background:** Add background contribution: $\mathbf{C} \leftarrow \mathbf{C} + T \cdot \mathbf{c}_{\text{bg}}$ [S1, unofficial].

**Formulas:**
- Gaussian falloff: $G = e^{-\frac{1}{2} d^T \Sigma'^{-1} d}$ [S1, unofficial].
- Effective alpha: $\alpha_i(\mathbf{x}) = \alpha_i \cdot G_i(\mathbf{x})$ [S1, unofficial].
- Color accumulation: $\mathbf{C} = \sum_i \mathbf{c}_i \alpha_i T_i$, where $T_i = \prod_{j<i} (1-\alpha_j)$ [S1, unofficial].

**Parameters:**
- Alpha clipping upper bound: $0.999$ [S1, unofficial].
- Background color: $[0.05, 0.05, 0.08]$ (dark navy) [S1, unofficial].
- Low alpha threshold: $\approx 0.004$ ($1/255$) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def vectorized_front_to_back_compositing(splats, H, W, background_color):
    """
    splats: list of dicts with keys 'center' (2,), 'inv' (2x2), 'color' (3,), 'opacity' (float), 'depth' (float)
    H, W: int
    background_color: array (3,)
    """
    # 1. Sort splats by depth (near first)
    splats_sorted = sorted(splats, key=lambda g: g["depth"])
    
    # 2. Initialize pixel grid and accumulators
    vv, uu = np.mgrid[0:H, 0:W]
    pix = np.stack([uu, vv], axis=-1).astype(float)  # (H, W, 2)
    
    C = np.zeros((H, W, 3))  # Accumulated color
    T = np.ones((H, W))      # Accumulated transmittance
    
    # 3. Iterate over splats
    for g in splats_sorted:
        # a. Pixel offsets
        d = pix - g["center"]  # (H, W, 2)
        
        # b. Quadratic form d^T Sigma'^-1 d
        q = np.einsum('...i,ij,...j->...', d, g["inv"], d)
        
        # c. Gaussian falloff
        G = np.exp(-0.5 * q)  # (H, W)
        
        # d. Effective alpha
        alpha = g["opacity"] * G
        
        # e. Clip alpha to prevent gradient vanishing
        alpha = np.clip(alpha, 0.0, 0.999)
        
        # f. Weight
        w = alpha * T  # (H, W)
        
        # g. Update color
        C += w[:, :, None] * g["color"]  # Broadcasting color to (H, W, 3)
        
        # h. Update transmittance
        T *= (1.0 - alpha)
        
        # Skip splats with very low alpha to save computation and gradient traffic
        if np.max(alpha) < 0.004:
            continue

    # 4. Add background
    C += T[:, :, None] * background_color
    
    # Clip final image
    img = np.clip(C, 0, 1)
    return img
```

**Edge Cases and Pitfalls:**
- **Alpha = 1.0:** If $\alpha$ reaches exactly 1.0, $T$ becomes 0, stopping further contributions. This is acceptable for forward rendering but causes vanishing gradients during backpropagation [S1, unofficial]. Clamping to 0.999 prevents this [S1, unofficial].
- **Low Alpha:** Effective alpha below $1/255 \approx 0.004$ contributes nothing to 8-bit pixels [S1, unofficial]. Evaluating these splats wastes computation and gradient traffic [S1, unofficial]. Real implementations skip these [S1, unofficial].
- **No Early Termination:** Unlike GPU implementations, this vectorized approach cannot break the loop early when $T < \epsilon$ for specific pixels [S1, unofficial]. All pixels process all splats regardless of opacity [S1, unofficial].

## Operating characteristics
Time and memory complexity, measured performance, scaling, strengths, limitations, and usage guidelines are Not given in the sources [S1, unofficial]. The method processes all splats for all pixels without per-pixel early termination, unlike GPU implementations [S1, unofficial].

## Background
Described in an unofficial GitHub repository "3DGS-101" by Chamud [S1, unofficial]. The underlying tiled sorting and compositing pipeline is implemented in the reference CUDA rasterizer for 3D Gaussian Splatting and libraries like gsplat [S1, unofficial].

## Sources
[S1] 3DGS-101/textbook/05-alpha-compositing.md at main - GitHub, https://github.com/Chamud/3DGS-101/blob/main/textbook/05-alpha-compositing.md, fetched 2026-10-28 (unofficial)
