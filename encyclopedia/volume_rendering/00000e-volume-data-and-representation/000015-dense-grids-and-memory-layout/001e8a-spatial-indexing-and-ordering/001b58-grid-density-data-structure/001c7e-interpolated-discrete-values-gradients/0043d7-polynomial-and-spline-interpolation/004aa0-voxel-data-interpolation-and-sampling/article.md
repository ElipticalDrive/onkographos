# Voxel Data Interpolation and Sampling

Also called: getNextIntersectColor, getNextIntersectionGradient, getNextIntersectionScalar

## Overview
This method retrieves scalar values, gradients, and transformed colors at arbitrary 3D positions within a volume grid, supporting configurable interpolation and transfer function modes [S1, unofficial]. It differs from its siblings by explicitly exposing the raw scalar and gradient data structures and providing a specific transfer function input mode that utilizes gradient magnitude, whereas [B-spline Field Interpolation](11066-b-spline-field-interpolation.md) focuses on higher-order continuity and [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md) focuses on index calculation without the associated transfer function logic.

## How it works
**Inputs and Outputs:**
*   **Input:** Query point $\mathbf{p} = (x, y, z)$ in voxel coordinates; grid dimensions $XDIM, YDIM, ZDIM$; interpolation mode `INTER_MODE`; transfer function mode `TRN_MODE`.
*   **Output:** A 4D color/alpha vector (RGBA) or scalar/gradient pair depending on the calling context.

**Data Structures:**
The volume data is stored in two contiguous vectors:
1.  `scalars`: `std::vector<float>` containing the voxel values.
2.  `gradients`: `std::vector<glm::vec3>` containing the gradient vectors.

The index for a voxel at integer coordinates $(x, y, z)$ is calculated as:
$$
idx = x + y \cdot XDIM + z \cdot XDIM \cdot YDIM
$$

**Parameters:**
*   `INTER_MODE`: Controls interpolation.
    *   `1`: Trilinear interpolation.
    *   `2`: Nearest neighbor interpolation.
*   `TRN_MODE`: Controls transfer function input arguments.
    *   `1`: Passes `(Voxel Value, Voxel Value)` to the transformer.
    *   `2`: Passes `(Voxel Value, Gradient Magnitude)` to the transformer.

**Steps:**
1.  **Boundary Check:** If the query coordinates $(x, y, z)$ are not within the bounds $[0, XDIM-1] \times [0, YDIM-1] \times [0, ZDIM-1]$, return $(0, 0, 0, 0)$ [S1, unofficial].
2.  **Value Retrieval:**
    *   If `INTER_MODE` is `2` (Nearest Neighbor): Retrieve the scalar value $V$ at the nearest integer coordinate. Retrieve the gradient $\mathbf{G}$ at the same coordinate.
    *   If `INTER_MODE` is `1` (Trilinear): Perform trilinear interpolation using the 8 neighboring voxels to compute $V$ and $\mathbf{G}$ [S1, unofficial]. *Note: The source implementation currently defaults to nearest neighbor, stating "needs to implement voxel intersection" for full trilinear support* [S1, unofficial].
3.  **Transfer Function Input Preparation:**
    *   If `TRN_MODE` is `1`: Set input arguments $a = V, b = V$.
    *   If `TRN_MODE` is `2`: Compute the gradient magnitude $M = \|\mathbf{G}\|$. Clamp $M$ to $[0, 1]$. Scale by 256. Set input arguments $a = V, b = M \cdot 256$.
4.  **Transformation:** Pass $a$ and $b$ to the transfer function transformer to obtain the final RGBA color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel_data(p: Vector3, grid_dims: Tuple[int, int, int], 
                   scalars: List[float], gradients: List[Vector3],
                   inter_mode: int, trn_mode: int) -> Tuple[float, float, float, float]:
    x, y, z = p.x, p.y, p.z
    xd, yd, zd = grid_dims
    
    # 1. Boundary Check
    if not (0 <= x < xd and 0 <= y < yd and 0 <= z < zd):
        return (0.0, 0.0, 0.0, 0.0)

    # 2. Value Retrieval
    # Note: Source indicates trilinear is planned but nearest neighbor is implemented.
    # This pseudocode implements the described logic for both modes.
    
    if inter_mode == 2: # Nearest Neighbor
        ix, iy, iz = round(x), round(y), round(z)
        # Clamp to bounds for nearest neighbor
        ix = max(0, min(ix, xd - 1))
        iy = max(0, min(iy, yd - 1))
        iz = max(0, min(iz, zd - 1))
        
        idx = ix + iy * xd + iz * xd * yd
        scalar_val = scalars[idx]
        grad_vec = gradients[idx]
    else: # Trilinear (Mode 1)
        # Implementation of trilinear interpolation using 8 neighbors
        # is required here as per the comment "needs to implement voxel intersection"
        # [G] Standard trilinear interpolation logic:
        # 1. Find fractional parts fx, fy, fz
        # 2. Interpolate along x, then y, then z
        # 3. Apply same logic to gradients
        scalar_val = trilinear_interpolate(scalars, x, y, z, xd, yd, zd)
        grad_vec = trilinear_interpolate_vector(gradients, x, y, z, xd, yd, zd)

    # 3. Transfer Function Input Preparation
    if trn_mode == 1:
        arg1 = scalar_val
        arg2 = scalar_val
    elif trn_mode == 2:
        arg1 = scalar_val
        grad_mag = length(grad_vec)
        # Clamp to [0, 1]
        grad_mag = max(0.0, min(1.0, grad_mag))
        arg2 = grad_mag * 256.0
    else:
        # Default or error case
        arg1 = scalar_val
        arg2 = scalar_val

    # 4. Transformation
    # [G] The specific transfer function implementation is not provided in the source.
    # This step calls the user-defined transformer.
    color = transform_transfer_function(arg1, arg2)
    
    return color

def trilinear_interpolate(data: List[float], x: float, y: float, z: float, 
                          xd: int, yd: int, zd: int) -> float:
    # [G] Standard trilinear interpolation implementation
    ix = int(floor(x))
    iy = int(floor(y))
    iz = int(floor(z))
    
    fx = x - ix
    fy = y - iy
    fz = z - iz
    
    # Ensure indices are within bounds (clamping assumed for robustness)
    # In strict implementation, boundary checks should handle out-of-bounds before calling
    
    def get_val(ix, iy, iz):
        idx = ix + iy * xd + iz * xd * yd
        return data[idx]
        
    c000 = get_val(ix, iy, iz)
    c100 = get_val(ix + 1, iy, iz)
    c010 = get_val(ix, iy + 1, iz)
    c110 = get_val(ix + 1, iy + 1, iz)
    c001 = get_val(ix, iy, iz + 1)
    c101 = get_val(ix + 1, iy, iz + 1)
    c011 = get_val(ix, iy + 1, iz + 1)
    c111 = get_val(ix + 1, iy + 1, iz + 1)
    
    # Interpolate along x
    c00 = c000 * (1 - fx) + c100 * fx
    c01 = c010 * (1 - fx) + c110 * fx
    c10 = c001 * (1 - fx) + c101 * fx
    c11 = c011 * (1 - fx) + c111 * fx
    
    # Interpolate along y
    c0 = c00 * (1 - fy) + c01 * fy
    c1 = c10 * (1 - fy) + c11 * fy
    
    # Interpolate along z
    return c0 * (1 - fz) + c1 * fz

def trilinear_interpolate_vector(data: List[Vector3], x: float, y: float, z: float, 
                                 xd: int, yd: int, zd: int) -> Vector3:
    # [G] Interpolate each component of the vector independently
    # This is equivalent to interpolating the vector field
    return Vector3(
        trilinear_interpolate_component(data, x, y, z, xd, yd, zd, 0),
        trilinear_interpolate_component(data, x, y, z, xd, yd, zd, 1),
        trilinear_interpolate_component(data, x, y, z, xd, yd, zd, 2)
    )

def trilinear_interpolate_component(data: List[Vector3], x: float, y: float, z: float, 
                                    xd: int, yd: int, zd: int, comp: int) -> float:
    # [G] Helper to extract a single component for scalar interpolation
    # This requires flattening the vector data or accessing components directly
    # Assuming data is stored as interleaved or we access .x, .y, .z
    # For simplicity, we assume a helper that treats the component as a scalar field
    # In the source, gradients are glm::vec3, so we would access .x, .y, .z
    # This function is a placeholder for the specific memory layout handling
    pass 
```

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query for nearest neighbor. $O(1)$ for trilinear (constant number of arithmetic operations).
*   **Memory Complexity:** $O(N)$ where $N$ is the number of voxels, storing both scalar and gradient fields.
*   **Strengths:** Simple to implement; supports gradient-based transfer functions which can enhance edge visibility.
*   **Limitations:** The source implementation notes that full trilinear interpolation with voxel intersection is not yet implemented, defaulting to nearest neighbor which may cause aliasing artifacts [S1, unofficial].
*   **When to use:** When gradient information is needed for transfer functions and performance is critical, or when nearest neighbor is sufficient for the visual quality required.

## Background
This method originates from a student project for CSE-333 Computer Graphics (2020) involving volume ray casting [S1, unofficial]. It was designed to support a 1D transfer function that could optionally use gradient magnitude to control opacity, moving beyond simple scalar-based transfer functions.

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
