# Affine Grid Generator

## Overview
The Affine Grid Generator computes a transformed coordinate grid by applying an affine transformation matrix $\theta$ to a base grid of normalized coordinates [S1, unofficial]. It differs from [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md) by explicitly applying a learnable or provided affine matrix to the coordinates, whereas the sibling generates static normalized grids, and from [CT Slice Voxel Construction](8351-ct-slice-voxel-construction.md) by operating on continuous coordinate spaces rather than discrete voxel indices [S1, unofficial].

## How it works
**Inputs:**
1. `theta`: A tensor of shape $(N, 3, 4)$ representing $N$ affine transformation matrices [S1, unofficial].
2. `grid_size`: A tuple $(N, D, H, W)$ specifying the batch size, depth, height, and width of the output grid [S1, unofficial].

**Outputs:**
1. `grid`: A tensor of shape $(N, D, H, W, 3)$ containing the transformed 3D coordinates [S1, unofficial].

**Data Structures:**
* **Base Grid**: A tensor of shape $(N, D, H, W, 4)$ in homogeneous coordinates [S1, unofficial].
* **Transformed Grid**: A tensor of shape $(N, D, H, W, 3)$ in Cartesian coordinates [S1, unofficial].

**Steps:**
1. **Generate Base Grid**: Create a tensor of shape $(N, D, H, W, 4)$ [S1, unofficial].
    * The first three dimensions $(D, H, W)$ are populated with normalized coordinates ranging from $-1$ to $1$ using `torch.linspace` for width ($W$), height ($H$), and depth ($D$) [S1, unofficial].
    * The fourth dimension (homogeneous coordinate) is filled with $1$s [S1, unofficial].
    * Let $B$ denote this base grid tensor.
2. **Reshape for Batch Matrix Multiplication**: Reshape $B$ from $(N, D, H, W, 4)$ to $(N, D \cdot H \cdot W, 4)$ [S1, unofficial]. Let this flattened tensor be $B_{\text{flat}}$.
3. **Prepare Transformation Matrix**: The input `theta` has shape $(N, 3, 4)$ [S1, unofficial]. To perform the multiplication $B_{\text{flat}} \times \theta^{\top}$, we transpose `theta` using `theta.transpose(1, 2)`, yielding shape $(N, 4, 3)$ [S1, unofficial]. Let $T = \theta^{\top}$.
4. **Apply Transformation**: Compute the batch matrix multiplication $G_{\text{flat}} = B_{\text{flat}} \times T$, producing a tensor of shape $(N, D \cdot H \cdot W, 3)$ [S1, unofficial].
5. **Reshape Output**: Reshape $G_{\text{flat}}$ from $(N, D \cdot H \cdot W, 3)$ back to $(N, D, H, W, 3)$ [S1, unofficial]. This is the final output grid.

**Backward Pass (Gradient Computation):**
1. Let $\frac{\partial L}{\partial G}$ be the gradient of the loss with respect to the output grid, with shape $(N, D, H, W, 3)$ [S1, unofficial].
2. Reshape $\frac{\partial L}{\partial G}$ to $(N, D \cdot H \cdot W, 3)$ as $\frac{\partial L}{\partial G_{\text{flat}}}$ [S1, unofficial].
3. Regenerate $B_{\text{flat}}$ with shape $(N, D \cdot H \cdot W, 4)$ [S1, unofficial].
4. Compute the gradient with respect to the transformation matrix $\theta$ using the chain rule: $\frac{\partial L}{\partial \theta^{\top}} = B_{\text{flat}}^{\top} \times \frac{\partial L}{\partial G_{\text{flat}}}$ [S1, unofficial].
    * $B_{\text{flat}}^{\top}$ has shape $(N, 4, D \cdot H \cdot W)$.
    * $\frac{\partial L}{\partial G_{\text{flat}}}$ has shape $(N, D \cdot H \cdot W, 3)$.
    * The product has shape $(N, 4, 3)$ [S1, unofficial].
5. Transpose the result to match the shape of `theta` $(N, 3, 4)$:
    $$ \frac{\partial L}{\partial \theta} = \left( B_{\text{flat}}^{\top} \times \frac{\partial L}{\partial G_{\text{flat}}} \right)^{\top} $$ [S1, unofficial].

**Parameters:**
* CuDNN must be enabled for processing CUDA inputs [S1, unofficial].

**Edge Cases and Pitfalls:**
* **Precision**: Normalized coordinates in $[-1, 1]$ are standard for neural field training and spatial transformers; ensure `linspace` includes both endpoints [G].
* **Dimensions**: The homogeneous coordinate dimension must be handled correctly: the base grid is 4D (homogeneous), the transformation matrix maps 4D to 3D, and the output is 3D [G].
* **Batch Size**: The batch dimension $N$ is preserved throughout all operations [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def affine_grid_generator(theta, grid_size):
    """
    Generates a transformed coordinate grid using an affine transformation.
    
    Args:
        theta (torch.Tensor): Affine transformation matrices of shape (N, 3, 4).
        grid_size (tuple): Tuple (N, D, H, W) specifying the output grid dimensions.
        
    Returns:
        torch.Tensor: Transformed grid of shape (N, D, H, W, 3).
    """
    N, D, H, W = grid_size
    
    # Step 1: Generate base grid of shape (N, D, H, W, 4)
    x = torch.linspace(-1, 1, W)
    y = torch.linspace(-1, 1, H)
    z = torch.linspace(-1, 1, D)
    
    z_grid, y_grid, x_grid = torch.meshgrid(z, y, x, indexing='ij')
    base_coords = torch.stack((x_grid, y_grid, z_grid), dim=-1)  # (D, H, W, 3)
    ones = torch.ones_like(z_grid).unsqueeze(-1)                 # (D, H, W, 1)
    base_grid_homogeneous = torch.cat((base_coords, ones), dim=-1)  # (D, H, W, 4)
    base_grid = base_grid_homogeneous.unsqueeze(0).expand(N, -1, -1, -1, -1)  # (N, D, H, W, 4)
    
    # Step 2: Reshape to (N, D*H*W, 4)
    base_grid_flat = base_grid.view(N, D * H * W, 4)
    
    # Step 3: Prepare transformation matrix
    theta_T = theta.transpose(1, 2)  # (N, 4, 3)
    
    # Step 4: Batch Matrix Multiplication
    transformed_grid_flat = torch.bmm(base_grid_flat, theta_T)  # (N, D*H*W, 3)
    
    # Step 5: Reshape to (N, D, H, W, 3)
    transformed_grid = transformed_grid_flat.view(N, D, H, W, 3)
    
    return transformed_grid

def affine_grid_generator_backward(grad_output, grid_size):
    """
    Computes gradients for the Affine Grid Generator.
    
    Args:
        grad_output (torch.Tensor): Gradient of loss w.r.t. output grid, shape (N, D, H, W, 3).
        grid_size (tuple): Tuple (N, D, H, W).
        
    Returns:
        torch.Tensor: Gradient w.r.t. theta, shape (N, 3, 4).
    """
    N, D, H, W = grid_size
    
    # Reshape grad_output to (N, D*H*W, 3)
    grad_output_flat = grad_output.view(N, D * H * W, 3)
    
    # Regenerate base_grid_flat (N, D*H*W, 4)
    x = torch.linspace(-1, 1, W)
    y = torch.linspace(-1, 1, H)
    z = torch.linspace(-1, 1, D)
    z_grid, y_grid, x_grid = torch.meshgrid(z, y, x, indexing='ij')
    base_coords = torch.stack((x_grid, y_grid, z_grid), dim=-1)
    ones = torch.ones_like(z_grid).unsqueeze(-1)
    base_grid_homogeneous = torch.cat((base_coords, ones), dim=-1)
    base_grid = base_grid_homogeneous.unsqueeze(0).expand(N, -1, -1, -1, -1)
    base_grid_flat = base_grid.view(N, D * H * W, 4)
    
    # Compute gradient w.r.t theta^T: (N, 4, D*H*W) @ (N, D*H*W, 3) -> (N, 4, 3)
    grad_theta_T = torch.bmm(base_grid_flat.transpose(1, 2), grad_output_flat)
    
    # Transpose to get gradient w.r.t theta: (N, 3, 4)
    grad_theta = grad_theta_T.transpose(1, 2)
    
    return grad_theta
```

## Operating characteristics
* **Time Complexity**: $O(N \cdot D \cdot H \cdot W)$ for grid generation and transformation, dominated by the batch matrix multiplication [G].
* **Memory Complexity**: $O(N \cdot D \cdot H \cdot W)$ to store the base grid and transformed grid [G].
* **Performance**: Efficient on GPU due to batch matrix multiplication; requires CuDNN for CUDA inputs [S1, unofficial].
* **Strengths**: Fully differentiable, enabling end-to-end training of spatial transformations [G].
* **Limitations**: Only supports affine transformations; non-affine deformations require alternative methods [G].

## Background
The method is described in the `voxel2mesh` repository [S1, unofficial]. It is used to generate transformed coordinate grids for voxel-based mesh reconstruction tasks [S1, unofficial].

## Sources
[S1] cvlab-epfl/voxel2mesh, https://github.com/cvlab-epfl/voxel2mesh, fetched 2026-10-27 (unofficial)
