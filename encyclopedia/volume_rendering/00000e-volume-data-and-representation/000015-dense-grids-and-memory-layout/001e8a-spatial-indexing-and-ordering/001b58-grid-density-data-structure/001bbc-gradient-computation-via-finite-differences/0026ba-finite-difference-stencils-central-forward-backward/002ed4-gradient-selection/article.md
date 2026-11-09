# Gradient Selection

Also called: Central differences, Finite Difference Stencils, Gradient Mode Selection, Gradient estimation via finite differences

## Overview
Gradient Selection is a runtime dispatch mechanism that chooses between specific finite difference gradient estimation algorithms, primarily Central Difference and Sobel-based methods, based on a configuration uniform [S1, unofficial]. It solves the problem of adapting gradient computation to the noise characteristics and visual requirements of the volume data, providing a normalized gradient vector to substitute for surface normals in shading models [S2, unofficial]. It is distinguished from its siblings by acting as a selector rather than a standalone estimator, allowing dynamic switching between lower-cost central differences and higher-robustness Sobel filters [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field texture $V$, a sampling coordinate $\mathbf{t}$ (texture coordinates or voxel indices), and a configuration uniform `gradientMode`.
*   **Output:** A normalized gradient vector $\mathbf{G}_{norm} = (G_x, G_y, G_z)$.

**Data Structures:**
*   `gradientMode`: An integer or enum uniform. Common values include `GRADIENT_CENTRAL` and `GRADIENT_SOBEL` [S1, unofficial].

**Steps:**
1.  Evaluate the `gradientMode` uniform.
2.  If `gradientMode` indicates Sobel estimation, invoke the Sobel gradient estimator at $\mathbf{t}$ [S1, unofficial].
3.  Otherwise, invoke the Central Difference gradient estimator at $\mathbf{t}$ [S1, unofficial].
4.  Normalize the resulting gradient vector $\mathbf{G}$ to unit length:
    $$ \mathbf{G}_{norm} = \frac{\mathbf{G}}{\|\mathbf{G}\|} $$
    [S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_selection(texture, coord, gradient_mode):
    """
    Dispatches gradient computation based on mode.
    
    Args:
        texture: The scalar field volume texture.
        coord: The sampling coordinate (vec3).
        gradient_mode: Enum indicating the estimation method.
        
    Returns:
        vec3: The normalized gradient vector.
    """
    if gradient_mode == GRADIENT_SOBEL:
        g = gradient_sobel(texture, coord)
    else:
        # Default to Central Difference if mode is not Sobel
        g = gradient_central(texture, coord)
        
    # Normalize the gradient vector for use as a surface normal
    magnitude = length(g)
    if magnitude > 0.0:
        return g / magnitude
    else:
        return vec3(0.0, 0.0, 0.0)

def gradient_central(texture, coord):
    """
    Computes gradient using central differences.
    Assumes isotropic spacing and boundary handling (e.g., clamping).
    """
    # Partial derivative in x
    dx = sample(texture, coord + vec3(1.0, 0.0, 0.0)) - sample(texture, coord - vec3(1.0, 0.0, 0.0))
    # Partial derivative in y
    dy = sample(texture, coord + vec3(0.0, 1.0, 0.0)) - sample(texture, coord - vec3(0.0, 1.0, 0.0))
    # Partial derivative in z
    dz = sample(texture, coord + vec3(0.0, 0.0, 1.0)) - sample(texture, coord - vec3(0.0, 0.0, 1.0))
    
    return vec3(dx, dy, dz)

def gradient_sobel(texture, coord):
    """
    Computes gradient using Sobel operator.
    Uses 3x3x3 cubic window for robustness.
    """
    # Sobel weights for x, y, z directions are applied to the 3x3x3 neighborhood.
    # This involves weighted sums of neighbors to estimate derivatives.
    # Implementation details depend on specific Sobel kernel definitions.
    # For brevity, this is a placeholder for the specific Sobel kernel application.
    # [G] Standard Sobel in 3D typically uses separable filters or explicit 3x3x3 kernels.
    # The result is a gradient vector (Gx, Gy, Gz).
    pass # Implementation specific to the Sobel variant chosen
```

**Parameters:**
*   `gradientMode`: Controls the selection. Typical values: `GRADIENT_CENTRAL` (default), `GRADIENT_SOBEL` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Noisy Data:** Simple central differences are sensitive to noise. If the data is noisy, selecting Sobel or more complex estimation schemes is recommended to avoid artifacts [S2, unofficial].
*   **Boundary Conditions:** Both underlying methods require handling voxels near volume boundaries. Common strategies include clamping, zero-padding, or extrapolation [G].
*   **Zero Magnitude:** If the gradient magnitude is zero (e.g., in homogeneous regions), normalization fails. The pseudocode handles this by returning a zero vector [G].

## Subtopics

### [Cubic Shading](9681-cubic-shading.md)
Uses all 26 neighbors in a 3x3x3 window for smoother normals, closer to analytical spheres. Choose when visual smoothness is prioritized over computational cost.

### [Neumann Gradient](3342-neumann-gradient.md)
Uses Neumann boundary conditions for gradient estimation. Choose when boundary accuracy is critical.

### [Forward Difference Gradient](3362-forward-difference-gradient.md)
Uses forward differences. Choose when memory access patterns favor forward strides or for specific boundary treatments.

### [4th-Order Finite Difference Gradient](4889-4th-order-finite-difference-gradient.md)
Higher-order accuracy. Choose when high precision is required and noise is low.

### [Gauss-Seidel with Successive Over-Relaxation (SOR)](6904-gauss-seidel-with-successive-over-relaxation-sor.md)
Iterative method for solving gradient fields. Choose for global consistency in noisy data.

### [Brick-local Gradient Estimation](8810-brick-local-gradient-estimation.md)
Local estimation within brick structures. Choose for hierarchical data formats.

### [Sobel-based Gradient Estimation](9967-sobel-based-gradient-estimation.md)
Robust gradient estimation using Sobel filters. Choose for noisy data.

### [Central Difference Stencils](9968-central-difference-stencils.md)
Standard central difference approximation. Choose for speed and low noise.

## Operating characteristics
*   **Time Complexity:** Depends on the selected method. Central difference is $O(1)$ with 6 texture lookups. Sobel is $O(1)$ but with more lookups (typically 26 for cubic shading or 18 for standard 3D Sobel) [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory, as it operates on the input texture directly.
*   **Strengths:** Flexible; allows runtime adaptation to data quality. Central difference is fast; Sobel is robust to noise [S1, S2, unofficial].
*   **Limitations:** Central difference is sensitive to noise [S2, unofficial]. Sobel is more computationally expensive.
*   **When to use:** Use Central Difference for clean, high-resolution data. Use Sobel for noisy or lower-resolution data where smooth shading is desired [S1, S2, unofficial].

## Background
Gradient selection is a common pattern in volume rendering pipelines to balance shading quality and performance. The concept of using normalized gradients as surface normals for shading is fundamental to shaded volume rendering [S2, unofficial]. The specific implementation of selecting between central and Sobel methods is found in practical volume rendering libraries and educational resources [S1, unofficial].

## Sources
[S1] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Rendering - Isenberg, https://tobias.isenberg.cc/graphics/uploads/Class/07-volume-rendering.pdf, fetched 2026-10-28 (unofficial)
