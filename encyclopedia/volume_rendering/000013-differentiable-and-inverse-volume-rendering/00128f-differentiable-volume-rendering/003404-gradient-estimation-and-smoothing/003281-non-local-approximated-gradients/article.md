# Non-local Approximated Gradients

Also called: Approximate Gradient, Approximated Spatial Gradients, Loper and Black, OpenDR (Approximated Spatial Gradients), OpenDR Gradients

## Overview
Non-local approximated gradients are a technique for differentiable rendering that approximates the gradient of the rendering integrand with respect to geometric parameters by incorporating gradients backpropagated from a loss function [S1, unofficial]. Unlike standard differentiable rendering methods that rely on local geometric derivatives, this approach aims to provide non-zero, directed gradients that guide optimization away from local minima by leveraging the specific optimization objective (e.g., image reconstruction) [S1, unofficial].

## How it works
The method retains a standard rasterizer in the forward pass and computes approximated gradients in the backward pass [S2]. It addresses the limitations of the original OpenDR approach, which suffered from "localness" (only boundary pixels provided gradients) and a lack of "objective awareness" (gradients did not consider the loss function) [S1, unofficial].

### Inputs and Outputs
- **Inputs**: Vertex positions $V$, rendered image $I$, loss function $L$, and its gradient with respect to image pixels $\frac{\partial L}{\partial I}$ [S1, unofficial].
- **Outputs**: Approximated gradients $\frac{\partial L}{\partial V}$ with respect to vertex positions [S1, unofficial].

### Data Structures
Standard rasterization outputs (pixel colors, coverage) and a loss gradient map [S1, unofficial].

### Steps
1. **Forward Pass**: Render the scene using a standard rasterizer to obtain the image $I$ [S2].
2. **Loss Computation**: Compute the loss $L$ based on the rendered image $I$ and the target [S2].
3. **Backpropagate Loss**: Compute the gradient of the loss with respect to each pixel $v_p$ in the image, denoted as $\frac{\partial L}{\partial v_p}$ [S1, unofficial]. While standard geometric gradients are only non-zero for pixels on the triangle boundary, this method utilizes the loss gradient from all pixels to overcome poor local minima [S1, unofficial].
4. **Differential Filter Application**: For each vertex $v_i$, compute the approximate gradient by applying differential filters (e.g., Sobel filter) to the loss gradients of neighboring pixels [S1, unofficial]. The pixel $v_p$ is represented as $v_p = \frac{\tilde{w}_i v_i}{\tilde{w}_i}$ with $\tilde{w}_i = 1$ [S1, unofficial]. The pixel derivatives with respect to $v_p$ are computed using differential filters:
   $$
   \left\{ \frac{\partial c_v}{\partial x_p}, \frac{\partial c_v}{\partial y_p} \right\} = \frac{\partial c_v}{\partial v_p}
   $$
   where $\frac{\partial c_v}{\partial x_p}$ and $\frac{\partial c_v}{\partial y_p}$ are the spatial gradients of the pixel color/value $c_v$ with respect to pixel coordinates $x_p, y_p$, and $\frac{\partial c_v}{\partial v_p}$ represents the gradient with respect to the vertex position $v_p$ [S1, unofficial]. This formulation ensures the gradient flow is objective-aware, directing vertex displacement to minimize the specific loss rather than following arbitrary geometric derivatives [S1, unofficial].
5. **Aggregation**: Aggregate these gradients to update the vertex positions [S1, unofficial]. The key distinction is that the gradient flow is aware of the specific optimization objective because it uses $\frac{\partial L}{\partial v_p}$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def _convolve2d(image, kernel, mode='same'):
    """Simple 2D convolution for self-containment."""
    kh, kw = kernel.shape
    ph, pw = kh // 2, kw // 2
    padded = np.pad(image, ((ph, ph), (pw, pw)), mode='constant')
    out = np.zeros_like(image)
    for i in range(image.shape[0]):
        for j in range(image.shape[1]):
            out[i, j] = np.sum(padded[i:i+kh, j:j+kw] * kernel)
    return out

def compute_non_local_approximated_gradients(vertices, image, loss_gradient, triangle_pixel_map):
    """
    Compute approximated gradients for vertices using loss-aware differential filters.
    
    Args:
        vertices: ndarray of shape (N, 3) containing vertex positions.
        image: ndarray of shape (H, W, C) containing rendered pixel values.
        loss_gradient: ndarray of shape (H, W, C) containing dL/dI.
        triangle_pixel_map: dict mapping triangle index to list of pixel coordinates (y, x).
        
    Returns:
        vertex_gradients: ndarray of shape (N, 3) containing dL/dV.
    """
    vertex_gradients = np.zeros_like(vertices)
    
    # Sobel kernels for estimating spatial gradients [G]
    sobel_x = np.array([[-1, 0, 1], [-2, 0, 2], [-1, 0, 1]])
    sobel_y = np.array([[-1, -2, -1], [0, 0, 0], [1, 2, 1]])
    
    # Apply differential filters to the loss gradient map [S1, unofficial]
    loss_grad_x = np.zeros_like(loss_gradient)
    loss_grad_y = np.zeros_like(loss_gradient)
    for c in range(loss_gradient.shape[2]):
        loss_grad_x[:, :, c] = _convolve2d(loss_gradient[:, :, c], sobel_x)
        loss_grad_y[:, :, c] = _convolve2d(loss_gradient[:, :, c], sobel_y)
        
    # Aggregate contributions per triangle/vertex [S1, unofficial]
    for tri_idx, pixels in triangle_pixel_map.items():
        grad_accum = np.zeros(vertices.shape[1])
        for y, x in pixels:
            # The gradient w.r.t vertex position is approximated by the 
            # spatial gradient of the pixel value, scaled by the loss gradient [S1, unofficial]
            spatial_grad = np.concatenate([loss_grad_x[y, x, :], loss_grad_y[y, x, :]])
            grad_accum += spatial_grad
        vertex_gradients[tri_idx] = grad_accum
        
    return vertex_gradients
```

### Parameters
- **Differential Filter**: Typically a Sobel filter or similar kernel used to estimate spatial gradients [S1, unofficial].

### Edge Cases and Pitfalls
- **Boundary Pixels**: Gradients are only non-zero for pixels located on the boundary of the triangle; gradients at interior pixels are zero because they do not depend on vertex positions in the same way [S1, unofficial].
- **Local Minima**: Standard gradients at pixels not covered by primitives cannot be used, leading to poor local minima [S1, unofficial].
- **Objective Awareness**: Standard derivatives do not leverage the loss gradient of the target application, meaning they may displace vertices in directions that do not minimize the specific objective [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources [S1, unofficial].
- **Memory Complexity**: Not given in the sources [S1, unofficial].
- **Strengths**: Provides non-zero gradients for optimization, even in regions where standard differentiable rendering would yield zero gradients [S1, unofficial]. It is objective-aware, leading to better convergence in tasks like image reconstruction [S1, unofficial].
- **Limitations**: The approximation may not be as accurate as full differentiable rasterization. The "localness" of the differential filters can still limit the gradient flow to boundary pixels, although the loss-awareness helps mitigate this [S1, unofficial].
- **When to Use**: Suitable for optimization tasks where the goal is to fit a rendered image to a target, and where standard differentiable rendering fails due to lack of gradients [S1, unofficial].

## Background
Non-local approximated gradients were introduced by the authors of OpenDR (Loper and Black) in 2014 as part of the first general-purpose differentiable renderer [S1, unofficial][S2]. The method was later refined by Kato et al. in the Neural 3D Mesh Renderer (NMR) in 2018, who proposed non-local approximated gradients to address the limitations of OpenDR's local gradient computation [S1, unofficial]. The authors of NMR later replaced non-local gradients with local gradients similar to OpenDR to reduce computational complexity [S1, unofficial]. The method is used in various differentiable rendering pipelines, particularly in tasks involving 3D shape optimization from 2D images [S1, unofficial][S2].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S2] Differentiable Rendering - 可微渲染：从2D 学习3D, https://www.ultralytics.com/zh/glossary/differentiable-rendering, fetched 2026-10-30
