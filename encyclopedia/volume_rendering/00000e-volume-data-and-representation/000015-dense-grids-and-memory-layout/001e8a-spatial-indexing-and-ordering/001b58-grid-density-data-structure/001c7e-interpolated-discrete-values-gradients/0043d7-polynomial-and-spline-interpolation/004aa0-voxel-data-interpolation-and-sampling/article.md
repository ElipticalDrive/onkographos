# Voxel Data Interpolation and Sampling

Also called: getNextIntersectColor, getNextIntersectionGradient, getNextIntersectionScalar

## Overview
A method to retrieve scalar values, gradients, and transformed colors at arbitrary 3D positions within a volume grid, supporting different interpolation modes and transformation modes [S1, unofficial]. It differs from its siblings by explicitly coupling the sampling of the scalar field and its spatial gradient to a configurable transfer function input mode, rather than just returning raw field values [S1, unofficial].

## How it works
**Inputs:**
*   Query point $\mathbf{p} = (x, y, z)$ in grid index space (floating point).
*   Volume data:
    *   `scalars`: `vector<float>` of size $X_{DIM} \times Y_{DIM} \times Z_{DIM}$.
    *   `gradients`: `vector<glm::vec3>` of size $X_{DIM} \times Y_{DIM} \times Z_{DIM}$.
*   Parameters:
    *   `INTER_MODE`: Integer. 1 for trilinear interpolation, 2 for nearest neighbor [S1, unofficial].
    *   `TRN_MODE`: Integer. 1 for (Voxel Value, Voxel Value), 2 for (Voxel Value, Gradient Magnitude) [S1, unofficial].
*   Grid dimensions: $X_{DIM}, Y_{DIM}, Z_{DIM}$.

**Outputs:**
*   A tuple containing the interpolated scalar value, the interpolated gradient vector, and the resulting color/opacity pair passed to the transfer function.

**Data Structures:**
The volume data is stored in two parallel vectors. The index for a voxel at integer coordinates $(x, y, z)$ is calculated as:
$$ \text{idx} = x + (y \cdot X_{DIM}) + (z \cdot X_{DIM} \cdot Y_{DIM}) $$
[S1, unofficial].

**Steps:**
1.  **Boundary Check**: Check if the integer coordinates $(\lfloor x \rfloor, \lfloor y \rfloor, \lfloor z \rfloor)$ are within bounds $[0, X_{DIM}-1] \times [0, Y_{DIM}-1] \times [0, Z_{DIM}-1]$. If not, return $(0, 0, 0, 0)$ [S1, unofficial].
2.  **Interpolation Mode Selection**:
    *   If `INTER_MODE` is 2 (Nearest Neighbor): Retrieve the scalar $S$ and gradient $\mathbf{G}$ at the nearest integer grid point.
    *   If `INTER_MODE` is 1 (Trilinear): Retrieve the scalar $S$ and gradient $\mathbf{G}$ using trilinear interpolation over the 8 nearest neighbors [S1, unofficial].
3.  **Transfer Function Input Preparation**:
    *   If `TRN_MODE` is 1: Pass $(S, S)$ to the transformer.
    *   If `TRN_MODE` is 2: Compute the gradient magnitude $M = \|\mathbf{G}\|$. Clamp $M$ to $[0, 1]$. Scale by 256. Pass $(S, M \cdot 256)$ to the transformer [S1, unofficial].
4.  **Transformation**: Apply the 1D transfer function to the prepared inputs to get color and opacity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel_data(p: Vector3f, scalars: List[float], gradients: List[Vector3f], 
                   dims: Tuple[int, int, int], inter_mode: int, trn_mode: int) -> Tuple[float, Vector3f, float, float]:
    x, y, z = p.x, p.y, p.z
    xd, yd, zd = dims[0], dims[1], dims[2]
    
    # Boundary check for integer coordinates
    ix, iy, iz = int(floor(x)), int(floor(y)), int(floor(z))
    if not (0 <= ix < xd and 0 <= iy < yd and 0 <= iz < zd):
        return 0.0, Vector3f(0,0,0), 0.0, 0.0

    # Helper to get scalar at integer index
    def get_scalar(ix, iy, iz):
        idx = ix + (iy * xd) + (iz * xd * yd)
        return scalars[idx]

    # Helper to get gradient at integer index
    def get_gradient(ix, iy, iz):
        idx = ix + (iy * xd) + (iz * xd * yd)
        return gradients[idx]

    # Interpolation
    if inter_mode == 2: # Nearest Neighbor
        s_val = get_scalar(ix, iy, iz)
        g_val = get_gradient(ix, iy, iz)
    elif inter_mode == 1: # Trilinear
        # Note: Implementation details for trilinear interpolation of scalars and gradients
        # are not explicitly provided in the source, but the structure supports it.
        # [G] Standard trilinear interpolation would involve weighting the 8 corners
        # based on fractional distances (fx, fy, fz).
        s_val = trilinear_interpolate_scalar(x, y, z, get_scalar, xd, yd, zd)
        g_val = trilinear_interpolate_vector(x, y, z, get_gradient, xd, yd, zd)
    else:
        raise ValueError("Unsupported INTER_MODE")

    # Transfer Function Input Preparation
    if trn_mode == 1:
        input1 = s_val
        input2 = s_val
    elif trn_mode == 2:
        input1 = s_val
        mag = length(g_val)
        mag = clamp(mag, 0.0, 1.0)
        input2 = mag * 256.0
    else:
        raise ValueError("Unsupported TRN_MODE")

    # Apply Transfer Function (1D)
    # [G] The specific transfer function implementation is not detailed, 
    # but it maps (input1, input2) to (color, opacity).
    color, opacity = apply_transfer_function(input1, input2)

    return s_val, g_val, color, opacity
```

**Parameters:**
*   `INTER_MODE`: 1 (Trilinear), 2 (Nearest Neighbor) [S1, unofficial].
*   `TRN_MODE`: 1 (Scalar, Scalar), 2 (Scalar, Gradient Magnitude * 256) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The source notes that the implementation currently uses nearest neighbor only and requires implementation of voxel intersection for trilinear interpolation [S1, unofficial].
*   The code comments mention trilinear interpolation with nearest 8 neighbors, but the actual logic in `getNextIntersectColor` is not fully shown [S1, unofficial].
*   Out-of-bounds access returns $(0,0,0,0)$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per query, dependent on the number of neighbors (8 for trilinear, 1 for nearest).
*   **Memory Complexity**: $O(N)$ for storing scalars and gradients, where $N$ is the number of voxels.
*   **Strengths**: Simple to implement, supports gradient-based transfer functions.
*   **Limitations**: Trilinear interpolation implementation is noted as incomplete in the source [S1, unofficial]. Only 1D transfer functions are supported [S1, unofficial].
*   **When to use**: When gradient information is needed for opacity or color mapping and computational simplicity is preferred.

## Background
This method originates from a student project for CSE-333 Computer Graphics (2020) [S1, unofficial]. It was developed as part of a volume ray casting implementation that reads `.ctscan` files and computes gradients [S1, unofficial]. The project uses a 1D transfer function and currently implements average rendering (X-Ray) [S1, unofficial].

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
