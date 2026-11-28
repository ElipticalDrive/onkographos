# SortFree Weighting

Also called: SortFree, Weighted-sum blending OIT

## Overview
SortFree Weighting is an Order-Independent Transparency (OIT) approximation for 3D Gaussian Splatting that eliminates the computational bottleneck of depth sorting by replacing non-commutative alpha blending with a commutative weighted-sum blending scheme [S1, unofficial]. It allows parallel accumulation of Gaussian colors in a single pass, trading strict physical accuracy for rendering efficiency and temporal stability [S1, unofficial]. It sets itself apart from siblings like [Depth-aware Order-independent Rendering](9460-depth-aware-order-independent-rendering.md) by using a fixed analytical depth falloff rather than neural networks to predict view-dependent weights [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians, each defined by position, covariance, opacity, and spherical harmonic coefficients [G]. For each pixel, the set of projected 2D Gaussian splats overlapping the pixel [G].
*   **Output:** The final color $C$ for the pixel [G].

**Data Structures:**
*   **Gaussian Splat:** Contains screen-space center $(x, y)$, covariance matrix $\Sigma$, opacity $\alpha$, and spherical harmonic coefficients $\mathbf{c}$ [G].
*   **Scene Parameter:** A single learnable scalar $\sigma$ representing the depth falloff range [S1, unofficial].

**Method:**
The core mechanism replaces the standard alpha compositing equation $C = \sum c_i \alpha_i \prod_{j<i} (1-\alpha_j)$ with a weighted sum [S1, unofficial]. The weight $w$ for each Gaussian depends on its depth $d$ and view direction $\mathbf{r}$ [S1, unofficial].

1.  **Project Gaussians:** Project each 3D Gaussian to 2D screen space to obtain the 2D covariance matrix $\Sigma'$ and center [G].
2.  **Compute Depth:** For each Gaussian $i$, determine its depth $d_i$ in camera space [G].
3.  **Compute View Direction:** Determine the view direction vector $\mathbf{r}_i$ from the camera center to the Gaussian center [G].
4.  **Calculate Weight:** Compute the weighting factor $w_i$ using the formula:
    $$ w_i = \max\left(0, 1 - \frac{d_i}{\sigma}\right) v(\mathbf{r}_i) $$
    Where:
    *   $d_i$ is the depth of the $i$-th Gaussian [S1, unofficial].
    *   $\sigma$ is the learnable scene parameter controlling the depth falloff [S1, unofficial].
    *   $v(\mathbf{r}_i)$ is a view-dependent term derived from spherical harmonics [S1, unofficial].
5.  **Accumulate Color:** Compute the final color $C$ by summing the weighted colors of all overlapping splats:
    $$ C = \sum_{i \in \text{overlapping}} w_i $$
    Where $c_i$ is the color of the $i$-th Gaussian evaluated at $\mathbf{r}_i$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sortfree_weighting_render(gaussians, camera_origin, view_matrix, proj_matrix, sigma, image_shape):
    """
    Render a frame using SortFree Weighting.
    
    Args:
        gaussians: List of dicts with keys: 'pos' (3,), 'sh_coeffs' (degree-3 SH array)
        camera_origin: (3,) camera position in world space
        view_matrix: (4,4) world-to-camera transformation
        proj_matrix: (4,4) perspective projection matrix
        sigma: float, learnable depth falloff parameter
        image_shape: tuple (height, width)
        
    Returns:
        image: (H, W, 3) rendered image
    """
    H, W = image_shape
    image = np.zeros((H, W, 3))
    
    for g in gaussians:
        # 1. Project to camera space
        pos_world = np.append(g['pos'], 1.0)
        pos_cam = view_matrix @ pos_world
        pos_cam = pos_cam[:3] / pos_cam[3]
        
        # 2. Project to screen space
        pos_clip = proj_matrix @ np.append(pos_cam, 1.0)
        pos_clip = pos_clip[:3] / pos_clip[3]
        x_screen = (pos_clip[0] + 1.0) * 0.5 * W
        y_screen = (1.0 - pos_clip[1]) * 0.5 * H
        
        # Guard: skip if outside framebuffer
        if not (0 <= x_screen < W and 0 <= y_screen < H):
            continue
            
        # 3. Compute Depth
        d = pos_cam[2]
        
        # 4. Compute View Direction
        view_vec = pos_cam - camera_origin
        view_vec = view_vec / np.linalg.norm(view_vec)
        
        # 5. Calculate Weight
        depth_term = max(0.0, 1.0 - d / sigma)
        sh_color = evaluate_spherical_harmonics(g['sh_coeffs'], view_vec)
        contribution = depth_term * sh_color
        
        # 6. Accumulate (commutative, order-independent)
        image[int(y_screen), int(x_screen)] += contribution
        
    return image

def evaluate_spherical_harmonics(sh_coeffs, direction):
    """Evaluate degree-3 spherical harmonics for a given direction vector."""
    x, y, z = direction
    # Standard degree-3 SH basis evaluation (9 coefficients per channel)
    # Coefficients layout: [R0, R1, R2, R3, R4, R5, R6, R7, R8]
    sh_basis = [
        0.282095,
        -0.488603 * y, 0.488603 * z, -0.488603 * x,
        1.092548 * x * y, -1.092548 * y * z, 0.315392 * (3 * z**2 - 1),
        -1.092548 * x * z, 0.546274 * (x**2 - y**2)
    ]
    # Multiply basis by SH coefficients and sum across channels
    color = np.zeros(3)
    for c in range(3):
        coeff = sh_coeffs[c * 9 : (c + 1) * 9]
        color[c] = sum(b * a for b, a in zip(sh_basis, coeff))
    return color
```

**Parameters:**
*   $\sigma$: A learnable scalar parameter controlling the depth falloff range. Typical value not specified in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth $d > \sigma$:** The term $\max(0, 1 - d/\sigma)$ becomes 0, effectively culling distant Gaussians [S1, unofficial].
*   **Precision:** Accumulating many small weights can lead to floating-point precision issues if not handled carefully (e.g., using Kahan summation or higher precision accumulators) [G].
*   **View-Dependence:** The method relies on SH for view-dependent appearance [S1, unofficial]. If SH coefficients are not properly trained, the view-dependent compensation $v(\mathbf{r})$ may be inaccurate [G].

## Subtopics

### [Depth-aware Order-independent Rendering](9460-depth-aware-order-independent-rendering.md)
Uses neural networks to predict view-dependent weights and opacity, improving visual quality over SortFree's fixed linear falloff. Choose when higher visual fidelity is required and neural network inference overhead is acceptable.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel where $N$ is the number of overlapping splats, as no sorting is required [G]. The sorting step in standard 3DGS is $O(N \log N)$ [G].
*   **Memory Complexity:** $O(1)$ additional memory per pixel for accumulation, compared to $O(N)$ for storing sorted lists in OIT methods [G].
*   **Performance:** Significantly faster than sorted alpha blending due to the removal of the sorting bottleneck. However, it is generally slower than volumetric rendering methods due to the per-splat evaluation and accumulation overhead [S1, unofficial].
*   **Strengths:** Eliminates sorting overhead; provides temporal stability (no popping artifacts from sorting changes); simple implementation [S1, unofficial].
*   **Limitations:** Approximates alpha blending, leading to potential visual artifacts (e.g., incorrect transparency handling); quality can suffer at novel viewpoints compared to sorted methods [S1, unofficial].
*   **When to Use:** Real-time applications on edge devices where sorting is too expensive and slight visual inaccuracies are acceptable [S1, unofficial].
*   **When Not to Use:** High-fidelity offline rendering where physical accuracy is critical [G].

## Background
SortFree Weighting was proposed in the paper "SortFree" (cited as [14] in [S1, unofficial]), which introduced a weighted-sum blending formulation to approximate standard alpha compositing in a commutative manner. It is part of a broader class of Order-Independent Transparency (OIT) techniques applied to Gaussian-based representations. The method aims to reduce rendering overhead and improve temporal stability by removing the sorting step [S1, unofficial].

## Sources
[S1] SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method, https://doi.org/10.48550/arXiv.2605.13855, fetched 2026-10-29 (unofficial)
