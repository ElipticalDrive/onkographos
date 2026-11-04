# Front-to-Back Alpha Compositing

Also called: Alpha Compositing, Alpha Compositing with Front-to-Back Blending, Front-to-Back Blending, Over Operator, Porter-Duff Over, Running Transmittance Compositing

## Overview
Front-to-Back Alpha Compositing is a rendering technique that blends multiple semi-transparent primitives (such as 2D Gaussians or splats) into a final image by processing them in depth order from nearest to farthest, ensuring correct occlusion where near primitives paint over far ones [S1, unofficial]. It is distinguished from back-to-front compositing by its ability to use early termination: once the accumulated transmittance drops below a threshold, the pixel is considered fully opaque and remaining splats are skipped, providing a significant computational speedup [S1, unofficial].

## How it works
**Inputs:**
- A set of $N$ splats (Gaussians), each defined by:
  - Center $\mathbf{x}_i = (u_i, v_i)$ in screen space.
  - Inverse 2D covariance matrix $\Sigma'^{-1}_i$ (defining the ellipse shape/orientation).
  - Intrinsic opacity scalar $\alpha_i \in [0, 1]$.
  - Color vector $\mathbf{c}_i$ (typically RGB).
  - Depth $Z_c$ (used for sorting).
- Image dimensions (width, height).
- Early termination threshold $\epsilon$ (typically $10^{-4}$) [S1, unofficial].

**Outputs:**
- Final image $\mathbf{C}$ with dimensions matching the input image.

**Data Structures:**
- Splat list: Array of splat objects sorted by depth $Z_c$ in ascending order (nearest first) [S1, unofficial].
- Per-pixel accumulators:
  - $\mathbf{C}$: Accumulated color, initialized to $\mathbf{0}$ [S1, unofficial].
  - $T$: Running transmittance, initialized to $1$ [S1, unofficial].

**Steps:**
1. **Sort Splats:** Sort all splats by their center depth $Z_c$ in ascending order (nearest to farthest) [S1, unofficial].
2. **Initialize:** For each pixel $\mathbf{x}$, set $\mathbf{C} \leftarrow \mathbf{0}$ and $T \leftarrow 1$ [S1, unofficial].
3. **Iterate Splats:** For each splat $i$ in the sorted list:
   a. **Compute Effective Alpha:** Calculate the effective per-pixel alpha $\alpha_i(\mathbf{x})$ using the 2D Gaussian falloff:
      $$
      \alpha_i(\mathbf{x}) = \alpha_i \cdot \exp\left(-\tfrac{1}{2}(\mathbf{x} - \mathbf{x}_i)^\top \Sigma'^{-1}_i (\mathbf{x} - \mathbf{x}_i)\right)
      $$
      where $\alpha_i$ is the intrinsic opacity, $\mathbf{x}_i$ is the splat center, and $\Sigma'^{-1}_i$ is the inverse covariance [S1, unofficial].
   b. **Update Color:** Add the splat's contribution to the accumulated color:
      $$
      \mathbf{C} \leftarrow \mathbf{C} + T \cdot \alpha_i(\mathbf{x}) \cdot \mathbf{c}_i
      $$
   c. **Update Transmittance:** Multiply the running transmittance by the fraction of light blocked by the current splat:
      $$
      T \leftarrow T \cdot (1 - \alpha_i(\mathbf{x}))
      $$
   d. **Early Termination:** If $T < \epsilon$, break the loop for this pixel, as the pixel is effectively fully opaque [S1, unofficial].
4. **Final Color:** The value $\mathbf{C}$ after the loop (or break) is the final pixel color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def front_to_back_compositing(splats, image_width, image_height, epsilon=1e-4):
    """
    Composites splats front-to-back for a given image.
    
    Args:
        splats: List of dicts with keys:
            'center': (u, v) tuple
            'cov_inv': 2x2 numpy array (inverse covariance)
            'alpha': float (intrinsic opacity)
            'color': (r, g, b) tuple or array
            'depth': float (for sorting)
        image_width: int
        image_height: int
        epsilon: float, threshold for early termination
        
    Returns:
        final_image: numpy array of shape (H, W, 3)
    """
    # Sort splats by depth (nearest first)
    sorted_splats = sorted(splats, key=lambda s: s['depth'])
    
    # Initialize output image
    final_image = np.zeros((image_height, image_width, 3), dtype=np.float32)
    
    # Pre-allocate transmittance and color accumulators for all pixels
    # T[i, j] is the transmittance for pixel (j, i)
    T = np.ones((image_height, image_width), dtype=np.float32)
    
    # Iterate through each splat
    for splat in sorted_splats:
        cx, cy = splat['center']
        cov_inv = splat['cov_inv']
        alpha_intrinsic = splat['alpha']
        color = np.array(splat['color'])
        
        # Create meshgrid for pixel coordinates
        # y corresponds to rows (height), x to columns (width)
        y, x = np.mgrid[0:image_height, 0:image_width]
        
        # Vectorized computation of effective alpha for all pixels
        # dx, dy are vectors from pixel to splat center
        dx = x - cx
        dy = y - cy
        
        # Quadratic form: (x - x_i)^T Sigma'^-1 (x - x_i)
        # Stack dx, dy into a 2xN matrix for each pixel? 
        # Better: use broadcasting for 2D grid
        # Let v = [dx, dy]^T. We compute v^T cov_inv v.
        # cov_inv is 2x2.
        # term1 = cov_inv[0,0]*dx^2 + cov_inv[1,1]*dy^2 + 2*cov_inv[0,1]*dx*dy
        term1 = cov_inv[0, 0] * dx**2 + cov_inv[1, 1] * dy**2 + 2 * cov_inv[0, 1] * dx * dy
        
        # Gaussian falloff
        gaussian_falloff = np.exp(-0.5 * term1)
        
        # Effective alpha
        alpha_eff = alpha_intrinsic * gaussian_falloff
        
        # Update color: C += T * alpha_eff * color
        # Only update pixels where alpha_eff is significant? 
        # The formula applies everywhere, but alpha_eff ~ 0 outside ellipse.
        final_image += (T[:, :, np.newaxis] * alpha_eff[:, :, np.newaxis] * color[np.newaxis, np.newaxis, :])
        
        # Update transmittance: T *= (1 - alpha_eff)
        T *= (1 - alpha_eff)
        
        # Early termination check
        # If T is below epsilon, no further splats will contribute significantly.
        # We can mask out these pixels to save computation in subsequent splats.
        # However, in a simple loop over splats, we just continue. 
        # Optimization: if all T < epsilon, we can stop entirely.
        if np.all(T < epsilon):
            break
            
    return final_image
```

**Edge Cases and Pitfalls:**
- **Sort Order:** Splats must be processed from nearest to farthest. If processed far-to-near without adjusting the compositing math, occlusion inverts, causing far splats to wrongly paint over near ones [S1, unofficial].
- **Sort-Order Flips (Popping Artifacts):** 3DGS sorts splats by their center depth $Z_c$. Since splats are 3D blobs with extent, overlapping splats can swap order as the camera moves, causing flickering. This is a speed/quality trade-off [S1, unofficial].
- **Precision:** The transmittance $T$ decays exponentially. If $T$ becomes very small, floating-point precision issues may arise, though early termination mitigates this.

## Subtopics

### [Vectorized Front-to-Back Compositing](4305-vectorized-front-to-back-compositing.md)
A Python/NumPy implementation that processes all pixels simultaneously using vectorized operations, trading computational efficiency for code simplicity and gradient stability.

## Operating characteristics
- **Time Complexity:** $O(N \cdot P)$ in the worst case, where $N$ is the number of splats and $P$ is the number of pixels. However, with early termination, the average case is significantly faster, especially in dense scenes where pixels become opaque quickly [S1, unofficial].
- **Memory Complexity:** $O(P)$ for storing the accumulated color and transmittance per pixel [S1, unofficial].
- **Performance:** Early termination provides a significant speedup, often stopping most pixels after only a handful of contributions [S1, unofficial]. Tiling further optimizes this by reducing the number of splats considered per pixel (most tiles touch only 2–4 splats) [S1, unofficial].
- **Strengths:** Correct occlusion handling, efficient early termination, simple algebra.
- **Limitations:** Sort-order flips can cause visual artifacts (popping) [S1, unofficial]. Vectorized implementations may process all splats for all pixels, losing the early termination benefit [S1, unofficial].

## Background
Front-to-Back Alpha Compositing is based on the "over" operator from Porter-Duff compositing, widely used in image editing (e.g., Photoshop) and CGI [S1, unofficial]. It is a fundamental building block in volume rendering and neural rendering methods like 3D Gaussian Splatting (3DGS) [S1, unofficial]. The method was popularized in the context of 3DGS by Kerbl et al. (2023), though the underlying compositing math is standard in computer graphics [S1, unofficial].

## Sources
[S1] 3DGS-101/textbook/05-alpha-compositing.md at main - GitHub, https://github.com/Chamud/3DGS-101/blob/main/textbook/05-alpha-compositing.md, fetched 2026-10-28 (unofficial)
