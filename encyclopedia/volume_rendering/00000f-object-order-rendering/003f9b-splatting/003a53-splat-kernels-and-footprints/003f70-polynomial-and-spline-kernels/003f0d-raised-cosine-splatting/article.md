# Raised Cosine Splatting

Also called: 3DRCS, Modified Raised Cosine

## Overview
Raised Cosine Splatting is a volume rendering technique that replaces the standard Gaussian radial basis function with a modified raised cosine kernel to improve the reconstruction of fine geometric details [S1]. It distinguishes itself from Gaussian splatting by using a kernel with compact support and specific decay properties, which reduces reconstruction error in 1D simulations and enhances visual fidelity for high-frequency features like edges and small textures [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D primitives (splats), each defined by a mean position $\mathbf{\mu}$, a 3D covariance matrix $\Sigma$, and an opacity/density weight $w$. The primitives are transformed into screen space, resulting in a 2D covariance matrix $\Sigma'$ (or its inverse, the conic matrix) and a projected mean $\mathbf{\mu}'$ [S1].
*   **Output:** A 2D image where each pixel $(x,y)$ receives an alpha value $\alpha$ and a color contribution based on the kernel evaluation [S1].

**Data Structures:**
The implementation relies on the 2D conic matrix (inverse covariance) $\mathbf{C} = \begin{bmatrix} a & b \\ b & c \end{bmatrix}$ and the determinant $det = ac - b^2$ [S1]. The code snippet uses `con_o` to represent the conic matrix entries and `d` for the displacement vector from the splat center to the pixel [S1].

**Parameters:**
*   `correctionFactor`: A scalar multiplier applied to the 3D covariance matrix before projection to adjust the kernel's spread. Value: $0.655$ [S1].
*   `ellipseLimit`: The maximum squared distance for the kernel support. Value: $6.25$ [S1].
*   `alphaClamp`: The maximum alpha value to prevent over-saturation. Value: $0.99$ [S1].

**Algorithm Steps:**
1.  **Covariance Correction:** Multiply the 3D covariance matrix $\Sigma$ by `correctionFactor` ($0.655$) to obtain the adjusted covariance $\Sigma_{adj}$ [S1].
2.  **Projection:** Transform $\Sigma_{adj}$ into the 2D screen-space covariance $\Sigma'$ using the view projection and Jacobian of the transformation. Compute the conic matrix $\mathbf{C} = \Sigma'^{-1}$ [S1].
3.  **Displacement Calculation:** For each pixel $(x,y)$ within the bounding box of the splat, calculate the displacement vector $\mathbf{d} = [x - \mu'_x, y - \mu'_y]^T$ [S1].
4.  **Ellipse Evaluation:** Compute the squared Mahalanobis distance (ellipse term) using the conic matrix:
    $$ ellipse = a \cdot d_x^2 + c \cdot d_y^2 + 2b \cdot d_x \cdot d_y $$
    where $a, b, c$ are the elements of the conic matrix $\mathbf{C}$ [S1].
5.  **Boundary Check:** If $ellipse < 0$ or $ellipse > 6.25$, skip the pixel (the splat contribution is zero outside this support) [S1].
6.  **Kernel Evaluation:**
    *   Calculate the square root of the ellipse: $d_M = \sqrt{ellipse}$ [S1].
    *   Calculate the scaling factor $\xi = \frac{5}{2\pi}$, which implies the argument scaling factor is $\frac{\pi}{2.5}$ [S1].
    *   Compute the ellipse factor: $ellipseFactor = \frac{\pi \cdot d_M}{2.5}$ [S1].
    *   Compute the base weight: $Cosine = 0.5 + 0.5 \cdot \cos(ellipseFactor)$ [S1].
7.  **Alpha Compositing:**
    *   Compute the final alpha: $\alpha = \min(0.99, w \cdot Cosine)$ [S1].
    *   Write the color and alpha to the framebuffer using standard blending [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def raised_cosine_splat_forward(conic_a, conic_b, conic_c, weight, pixel_x, pixel_y, splat_center_x, splat_center_y):
    """
    Computes the alpha contribution of a raised cosine splat at a specific pixel.
    
    Args:
        conic_a, conic_b, conic_c: Elements of the 2D conic matrix (inverse covariance).
        weight: The opacity/density scalar of the splat.
        pixel_x, pixel_y: The screen coordinates of the current pixel.
        splat_center_x, splat_center_y: The projected 2D center of the splat.
        
    Returns:
        alpha: The computed alpha value for the pixel.
    """
    # 1. Calculate displacement vector d
    dx = pixel_x - splat_center_x
    dy = pixel_y - splat_center_y
    
    # 2. Compute ellipse term (squared Mahalanobis distance)
    # ellipse = a*dx^2 + c*dy^2 + 2*b*dx*dy
    ellipse = conic_a * (dx * dx) + conic_c * (dy * dy) + 2.0 * conic_b * dx * dy
    
    # 3. Boundary check
    # Limit set on 2D ellipse to restrict to one pulse
    if ellipse < 0.0 or ellipse > 6.25:
        return 0.0
    
    # 4. Compute kernel value
    ellipse_root = math.sqrt(ellipse)
    
    # Scaling factor xi = 5 / (2 * pi) -> argument multiplier = pi / 2.5
    ellipse_factor = (math.pi * ellipse_root) / 2.5
    
    # Base weight: 0.5 + 0.5 * cos(ellipse_factor)
    cosine_weight = 0.5 + 0.5 * math.cos(ellipse_factor)
    
    # 5. Final alpha with clamping
    alpha = min(0.99, weight * cosine_weight)
    
    return alpha

def raised_cosine_splat_backward(dL_dalpha, conic_a, conic_b, conic_c, weight, pixel_x, pixel_y, splat_center_x, splat_center_y):
    """
    Computes gradients for the raised cosine splat.
    Note: The source provides gradients w.r.t the conic matrix entries given dL/d_conic.
    This function demonstrates the forward pass logic required to compute the intermediate 'ellipse' value
    needed for the gradient calculation provided in the source.
    """
    dx = pixel_x - splat_center_x
    dy = pixel_y - splat_center_y
    ellipse = conic_a * (dx * dx) + conic_c * (dy * dy) + 2.0 * conic_b * dx * dy
    
    if ellipse < 0.0 or ellipse > 6.25:
        return 0.0, 0.0, 0.0 # Gradients are zero outside support
    
    # The source code for backward pass assumes dL_dconic is already computed.
    # The forward pass here is necessary to determine if the gradient is non-zero.
    # Actual gradient computation w.r.t conic entries is handled by the specific 
    # dL_dconic -> dL_dconic_a/b/c mapping in the source [S1].
    return 0.0, 0.0, 0.0
```

**Edge Cases and Pitfalls:**
*   **Precision:** The boundary check `ellipse > 6.25` is critical. If the ellipse value exceeds this limit, the kernel value becomes negative or undefined in the context of the single-pulse restriction, so the splat must be skipped [S1].
*   **Covariance Scaling:** The `correctionFactor` of $0.655$ must be applied to the 3D covariance *before* projection to 2D. Applying it afterwards yields incorrect kernel shapes [S1].
*   **Alpha Clamping:** The alpha is clamped to $0.99$ to prevent numerical instability during blending [S1].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of splats and $K$ is the number of pixels covered by each splat's footprint. The footprint is determined by the support limit $ellipse \le 6.25$.
*   **Memory Complexity:** $O(N)$ to store the splat parameters (position, covariance, weight) [S1].
*   **Performance:** The method is hardware-accelerated via GPU rasterization, similar to Gaussian Splatting [S1].
*   **Strengths:** Improved reconstruction of fine details (buttons, marks, strings, pot edges) compared to Gaussians [S1]. Lower 1D reconstruction error (MSE) with fewer components (N=8 vs N=10 for Gaussians) [S1].
*   **Limitations:** The compact support may require more splats to cover large smooth areas compared to the infinite support of Gaussians, potentially increasing memory usage for very large, smooth volumes [S1].
*   **When to use:** When high-frequency detail reconstruction is prioritized over smooth, broad-area blending, and when computational resources allow for the rasterization of potentially more numerous splats [S1].

## Background
Raised Cosine Splatting, also known as 3DRCS or Modified Raised Cosine, was introduced in the paper "DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions" [S1]. The method generalizes splatting by allowing various radial basis functions, with the raised cosine being a key example that offers better performance than Gaussians in specific reconstruction tasks [S1]. It was evaluated on datasets such as Mip-NeRF 360, Tanks&Temples, and Deep Blending [S1].

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
