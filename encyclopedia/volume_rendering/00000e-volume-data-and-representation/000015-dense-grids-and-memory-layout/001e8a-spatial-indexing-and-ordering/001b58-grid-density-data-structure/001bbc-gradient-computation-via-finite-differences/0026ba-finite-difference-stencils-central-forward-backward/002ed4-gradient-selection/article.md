# Gradient Selection

Also called: Central differences, Finite Difference Stencils, Gradient Mode Selection, Gradient estimation via finite differences

## Overview
Gradient selection is the runtime decision mechanism that chooses between available finite difference gradient estimation schemes, primarily central differences and Sobel-based estimation, to compute the normalized gradient vector of a scalar field for shading [S1, S2, unofficial]. It solves the problem of selecting the appropriate numerical derivative method based on data characteristics or user configuration, where noisy data may necessitate more complex estimation schemes than simple central differences [S2, unofficial]. It distinguishes itself from sibling methods by acting as a dispatcher or mode selector rather than implementing a specific stencil kernel itself, allowing dynamic switching between accuracy-focused (Sobel) and performance-focused (Central) estimators [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A scalar volume texture, a sampling coordinate (texture coordinate or voxel index), and a configuration parameter `gradientMode` (or boundary context).
- Output: A normalized 3D gradient vector $\mathbf{G} = (G_x, G_y, G_z)$ representing the surface normal approximation at the query point.

**Data Structures:**
- `gradientMode`: An integer or enum uniform variable. Common values include `GRADIENT_CENTRAL` and `GRADIENT_SOBEL` [S1, unofficial].

**Steps:**
1. Evaluate the `gradientMode` parameter.
2. If `gradientMode` indicates Sobel estimation:
   - Invoke the Sobel gradient estimation routine using the provided texture coordinate [S1, unofficial].
3. Otherwise (defaulting to Central Difference):
   - Invoke the Central Difference gradient estimation routine using the provided texture coordinate [S1, unofficial].
4. Normalize the resulting gradient vector $\mathbf{G}$ to unit length [S2, unofficial].

**Pseudocode:**
The following pseudocode implements the selection logic described in [S1, unofficial]. It assumes helper functions `gradientSobel` and `gradientCentral` are implemented according to their respective articles.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_gradient_mode(texture_coord, gradient_mode):
    """
    Selects and returns the gradient vector based on the specified mode.
    
    Args:
        texture_coord: 3D coordinate (x, y, z) in normalized [0, 1] or voxel space.
        gradient_mode: Integer constant indicating the estimation method.
                       Expected values: GRADIENT_SOBEL, GRADIENT_CENTRAL.
    
    Returns:
        Normalized 3D gradient vector (gx, gy, gz).
    """
    # Constants for mode selection
    GRADIENT_SOBEL = 1
    GRADIENT_CENTRAL = 0
    
    if gradient_mode == GRADIENT_SOBEL:
        # Delegate to Sobel estimation which handles its own boundary conditions
        # and uses larger stencils for noise robustness.
        return gradient_sobel(texture_coord)
    else:
        # Default to Central Difference for standard cases.
        # This is the standard first-order derivative approximation.
        return gradient_central(texture_coord)

def gradient_central(texture_coord):
    """
    Placeholder for Central Difference implementation.
    Computes partial derivatives using immediate neighbors.
    See [Parent](3858-finite-difference-stencils-central-forward-backward.md) for details.
    """
    # Implementation details are in the parent article.
    # Returns unnormalized gradient vector.
    pass

def gradient_sobel(texture_coord):
    """
    Placeholder for Sobel-based estimation.
    Uses a 3x3x3 cubic window (26 neighbors) for smoother normals.
    See [Sobel-based Gradient Estimation](9967-sobel-based-gradient-estimation.md) for details.
    """
    # Implementation details are in the sibling article.
    # Returns unnormalized gradient vector.
    pass
```

**Boundary Conditions and Edge Cases:**
- When using Central Differences, boundary voxels require clamping, zero-padding, or extrapolation as defined in the parent article [G].
- Sobel estimation typically handles boundaries differently, often by extending the kernel or using specific boundary weights, which is why it is preferred for noisy data where simple central differences might fail or produce artifacts [S2, unofficial].
- If the volume data is noisy, the selection logic should ideally default to or force `GRADIENT_SOBEL` to mitigate noise amplification inherent in simple finite differences [S2, unofficial].

## Subtopics

### [Cubic Shading](9681-cubic-shading.md)
Cubic shading is a gradient estimation technique that calculates surface normals using all 26 neighbors of a center voxel within a 3x3x3 cubic window, rather than relying solely on the 6 immediate orthogonal neighbors. It produces images with smoother shading and normals closer to those of analytically defined spheres compared to standard grey-level gradient shading. This method is often associated with the Sobel mode in gradient selection contexts due to its use of a larger neighborhood.

### Gradient Selection — Central/Forward/Backward
This variant estimates the gradient/slope at a specified position using central, forward, or backward finite differences depending on boundary conditions [S3, unofficial]. For integer z-axis positions: if the index $i$ is strictly between the minimum and maximum-1 bounds ($min < i < max-1$), it returns the central difference `next - prev` [S3, unofficial]. If $i \le min$, it returns the forward difference `next - current` [S3, unofficial]. If $i \ge max-1$, it returns the backward difference `current - prev` [S3, unofficial] [S3, unofficial]. For non-integer z-axis positions, it uses `getRealVoxel` for interpolation instead of `getVoxel` [S3, unofficial]. The parameters are: $i$ (the axis index being altered, typically z), $min$ (minimum valid axis value, normally 0), and $max$ (maximum valid axis value, normally axis length - 1) [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel for the selection step itself. The cost depends on the selected estimator: Central Difference is $O(1)$ with a small constant (6-7 taps), while Sobel is $O(1)$ with a larger constant (26-27 taps or equivalent convolution cost).
- **Memory Complexity:** $O(1)$ auxiliary memory for the selection logic. The gradient vectors are stored in the output buffer.
- **Performance:** Central Difference is faster due to fewer memory accesses and simpler arithmetic. Sobel is slower but more robust to noise [S2, unofficial].
- **Strengths:** Allows dynamic adaptation to data quality (noise vs. smoothness) and hardware capabilities.
- **Limitations:** Requires the application to manage the `gradientMode` state. Switching modes may require re-evaluation of all voxels if the gradient field is precomputed.
- **When to use:** Use Central Difference for smooth, high-resolution data where performance is critical. Use Sobel or Cubic Shading for noisy data or when smoother, more accurate normals are required for shading [S1, S2, unofficial].

## Background
Gradient estimation via finite differences is a fundamental technique in volume rendering for approximating surface normals from scalar field data [S2, unofficial]. The specific selection mechanism between Central and Sobel modes is documented in community-driven volume rendering implementations [S1, unofficial]. The boundary-aware selection (Central/Forward/Backward) is also found in educational and open-source volume rendering projects [S3, unofficial]. The use of gradient magnitude to modulate opacity and interpolate between unshaded and shaded colors is a standard practice in volume rendering pipelines [S2, unofficial].

## Sources
[S1] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Rendering - Isenberg, https://tobias.isenberg.cc/graphics/uploads/Class/07-volume-rendering.pdf, fetched 2026-10-28 (unofficial)
[S3] joshcodd/volume-rendering, https://github.com/joshcodd/volume-rendering, fetched 2026-10-26 (unofficial)
