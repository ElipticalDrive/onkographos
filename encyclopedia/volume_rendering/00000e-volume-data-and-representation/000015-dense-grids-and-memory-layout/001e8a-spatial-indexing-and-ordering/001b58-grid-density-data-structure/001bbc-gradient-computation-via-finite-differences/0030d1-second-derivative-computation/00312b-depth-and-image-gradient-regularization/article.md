# Depth and Image Gradient Regularization

Also called: Curvature Regularization with Gradient Masking, Depth Regularization with Image Gradient Weighting

## Overview
Depth and Image Gradient Regularization is a loss term used in differentiable rendering and 3D reconstruction to enforce geometric smoothness in predicted depth maps while preserving sharp features. It solves the problem of over-smoothing depth boundaries by weighting the second-order depth derivatives (curvature) with the local image gradient magnitude. Unlike standard Laplacian smoothing which penalizes curvature uniformly, this method suppresses regularization in regions of high image intensity change (edges), allowing the depth map to remain discontinuous or steep where the visual content indicates an object boundary [S1].

## How it works
The method operates on a 2D grid representing a depth map $D$ and a corresponding RGB image $\mathbf{I}$. It computes second-order finite differences for depth and first-order magnitude differences for the image, then combines them into a weighted loss.

**Inputs:**
- $D$: 2D grid of depth values, shape $(H, W)$.
- $\mathbf{I}$: 2D grid of image intensities (or per-channel RGB), shape $(H, W, C)$.
- $\mathbf{u}$: Pixel coordinate $(x, y)$.

**Outputs:**
- $\mathcal{L}_{\text{reg. depth}}$: Scalar loss value.

**Data Structures:**
- The depth grid $D$ and image grid $\mathbf{I}$ are stored as contiguous arrays.
- Boundary handling: The sources do not specify explicit boundary padding for the finite difference operations. Standard central difference assumptions apply, requiring neighbors to exist. In implementation, this typically implies zero-padding or clamping at boundaries, but the exact behavior is "Not given in the sources" [S1].

**Steps:**

1.  **Compute Horizontal Depth Curvature ($D_x^{\Delta}$):**
    Apply a central difference stencil to the depth map along the x-axis.
    $$ D_x^{\Delta}[\mathbf{u}] = D[\mathbf{u}_{x-1}, \mathbf{u}_y] - 2D[\mathbf{u}_x, \mathbf{u}_y] + D[\mathbf{u}_{x+1}, \mathbf{u}_y] $$
    where $\mathbf{u}_{x-1}$ denotes the pixel at $(x-1, y)$ [S1].

2.  **Compute Vertical Depth Curvature ($D_y^{\Delta}$):**
    Apply a central difference stencil to the depth map along the y-axis.
    $$ D_y^{\Delta}[\mathbf{u}] = D[\mathbf{u}_x, \mathbf{u}_{y-1}] - 2D[\mathbf{u}_x, \mathbf{u}_y] + D[\mathbf{u}_x, \mathbf{u}_{y+1}] $$
    where $\mathbf{u}_{y-1}$ denotes the pixel at $(x, y-1)$ [S1].

3.  **Compute Horizontal Image Gradient Magnitude ($\mathbf{I}_x^{\Delta}$):**
    Compute the absolute difference between the center pixel and its horizontal neighbors. Use the L-infinity norm (maximum) to determine the magnitude.
    $$ \mathbf{I}_x^{\Delta}[\mathbf{u}] = \max\left(|\mathbf{I}[\mathbf{u}_{x-1}, \mathbf{u}_y] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|, |\mathbf{I}[\mathbf{u}_{x+1}, \mathbf{u}_y] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|\right) $$
    If $\mathbf{I}$ is multi-channel (RGB), the magnitude is typically computed per channel or on a grayscale conversion; the source notation $\mathbf{I}$ implies the intensity value at the pixel [S1].

4.  **Compute Vertical Image Gradient Magnitude ($\mathbf{I}_y^{\Delta}$):**
    Compute the absolute difference between the center pixel and its vertical neighbors.
    $$ \mathbf{I}_y^{\Delta}[\mathbf{u}] = \max\left(|\mathbf{I}[\mathbf{u}_x, \mathbf{u}_{y-1}] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|, |\mathbf{I}[\mathbf{u}_x, \mathbf{u}_{y+1}] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|\right) $$
    [S1].

5.  **Compute Regularization Loss:**
    Combine the depth curvatures with exponential weights derived from the image gradients. The weight factor is 8.
    $$ \mathcal{L}_{\text{reg. depth}}[\mathbf{u}] = D_x^{\Delta}[\mathbf{u}] \exp\left(8 \cdot \mathbf{I}_x^{\Delta}[\mathbf{u}]\right) + D_y^{\Delta}[\mathbf{u}] \exp\left(8 \cdot \mathbf{I}_y^{\Delta}[\mathbf{u}]\right) $$
    The total loss is the sum (or mean) of this term over all pixels $\mathbf{u}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def depth_image_gradient_regularization(depth: np.ndarray, image: np.ndarray) -> float:
    """
    Computes the Depth and Image Gradient Regularization loss.
    
    Args:
        depth: 2D numpy array of shape (H, W) representing the depth map D.
        image: 2D or 3D numpy array of shape (H, W) or (H, W, C) representing intensity I.
               If 3D, it is assumed to be pre-processed to a single intensity channel 
               or the operation is applied per-channel and summed. 
               For strict adherence to [S1], we assume scalar intensity I.
    
    Returns:
        The scalar regularization loss value.
    """
    # Ensure image is 2D (intensity)
    if image.ndim == 3:
        # Convert RGB to grayscale using standard luminance or mean if not specified.
        # Source [S1] uses I as a scalar field. We assume grayscale input or conversion.
        image = np.mean(image, axis=2)

    H, W = depth.shape
    
    # Initialize loss
    loss = 0.0
    
    # Iterate over valid interior pixels to avoid boundary checks not defined in source
    # Source [S1] implies central differences, so boundaries x=0, x=W-1, y=0, y=H-1 
    # are problematic. We assume zero-padding or that the grid is padded.
    # Here we compute for interior pixels only to match the stencil requirements.
    
    # Horizontal Depth Derivative D_x^Delta
    # D[x-1, y] - 2*D[x, y] + D[x+1, y]
    # Slice: depth[0:-2, :] is x-1, depth[1:-1, :] is x, depth[2:, :] is x+1
    if H > 2 and W > 2:
        d_x = depth[0:-2, 1:-1] - 2 * depth[1:-1, 1:-1] + depth[2:, 1:-1]
        d_y = depth[1:-1, 0:-2] - 2 * depth[1:-1, 1:-1] + depth[1:-1, 2:]
        
        # Horizontal Image Gradient I_x^Delta
        # max(|I[x-1, y] - I[x, y]|, |I[x+1, y] - I[x, y]|)
        i_x_left = np.abs(image[0:-2, 1:-1] - image[1:-1, 1:-1])
        i_x_right = np.abs(image[2:, 1:-1] - image[1:-1, 1:-1])
        i_x = np.maximum(i_x_left, i_x_right)
        
        # Vertical Image Gradient I_y^Delta
        # max(|I[x, y-1] - I[x, y]|, |I[x, y+1] - I[x, y]|)
        i_y_up = np.abs(image[1:-1, 0:-2] - image[1:-1, 1:-1])
        i_y_down = np.abs(image[1:-1, 2:] - image[1:-1, 1:-1])
        i_y = np.maximum(i_y_up, i_y_down)
        
        # Weights
        # exp(8 * I)
        w_x = np.exp(8.0 * i_x)
        w_y = np.exp(8.0 * i_y)
        
        # Loss term
        # D_x^Delta * exp(8 * I_x^Delta) + D_y^Delta * exp(8 * I_y^Delta)
        loss_term = d_x * w_x + d_y * w_y
        
        # Sum over all pixels
        loss = np.sum(loss_term)
    else:
        # Grid too small for central differences
        loss = 0.0
        
    return loss
```

**Parameters:**
- Weight multiplier in exponential: $8$ [S1].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The central difference stencils require neighbors outside the image for boundary pixels. The source does not specify padding. Implementers must choose (e.g., zero-padding, replication) which affects boundary accuracy [S1].
- **Intensity Scale:** The exponential weight $\exp(8 \cdot \mathbf{I})$ is sensitive to the scale of $\mathbf{I}$. If $\mathbf{I}$ is not normalized (e.g., [0, 1]), the weights may explode or vanish. The source assumes a specific intensity range consistent with the training data [S1].
- **Depth Sign:** The formula uses raw depth values. If depth is signed (e.g., camera space), the curvature interpretation depends on the sign convention. The source assumes $D$ is a standard depth map [S1].

## Subtopics

### Auxiliary Gradient Constraint
A penalty term used in Neural Density-Distance Fields to constrain the shape of an auxiliary gradient using a hyperparameter $\alpha$. It ensures the auxiliary gradient becomes active in a narrower range than the distance field and concentrates near cusps. When $\alpha \le 1$, the range is narrower; as $\alpha$ increases, concentration near cusps increases. Without this constraint, incorrect volume densities occur in empty regions. The auxiliary gradient is strongly activated near the cusp of the distance field where distances from several objects are similar, unlike the smoother distance field [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(H \cdot W)$ for a grid of size $H \times W$, as it involves constant-time finite difference operations per pixel.
- **Memory Complexity:** $O(H \cdot W)$ to store intermediate derivative grids if not computed in-place or streamed.
- **Strengths:** Preserves sharp depth edges corresponding to image boundaries, preventing the "smearing" of depth discontinuities common in pure Laplacian regularization.
- **Limitations:** Sensitive to image noise (which creates high gradients) and intensity scaling. Requires paired depth and image data.
- **When to use:** In differentiable rendering pipelines (e.g., Gaussian Splatting, NeRF variants) where depth consistency is enforced via auxiliary losses, particularly when dealing with scenes containing sharp object boundaries [S1].
- **When not to use:** When only geometric smoothness is desired without regard to visual edges, or when image data is unavailable [S1].

## Background
The method was introduced in the context of 3D Gaussian Splatting from image pairs, specifically in the PixelSplat framework. It leverages the correlation between image gradients and depth discontinuities to regularize the predicted depth map during the optimization process [S1]. The auxiliary gradient variant originates from Neural Density-Distance Fields, which use distance fields to represent volumes and employ auxiliary gradients to handle complex topologies like cusps [S2, unofficial].

## Sources
[S1] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
[S2] Neural Density-Distance Fields, https://doi.org/10.48550/arXiv.2207.14455, fetched 2026-10-27 (unofficial)
