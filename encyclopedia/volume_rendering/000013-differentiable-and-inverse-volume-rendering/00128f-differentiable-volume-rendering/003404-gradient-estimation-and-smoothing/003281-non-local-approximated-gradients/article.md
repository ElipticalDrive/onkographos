# Non-local Approximated Gradients

Also called: Loper and Black, OpenDR (Approximated Spatial Gradients), OpenDR gradients

## Overview
Non-local approximated gradients are a gradient estimation technique used in differentiable rendering to provide non-zero, directed gradients that guide optimization away from local minima caused by the "localness" of standard rasterizer derivatives [S1, unofficial]. While standard differentiable rendering methods often fail because the rendering integrand is discontinuous with respect to scene parameters (such as vertex positions), causing the derivative of the integral to be zero almost everywhere, this method addresses the specific limitations of early differentiable renderers like OpenDR [S1, unofficial]. It sets itself apart from its sibling [Smooth Gradient Estimation via Convolution](6449-smooth-gradient-estimation-via-convolution.md) by explicitly incorporating backpropagated loss gradients to make the gradient flow "objective-aware," whereas convolution-based methods typically apply a fixed smoothing kernel to the integrand [S1, unofficial].

## How it works
The method computes pixel derivatives with respect to vertex positions using differential filters, but extends the basic spatial gradient approach by incorporating gradients of pixels backpropagated from a loss function [S1, unofficial]. This allows gradient flow from pixels outside the immediate primitive boundary to influence parameters, addressing the "localness" issue where only boundary pixels contribute to gradients in standard differential filter approaches [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Vertex positions $v_i$, rendered pixel values $v_p$, and the gradient of the loss function with respect to the pixel values $\frac{\partial \mathcal{L}}{\partial v_p}$ [S1, unofficial].
*   **Output:** Gradients of the loss with respect to vertex positions $\frac{\partial \mathcal{L}}{\partial v_i}$ [S1, unofficial].

**Data Structures:**
*   **Pixel Representation:** A pixel $v_p$ is represented as a weighted average of vertex positions:
    $$ v_p = \frac{\sum \tilde{w}_i v_i}{\sum \tilde{w}_i} $$
    where $\tilde{w}_i = 1$ for all vertices $i$ contributing to the pixel [S1, unofficial].
*   **Differential Filters:** Standard spatial differential filters (e.g., Sobel filters) are used to approximate spatial derivatives [S1, unofficial].

**Steps:**
1.  **Compute Spatial Gradients:** Calculate the spatial derivatives of the pixel value $v_p$ with respect to its screen coordinates $(\tilde{x}_p, \tilde{y}_p)$ using differential filters. The pixel derivatives with respect to the vertex position $v_p$ are computed as:
    $$ \left\{ \frac{\partial c_v}{\partial \tilde{x}_p}, \frac{\partial c_v}{\partial \tilde{y}_p} \right\} = \frac{\partial c_v}{\partial v_p} $$
    where $c_v$ represents the color or intensity value associated with the pixel [S1, unofficial].
2.  **Incorporate Loss Gradients:** Instead of using only the spatial derivatives, incorporate the gradients of pixels backpropagated from the loss function $\frac{\partial \mathcal{L}}{\partial v_p}$ [S1, unofficial]. This step ensures that the gradient flow is aware of the specific optimization objective (e.g., image reconstruction) [S1, unofficial].
3.  **Backpropagate to Vertices:** Propagate the combined gradients (spatial and loss-aware) back to the vertex positions $v_i$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_non_local_approximated_gradients(vertices, pixels, loss_gradients, filter_kernel):
    """
    Compute gradients for vertices using non-local approximated gradients.
    
    Args:
        vertices: Array of vertex positions (N, 3).
        pixels: Array of rendered pixel values (H, W).
        loss_gradients: Array of loss gradients with respect to pixels (H, W).
        filter_kernel: Differential filter kernel (e.g., Sobel).
        
    Returns:
        vertex_gradients: Gradients of loss with respect to vertices (N, 3).
    """
    # Step 1: Compute spatial gradients using differential filters
    # Approximate spatial derivatives of pixel values
    spatial_gradients_x = apply_filter(pixels, filter_kernel_x)
    spatial_gradients_y = apply_filter(pixels, filter_kernel_y)
    
    # Step 2: Incorporate loss gradients
    # Multiply spatial gradients by loss gradients to make them objective-aware
    # This allows gradients from pixels outside the immediate boundary to influence vertices
    weighted_gradients_x = spatial_gradients_x * loss_gradients
    weighted_gradients_y = spatial_gradients_y * loss_gradients
    
    # Step 3: Backpropagate to vertices
    # Initialize vertex gradients
    vertex_gradients = np.zeros_like(vertices)
    
    # For each vertex, accumulate gradients from all pixels it influences
    # Note: The exact accumulation depends on the rasterization model
    # For simplicity, assuming a direct mapping or barycentric weighting
    for i in range(len(vertices)):
        # Get pixels influenced by vertex i
        # In a real implementation, this would involve rasterization data structures
        influenced_pixels = get_influenced_pixels(vertices[i], rasterization_data)
        
        # Accumulate gradients
        for pixel in influenced_pixels:
            # Weight by the contribution of the vertex to the pixel
            # In OpenDR, w_i = 1, so the contribution is uniform
            vertex_gradients[i] += weighted_gradients_x[pixel] * spatial_gradient_x_contribution
            vertex_gradients[i] += weighted_gradients_y[pixel] * spatial_gradient_y_contribution
            
    return vertex_gradients

def apply_filter(image, kernel):
    """Apply a convolution filter to an image."""
    # Standard convolution operation
    return convolve(image, kernel)

def get_influenced_pixels(vertex, rasterization_data):
    """Get all pixels that are influenced by a given vertex."""
    # This function would typically use rasterization data to find pixels
    # covered by triangles containing the vertex
    # Implementation detail: [G]
    pass
```

**Parameters:**
*   **Filter Kernel:** Typically a Sobel filter or similar differential filter [S1, unofficial].
*   **Weights:** $\tilde{w}_i = 1$ for all vertices [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Pixels:** Gradients are only non-zero for pixels located on the boundary of the triangle; gradients at interior pixels are zero because they do not depend on the vertex positions in the same way [S1, unofficial].
*   **Standard Gradients:** Standard gradients at pixels not covered by primitives cannot be used, leading to poor local minima [S1, unofficial].
*   **Objective Awareness:** Standard derivatives do not leverage the loss gradient of the target application (e.g., image reconstruction), meaning they may displace vertices in directions that do not minimize the specific objective [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Provides non-zero gradients for optimization, addressing the "localness" issue of standard differentiable rendering [S1, unofficial]. Incorporates loss gradients to make the gradient flow objective-aware [S1, unofficial].
*   **Limitations:** Only boundary pixels contribute to gradients in the basic spatial gradient approach, which can lead to poor local minima [S1, unofficial]. The method was later replaced by local gradients similar to OpenDR to reduce computation complexity [S1, unofficial].
*   **When to Use:** When standard differentiable rendering methods fail due to discontinuous integrands and local minima [S1, unofficial].
*   **When Not to Use:** When computation complexity is a concern, as the method was later replaced by simpler local gradients [S1, unofficial].

## Background
Non-local approximated gradients were proposed by the authors of OpenDR, the first general purpose differentiable renderer [S1, unofficial]. The method was introduced to address the limitations of standard differentiable rendering, such as the "localness" of gradient computation and the lack of objective awareness [S1, unofficial]. The authors later replaced non-local gradients with local gradients similar to OpenDR to reduce computation complexity [S1, unofficial].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
