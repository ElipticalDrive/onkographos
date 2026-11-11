# Voxel Data Interpolation and Sampling

Also called: getNextIntersectColor, getNextIntersectionGradient, getNextIntersectionScalar

## Overview
Voxel Data Interpolation and Sampling is a method for retrieving scalar values, gradients, and transformed colors at arbitrary 3D positions within a volume grid, supporting configurable interpolation and transfer function modes [S1, unofficial]. It differs from its siblings by explicitly decoupling the spatial sampling (interpolation) from the transfer function input preparation (gradient magnitude calculation), allowing the renderer to choose between simple value-based or gradient-enhanced opacity mapping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Query point $\mathbf{p} = (x, y, z)$ in voxel coordinates; Volume data structure containing scalar and gradient fields; Parameters `INTER_MODE` and `TRN_MODE`.
*   **Output:** A 4-tuple $(R, G, B, A)$ representing the transformed color and opacity.

**Data Structures:**
The volume data is stored in two contiguous vectors:
1.  `scalars`: `std::vector<float>` storing scalar values.
2.  `gradients`: `std::vector<glm::vec3>` storing 3D gradient vectors.

The index for a voxel at integer coordinates $(x, y, z)$ in a grid with dimensions $X_{DIM}, Y_{DIM}, Z_{DIM}$ is calculated as:
$$ \text{index} = x + (y \cdot X_{DIM}) + (z \cdot X_{DIM} \cdot Y_{DIM}) $$

**Parameters:**
*   `INTER_MODE`: Controls spatial interpolation.
    *   `1`: Trilinear interpolation (using nearest 8 neighbors) [S1, unofficial].
    *   `2`: Nearest neighbor interpolation [S1, unofficial].
*   `TRN_MODE`: Controls the input passed to the transfer function transformer.
    *   `1`: Passes $(V, V)$, where $V$ is the interpolated scalar value [S1, unofficial].
    *   `2`: Passes $(V, M)$, where $V$ is the interpolated scalar value and $M$ is the clamped gradient magnitude [S1, unofficial].

**Steps:**
1.  **Boundary Check:** If the query point $\mathbf{p}$ has integer coordinates outside the volume bounds $[0, X_{DIM}-1] \times [0, Y_{DIM}-1] \times [0, Z_{DIM}-1]$, return $(0, 0, 0, 0)$ [S1, unofficial].
2.  **Interpolation:**
    *   If `INTER_MODE` is `2` (Nearest Neighbor): Retrieve the scalar value $V$ at the nearest integer voxel coordinate.
    *   If `INTER_MODE` is `1` (Trilinear): Perform trilinear interpolation using the 8 neighboring voxels to compute $V$ [S1, unofficial].
3.  **Gradient Retrieval (if `TRN_MODE` == 2):**
    *   Retrieve the gradient vector $\mathbf{g}$ at the same location (interpolated if necessary, though the source notes imply nearest neighbor for gradients in the current implementation context) [S1, unofficial].
    *   Compute the magnitude $M = \|\mathbf{g}\|$.
    *   Clamp $M$ to the range $[0, 1]$.
    *   Scale the magnitude: $M_{scaled} = M \cdot 256$.
4.  **Transformation:**
    *   If `TRN_MODE` is `1`: Call the transformer function $T(V, V)$.
    *   If `TRN_MODE` is `2`: Call the transformer function $T(V, M_{scaled})$.
5.  **Return:** The result of the transformer function as the final color and alpha.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel_color(p: Vector3, volume: Volume, inter_mode: int, trn_mode: int) -> Color:
    # 1. Boundary Check
    if not volume.is_in_bounds(p):
        return Color(0, 0, 0, 0)

    # 2. Interpolation
    # Note: The source notes state "currently do the nearest neighbour only" 
    # but the structure supports INTER_MODE=1 for trilinear. 
    # We implement the logic described for INTER_MODE=1 as the general case.
    
    if inter_mode == 2:
        # Nearest Neighbor
        ix, iy, iz = round(p.x), round(p.y), round(p.z)
        scalar_val = volume.get_scalar(ix, iy, iz)
        grad_vec = volume.get_gradient(ix, iy, iz)
    else:
        # Trilinear Interpolation (INTER_MODE == 1)
        # Implementation detail: Trilinear interpolation requires fetching 
        # 8 corner values and weights based on fractional part of p.
        # Since the source code snippet does not provide the trilinear 
        # implementation, we assume a standard trilinear function exists.
        scalar_val = volume.trilinear_interpolate_scalar(p)
        grad_vec = volume.trilinear_interpolate_gradient(p)

    # 3. Gradient Processing (if TRN_MODE == 2)
    if trn_mode == 2:
        grad_magnitude = length(grad_vec)
        grad_magnitude = clamp(grad_magnitude, 0.0, 1.0)
        grad_magnitude_scaled = grad_magnitude * 256.0
        result = volume.transform(scalar_val, grad_magnitude_scaled)
    else:
        # TRN_MODE == 1: Pass scalar value twice
        result = volume.transform(scalar_val, scalar_val)

    return result
```

**Edge Cases and Pitfalls:**
*   **Implementation Gap:** The source code comments indicate that while `INTER_MODE=1` is defined, the actual trilinear interpolation logic may not be fully implemented in the referenced repository, defaulting to nearest neighbor [S1, unofficial].
*   **Gradient Clamping:** The gradient magnitude is explicitly clamped to $[0, 1]$ before scaling. This assumes the gradient vectors are normalized or scaled appropriately during volume construction [S1, unofficial].
*   **Coordinate System:** The index calculation assumes a specific memory layout (likely row-major or column-major depending on the order of $x, y, z$ multiplication). The formula $x + y \cdot X_{DIM} + z \cdot X_{DIM} \cdot Y_{DIM}$ implies $x$ varies fastest [S1, unofficial].

## Subtopics

### Grid Cell Collection via Corner Indices
Determines the 8 voxel indices surrounding a query point for trilinear interpolation. It sets apart by providing the discrete indices required for the weighted sum, whereas this method handles the continuous value retrieval. Choose when you need to decouple index calculation from value fetching. [1518-grid-cell-collection-via-corner-indices.md]

### Piecewise Quadratic Hermite Interpolation
Uses quadratic polynomials with derivative information for smoother interpolation. It sets apart by offering $C^1$ continuity with fewer control points than cubic methods. Choose when smoothness is critical but cubic costs are too high. [4001-piecewise-quadratic-hermite-interpolation.md]

### B-spline Field Interpolation
Uses B-spline basis functions for interpolation. It sets apart by providing higher-order continuity ($C^2$) and better frequency response. Choose when high-quality smoothness is required. [11066-b-spline-field-interpolation.md]

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, as it accesses a fixed number of neighbors (8 for trilinear, 1 for nearest) regardless of grid size [11066-b-spline-field-interpolation.md].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of voxels, storing both scalar and gradient fields [S1, unofficial].
*   **Strengths:** Simple to implement; supports gradient-based opacity mapping which enhances surface detail perception [S1, unofficial].
*   **Limitations:** Trilinear interpolation is only $C^0$ continuous; gradient calculation can be noisy if the volume data is not smoothed [S1, unofficial].
*   **When to use:** Real-time volume rendering where gradient-enhanced opacity is desired.
*   **When not to use:** When high-order continuity is required (use B-spline) or when memory bandwidth is extremely constrained (use nearest neighbor).

## Background
This method originates from the implementation of a volume ray casting system described in the CSE-333 Computer Graphics 2020 course repository [S1, unofficial]. It was developed to support 1D transfer functions with optional gradient-based opacity mapping, allowing users to customize how scalar values and gradient magnitudes influence the final color and alpha [S1, unofficial].

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
