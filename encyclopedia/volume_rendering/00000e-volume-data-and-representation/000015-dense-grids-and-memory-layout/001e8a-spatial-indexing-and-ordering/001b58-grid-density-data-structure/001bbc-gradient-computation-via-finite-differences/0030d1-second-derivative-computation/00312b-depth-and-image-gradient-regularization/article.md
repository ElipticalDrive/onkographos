# Depth and Image Gradient Regularization

Also called: Curvature Regularization with Gradient Masking, Depth Regularization with Image Gradient Weighting

## Overview
This method applies a regularization loss to a predicted depth map by penalizing second-order spatial derivatives (curvature) weighted by the local image gradient magnitude. It solves the problem of noisy or artifact-prone depth predictions in view synthesis by enforcing smoothness in textureless regions while preserving sharp discontinuities at object boundaries. It differs from standard Laplacian smoothing [Laplacian Grid Computation](3160-laplacian-grid-computation.md) by using the image gradient as a spatially varying mask, ensuring that depth edges align with visual edges rather than being blurred.

## How it works
The method operates on a 2D depth map $D$ and a corresponding 2D image $\mathbf{I}$ (typically RGB or grayscale), both defined on a pixel grid with coordinates $\mathbf{u} = (u_x, u_y)$.

**Inputs:**
- $D$: A 2D array of depth values, shape $(H, W)$.
- $\mathbf{I}$: A 2D array of image intensity or color values, shape $(H, W)$.

**Outputs:**
- $\mathcal{L}_{\text{reg. depth}}$: A 2D array of regularization loss values, shape $(H, W)$.

**Data Structures:**
- The depth map $D$ and image $\mathbf{I}$ are stored as dense grids.
- Intermediate gradient maps $\mathbf{I}_x^{\Delta}$ and $\mathbf{I}_y^{\Delta}$ are computed per-pixel.

**Steps:**

1. **Compute Horizontal Depth Curvature:**
   For each pixel $(u_x, u_y)$, compute the second-order finite difference in the horizontal direction:
   $$ D_x^{\Delta}[\mathbf{u}] = D[\mathbf{u}_{x-1}, \mathbf{u}_y] - 2D[\mathbf{u}_x, \mathbf{u}_y] + D[\mathbf{u}_{x+1}, \mathbf{u}_y] $$
   where $\mathbf{u}_{x-1} = (u_x-1, u_y)$ and $\mathbf{u}_{x+1} = (u_x+1, u_y)$.

2. **Compute Vertical Depth Curvature:**
   For each pixel $(u_x, u_y)$, compute the second-order finite difference in the vertical direction:
   $$ D_y^{\Delta}[\mathbf{u}] = D[\mathbf{u}_x, \mathbf{u}_{y-1}] - 2D[\mathbf{u}_x, \mathbf{u}_y] + D[\mathbf{u}_x, \mathbf{u}_{y+1}] $$
   where $\mathbf{u}_{y-1} = (u_x, u_y-1)$ and $\mathbf{u}_{y+1} = (u_x, u_y+1)$.

3. **Compute Horizontal Image Gradient Magnitude:**
   Calculate the L-infinity norm of the horizontal intensity difference:
   $$ \mathbf{I}_x^{\Delta}[\mathbf{u}] = \max\left(|\mathbf{I}[\mathbf{u}_{x-1}, \mathbf{u}_y] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|, |\mathbf{I}[\mathbf{u}_{x+1}, \mathbf{u}_y] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|\right) $$

4. **Compute Vertical Image Gradient Magnitude:**
   Calculate the L-infinity norm of the vertical intensity difference:
   $$ \mathbf{I}_y^{\Delta}[\mathbf{u}] = \max\left(|\mathbf{I}[\mathbf{u}_x, \mathbf{u}_{y-1}] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|, |\mathbf{I}[\mathbf{u}_x, \mathbf{u}_{y+1}] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|\right) $$

5. **Compute Weighted Regularization Loss:**
   Combine the depth curvatures with exponential weights derived from the image gradients:
   $$ \mathcal{L}_{\text{reg. depth}}[\mathbf{u}] = D_x^{\Delta}[\mathbf{u}] \exp\left(8 \cdot \mathbf{I}_x^{\Delta}[\mathbf{u}]\right) + D_y^{\Delta}[\mathbf{u}] \exp\left(8 \cdot \mathbf{I}_y^{\Delta}[\mathbf{u}]\right) $$

**Parameters:**
- The exponential scaling factor is fixed at $8$ [S1].

**Edge Cases and Pitfalls:**
- **Boundary Pixels:** The stencils require neighbors outside the image domain. Standard boundary handling (e.g., zero-padding, clamping, or ignoring boundary pixels during loss computation) must be applied. The source does not specify the exact boundary strategy, but implies indexing $\mathbf{u}_{x\pm1}$ and $\mathbf{u}_{y\pm1}$ [S1].
- **Gradient Magnitude:** The use of the L-infinity norm ($\max$) rather than L2 norm simplifies computation and avoids square roots, but may be less sensitive to diagonal gradients.
- **Exponential Overflow:** If image gradients are large, $\exp(8 \cdot \mathbf{I}^{\Delta})$ can overflow. In practice, image intensities are often normalized, or the gradient magnitude is clamped.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def depth_gradient_regularization(D: np.ndarray, I: np.ndarray) -> np.ndarray:
    """
    Compute depth regularization loss weighted by image gradients.
    
    Args:
        D: Depth map of shape (H, W).
        I: Image intensity map of shape (H, W).
        
    Returns:
        Loss map of shape (H, W).
    """
    H, W = D.shape
    
    # Initialize output
    loss = np.zeros_like(D)
    
    # Compute depth curvatures
    # D_x^Delta: horizontal second derivative
    # D_y^Delta: vertical second derivative
    # Using slicing for interior pixels to avoid boundary issues in this example
    # Note: Boundary handling is not specified in source [S1], assuming interior only or padding
    
    # Horizontal depth curvature
    D_x_delta = D[1:-1, :-2] - 2 * D[1:-1, 1:-1] + D[1:-1, 2:]
    
    # Vertical depth curvature
    D_y_delta = D[:-2, 1:-1] - 2 * D[1:-1, 1:-1] + D[2:, 1:-1]
    
    # Compute image gradient magnitudes (L-infinity norm)
    # Horizontal gradient magnitude
    I_x_diff_left = np.abs(I[1:-1, :-2] - I[1:-1, 1:-1])
    I_x_diff_right = np.abs(I[1:-1, 2:] - I[1:-1, 1:-1])
    I_x_delta = np.maximum(I_x_diff_left, I_x_diff_right)
    
    # Vertical gradient magnitude
    I_y_diff_up = np.abs(I[:-2, 1:-1] - I[1:-1, 1:-1])
    I_y_diff_down = np.abs(I[2:, 1:-1] - I[1:-1, 1:-1])
    I_y_delta = np.maximum(I_y_diff_up, I_y_diff_down)
    
    # Compute weighted loss
    # Weight factor is 8
    weight_x = np.exp(8 * I_x_delta)
    weight_y = np.exp(8 * I_y_delta)
    
    loss_interior = D_x_delta * weight_x + D_y_delta * weight_y
    
    # Assign to output (boundaries remain 0)
    loss[1:-1, 1:-1] = loss_interior
    
    return loss
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(HW)$, where $H$ and $W$ are the height and width of the image. The operations are element-wise and involve constant-size stencils.
- **Memory Complexity:** $O(HW)$ for storing the input maps and intermediate gradient maps.
- **Strengths:** Effectively preserves depth edges where image gradients are high, preventing the "smearing" of depth discontinuities common in unweighted Laplacian regularization.
- **Limitations:** Relies on the quality of the input image $\mathbf{I}$. If the image is noisy, the gradient mask may be unreliable. The fixed exponential factor (8) may need tuning for different datasets or intensity scales.
- **When to use:** When optimizing depth maps from image pairs or sparse views where visual consistency is critical.
- **When not to use:** When the image is heavily corrupted or when depth smoothness is required regardless of visual content (e.g., in textureless synthetic scenes).

## Background
This method was introduced in the context of 3D Gaussian Splatting for scalable generalizable 3D reconstruction. It is used in the PixelSplat model to regularize the predicted depth maps derived from Gaussian parameters. The approach leverages the correlation between image edges and depth discontinuities to guide the smoothness prior [S1].

## Sources
[S1] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
