# Depth and Image Gradient Regularization

Also called: Curvature Regularization with Gradient Masking, Depth Regularization with Image Gradient Weighting

## Overview
Depth and Image Gradient Regularization is a regularization loss term that weights second-order depth derivatives by exponential image gradients [S1]. It computes second-order finite differences for a depth map and first-order magnitude differences for a corresponding image, then combines them into a weighted loss [S1]. Builds on [Second Derivative Computation](6353-second-derivative-computation.md).

## How it works
**Inputs:**
- $D$: 2D grid of depth values, shape $(H, W)$ [S1].
- $\mathbf{I}$: 2D grid of image intensities, shape $(H, W)$ [S1].
- $\mathbf{u}$: Pixel coordinate $(x, y)$ [S1].

**Outputs:**
- $\mathcal{L}_{\text{reg. depth}}[\mathbf{u}]$: Per-pixel regularization loss value [S1].

**Data Structures:**
- The depth grid $D$ and image grid $\mathbf{I}$ are stored as 2D arrays [S1].
- Boundary handling: The sources do not specify explicit boundary padding for the finite difference operations [S1].

**Steps:**
1. **Compute Horizontal Depth Curvature ($D_x^{\Delta}$):**
   Apply a central difference stencil to the depth map along the x-axis.
   $$ D_x^{\Delta}[\mathbf{u}] = D[\mathbf{u}_{x-1}, \mathbf{u}_y] - 2D[\mathbf{u}_x, \mathbf{u}_y] + D[\mathbf{u}_{x+1}, \mathbf{u}_y] $$
   where $\mathbf{u}_{x-1}$ denotes the pixel at $(x-1, y)$ [S1].
2. **Compute Vertical Depth Curvature ($D_y^{\Delta}$):**
   Apply a central difference stencil to the depth map along the y-axis.
   $$ D_y^{\Delta}[\mathbf{u}] = D[\mathbf{u}_x, \mathbf{u}_{y-1}] - 2D[\mathbf{u}_x, \mathbf{u}_y] + D[\mathbf{u}_x, \mathbf{u}_{y+1}] $$
   where $\mathbf{u}_{y-1}$ denotes the pixel at $(x, y-1)$ [S1].
3. **Compute Horizontal Image Gradient Magnitude ($\mathbf{I}_x^{\Delta}$):**
   Compute the absolute difference between the center pixel and its horizontal neighbors. Use the L-infinity norm (maximum) to determine the magnitude.
   $$ \mathbf{I}_x^{\Delta}[\mathbf{u}] = \max\left(|\mathbf{I}[\mathbf{u}_{x-1}, \mathbf{u}_y] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|, |\mathbf{I}[\mathbf{u}_{x+1}, \mathbf{u}_y] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|\right) $$
   [S1].
4. **Compute Vertical Image Gradient Magnitude ($\mathbf{I}_y^{\Delta}$):**
   Compute the absolute difference between the center pixel and its vertical neighbors.
   $$ \mathbf{I}_y^{\Delta}[\mathbf{u}] = \max\left(|\mathbf{I}[\mathbf{u}_x, \mathbf{u}_{y-1}] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|, |\mathbf{I}[\mathbf{u}_x, \mathbf{u}_{y+1}] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|\right) $$
   [S1].
5. **Compute Regularization Loss:**
   Combine the depth curvatures with exponential weights derived from the image gradients. The weight factor is 8.
   $$ \mathcal{L}_{\text{reg. depth}}[\mathbf{u}] = D_x^{\Delta}[\mathbf{u}] \exp\left(8 \cdot \mathbf{I}_x^{\Delta}[\mathbf{u}]\right) + D_y^{\Delta}[\mathbf{u}] \exp\left(8 \cdot \mathbf{I}_y^{\Delta}[\mathbf{u}]\right) $$
   [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def depth_image_gradient_regularization(depth: np.ndarray, image: np.ndarray) -> np.ndarray:
    """
    Computes the Depth and Image Gradient Regularization loss per pixel.
    
    Args:
        depth: 2D numpy array of shape (H, W) representing the depth map D.
        image: 2D numpy array of shape (H, W) representing intensity I.
    
    Returns:
        2D numpy array of shape (H-2, W-2) containing the per-pixel loss values.
        Aggregation over the grid is not specified in the sources [S1].
    """
    H, W = depth.shape
    if H < 3 or W < 3:
        return np.array([])

    # Horizontal Depth Derivative D_x^Delta: D[:, x-1] - 2*D[:, x] + D[:, x+1]
    d_x = depth[:, :-2] - 2 * depth[:, 1:-1] + depth[:, 2:]
    
    # Vertical Depth Derivative D_y^Delta: D[y-1, :] - 2*D[y, :] + D[y+1, :]
    d_y = depth[:-2, :] - 2 * depth[1:-1, :] + depth[2:, :]
    
    # Horizontal Image Gradient I_x^Delta: max(|I[:, x-1] - I[:, x]|, |I[:, x+1] - I[:, x]|)
    i_x_left = np.abs(image[:, :-2] - image[:, 1:-1])
    i_x_right = np.abs(image[:, 2:] - image[:, 1:-1])
    i_x = np.maximum(i_x_left, i_x_right)
    
    # Vertical Image Gradient I_y^Delta: max(|I[y-1, :] - I[y, :]|, |I[y+1, :] - I[y, :]|)
    i_y_up = np.abs(image[:-2, :] - image[1:-1, :])
    i_y_down = np.abs(image[2:, :] - image[1:-1, :])
    i_y = np.maximum(i_y_up, i_y_down)
    
    # Weights: exp(8 * I)
    w_x = np.exp(8.0 * i_x)
    w_y = np.exp(8.0 * i_y)
    
    # Loss term: D_x^Delta * exp(8 * I_x^Delta) + D_y^Delta * exp(8 * I_y^Delta)
    loss_term = d_x * w_x + d_y * w_y
    
    return loss_term
```

**Parameters:**
- Weight multiplier in exponential: $8$ [S1].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The central difference stencils require neighbors outside the image for boundary pixels [G]. The sources do not specify padding, so implementers must choose an assumption (e.g., zero-padding, replication) [G].
- **Intensity Scale:** The exponential weight $\exp(8 \cdot \mathbf{I})$ is sensitive to the scale of $\mathbf{I}$ [G]. If $\mathbf{I}$ is not normalized (e.g., to $[0, 1]$), the weights may explode or vanish [G].
- **Depth Sign:** The formula uses raw depth values [S1]. If depth is signed (e.g., camera space), the curvature interpretation depends on the sign convention [G].

## Subtopics

### Auxiliary Gradient Constraint
A penalty term used in Neural Density-Distance Fields to constrain the shape of an auxiliary gradient using a hyperparameter $\alpha$ [S2, unofficial]. It ensures the auxiliary gradient becomes active in a narrower range than the distance field and concentrates near cusps [S2, unofficial]. When $\alpha \le 1$, the range is narrower; as $\alpha$ increases, concentration near cusps increases [S2, unofficial]. Without this constraint, incorrect volume densities occur in empty regions [S2, unofficial]. The auxiliary gradient is strongly activated near the cusp of the distance field where distances from several objects are similar, unlike the smoother distance field [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(H \cdot W)$ for a grid of size $H \times W$, as it involves constant-time finite difference operations per pixel [G].
- **Memory Complexity:** $O(H \cdot W)$ to store intermediate derivative grids if not computed in-place or streamed [G].
- **Strengths:** Preserves sharp depth edges corresponding to image boundaries, preventing the "smearing" of depth discontinuities common in pure Laplacian regularization [G].
- **Limitations:** Sensitive to image noise (which creates high gradients) and intensity scaling. Requires paired depth and image data [G].
- **Measured performance, scaling, and specific hardware benchmarks:** Not given in the sources.

## Background
The method was introduced in the context of 3D Gaussian Splatting from image pairs, specifically in the PixelSplat framework [S1]. It leverages the correlation between image gradients and depth discontinuities to regularize the predicted depth map during the optimization process [S1]. The auxiliary gradient variant originates from Neural Density-Distance Fields, which use distance fields to represent volumes and employ auxiliary gradients to handle complex topologies like cusps [S2, unofficial].

## Sources
[S1] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
[S2] Neural Density-Distance Fields, https://doi.org/10.48550/arXiv.2207.14455, fetched 2026-10-27 (unofficial)
