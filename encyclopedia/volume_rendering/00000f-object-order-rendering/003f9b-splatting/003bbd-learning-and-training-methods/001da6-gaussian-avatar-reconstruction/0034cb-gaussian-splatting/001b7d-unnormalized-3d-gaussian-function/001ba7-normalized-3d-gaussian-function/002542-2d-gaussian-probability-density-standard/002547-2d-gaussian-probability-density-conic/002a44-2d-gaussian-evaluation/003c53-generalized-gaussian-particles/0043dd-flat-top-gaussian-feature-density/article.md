# Flat-top Gaussian Feature Density

Also called: Flat-top Gaussian, Generalized Gaussian with high degree, Super-gaussian feature density

## Overview
Flat-top Gaussian Feature Density provides a differentiable approximation of feature density for adaptive grid-based scene representations, replacing non-differentiable box functions with a smooth kernel [S1]. It sets itself apart from standard Gaussian splatting by using a high-degree exponent parameter $p$ to create a box-like density profile, which better matches the discrete nature of feature grids while maintaining gradient flow for training [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $M$ grid cells (particles), each defined by a transformation matrix $G_i$ and a flat-top strength parameter $p$.
*   **Output:** The scalar feature density $\rho(x)$ at an arbitrary 3D coordinate $x$.

**Data Structures**
*   **Grid Cell $i$:** Contains a $3 \times 3$ transformation matrix $G_{i,0:3,0:3}$ representing the local coordinate transformation (scale, rotation, shear). The determinant $\det(G_{i,0:3,0:3})$ accounts for volume scaling.
*   **Parameter $p$:** A scalar float controlling the flatness of the Gaussian.

**Formulas**
The core building block is the 1D flat-top Gaussian function [S1]:
$$g(x, p) = A \exp \left( - \left( \frac{(x - \mu)^{2p}}{2\sigma^2} \right) \right)$$
where:
*   $A$: Normalizing coefficient.
*   $x$: Location.
*   $\mu$: Center of the distribution.
*   $\sigma$: Standard deviation.
*   $p$: Strength of the flat-top.

The total feature density $\rho(x)$ at a point $x$ is the sum of contributions from all $M$ grid cells [S1]:
$$\rho(x) = \sum_{i=0}^{M-1} \left( \det(G_{i,0:3,0:3}) \cdot \exp \left( - \left( \sum_{d=0}^2 (G_i(x)_d)^{2p} \right) \right) \right)$$
where $G_i(x)_d$ is the transformed coordinate for axis $d \in \{0, 1, 2\}$ (x, y, z). In this transformed local space, the center is 0 and the standard deviation is 1, so $\mu$ and $\sigma$ are omitted [S1].

**Parameters**
*   **$p$ (Flat-top strength):** Experimentally set to $p = 10$ [S1]. This value is strong enough to approximate the box shape of the feature grid but avoids exploding gradients on the ramps leading to and from the flat-top [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_flat_top_feature_density(x, grid_cells, p=10.0):
    """
    Computes the feature density at point x using flat-top Gaussians.
    
    Args:
        x: 3D vector (x, y, z) coordinate.
        grid_cells: List of grid cell objects. Each object has:
            - transform_matrix: 3x3 matrix G_i.
            - det_G: Precomputed determinant of G_i.
        p: Flat-top strength parameter.
        
    Returns:
        float: Total feature density rho(x).
    """
    total_density = 0.0
    
    for cell in grid_cells:
        # Transform point x into the local coordinate space of the cell
        # G_i(x) implies applying the inverse transformation or mapping x to local coords.
        # Based on the formula, G_i(x)_d are the coordinates in the local space.
        # Assuming G_i represents the transformation from local to world,
        # we need the inverse to get local coords, or G_i(x) denotes the result of the transform.
        # The source states: "Gi(x)d is the transformed coordinate... center... is 0".
        # We assume G_i is the matrix that maps local to world, so we solve for local.
        # However, standard splatting often pre-computes the inverse or uses the matrix directly if defined as local-to-world inverse.
        # Given the formula structure exp(-(sum (G_i(x)_d)^2p)), G_i(x)_d must be the local coordinates.
        
        # Let's assume G_i is the transformation matrix such that local_coords = G_i * x (or similar linear mapping).
        # To be precise and self-contained, we define the transformation step clearly.
        # If G_i is the 3x3 matrix provided in the grid cell data:
        local_coords = apply_transformation(cell.transform_matrix, x)
        
        # Calculate the sum of powers for the 3 dimensions
        sum_powers = 0.0
        for d in range(3):
            coord_d = local_coords[d]
            sum_powers += coord_d ** (2 * p)
            
        # Calculate the exponential term
        # Note: The formula has a negative sign inside the exp
        density_contribution = cell.det_G * math.exp(-sum_powers)
        
        total_density += density_contribution
        
    return total_density

def apply_transformation(matrix, point):
    """
    Applies a 3x3 linear transformation to a 3D point.
    Assumes the matrix maps the point to the local coordinate system.
    """
    # Standard matrix-vector multiplication
    x, y, z = point
    m = matrix
    
    # If the matrix is intended to transform world to local, use it directly.
    # If it transforms local to world, we would need the inverse.
    # The source text says "Gi(x)d is the transformed coordinate...".
    # In many adaptive grid contexts, the matrix defines the cell's orientation/scale.
    # We assume the provided matrix allows direct computation of local coordinates.
    # If the matrix is Local->World, we must invert it.
    # To avoid ambiguity, we assume the input matrix is the one that yields G_i(x)_d.
    
    # Simple linear transform: y = M * x
    lx = m[0][0]*x + m[0][1]*y + m[0][2]*z
    ly = m[1][0]*x + m[1][1]*y + m[1][2]*z
    lz = m[2][0]*x + m[2][1]*y + m[2][2]*z
    
    return [lx, ly, lz]
```

**Edge Cases and Pitfalls**
*   **Gradient Explosion:** If $p$ is too large, the gradients on the "ramps" of the flat-top function can explode, destabilizing training [S1].
*   **Non-Differentiability:** Standard box functions are non-differentiable at boundaries. The flat-top Gaussian ensures differentiability everywhere, which is critical for learning transformation matrices [S1].
*   **Coordinate System:** The formula assumes the transformed coordinates are centered at 0 with unit standard deviation. Incorrect transformation matrices will break this assumption [S1].

## Subtopics

### [Target Feature Density Calculation](11134-target-feature-density-calculation.md)
Defines the dynamic target distribution $\rho^*(x)$ used to steer the current feature density $\rho(x)$ during training. It allocates computational resources by increasing density in high-error regions and decreasing it in low-error regions, using logarithmic scaling based on relative error.

## Operating characteristics
*   **Time Complexity:** $O(M)$ per query point, where $M$ is the number of grid cells. Each cell requires a matrix-vector multiplication and a power/exponential operation.
*   **Memory Complexity:** $O(M)$ to store the grid cells and their transformation matrices.
*   **Strengths:** Differentiable everywhere, allowing gradient-based optimization of grid transformations and scales. The box-like shape ($p=10$) closely approximates the discrete feature grid structure [S1].
*   **Limitations:** Requires careful tuning of $p$ to balance approximation quality and gradient stability. Performance degrades linearly with the number of grid cells.
*   **When to use:** When training adaptive grid-based scene representations where differentiable density estimation is required to update grid parameters based on reconstruction error [S1].
*   **When not to use:** For static rendering where differentiability is not needed, or when real-time performance is critical and $M$ is very large (consider simpler splatting methods).

## Background
The method originates from the work on Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization [S1]. It was developed to solve the problem of learning transformation matrices in adaptive grids, which is impossible with non-differentiable box functions. The flat-top Gaussian (or super-Gaussian) is a known class of functions in signal processing and optics, characterized by a flat peak and steep sides, controlled by the exponent $p$ [S1].

## Sources
[S1] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
