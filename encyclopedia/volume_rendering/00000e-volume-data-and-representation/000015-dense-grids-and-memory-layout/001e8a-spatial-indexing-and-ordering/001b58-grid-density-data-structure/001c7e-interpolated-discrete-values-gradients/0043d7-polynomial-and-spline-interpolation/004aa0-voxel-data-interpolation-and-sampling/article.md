# Voxel Data Interpolation and Sampling

Also called: getNextIntersectColor, getNextIntersectionGradient, getNextIntersectionScalar

## Overview
This method retrieves scalar values, gradients, and transformed colors at arbitrary 3D positions within a volume grid, supporting configurable interpolation and transfer function input modes [S1, unofficial]. It differs from its siblings by explicitly coupling the interpolation mode (`INTER_MODE`) with a transfer function input mode (`TRN_MODE`) that determines whether the transfer function receives raw voxel values or gradient-magnitude-enhanced inputs, a specific implementation detail found in the reference codebase [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Query point $\mathbf{p} = (x, y, z)$ in voxel space; Volume dimensions $XDIM, YDIM, ZDIM$; Interpolation mode `INTER_MODE`; Transfer function mode `TRN_MODE`; Volume scalar data array; Volume gradient data array.
*   **Output:** A 4-tuple $(R, G, B, A)$ representing the transformed color and opacity [S1, unofficial].

**Data Structures**
The volume data is stored in two contiguous vectors:
1.  `scalars`: `std::vector<float>` storing scalar values.
2.  `gradients`: `std::vector<glm::vec3>` storing 3D gradient vectors.

The index for a voxel at integer coordinates $(x, y, z)$ is calculated as:
$$ \text{index} = x + (y \cdot XDIM) + (z \cdot XDIM \cdot YDIM) $$
[S1, unofficial]

**Parameters**
*   `INTER_MODE`: Controls interpolation. Value `1` enables trilinear interpolation; Value `2` enables nearest neighbor sampling [S1, unofficial].
*   `TRN_MODE`: Controls transfer function input. Value `1` passes `(Voxel Value, Voxel Value)` to the transformer; Value `2` passes `(Voxel Value, Gradient Magnitude)` to the transformer [S1, unofficial].

**Steps**
1.  **Boundary Check:** Check if the integer coordinates $(x, y, z)$ are within the volume bounds $[0, XDIM-1] \times [0, YDIM-1] \times [0, ZDIM-1]$. If not, return $(0, 0, 0, 0)$ [S1, unofficial].
2.  **Data Retrieval:**
    *   Retrieve the scalar value $V$ at $(x, y, z)$ using the specified `INTER_MODE`.
    *   If `TRN_MODE` is `2`, retrieve the gradient vector $\mathbf{G}$ at $(x, y, z)$ using the specified `INTER_MODE`.
3.  **Gradient Processing (if `TRN_MODE` == 2):**
    *   Compute the gradient magnitude $M = \|\mathbf{G}\|$.
    *   Clamp $M$ to the range $[0, 1]$.
    *   Scale the clamped magnitude: $M' = M \cdot 256$.
    *   Pass $(V, M')$ to the transfer function transformer [S1, unofficial].
4.  **Transfer Function Application:**
    *   If `TRN_MODE` is `1`, pass $(V, V)$ to the transfer function transformer.
    *   If `TRN_MODE` is `2`, pass $(V, M')$ to the transfer function transformer.
    *   The transformer outputs the final color $(R, G, B)$ and opacity $A$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel_color(p: Vector3, volume: Volume) -> Color:
    x, y, z = p.x, p.y, p.z
    
    # 1. Boundary Check
    if not (0 <= x < volume.XDIM and 
            0 <= y < volume.YDIM and 
            0 <= z < volume.ZDIM):
        return Color(0, 0, 0, 0)

    # 2. Data Retrieval based on INTER_MODE
    # Note: The source code comment states "currently do the nearest neighbour only"
    # but the structure supports INTER_MODE. We assume nearest neighbor for 
    # the implementation details provided, or trilinear if INTER_MODE=1 is implemented.
    # [G] Standard trilinear interpolation requires fetching 8 neighbors and 
    # computing weights based on fractional part of p. 
    # [G] Nearest neighbor requires rounding p to nearest integer.
    
    if volume.INTER_MODE == 1:
        # Trilinear interpolation logic would go here
        # For the purpose of this article, we assume the source 
        # implementation defaults to nearest neighbor as noted.
        scalar_val = volume.get_scalar_nearest(x, y, z)
        grad_vec = volume.get_gradient_nearest(x, y, z)
    else:
        scalar_val = volume.get_scalar_nearest(x, y, z)
        grad_vec = volume.get_gradient_nearest(x, y, z)

    # 3. Transfer Function Input Preparation
    if volume.TRN_MODE == 1:
        # Pass (Voxel Value, Voxel Value)
        tf_input_val1 = scalar_val
        tf_input_val2 = scalar_val
    elif volume.TRN_MODE == 2:
        # Compute gradient magnitude
        grad_mag = length(grad_vec)
        # Clamp to [0, 1]
        grad_mag = clamp(grad_mag, 0.0, 1.0)
        # Scale by 256
        grad_mag_scaled = grad_mag * 256.0
        tf_input_val1 = scalar_val
        tf_input_val2 = grad_mag_scaled
    else:
        # Default or undefined mode
        tf_input_val1 = scalar_val
        tf_input_val2 = scalar_val

    # 4. Apply Transfer Function
    # The source mentions a 1D transfer function but notes it needs 
    # implementation to use gradient for opacity. 
    # [G] Assuming a generic transform function that takes two floats 
    # and returns RGBA.
    color = volume.transform(tf_input_val1, tf_input_val2)
    
    return color
```

**Edge Cases and Pitfalls**
*   **Interpolation Implementation Gap:** The source code comments explicitly state "currently do the nearest neighbour only (needs to implement voxel intersection)" despite defining `INTER_MODE` for trilinear interpolation [S1, unofficial]. Implementers must be aware that `INTER_MODE=1` may not be fully functional in the reference implementation.
*   **Gradient Clamping:** The gradient magnitude is clamped to $[0, 1]$ before scaling. If the raw gradient magnitude exceeds 1, information is lost. This assumes normalized gradients or specific data ranges [S1, unofficial].
*   **Transfer Function Limitations:** The reference implementation uses a 1D transfer function but notes the need to implement gradient-based opacity mapping [S1, unofficial]. Using `TRN_MODE=2` with an unmodified 1D TF may yield unexpected results if the TF does not account for the second input channel.

## Subtopics

### Grid Cell Collection via Corner Indices
Identifies the 8 corner indices of the enclosing voxel cell. This is a prerequisite for trilinear interpolation but does not perform the interpolation itself. [Link](1518-grid-cell-collection-via-corner-indices.md)

### Piecewise Quadratic Hermite Interpolation
Provides $C^1$ continuity using quadratic polynomials. More complex than linear/nearest neighbor but smoother. [Link](4001-piecewise-quadratic-hermite-interpolation.md)

### Hermite Interpolation of Scalar Fields
Uses both values and derivatives for interpolation. Complements this method by providing the gradient data if not precomputed. [Link](9450-hermite-interpolation-of-scalar-fields.md)

### B-spline Field Interpolation
Provides higher-order continuity ($C^2$) using B-spline basis functions. More computationally expensive but smoother results. [Link](11066-b-spline-field-interpolation.md)

### Third Order Elimination (Scaling)
A technique for optimizing or eliminating terms in higher-order interpolation. [Link](13712-third-order-elimination-scaling.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query point. The cost is dominated by the interpolation (8 neighbors for trilinear, 1 for nearest) and the transfer function evaluation.
*   **Memory Complexity:** $O(N)$ where $N$ is the number of voxels, storing both scalar and gradient data.
*   **Strengths:** Simple to implement; supports flexible transfer function inputs via `TRN_MODE`; efficient for real-time rendering if nearest neighbor is used.
*   **Limitations:** The reference implementation lacks full trilinear interpolation support; gradient clamping may distort high-frequency features; 1D transfer function limitation noted in source.
*   **When to use:** When simple, fast sampling is required and gradient-based transfer functions are not critical or are approximated.
*   **When not to use:** When high-quality smooth interpolation is required and the reference implementation's lack of trilinear support is a blocker; when gradient magnitude is large and clamping causes significant data loss.

## Background
This method originates from a student project for CSE-333 Computer Graphics (2020) involving volume ray casting [S1, unofficial]. It was designed to support X-ray rendering techniques and 1D transfer functions, with noted plans for gradient-based opacity mapping [S1, unofficial].

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
