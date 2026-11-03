# VoxelGrid

Also called: 3D Voxel Grid, Differentiable Voxel Grid, Learnable Voxel Grid, Neural Voxel Grid

## Overview
VoxelGrid is a dense, learnable 3D data structure that defines a parameterized function in 3D space by storing learnable values at the middle of voxels but effectively storing corner values for interpolation [S1, unofficial]. It enables differentiable trilinear interpolation of density and color, replacing coordinate-based MLPs with a fixed, axis-aligned tensor layout for efficient neural rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A tensor $P$ of query points with coordinates $(x, y, z)$ [S1, unofficial].
- Output: A tensor of interpolated values (e.g., density, RGB) at those points [S1, unofficial].

**Data Structures:**
- Layout: The grid stores corner values to facilitate trilinear interpolation [S1, unofficial].
- Dimensions: If the logical grid dimensions are `shape` = $(S_x, S_y, S_z)$, the storage tensor has shape $(S_x+1, S_y+1, S_z+1, d)$ [S1, unofficial].
- Fields: The tensor contains floating-point values (typically `float32`) [S1, unofficial]. $d$ is the output dimensionality (e.g., 1 for scalar density, 3 for RGB color) [S1, unofficial].

**Parameters:**
- `shape`: Tuple of integers defining the logical grid dimensions (e.g., $(6, 6, 6)$) [S1, unofficial].
- `d`: Integer defining the output dimensionality (default 1) [S1, unofficial].
- `max`: Float defining the maximum value for random initialization (default 1.0) [S1, unofficial].
- `fill`: Optional tensor or value used for initialization instead of random values [S1, unofficial].

**Algorithm Steps:**
1. **Initialization**: Create the `grid` parameter with shape $(S_x+1, S_y+1, S_z+1, d)$ [S1, unofficial]. Initialize with random values scaled by `max` or provided `fill` [S1, unofficial].
2. **Coordinate Extraction**: Extract $x, y, z$ coordinate vectors from the input points $P$ [S1, unofficial].
3. **Bracketing**: For each coordinate axis, determine the lower and upper grid indices and the interpolation weight [S1, unofficial].
   - Compute $x_0 = \lfloor x \rfloor$.
   - Set $X_0 = \text{clamp}(x_0, 0, S_x)$ and $X_1 = \text{clamp}(x_0 + 1, 0, S_x)$ [S1, unofficial].
   - Compute weight $\alpha = \text{clamp}(x - x_0, 0.0, 1.0)$ [S1, unofficial].
   - Repeat for $y$ (indices $Y_0, Y_1$, weight $\beta$) and $z$ (indices $Z_0, Z_1$, weight $\gamma$) [S1, unofficial].
4. **Trilinear Interpolation**:
   - **X-direction**: Interpolate along the x-axis for all 4 combinations of $(Y, Z)$ corners:
     $$ v_{y0z0} = v(X_0, Y_0, Z_0) \cdot (1-\alpha) + v(X_1, Y_0, Z_0) \cdot \alpha $$
     $$ v_{y1z0} = v(X_0, Y_1, Z_0) \cdot (1-\alpha) + v(X_1, Y_1, Z_0) \cdot \alpha $$
     $$ v_{y0z1} = v(X_0, Y_0, Z_1) \cdot (1-\alpha) + v(X_1, Y_0, Z_1) \cdot \alpha $$
     $$ v_{y1z1} = v(X_0, Y_1, Z_1) \cdot (1-\alpha) + v(X_1, Y_1, Z_1) \cdot \alpha $$
   - **Y-direction**: Interpolate the results from the previous step along the y-axis:
     $$ v_{z0} = v_{y0z0} \cdot (1-\beta) + v_{y1z0} \cdot \beta $$
     $$ v_{z1} = v_{y0z1} \cdot (1-\beta) + v_{y1z1} \cdot \beta $$
   - **Z-direction**: Interpolate the results along the z-axis:
     $$ v = v_{z0} \cdot (1-\gamma) + v_{z1} \cdot \gamma $$
5. **Output**: Return the final interpolated values, removing the trailing dimension if necessary [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def bracket(x, n):
    """Return the indices of the nearest grid points to x, as well as weights."""
    x0 = torch.floor(x).long()
    X0 = torch.clamp(x0, min=0, max=(n - 1))
    X1 = torch.clamp(x0 + 1, min=0, max=(n - 1))
    return X0, X1, torch.clamp(x - x0.float(), min=0.0, max=1.0)

def interpolate(v0, v1, alpha):
    """Interpolate between v0 and v1 using alpha, using unsqueeze to properly handle batches."""
    return v0 * (1 - alpha.unsqueeze(-1)) + v1 * alpha.unsqueeze(-1)

class VoxelGrid(nn.Module):
    def __init__(self, shape, d=1, max=1.0, fill=None):
        """A 3D voxel grid with given `shape` with learnable values at the middle of the voxels."""
        super(VoxelGrid, self).__init__()
        # Note that we store *corner* values, so we need one more point in each dimension:
        storage_shape = tuple(s + 1 for s in shape)
        if fill is not None:
            self.grid = nn.Parameter(fill.view(1, 1, 1, d).repeat(*storage_shape, 1))
        else:
           self.grid = nn.Parameter(torch.rand(*storage_shape, d, dtype=torch.float32) * max)

    def forward(self, P):
        """Implement trilinear interpolation at the points P."""
        x, y, z = P[..., 0], P[..., 1], P[..., 2]

        # Get indices of the corners, clamping to the grid size where needed:
        X0, X1, a = bracket(x, self.grid.shape[0])
        Y0, Y1, b = bracket(y, self.grid.shape[1])
        Z0, Z1, c = bracket(z, self.grid.shape[2])

        # Interpolate in the x direction:
        y0z0 = interpolate(self.grid[X0, Y0, Z0, :], self.grid[X1, Y0, Z0, :], a)
        y1z0 = interpolate(self.grid[X0, Y1, Z0, :], self.grid[X1, Y1, Z0, :], a)
        y0z1 = interpolate(self.grid[X0, Y0, Z1, :], self.grid[X1, Y0, Z1, :], a)
        y1z1 = interpolate(self.grid[X0, Y1, Z1, :], self.grid[X1, Y1, Z1, :], a)

        # Interpolate in the y direction:
        z0 = interpolate(y0z0, y1z0, b)
        z1 = interpolate(y0z1, y1z1, b)

        # Interpolate in the z direction:
        return interpolate(z0, z1, c).squeeze(-1)
```

**Edge Cases and Pitfalls:**
- **Boundary Handling**: Query points outside the grid bounds must be clamped to the nearest valid index $[0, n-1]$ to prevent out-of-bounds memory access [S1, unofficial].
- **Precision**: Floating-point precision in coordinate extraction ($\lfloor x \rfloor$) and weight calculation ($x - \lfloor x \rfloor$) can introduce small errors, but clamping weights to $[0, 1]$ mitigates instability [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: DVGO uses this approach as a much faster alternative to fully connected MLPs for representing density and color [S1, unofficial].
- **Strengths**: Differentiable, enables efficient training and rendering via tensor indexing and interpolation [S1, unofficial].
- **Limitations**: Not given in the sources.

## Background
VoxelGrid was introduced in Differentiable Volumetric Graphics (DVGO) as a faster alternative to the original Neural Radiance Field (NeRF), which used a fully connected multi-layer perceptron (MLP) [S1, unofficial]. By storing density and color in a voxel grid and using differentiable trilinear interpolation, DVGO enabled efficient training and rendering of neural radiance fields [S1, unofficial].

## Sources
[S1] 7.6. Neural Radiance Fields for Drones, https://www.roboticsbook.org/S76_drone_learning.html, fetched 2026-10-29 (unofficial)
