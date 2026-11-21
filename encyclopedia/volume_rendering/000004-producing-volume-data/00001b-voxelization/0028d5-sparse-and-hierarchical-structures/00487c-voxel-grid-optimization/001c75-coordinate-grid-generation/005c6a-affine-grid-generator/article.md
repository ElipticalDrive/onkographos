# Affine Grid Generator


## Overview
The Affine Grid Generator produces a transformed coordinate grid by applying an affine transformation to a base grid of normalized coordinates. It solves the problem of spatially warping input data (such as images or voxel grids) for tasks like spatial transformer networks or deformable voxelization. It differs from its sibling [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md) by explicitly applying a learnable or provided affine matrix $\theta$ to the coordinates, whereas the sibling generates static normalized grids. It differs from [CT Slice Voxel Construction](8351-ct-slice-voxel-construction.md) by operating on continuous coordinate spaces rather than discrete voxel indices derived from projection geometry.

## How it works
**Inputs:**
1.  `theta`: A tensor of shape $(N, 3, 4)$ representing $N$ affine transformation matrices. Each matrix maps homogeneous 3D coordinates to 3D coordinates.
2.  `grid_size`: A tuple $(N, D, H, W)$ specifying the batch size, depth, height, and width of the output grid.

**Outputs:**
1.  `grid`: A tensor of shape $(N, D, H, W, 3)$ containing the transformed 3D coordinates.

**Data Structures:**
*   **Base Grid**: A tensor of shape $(N, D, H, W, 4)$ in homogeneous coordinates.
*   **Transformed Grid**: A tensor of shape $(N, D, H, W, 3)$ in Cartesian coordinates.

**Steps:**
1.  **Generate Base Grid**: Create a tensor of shape $(N, D, H, W, 4)$.
    *   The first three dimensions $(D, H, W)$ are populated with normalized coordinates ranging from $-1$ to $1$. Specifically, for each axis, use `torch.linspace(-1, 1, dim)` where `dim` is $D$, $H$, or $W$.
    *   The fourth dimension (homogeneous coordinate) is filled with $1$s.
    *   Let $B$ be this base grid tensor.
2.  **Reshape for Batch Matrix Multiplication**: Reshape $B$ from $(N, D, H, W, 4)$ to $(N, D \cdot H \cdot W, 4)$. Let this be $B_{flat}$.
3.  **Prepare Transformation Matrix**: The input `theta` has shape $(N, 3, 4)$. To perform the multiplication $B_{flat} \times \theta^T$, we need the transpose of `theta`. However, standard batch matrix multiplication `bmm` expects inputs $(N, K, M)$ and $(N, M, P)$. Here, $B_{flat}$ is $(N, P, 4)$ where $P = D \cdot H \cdot W$. The transformation matrix must be $(N, 4, 3)$ to yield an output of $(N, P, 3)$.
    *   The source notes indicate `theta.transpose(1, 2)` is used. If `theta` is $(N, 3, 4)$, `transpose(1, 2)` results in $(N, 4, 3)$. Let $T = \theta^T$.
4.  **Apply Transformation**: Compute the batch matrix multiplication:
    $$ G_{flat} = B_{flat} \times T $$
    Where $G_{flat}$ has shape $(N, D \cdot H \cdot W, 3)$.
5.  **Reshape Output**: Reshape $G_{flat}$ from $(N, D \cdot H \cdot W, 3)$ back to $(N, D, H, W, 3)$. This is the final output grid.

**Backward Pass (Gradient Computation):**
1.  Let $\frac{\partial L}{\partial G}$ be the gradient of the loss with respect to the output grid, with shape $(N, D, H, W, 3)$.
2.  Reshape $\frac{\partial L}{\partial G}$ to $(N, D \cdot H \cdot W, 3)$. Let this be $\frac{\partial L}{\partial G_{flat}}$.
3.  Reshape the base grid $B$ to $(N, D \cdot H \cdot W, 4)$ as $B_{flat}$.
4.  Compute the gradient with respect to the transformation matrix $\theta$. The gradient $\frac{\partial L}{\partial \theta}$ is derived from the chain rule. Since $G_{flat} = B_{flat} \times \theta^T$, the gradient with respect to $\theta^T$ is $B_{flat}^T \times \frac{\partial L}{\partial G_{flat}}$.
    *   $B_{flat}^T$ has shape $(N, 4, D \cdot H \cdot W)$.
    *   $\frac{\partial L}{\partial G_{flat}}$ has shape $(N, D \cdot H \cdot W, 3)$.
    *   The product $B_{flat}^T \times \frac{\partial L}{\partial G_{flat}}$ has shape $(N, 4, 3)$.
5.  Transpose the result to match the shape of `theta` $(N, 3, 4)$:
    $$ \frac{\partial L}{\partial \theta} = \left( B_{flat}^T \times \frac{\partial L}{\partial G_{flat}} \right)^T $$

**Parameters:**
*   CuDNN must be enabled for processing CUDA inputs [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision**: Normalized coordinates in $[-1, 1]$ are standard for neural field training and spatial transformers. Ensure `linspace` includes both endpoints.
*   **Dimensions**: The homogeneous coordinate dimension must be handled correctly. The base grid is 4D (homogeneous), the transformation matrix maps 4D to 3D, and the output is 3D.
*   **Batch Size**: The batch dimension $N$ is preserved throughout.

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
    # Create normalized coordinates for each axis
    x = torch.linspace(-1, 1, W)
    y = torch.linspace(-1, 1, H)
    z = torch.linspace(-1, 1, D)
    
    # Create meshgrid
    # Note: meshgrid returns (W, H, D) or similar depending on indexing. 
    # We need to construct the grid carefully to match (N, D, H, W, 4)
    # Using torch.meshgrid with indexing='ij' to match (D, H, W) order
    z_grid, y_grid, x_grid = torch.meshgrid(z, y, x, indexing='ij')
    
    # Stack to form (D, H, W, 3)
    base_coords = torch.stack((x_grid, y_grid, z_grid), dim=-1) # (D, H, W, 3)
    
    # Add homogeneous coordinate 1
    ones = torch.ones_like(z_grid).unsqueeze(-1) # (D, H, W, 1)
    base_grid_homogeneous = torch.cat((base_coords, ones), dim=-1) # (D, H, W, 4)
    
    # Expand to batch dimension (N, D, H, W, 4)
    base_grid = base_grid_homogeneous.unsqueeze(0).expand(N, -1, -1, -1, -1)
    
    # Step 2: Reshape to (N, D*H*W, 4)
    base_grid_flat = base_grid.view(N, D * H * W, 4)
    
    # Step 3: Prepare transformation matrix
    # theta is (N, 3, 4). We need (N, 4, 3) for multiplication with base_grid_flat (N, P, 4)
    # theta.transpose(1, 2) gives (N, 4, 3)
    theta_T = theta.transpose(1, 2)
    
    # Step 4: Batch Matrix Multiplication
    # (N, P, 4) @ (N, 4, 3) -> (N, P, 3)
    transformed_grid_flat = torch.bmm(base_grid_flat, theta_T)
    
    # Step 5: Reshape to (N, D, H, W, 3)
    transformed_grid = transformed_grid_flat.view(N, D, H, W, 3)
    
    return transformed_grid

def affine_grid_generator_backward(grad_output, theta, grid_size):
    """
    Computes gradients for the Affine Grid Generator.
    
    Args:
        grad_output (torch.Tensor): Gradient of loss w.r.t. output grid, shape (N, D, H, W, 3).
        theta (torch.Tensor): Original theta tensor, shape (N, 3, 4).
        grid_size (tuple): Tuple (N, D, H, W).
        
    Returns:
        torch.Tensor: Gradient w.r.t. theta, shape (N, 3, 4).
    """
    N, D, H, W = grid_size
    
    # Reshape grad_output to (N, D*H*W, 3)
    grad_output_flat = grad_output.view(N, D * H * W, 3)
    
    # Reconstruct base_grid_flat (N, D*H*W, 4)
    # This requires regenerating the base grid as in the forward pass
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
    # base_grid_flat.transpose(1, 2) is (N, 4, D*H*W)
    grad_theta_T = torch.bmm(base_grid_flat.transpose(1, 2), grad_output_flat)
    
    # Transpose to get gradient w.r.t theta: (N, 3, 4)
    grad_theta = grad_theta_T.transpose(1, 2)
    
    return grad_theta
```

## Subtopics

### 3D Coordinate Grid Generation
Generates static normalized 3D grids without transformation. Use when no spatial deformation is needed. [Link](2333-3d-coordinate-grid-generation.md)

### CT Slice Voxel Construction
Constructs voxel grids from CT projections using ray casting. Use for medical imaging voxelization. [Link](8351-ct-slice-voxel-construction.md)

## Operating characteristics
*   **Time Complexity**: $O(N \cdot D \cdot H \cdot W)$ for grid generation and transformation. The dominant operation is the batch matrix multiplication.
*   **Memory Complexity**: $O(N \cdot D \cdot H \cdot W)$ to store the base grid and transformed grid.
*   **Performance**: Efficient on GPU due to batch matrix multiplication. Requires CuDNN for CUDA inputs [S1, unofficial].
*   **Strengths**: Differentiable, allowing end-to-end training of spatial transformations.
*   **Limitations**: Only supports affine transformations. Non-affine deformations require other methods.

## Background
The Affine Grid Generator is implemented as a PyTorch Autograd Function in the `voxel2mesh` repository [S1, unofficial]. It is used to generate transformed coordinate grids for voxel-based mesh reconstruction tasks.

## Sources
[S1] cvlab-epfl/voxel2mesh, https://github.com/cvlab-epfl/voxel2mesh, fetched 2026-10-27 (unofficial)
