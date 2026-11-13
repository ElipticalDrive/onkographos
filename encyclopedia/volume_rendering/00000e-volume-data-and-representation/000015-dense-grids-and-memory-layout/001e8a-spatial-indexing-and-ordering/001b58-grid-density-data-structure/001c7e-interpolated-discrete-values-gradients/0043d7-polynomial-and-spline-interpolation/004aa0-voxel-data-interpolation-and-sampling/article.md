# Voxel Data Interpolation and Sampling

Also called: getNextIntersectColor, getNextIntersectionGradient, getNextIntersectionScalar

## Overview
This method retrieves scalar values, gradients, and transformed colors at arbitrary 3D positions within a volume grid, supporting configurable interpolation and transfer function input modes [S1, unofficial]. It differs from sibling methods by explicitly exposing the mechanism for selecting transfer function inputs (scalar-only vs. scalar-plus-gradient) and managing the boundary checks for voxel indices, whereas siblings like [B-spline Field Interpolation](11066-b-spline-field-interpolation.md) focus on higher-order continuity or [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md) on index generation.

## How it works
**Inputs and Outputs:**
*   **Input:** Query point $\mathbf{p} = (x, y, z)$ in grid coordinates; Volume data structure containing scalar and gradient fields; Parameters `INTER_MODE` and `TRN_MODE`.
*   **Output:** A 4-component color/alpha vector (RGBA) or scalar/gradient pair, depending on the calling context.

**Data Structures:**
The volume data is stored in two contiguous vectors [S1, unofficial]:
1.  `scalars`: `std::vector<float>` storing scalar values.
2.  `gradients`: `std::vector<glm::vec3>` storing 3D gradient vectors.

The index for a voxel at integer coordinates $(x, y, z)$ in a grid with dimensions $X_{DIM}, Y_{DIM}, Z_{DIM}$ is calculated as [S1, unofficial]:
$$
\text{index} = x + (y \cdot X_{DIM}) + (z \cdot X_{DIM} \cdot Y_{DIM})
$$

**Parameters:**
*   `INTER_MODE`: Controls interpolation. Value `1` enables trilinear interpolation; Value `2` enables nearest neighbor sampling [S1, unofficial].
*   `TRN_MODE`: Controls transfer function input. Value `1` passes (Voxel Value, Voxel Value); Value `2` passes (Voxel Value, Gradient Magnitude) [S1, unofficial].

**Steps:**
1.  **Boundary Check:** Determine if the query coordinates $(x, y, z)$ are within the grid bounds $[0, X_{DIM}-1] \times [0, Y_{DIM}-1] \times [0, Z_{DIM}-1]$. If any coordinate is out of bounds, return $(0, 0, 0, 0)$ [S1, unofficial].
2.  **Value Retrieval:**
    *   If `INTER_MODE` is `2` (Nearest Neighbor): Retrieve the scalar $v$ and gradient $\mathbf{g}$ at the nearest integer voxel index.
    *   If `INTER_MODE` is `1` (Trilinear): Compute weights for the 8 surrounding voxels based on the fractional part of $(x, y, z)$ and blend the scalar values $v_i$ and gradient vectors $\mathbf{g}_i$ from the 8 neighbors [S1, unofficial]. *Note: The provided source implementation currently defaults to nearest neighbor, noting that trilinear logic requires voxel intersection implementation* [S1, unofficial].
3.  **Transfer Function Input Preparation:**
    *   If `TRN_MODE` is `1`: Pass the retrieved scalar value $v$ twice to the transfer function transformer: $(v, v)$ [S1, unofficial].
    *   If `TRN_MODE` is `2`: Compute the gradient magnitude $m = \|\mathbf{g}\|$. Clamp $m$ to the range $[0, 1]$. Scale the clamped magnitude by 256. Pass the scalar $v$ and the scaled magnitude $m \cdot 256$ to the transformer: $(v, m \cdot 256)$ [S1, unofficial].
4.  **Transformation:** Apply the 1D transfer function (or transformer) to the prepared inputs to generate the final RGBA color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel_data(p: Vector3, scalars: List[float], gradients: List[Vector3], 
                   x_dim: int, y_dim: int, z_dim: int, 
                   inter_mode: int, trn_mode: int) -> Vector4:
    """
    Retrieves voxel data and prepares it for transfer function evaluation.
    
    Args:
        p: Query point (x, y, z). Note: In the source, integer coordinates are checked directly.
           For trilinear, fractional coordinates are used for weighting.
        scalars: Flat list of scalar values.
        gradients: Flat list of 3D gradient vectors.
        x_dim, y_dim, z_dim: Grid dimensions.
        inter_mode: 1 for trilinear, 2 for nearest neighbor.
        trn_mode: 1 for (V, V), 2 for (V, |grad|*256).
        
    Returns:
        Vector4 (RGBA) result from transfer function.
    """
    x, y, z = p.x, p.y, p.z
    
    # 1. Boundary Check
    # Source checks integer coordinates for bounds. 
    # For trilinear, we assume p is already in grid space.
    # If out of bounds, return black/transparent.
    if not (0 <= x < x_dim and 0 <= y < y_dim and 0 <= z < z_dim):
        return Vector4(0.0, 0.0, 0.0, 0.0)

    # 2. Value Retrieval
    if inter_mode == 2:
        # Nearest Neighbor
        ix, iy, iz = round(x), round(y), round(z)
        # Clamp indices to valid range for nearest neighbor if p is near edge
        ix = max(0, min(x_dim - 1, ix))
        iy = max(0, min(y_dim - 1, iy))
        iz = max(0, min(z_dim - 1, iz))
        
        idx = ix + (iy * x_dim) + (iz * x_dim * y_dim)
        scalar_val = scalars[idx]
        grad_val = gradients[idx]
    else:
        # Trilinear Interpolation (Logic implied by source comment, not fully implemented in snippet)
        # Calculate fractional parts
        fx = x - floor(x)
        fy = y - floor(y)
        fz = z - floor(z)
        
        # Integer coordinates of the 8 corners
        x0, x1 = floor(x), ceil(x)
        y0, y1 = floor(y), ceil(y)
        z0, z1 = floor(z), ceil(z)
        
        # Handle boundary conditions for corners (clamp or wrap)
        # Assuming clamp for this implementation
        x0 = max(0, min(x_dim - 1, x0))
        x1 = max(0, min(x_dim - 1, x1))
        y0 = max(0, min(y_dim - 1, y0))
        y1 = max(0, min(y_dim - 1, y1))
        z0 = max(0, min(z_dim - 1, z0))
        z1 = max(0, min(z_dim - 1, z1))
        
        # Retrieve 8 scalars and gradients
        # Helper to get index
        def get_idx(ix, iy, iz):
            return ix + (iy * x_dim) + (iz * x_dim * y_dim)
            
        s000 = scalars[get_idx(x0, y0, z0)]
        s100 = scalars[get_idx(x1, y0, z0)]
        s010 = scalars[get_idx(x0, y1, z0)]
        s110 = scalars[get_idx(x1, y1, z0)]
        s001 = scalars[get_idx(x0, y0, z1)]
        s101 = scalars[get_idx(x1, y0, z1)]
        s011 = scalars[get_idx(x0, y1, z1)]
        s111 = scalars[get_idx(x1, y1, z1)]
        
        g000 = gradients[get_idx(x0, y0, z0)]
        g100 = gradients[get_idx(x1, y0, z0)]
        g010 = gradients[get_idx(x0, y1, z0)]
        g110 = gradients[get_idx(x1, y1, z0)]
        g001 = gradients[get_idx(x0, y0, z1)]
        g101 = gradients[get_idx(x1, y0, z1)]
        g011 = gradients[get_idx(x0, y1, z1)]
        g111 = gradients[get_idx(x1, y1, z1)]
        
        # Trilinear blend for scalar
        scalar_val = (s000 * (1-fx) * (1-fy) * (1-fz) + 
                      s100 * fx * (1-fy) * (1-fz) + 
                      s010 * (1-fx) * fy * (1-fz) + 
                      s110 * fx * fy * (1-fz) + 
                      s001 * (1-fx) * (1-fy) * fz + 
                      s101 * fx * (1-fy) * fz + 
                      s011 * (1-fx) * fy * fz + 
                      s111 * fx * fy * fz)
                      
        # Trilinear blend for gradient
        grad_val = (g000 * (1-fx) * (1-fy) * (1-fz) + 
                    g100 * fx * (1-fy) * (1-fz) + 
                    g010 * (1-fx) * fy * (1-fz) + 
                    g110 * fx * fy * (1-fz) + 
                    g001 * (1-fx) * (1-fy) * fz + 
                    g101 * fx * (1-fy) * fz + 
                    g011 * (1-fx) * fy * fz + 
                    g111 * fx * fy * fz)

    # 3. Transfer Function Input Preparation
    if trn_mode == 1:
        input1 = scalar_val
        input2 = scalar_val
    elif trn_mode == 2:
        input1 = scalar_val
        grad_mag = length(grad_val)
        # Clamp to [0, 1]
        grad_mag = max(0.0, min(1.0, grad_mag))
        input2 = grad_mag * 256.0
    else:
        # Default or error case, fallback to mode 1
        input1 = scalar_val
        input2 = scalar_val

    # 4. Transformation
    # Apply 1D transfer function transformer
    # Note: The source mentions a "transformer" function. 
    # We assume a generic function `apply_transfer_function` exists.
    color = apply_transfer_function(input1, input2)
    
    return color
```

## Operating characteristics
*   **Time Complexity:** $O(1)$ for nearest neighbor. $O(1)$ for trilinear (constant number of arithmetic operations and memory accesses, specifically 8 reads and blends).
*   **Memory Complexity:** $O(N)$ where $N$ is the number of voxels, for storing the scalar and gradient vectors.
*   **Strengths:** Simple to implement; supports gradient-based transfer functions which are common in volume visualization for edge enhancement.
*   **Limitations:** The source implementation notes that trilinear interpolation is not fully implemented in the specific codebase cited, relying on nearest neighbor [S1, unofficial]. Performance is limited by memory bandwidth when accessing scattered gradient vectors if not cached efficiently.
*   **When to use:** When using 1D transfer functions that benefit from gradient magnitude information, or when simple interpolation suffices.
*   **When not to use:** When higher-order continuity (e.g., $C^2$) is required, or when the volume is extremely large and cache misses dominate performance.

## Background
This method originates from a student project for a Computer Graphics course (CSE-333) in 2020, implemented as part of a volume ray casting engine [S1, unofficial]. It was designed to support X-ray rendering techniques and 1D transfer functions, with plans to extend to gradient-based opacity. The codebase is hosted on GitHub as an unofficial repository [S1, unofficial].

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
