# Non-local Approximated Gradients

Also called: Loper and Black, OpenDR (Approximated Spatial Gradients), OpenDR gradients

## Overview
Non-local approximated gradients are a gradient estimation technique for differentiable volume and mesh rendering that computes pixel derivatives with respect to vertex positions using differential filters, allowing gradient flow from pixels outside the immediate primitive boundary [S1, S1, unofficial]. Unlike standard differentiable rendering which only propagates gradients through covered pixels, this method enables optimization objectives to influence vertices via boundary pixels, addressing the issue of poor local minima caused by the "localness" of standard rasterizer derivatives [S1, unofficial]. It sets itself apart from sibling methods like [Smooth Gradient Estimation via Convolution](6449-smooth-gradient-estimation-via-convolution.md) by explicitly leveraging backpropagated loss gradients to make the gradient flow objective-aware, rather than relying solely on geometric or intensity-based smoothing [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input:** Vertex positions $v_i$, pixel coordinates $\tilde{x}_p, \tilde{y}_p$, and the backpropagated gradient of the loss function with respect to the pixel values $\frac{\partial L}{\partial v_p}$ (or $\tilde{c}_v$ in the source notation).
- **Output:** Approximated gradients of the pixel value $v_p$ with respect to vertex positions, $\frac{\partial v_p}{\partial v_i}$, which are then used to update vertices.

**Data Structures**
- **Pixel Representation:** A pixel $v_p$ is represented as a weighted average of vertex contributions. In the specific formulation by Loper and Black [19], the weight $\tilde{w}_i$ is set to 1 [S1, unofficial].
- **Differential Filters:** Standard spatial differential filters (e.g., Sobel filters) are applied to the rendered image to estimate spatial derivatives [S1, unofficial].

**Formulas**
The pixel value $v_p$ is defined as:
$$ v_p = \frac{\tilde{w}_i v_i}{\tilde{w}_i} $$
where $\tilde{w}_i = 1$ [S1, unofficial].

The derivatives of the pixel value $v_p$ with respect to its spatial coordinates $\tilde{x}_p, \tilde{y}_p$ are computed using differential filters. These spatial derivatives are equated to the derivatives with respect to the vertex position $v_p$ (note: the source notation $\frac{\partial c_v}{\partial v_p}$ appears to denote the gradient flow with respect to the pixel value, which is then mapped to vertex updates) [S1, unofficial]:
$$ \left\{ \frac{\partial c_v}{\partial \tilde{x}_p}, \frac{\partial c_v}{\partial \tilde{y}_p} \right\} = \frac{\partial c_v}{\partial v_p} $$
In practice, this implies that the gradient with respect to the vertex is derived from the spatial gradient of the pixel intensity, propagated through the differential filter [S1, unofficial].

**Steps**
1. **Render:** Perform standard rasterization to obtain pixel values $v_p$ for all pixels.
2. **Backpropagate Loss:** Compute the gradient of the loss function $L$ with respect to each pixel value, $\frac{\partial L}{\partial v_p}$ [S1, unofficial].
3. **Apply Differential Filters:** For each pixel $v_p$, apply a differential filter (such as a Sobel filter) to the image to compute spatial gradients $\frac{\partial c_v}{\partial \tilde{x}_p}$ and $\frac{\partial c_v}{\partial \tilde{y}_p}$ [S1, unofficial].
4. **Map to Vertex Gradients:** Equate the spatial gradients to the vertex gradients. The gradient flow from the loss is thus extended to pixels that are not strictly inside the primitive but are adjacent to the boundary, allowing the gradient to "reach" the vertices [S1, unofficial].
5. **Update Vertices:** Use the computed gradients to update vertex positions $v_i$ to minimize the loss [S1, unofficial].

**Parameters**
- **Filter Type:** Sobel filter is explicitly mentioned as an example [S1, unofficial].
- **Weight $\tilde{w}_i$:** Set to 1 [S1, unofficial].

**Edge Cases and Pitfalls**
- **Boundary-Only Gradients:** Gradients are only non-zero for pixels located on the boundary of the triangle. Interior pixels do not depend on vertex positions in the same way, so their gradients are zero [S1, unofficial].
- **Poor Local Minima:** Standard gradients at pixels not covered by primitives cannot be used, leading to poor local minima [S1, unofficial].
- **Objective Awareness:** Standard derivatives do not leverage the loss gradient of the target application (e.g., image reconstruction). For example, if the objective is to decrease the intensity of a pixel, the vertex should be displaced accordingly; if the objective is to increase it, it should not. Non-local gradients address this by incorporating the loss gradient [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_non_local_gradients(vertices, pixels, loss_gradient, filter_kernel):
    """
    Compute gradients for vertices using non-local approximated gradients.
    
    Args:
        vertices: Array of vertex positions.
        pixels: Rendered image pixels.
        loss_gradient: Gradient of loss w.r.t. pixels.
        filter_kernel: Differential filter kernel (e.g., Sobel).
        
    Returns:
        vertex_gradients: Gradients w.r.t. vertex positions.
    """
    # Step 1: Compute spatial gradients of the loss using differential filters
    # This extends gradient flow to boundary pixels
    spatial_gradients_x = apply_filter(loss_gradient, filter_kernel_x)
    spatial_gradients_y = apply_filter(loss_gradient, filter_kernel_y)
    
    # Step 2: Map spatial gradients to vertex gradients
    # In the OpenDR formulation, pixel derivatives are computed using differential filters
    # and equated to vertex derivatives.
    # Note: The exact mapping from spatial gradient to vertex gradient depends on the 
    # rasterization Jacobian, which is approximated here by the filter output.
    
    vertex_gradients = compute_vertex_gradients_from_spatial(
        vertices, 
        spatial_gradients_x, 
        spatial_gradients_y
    )
    
    return vertex_gradients

def apply_filter(image, kernel):
    """Apply a convolution filter to an image."""
    # Standard convolution operation
    return convolve(image, kernel)

def compute_vertex_gradients_from_spatial(vertices, grad_x, grad_y):
    """
    Approximate vertex gradients from spatial gradients.
    This is a placeholder for the specific Jacobian approximation used in OpenDR.
    The source notes that { dcv/dxp, dcv/dyp } = dcv/dvp, implying a direct mapping
    or approximation via the filter.
    """
    # Implementation detail: The gradient is propagated to vertices based on 
    # which pixels are influenced by the vertices.
    # For boundary pixels, the gradient is non-zero.
    # This function aggregates the spatial gradients to vertex space.
    pass
```

## Subtopics

### Local Gradients (OpenDR)
A later variant used by the OpenDR authors to reduce computation complexity. It is similar to non-local gradients but restricts the gradient flow to local neighborhoods, sacrificing some objective awareness for speed [S1, unofficial].

### Neural 3D Mesh Renderer (NMR)
A method that critiques OpenDR's localness and proposes non-local approximated gradients to overcome poor local minima and lack of objective awareness [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources. However, the application of differential filters (e.g., Sobel) is $O(N)$ where $N$ is the number of pixels, and the mapping to vertices is typically $O(V)$ where $V$ is the number of vertices [S1, unofficial].
- **Memory Complexity:** Not given in the sources. Requires storage for the image, loss gradients, and intermediate filter outputs [S1, unofficial].
- **Strengths:** Overcomes the limitation of standard differentiable rendering where gradients only flow through covered pixels. It allows the optimization to be aware of the specific objective (e.g., image reconstruction) by leveraging backpropagated loss gradients [S1, unofficial].
- **Limitations:** Gradients are only non-zero for boundary pixels, which can still lead to poor local minima if the boundary is not well-defined or if the object is small [S1, unofficial]. The method was later replaced by local gradients in OpenDR to reduce computation complexity, suggesting it may be computationally expensive or complex to implement [S1, unofficial].
- **When to Use:** When standard differentiable rendering fails due to poor local minima or when the optimization objective requires gradient flow from pixels outside the immediate primitive boundary [S1, unofficial].
- **When Not to Use:** When computational efficiency is critical, as local gradients may be preferred [S1, unofficial].

## Background
Non-local approximated gradients were proposed by Loper and Black in the context of OpenDR, the first general-purpose differentiable renderer [S1, unofficial]. The method was later critiqued by Kato et al. (NMR) for its localness and lack of objective awareness, leading to the proposal of non-local approximated gradients that incorporate loss gradients [S1, unofficial]. The OpenDR authors later replaced non-local gradients with local gradients to reduce computation complexity [S1, unofficial].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
