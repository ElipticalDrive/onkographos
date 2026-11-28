# Mean Curvature Grid Computation

Also called: Mean Curvature via BoxStencil, Divergence of Normalized Gradient

## Overview
Computes the mean curvature $\kappa = \nabla \cdot (\nabla f / |\nabla f|)$ at every active voxel using a single BoxStencil pass, avoiding intermediate gradient or Hessian grids [S1, unofficial]. It sets itself apart from sibling methods by computing the divergence of the normalized gradient directly in one pass rather than composing separate gradient and Laplacian grids [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A discrete grid tree representing a scalar field $f$ [S1, unofficial].
- Output: A new grid containing the mean curvature $\kappa$ at each active voxel, with a background value of zero [S1, unofficial].

**Data Structures:**
- `BoxStencil`: A temporary structure that holds a reference to the grid tree and provides access to voxel values at relative offsets via `value_at(s, dx, dy, dz)` and `center_value(s)` [S1, unofficial].
- `eps(T)`: A small threshold for the floating-point type $T$ used to prevent division by zero when the gradient magnitude vanishes [S1, unofficial].

**Parameters:**
- `half`: Scalar multiplier $0.5$ for first derivatives [S1, unofficial].
- `quarter`: Scalar multiplier $0.25$ for mixed second derivatives [S1, unofficial].
- `eps(T)`: Threshold constant for vanishing gradient checks [S1, unofficial].

**Steps:**
1. Initialize a `BoxStencil` with the input grid tree [S1, unofficial].
2. Iterate over all active voxels $(c, \dots)$ in the grid tree [S1, unofficial].
3. Move the stencil to the current coordinate $c$ [S1, unofficial].
4. Retrieve the center value $f_{center} = \text{center\_value}(s)$ [S1, unofficial].
5. Compute first derivatives $f_x, f_y, f_z$ using central differences with multiplier $0.5$:
   $$ f_x = (f(x+1, y, z) - f(x-1, y, z)) \times 0.5 $$
   $$ f_y = (f(x, y+1, z) - f(x, y-1, z)) \times 0.5 $$
   $$ f_z = (f(x, y, z+1) - f(x, y, z-1)) \times 0.5 $$
6. Compute the squared gradient magnitude:
   $$ \text{grad\_sq} = f_x^2 + f_y^2 + f_z^2 $$
7. If $\text{grad\_sq} < \epsilon(T)$, set the output value at $c$ to $0$ and continue to the next voxel [S1, unofficial].
8. Compute pure second derivatives $f_{xx}, f_{yy}, f_{zz}$ using the center value and multiplier $2$ (implicit in $f(x+1) + f(x-1) - 2f(x)$):
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
12. Store $\kappa$ in the output grid at $c$ [S1, unofficial].
13. Build the result grid using the computed data, a background value of zero, and the original grid's voxel size [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class BoxStencil:
    """Holds a reference to the grid tree and provides voxel access at offsets."""
    def __init__(self, grid_tree):
        self.grid = grid_tree
        self.coord = None
    def move_to(self, coord):
        self.coord = coord
    def value_at(self, dx, dy, dz):
        # Standard grid lookup; assumes zero-padding outside active domain [G]
        return self.grid.get_value(self.coord[0] + dx, self.coord[1] + dy, self.coord[2] + dz)
    def center_value(self):
        return self.value_at(0, 0, 0)

def compute_mean_curvature(grid_tree, eps_val=1e-12):
    """
    Computes mean curvature kappa = div(grad(f) / |grad(f)|) for all active voxels.
    Uses a single BoxStencil pass.
    """
    output_data = []
    stencil = BoxStencil(grid_tree)
    
    for coord in grid_tree.active_voxels():
        stencil.move_to(coord)
        f_center = stencil.center_value()
        
        # Compute first derivatives (central difference, multiplier 0.5)
        fx = (stencil.value_at(1, 0, 0) - stencil.value_at(-1, 0, 0)) * 0.5
        fy = (stencil.value_at(0, 1, 0) - stencil.value_at(0, -1, 0)) * 0.5
        fz = (stencil.value_at(0, 0, 1) - stencil.value_at(0, 0, -1)) * 0.5
        
        # Squared gradient magnitude
        grad_sq = fx*fx + fy*fy + fz*fz
        
        # Handle vanishing gradient to avoid division by zero
        if grad_sq < eps_val:
            output_data.append((coord, 0.0))
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
        curvature = numer / (grad_sq * math.sqrt(grad_sq))
        output_data.append((coord, curvature))
        
    # Build the result grid using the computed data, a background value of zero, 
    # and the original grid's voxel size [S1, unofficial]
    return output_data
```

## Operating characteristics
Time complexity is $O(N)$ where $N$ is the number of active voxels, as each voxel requires a fixed stencil evaluation [G]. Memory complexity is $O(N)$ for the output grid, with $O(1)$ auxiliary memory per voxel due to the single-pass BoxStencil design [G]. The method avoids explicit normalization vector storage and intermediate grid allocations [S1, unofficial].

## Background
The method is implemented in the Lyr.jl library [S1, unofficial].

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
