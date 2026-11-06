# Depth and Image Gradient Regularization

Also called: Curvature Regularization with Gradient Masking, Depth Regularization with Image Gradient Weighting

## Overview
This method computes a regularization loss for a predicted depth map by weighting second-order spatial derivatives (curvature) with the local image gradient magnitude. It solves the problem of enforcing geometric smoothness in depth estimation while preserving sharp boundaries, ensuring that depth discontinuities align with visible edges in the corresponding image. It differs from standard Laplacian regularization [Laplacian Grid Computation](3160-laplacian-grid-computation.md) by introducing an adaptive weighting mechanism based on image content, and from mean curvature methods [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md) by operating on 2D scalar depth fields rather than 3D implicit surfaces.

## How it works
**Inputs and Outputs**
- **Input**: A 2D depth map $D$ and a corresponding 2D image $\mathbf{I}$ (typically RGB or grayscale), both defined on a discrete grid of pixel coordinates $\mathbf{u} = (x, y)$.
- **Output**: A scalar regularization loss value $\mathcal{L}_{\text{reg. depth}}$ for each pixel, or the total loss summed over the domain.

**Data Structures**
- $D$: 2D array of floating-point values representing depth.
- $\mathbf{I}$: 2D array of floating-point values representing image intensity (or per-channel magnitude if RGB).
- Grid dimensions: $W \times H$.

**Steps**
1. **Compute Horizontal Depth Curvature**: Calculate the second-order derivative of depth in the x-direction using central differences.
   $$ D_x^{\Delta}[\mathbf{u}] = D[\mathbf{u}_{x-1}, \mathbf{u}_y] - 2D[\mathbf{u}_x, \mathbf{u}_y] + D[\mathbf{u}_{x+1}, \mathbf{u}_y] $$
2. **Compute Vertical Depth Curvature**: Calculate the second-order derivative of depth in the y-direction using central differences.
   $$ D_y^{\Delta}[\mathbf{u}] = D[\mathbf{u}_x, \mathbf{u}_{y-1}] - 2D[\mathbf{u}_x, \mathbf{u}_y] + D[\mathbf{u}_x, \mathbf{u}_{y+1}] $$
3. **Compute Horizontal Image Gradient Magnitude**: Compute the L-infinity norm of the horizontal image gradient.
   $$ \mathbf{I}_x^{\Delta}[\mathbf{u}] = \max\left(|\mathbf{I}[\mathbf{u}_{x-1}, \mathbf{u}_y] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|, |\mathbf{I}[\mathbf{u}_{x+1}, \mathbf{u}_y] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|\right) $$
4. **Compute Vertical Image Gradient Magnitude**: Compute the L-infinity norm of the vertical image gradient.
   $$ \mathbf{I}_y^{\Delta}[\mathbf{u}] = \max\left(|\mathbf{I}[\mathbf{u}_x, \mathbf{u}_{y-1}] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|, |\mathbf{I}[\mathbf{u}_x, \mathbf{u}_{y+1}] - \mathbf{I}[\mathbf{u}_x, \mathbf{u}_y]|\right) $$
5. **Compute Weighted Regularization Loss**: Combine the depth curvatures with exponential weights derived from the image gradients.
   $$ \mathcal{L}_{\text{reg. depth}}[\mathbf{u}] = D_x^{\Delta}[\mathbf{u}] \exp\left(8 \cdot \mathbf{I}_x^{\Delta}[\mathbf{u}]\right) + D_y^{\Delta}[\mathbf{u}] \exp\left(8 \cdot \mathbf{I}_y^{\Delta}[\mathbf{u}]\right) $$

**Parameters**
- The exponential scaling factor is fixed at $8$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_gradient_regularization(D: Array2D, I: Array2D) -> float:
    """
    Computes the depth regularization loss weighted by image gradients.
    D: 2D array of depth values.
    I: 2D array of image intensity values.
    Returns: Total scalar loss.
    """
    H, W = D.shape
    total_loss = 0.0
    
    # Iterate over interior pixels to avoid boundary checks for simplicity
    # In practice, boundary handling (clamping or exclusion) is required.
    # Here we assume clamping for neighbor access as per shared principles.
    
    for y in range(1, H - 1):
        for x in range(1, W - 1):
            # 1. Horizontal Depth Curvature
            d_x = D[y, x-1] - 2 * D[y, x] + D[y, x+1]
            
            # 2. Vertical Depth Curvature
            d_y = D[y-1, x] - 2 * D[y, x] + D[y+1, x]
            
            # 3. Horizontal Image Gradient Magnitude (L-infinity norm)
            grad_x = max(abs(I[y, x-1] - I[y, x]), abs(I[y, x+1] - I[y, x]))
            
            # 4. Vertical Image Gradient Magnitude (L-infinity norm)
            grad_y = max(abs(I[y-1, x] - I[y, x]), abs(I[y+1, x] - I[y, x]))
            
            # 5. Weighted Loss
            # Exponential weighting suppresses curvature penalty where image gradients are high
            loss_pixel = d_x * exp(8 * grad_x) + d_y * exp(8 * grad_y)
            total_loss += loss_pixel
            
    return total_loss
```

**Edge Cases and Pitfalls**
- **Boundary Pixels**: The stencil requires neighbors at $x\pm1$ and $y\pm1$. Pixels on the grid boundary must be handled via clamping (repeating boundary values) or exclusion. Exclusion is common in image regularization [S1].
- **Image Intensity Range**: The formula assumes $\mathbf{I}$ values are in a range where the exponential term behaves predictably. If $\mathbf{I}$ is normalized to $[0, 1]$, the max weight is $\exp(8) \approx 2980$. If $\mathbf{I}$ is not normalized, the weights may explode or vanish.
- **Depth Sign**: The formula does not explicitly handle the sign of depth curvature. If depth increases towards the camera vs away, the sign of $D_x^{\Delta}$ may flip. The loss penalizes magnitude of curvature, but the sign determines convexity/concavity.

## Subtopics

### [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md)
Computes mean curvature for 3D implicit surfaces. Choose when working with signed distance fields or level sets rather than 2D depth maps.

### [Divergence Grid Computation](3151-divergence-grid-computation.md)
Computes divergence of vector fields. Choose for fluid dynamics or flow regularization, not scalar depth smoothing.

### [Laplacian Grid Computation](3160-laplacian-grid-computation.md)
Computes standard Laplacian (sum of second derivatives) without gradient weighting. Choose when uniform smoothing is desired regardless of image edges.

## Operating characteristics
- **Time Complexity**: $O(W \cdot H)$, where $W$ and $H$ are the width and height of the depth map. Each pixel requires a constant number of arithmetic operations.
- **Memory Complexity**: $O(1)$ auxiliary space if computed in-place or streaming, or $O(W \cdot H)$ if storing intermediate gradient maps.
- **Strengths**: Preserves sharp depth edges corresponding to object boundaries in the image. Prevents over-smoothing in textured regions.
- **Limitations**: Sensitive to noise in the image gradient if $\mathbf{I}$ is noisy. The exponential weighting can amplify noise in high-gradient regions if not pre-filtered.
- **When to use**: Depth estimation from single or multi-view images where edge-aware smoothing is critical.
- **When not to use**: When the image has no texture or uniform regions, as the weighting provides no benefit over standard Laplacian regularization.

## Background
This method was introduced in the context of 3D Gaussian Splatting for scalable generalizable 3D reconstruction. It was proposed by the authors of PixelSplat to regularize predicted depth maps by leveraging the correlation between depth discontinuities and image edges. The approach ensures that the predicted 3D Gaussians align with visible structures in the input images.

## Sources
[S1] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
