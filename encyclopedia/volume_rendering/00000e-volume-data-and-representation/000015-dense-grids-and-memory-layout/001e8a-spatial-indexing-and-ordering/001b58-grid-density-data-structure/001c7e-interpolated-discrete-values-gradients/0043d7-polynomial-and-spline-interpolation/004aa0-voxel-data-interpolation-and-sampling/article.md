# Voxel Data Interpolation and Sampling

Also called: getNextIntersectColor, getNextIntersectionGradient, getNextIntersectionScalar

## Overview
This method retrieves scalar values, gradients, and transformed colors at arbitrary 3D positions within a volume grid, supporting configurable interpolation and transfer function input modes [S1, unofficial]. It differs from sibling methods by explicitly coupling the sampling of the scalar field with optional gradient magnitude extraction to drive transfer function inputs, rather than relying solely on the raw voxel value [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Query position $\mathbf{p} = (x, y, z)$; grid dimensions $XDIM, YDIM, ZDIM$; interpolation mode `INTER_MODE`; transfer function mode `TRN_MODE`.
- **Outputs**: A 4D vector $(R, G, B, A)$ representing the transformed color and opacity.

**Data Structures**
The volume data is stored in two contiguous vectors:
1.  `scalars`: `std::vector<float>` containing the voxel values.
2.  `gradients`: `std::vector<glm::vec3>` containing the gradient vectors $\nabla V$.

The linear index for a voxel at integer coordinates $(x, y, z)$ is calculated as:
$$ \text{index} = x + (y \cdot XDIM) + (z \cdot XDIM \cdot YDIM) $$
[G] This assumes row-major storage where $x$ varies fastest.

**Parameters**
- `INTER_MODE`: Controls interpolation. `1` for trilinear interpolation, `2` for nearest neighbor [S1, unofficial].
- `TRN_MODE`: Controls transfer function input. `1` for input $(V, V)$, `2` for input $(V, |\nabla V|)$ [S1, unofficial].

**Steps**
1.  **Boundary Check**: If the query coordinates $(x, y, z)$ are outside the grid bounds $[0, XDIM-1] \times [0, YDIM-1] \times [0, ZDIM-1]$, return $(0, 0, 0, 0)$ [S1, unofficial].
2.  **Sample Scalar**: Retrieve the scalar value $V$ at $(x, y, z)$ using the method specified by `INTER_MODE`.
3.  **Sample Gradient (if needed)**: If `TRN_MODE` is `2`, retrieve the gradient vector $\mathbf{g}$ at $(x, y, z)$ using the method specified by `INTER_MODE`.
4.  **Compute Gradient Magnitude**: If `TRN_MODE` is `2`, compute the magnitude $M = |\mathbf{g}| = \sqrt{g_x^2 + g_y^2 + g_z^2}$.
5.  **Clamp and Scale**: Clamp $M$ to $[0, 1]$ and scale by 256: $M' = \min(1.0, M) \cdot 256$ [S1, unofficial].
6.  **Transform**: Pass the values to the transfer function transformer:
    - If `TRN_MODE` is `1`: Input is $(V, V)$.
    - If `TRN_MODE` is `2`: Input is $(V, M')$.
7.  **Return**: Return the resulting color and alpha from the transformer.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel_color(p: Vector3, grid_dims: Tuple[int, int, int], 
                    scalars: List[float], gradients: List[Vector3],
                    inter_mode: int, trn_mode: int) -> Vector4:
    x, y, z = p.x, p.y, p.z
    xd, yd, zd = grid_dims
    
    # 1. Boundary Check
    if not (0 <= x < xd and 0 <= y < yd and 0 <= z < zd):
        return Vector4(0, 0, 0, 0)
        
    # 2. Sample Scalar
    # Note: Trilinear interpolation logic is not fully detailed in source, 
    # so we assume a helper function exists for the specific mode.
    if inter_mode == 2: # Nearest Neighbor
        v = scalars[voxel_index(x, y, z, xd, yd)]
    else: # Trilinear (Mode 1)
        v = trilinear_interpolate(scalars, p, grid_dims)
        
    # 3 & 4. Sample Gradient and Compute Magnitude
    if trn_mode == 2:
        if inter_mode == 2: # Nearest Neighbor
            g = gradients[voxel_index(x, y, z, xd, yd)]
        else: # Trilinear
            g = trilinear_interpolate_vector(gradients, p, grid_dims)
            
        mag = length(g)
        # 5. Clamp and Scale
        mag = min(1.0, mag) * 256.0
        input_val_2 = mag
    else:
        input_val_2 = v
        
    # 6. Transform
    # Assuming a global or passed transfer function object
    color = transfer_function.transform(v, input_val_2)
    
    # 7. Return
    return color

def voxel_index(x, y, z, xd, yd):
    return x + (y * xd) + (z * xd * yd)
```

**Edge Cases and Pitfalls**
- The source notes that while the structure supports trilinear interpolation, the implementation may default to nearest neighbor if voxel intersection logic is not fully implemented [S1, unofficial].
- Precision issues may arise if gradient magnitudes are not properly normalized before scaling, potentially causing overflow in the transfer function if $M > 1$ before clamping.

## Subtopics

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
Focuses on retrieving the 8 corner indices for a grid cell containing a query point, a prerequisite for trilinear interpolation.

### [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md)
Provides $C^1$ continuous interpolation using quadratic Hermite basis functions, suitable when gradient data is explicitly stored and interpolated.

### [B-spline Field Interpolation](11066-b-spline-field-interpolation.md)
Uses B-spline basis functions for higher-order continuity ($C^2$ for cubic), offering smoother gradients at the cost of larger support.

## Operating characteristics
- **Time Complexity**: $O(1)$ per query, as it accesses a constant number of neighbors (8 for trilinear, 1 for nearest).
- **Memory Complexity**: $O(N)$ for storing scalars and gradients, where $N$ is the number of voxels.
- **Strengths**: Simple to implement; supports flexible transfer function inputs via gradient magnitude.
- **Limitations**: Nearest neighbor interpolation causes aliasing; trilinear interpolation may be insufficient for high-frequency details compared to cubic methods.
- **When to use**: Real-time volume rendering where performance is critical and visual quality requirements are moderate.

## Background
This method originates from a student project for CSE-333 Computer Graphics (2020), implementing volume ray casting with a focus on transfer function customization [S1, unofficial]. It builds on the principles of polynomial and spline interpolation by providing a practical interface for sampling dense grid data.

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
