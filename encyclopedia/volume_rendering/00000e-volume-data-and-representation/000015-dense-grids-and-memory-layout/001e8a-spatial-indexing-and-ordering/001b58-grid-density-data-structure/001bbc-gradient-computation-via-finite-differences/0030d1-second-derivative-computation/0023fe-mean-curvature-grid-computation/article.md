# Mean Curvature Grid Computation

Also called: Mean Curvature via BoxStencil, Divergence of Normalized Gradient

## Overview
Computes the mean curvature $\kappa$ of a scalar field $f$ at every active voxel by evaluating the divergence of the normalized gradient field, $\kappa = \nabla \cdot (\nabla f / |\nabla f|)$ [S1, unofficial]. It sets itself apart from sibling methods like [Laplacian Grid Computation](3160-laplacian-grid-computation.md) by normalizing the gradient vector before differentiation, thereby measuring the average of the principal curvatures rather than the sum of second derivatives [S1, unofficial]. This method is implemented as a single pass using a `BoxStencil`, avoiding the creation of intermediate gradient or Hessian grids [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A discrete grid tree representing a scalar field $f$.
- Output: A new grid containing the mean curvature $\kappa$ at each active voxel, with a background value of zero.

**Data Structures:**
- `BoxStencil`: A temporary structure that holds references to the grid tree and provides access to voxel values at relative offsets. It supports `value_at(s, dx, dy, dz)` to retrieve $f(x+dx, y+dy, z+dz)$ and `center_value(s)` to retrieve $f(x, y, z)$.
- `eps(T)`: The machine epsilon for the floating-point type $T$ (typically `Float32` or `Float64`) used to prevent division by zero.

**Parameters:**
- `half`: Scalar multiplier $0.5$ for first derivatives.
- `quarter`: Scalar multiplier $0.25$ for mixed second derivatives.
- `eps(T)`: Small constant to check for vanishing gradients.

**Steps:**
1. Initialize a `BoxStencil` with the input grid tree.
2. Iterate over all active voxels $(c, \dots)$ in the grid tree.
3. Move the stencil to the current coordinate $c$.
4. Retrieve the center value $f_{center} = \text{center\_value}(s)$.
5. Compute first derivatives $f_x, f_y, f_z$ using central differences with multiplier $0.5$:
   $$ f_x = (f(x+1, y, z) - f(x-1, y, z)) \times 0.5 $$
   $$ f_y = (f(x, y+1, z) - f(x, y-1, z)) \times 0.5 $$
   $$ f_z = (f(x, y, z+1) - f(x, y, z-1)) \times 0.5 $$
6. Compute the squared gradient magnitude:
   $$ \text{grad\_sq} = f_x^2 + f_y^2 + f_z^2 $$
7. If $\text{grad\_sq} < \epsilon(T)$, set the output value at $c$ to $0$ and continue to the next voxel [S1, unofficial].
8. Compute pure second derivatives $f_{xx}, f_{yy}, f_{zz}$ using the center value and multiplier $2$ (implicit in the formula $f(x+1) + f(x-1) - 2f(x)$):
   $$ f_{xx} = f(x+1, y, z) + f(x-1, y, z) - 2 f_{center} $$
   $$ f_{yy} = f(x, y+1, z) + f(x, y-1, z) - 2 f_{center} $$
   $$ f_{zz} = f(x, y, z+1) + f(x, y, z-1) - 2 f_{center} $$
9. Compute mixed second derivatives $f_{xy}, f_{xz}, f_{yz}$ using the quarter multiplier $0.25$:
   $$ f_{xy} = (f(x+1, y+1, z) - f(x-1, y+1, z) - f(x+1, y-1, z) + f(x-1, y-1, z)) \times 0.25 $$
   $$ f_{xz} = (f(x+1, y, z+1) - f(x-1, y, z+1) - f(x+1, y, z-1) + f(x-1, y, z-1)) \times 0.25 $$
   $$ f_{yz} = (f(x, y+1, z+1) - f(x, y-1, z+1) - f(x, y+1, z-1) + f(x, y-1, z-1)) \times 0.25 $$
10. Compute the numerator of the curvature formula:
    $$ \text{numer} = f_x^2(f_{yy} + f_{zz}) + f_y^2(f_{xx} + f_{zz}) + f_z^2(f_{xx} + f_{yy}) - 2(f_x f_y f_{xy} + f_x f_z f_{xz} + f_y f_z f_{yz}) $$
11. Compute the final curvature:
    $$ \kappa = \frac{\text{numer}}{\text{grad\_sq} \times \sqrt{\text{grad\_sq}}} $$
12. Store $\kappa$ in the output grid at $c$.
13. Build the result grid using the computed data, a background value of zero, and the original grid's voxel size [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mean_curvature(grid_tree):
    """
    Computes mean curvature kappa = div(grad(f) / |grad(f)|) for all active voxels.
    Uses a single BoxStencil pass.
    """
    # Initialize output grid with background zero
    output_grid = GridTree(background=0.0, voxel_size=grid_tree.voxel_size)
    
    # Initialize stencil
    stencil = BoxStencil(grid_tree)
    
    # Get machine epsilon for the data type
    eps = get_epsilon(grid_tree.dtype)
    
    # Iterate over active voxels
    for coord in grid_tree.active_voxels():
        stencil.move_to(coord)
        
        # Retrieve center value
        f_center = stencil.center_value()
        
        # Compute first derivatives (central difference, multiplier 0.5)
        fx = (stencil.value_at(1, 0, 0) - stencil.value_at(-1, 0, 0)) * 0.5
        fy = (stencil.value_at(0, 1, 0) - stencil.value_at(0, -1, 0)) * 0.5
        fz = (stencil.value_at(0, 0, 1) - stencil.value_at(0, 0, -1)) * 0.5
        
        # Squared gradient magnitude
        grad_sq = fx*fx + fy*fy + fz*fz
        
        # Handle vanishing gradient to avoid division by zero
        if grad_sq < eps:
            output_grid.set_value(coord, 0.0)
            continue
            
        # Compute pure second derivatives
        fxx = stencil.value_at(1, 0, 0) + stencil.value_at(-1, 0, 0) - 2.0 * f_center
        fyy = stencil.value_at(0, 1, 0) + stencil.value_at(0, -1, 0) - 2.0 * f_center
        fzz = stencil.value_at(0, 0, 1) + stencil.value_at(0, 0, -1) - 2.0 * f_center
        
        # Compute mixed second derivatives (multiplier 0.25)
        fxy = (stencil.value_at(1, 1, 0) - stencil.value_at(-1, 1, 0) - 
               stencil.value_at(1, -1, 0) + stencil.value_at(-1, -1, 0)) * 0.25
        fxz = (stencil.value_at(1, 0, 1) - stencil.value_at(-1, 0, 1) - 
               stencil.value_at(1, 0, -1) + stencil.value_at(-1, 0, -1)) * 0.25
        fyz = (stencil.value_at(0, 1, 1) - stencil.value_at(0, -1, 1) - 
               stencil.value_at(0, 1, -1) + stencil.value_at(0, -1, -1)) * 0.25
        
        # Compute numerator of curvature formula
        numer = (fx*fx * (fyy + fzz) + 
                 fy*fy * (fxx + fzz) + 
                 fz*fz * (fxx + fyy) - 
                 2.0 * (fx*fy*fxy + fx*fz*fxz + fy*fz*fyz))
        
        # Compute final curvature: numer / |grad|^3
        # |grad|^3 = grad_sq * sqrt(grad_sq)
        curvature = numer / (grad_sq * sqrt(grad_sq))
        
        output_grid.set_value(coord, curvature)
        
    return output_grid
```

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of active voxels. Each voxel requires a fixed number of memory accesses (27 reads for the full stencil) and arithmetic operations.
- **Memory Complexity:** $O(N)$ for the output grid. The method avoids intermediate grids, requiring only the stencil state per voxel [S1, unofficial].
- **Strengths:** Single-pass computation reduces memory bandwidth pressure compared to multi-step approaches. Avoids explicit normalization vector storage.
- **Limitations:** Sensitive to noise in the input field due to the division by $|\nabla f|$ and the use of second derivatives. Requires careful boundary handling if the grid is not zero-padded or if active voxels are near the edge [S1, unofficial].
- **When to use:** When the mean curvature (average of principal curvatures) is specifically required, such as in surface smoothing or feature detection based on curvature magnitude.
- **When not to use:** If only the Laplacian (sum of principal curvatures) is needed, use [Laplacian Grid Computation](3160-laplacian-grid-computation.md). If the gradient is expected to be zero in significant regions, the division by zero guard is necessary, but the result will be zero, which may need post-processing.

## Background
The method is implemented in the Lyr.jl library, a Julia-based volume rendering and processing toolkit [S1, unofficial]. It leverages the `BoxStencil` abstraction to efficiently access neighboring voxels without manual index calculations. The formula for mean curvature as the divergence of the normalized gradient is a standard result in differential geometry [S1, unofficial].

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
