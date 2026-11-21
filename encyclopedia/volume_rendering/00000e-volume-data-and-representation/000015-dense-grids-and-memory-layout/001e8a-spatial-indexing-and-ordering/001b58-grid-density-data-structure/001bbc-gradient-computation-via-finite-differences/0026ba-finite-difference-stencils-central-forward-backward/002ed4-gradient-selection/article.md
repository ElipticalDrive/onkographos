# Gradient Selection

Also called: Central differences, Finite Difference Stencils, Gradient Mode Selection, Gradient estimation via finite differences

## Overview
Gradient Selection is a runtime dispatch mechanism that chooses between specific finite difference stencil implementations (typically Central Difference and Sobel) to estimate the gradient vector of a scalar volume field for shading purposes [S1, unofficial]. It solves the problem of adapting gradient estimation to the `gradientMode` uniform, allowing users to trade off between the isotropic accuracy of central differences and the noise-robustness of Sobel operators [S1, unofficial]. It distinguishes itself from precomputed gradient storage by computing derivatives on-the-fly and from 3D Sobel kernels by often relying on 1D stencils or specific hardware-accelerated texture gradient functions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: `textureCoord` (3D coordinate in volume space), `gradientMode` (enum or integer flag).
*   Output: Normalized gradient vector $\mathbf{g} = \nabla V / \|\nabla V\|$.

**Data Structures:**
*   `gradientMode`: An integer or enum. Common values include `GRADIENT_CENTRAL` and `GRADIENT_SOBEL` [S1, unofficial].

**Steps:**
1.  Check the value of `gradientMode`.
2.  If `gradientMode` equals `GRADIENT_SOBEL`, invoke the Sobel gradient estimation routine.
3.  Otherwise, invoke the Central Difference gradient estimation routine.
4.  Normalize the resulting gradient vector to obtain the surface normal approximation [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_selection(texture_coord, gradient_mode):
    """
    Dispatches gradient estimation based on the selected mode.
    
    Args:
        texture_coord: vec3, coordinates in the volume texture space.
        gradient_mode: int, 0 for Central Difference, 1 for Sobel (example mapping).
        
    Returns:
        vec3, the normalized gradient vector.
    """
    if gradient_mode == GRADIENT_SOBEL:
        grad = gradient_sobel(texture_coord)
    else:
        # Default to Central Difference if mode is not explicitly Sobel
        grad = gradient_central(texture_coord)
        
    # Normalize to get the surface normal approximation
    # Handle zero magnitude to avoid division by zero [G]
    magnitude = length(grad)
    if magnitude > 1e-6:
        return grad / magnitude
    else:
        return vec3(0.0, 0.0, 0.0)

def gradient_central(coord):
    """
    Estimates gradient using central differences.
    Assumes access to a texture sampling function sample_3d.
    """
    # Partial derivatives using central differencing schemes [S2, unofficial]
    # h is the voxel spacing, typically 1.0 in normalized texture coordinates
    h = 1.0 / texture_size # Or 0.5 if using texel offsets directly
    
    # Example implementation for central difference
    # dx = (V(x+h, y, z) - V(x-h, y, z)) / (2h)
    # This requires sampling neighbors. 
    # Note: Specific neighbor offsets depend on the texture coordinate system [G]
    dx = (sample_3d(coord + vec3(h, 0, 0)) - sample_3d(coord - vec3(h, 0, 0))) / (2.0 * h)
    dy = (sample_3d(coord + vec3(0, h, 0)) - sample_3d(coord - vec3(0, h, 0))) / (2.0 * h)
    dz = (sample_3d(coord + vec3(0, 0, h)) - sample_3d(coord - vec3(0, 0, h))) / (2.0 * h)
    
    return vec3(dx, dy, dz)

def gradient_sobel(coord):
    """
    Estimates gradient using Sobel operator.
    """
    # Implementation details for Sobel are not provided in the source notes.
    # Typically involves 3x3x3 convolution or separable 1D/2D passes.
    # Placeholder for actual Sobel logic [G]
    pass
```

**Parameters:**
*   `gradientMode`: Controls the selection. Typical values are not explicitly defined in the sources, but `GRADIENT_SOBEL` and a default (Central) are implied [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Noisy Data:** Simple central differences may produce inaccurate normals in noisy data; Sobel or more complex estimation schemes are recommended in such cases [S2, unofficial].
*   **Boundaries:** Boundary handling (clamping, zero-padding) is required when stencil neighbors fall outside the volume domain [G].

## Subtopics

### [Cubic Shading](9681-cubic-shading.md)
Uses all 26 neighbors in a 3x3x3 window for smoother normals, closer to analytical spheres. Choose when high-quality shading of smooth surfaces is needed.

### [Gradient Selection — Central/Forward/Backward](12132-gradient-selection-central-forward-backward.md)
Selects the appropriate 1D stencil based on axis index and boundary conditions. Choose when explicit control over directional derivative accuracy at boundaries is required.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel, as it involves a constant number of texture lookups and arithmetic operations.
*   **Memory Complexity:** $O(1)$ auxiliary memory, as gradients are computed on-the-fly.
*   **Strengths:** Flexible; allows runtime switching between accuracy (Central) and noise robustness (Sobel). No precomputation storage required.
*   **Limitations:** Central differences are sensitive to noise. Sobel implementation complexity varies. Boundary effects can distort normals if not handled correctly.
*   **When to use:** When shading quality needs to be tuned per-frame or per-volume, and precomputed gradients are too expensive.
*   **When not to use:** For very noisy data without Sobel fallback, or when hardware acceleration for specific gradient operations is unavailable.

## Background
Gradient estimation via finite differences is a fundamental technique in volume rendering to approximate surface normals for shading [S2, unofficial]. The selection between different stencil types (Central, Sobel) allows renderers to balance visual fidelity and noise sensitivity. The specific implementation described here appears in WebGL-based volume rendering contexts [S1, unofficial].

## Sources
[S1] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Rendering - Isenberg, https://tobias.isenberg.cc/graphics/uploads/Class/07-volume-rendering.pdf, fetched 2026-10-28 (unofficial)
