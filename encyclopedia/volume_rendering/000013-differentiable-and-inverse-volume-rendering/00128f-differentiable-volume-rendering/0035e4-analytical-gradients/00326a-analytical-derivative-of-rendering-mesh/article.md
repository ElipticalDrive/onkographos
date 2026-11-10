# Analytical Derivative of Rendering (Mesh)
Also called: Analytical Differentiable Mesh Rendering

## Overview
Analytical Derivative of Rendering (Mesh) is a differentiable rendering technique that computes exact gradients of the output image with respect to input parameters (shape, camera, material, lighting) by analytically differentiating the continuous steps of the rendering pipeline [S1, unofficial]. It sets itself apart from siblings like [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md) and [Differentiable Volume Rendering with Analytical Gradients](5983-differentiable-volume-rendering-with-analytical-gradients.md) by operating on explicit triangle meshes rather than volumetric representations, and from [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md) by relying on exact derivatives for continuous operations rather than approximating the rasterization visibility step [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Mesh geometry (vertex positions $\mathbf{v}_i$), material properties (vertex colors $\mathbf{c}_i$, normals $\mathbf{n}_i$), camera parameters (view/projection matrices), and lighting parameters.
*   **Outputs:** Rendered image color $\mathbf{C}$ and its analytical gradients $\frac{\partial \mathbf{C}}{\partial \mathbf{v}_i}, \frac{\partial \mathbf{C}}{\partial \mathbf{c}_i}, \dots$ with respect to input parameters.

**Data Structures:**
*   **Triangle:** Defined by three vertices $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$. Each vertex has position $\mathbf{p} \in \mathbb{R}^3$, color $\mathbf{c} \in \mathbb{R}^3$, and normal $\mathbf{n} \in \mathbb{R}^3$.
*   **Pixel:** Defined by screen coordinates $(u, v)$, depth $z$, and barycentric weights $(w_1, w_2, w_3)$.

**Steps:**
1.  **Projection:** Transform triangle vertices from world space to camera space using view matrix $\mathbf{V}$, then to screen space using projection matrix $\mathbf{P}$. These are differentiable matrix multiplications [S1, unofficial].
    $$ \mathbf{p}' = \mathbf{V} \mathbf{p}, \quad \mathbf{p}_{screen} = \mathbf{P} \mathbf{p}' $$
2.  **Barycentric Coordinate Computation:** For a pixel covered by a triangle, compute barycentric weights $w_i$ by solving a differentiable linear program or via analytic inversion of the triangle's screen-space basis [S1, unofficial].
    $$ \mathbf{p}_{pixel} = \sum_{i=1}^3 w_i \mathbf{p}_{i, screen}, \quad \sum_{i=1}^3 w_i = 1 $$
3.  **Depth Interpolation:** Compute pixel depth $z$ by interpolating the depth of the three vertices in camera space using the barycentric weights [S1, unofficial].
    $$ z = \sum_{i=1}^3 w_i z_i $$
4.  **Attribute Interpolation:** Express material and normal vectors at the pixel as a weighted sum of the vertex attributes [S1, unofficial].
    $$ \mathbf{c}_{pixel} = \sum_{i=1}^3 w_i \mathbf{c}_i, \quad \mathbf{n}_{pixel} = \sum_{i=1}^3 w_i \mathbf{n}_i $$
5.  **Shading:** Compute pixel color using a differentiable local illumination reflection model (e.g., Phong, Lambertian, Spherical Harmonics) given the interpolated material, lighting, and normal vectors [S1, unofficial].
    $$ \mathbf{C}_{pixel} = \text{Shading}(\mathbf{c}_{pixel}, \mathbf{n}_{pixel}, \mathbf{L}, \mathbf{M}) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def analytical_gradient_mesh_rendering(vertices, faces, camera_matrix, light_params):
    """
    Compute rendered image and analytical gradients for a mesh.
    vertices: Tensor of shape (N, 3)
    faces: Tensor of shape (M, 3) containing vertex indices
    camera_matrix: 4x4 projection/view matrix
    light_params: Lighting configuration
    """
    # 1. Project vertices to screen space
    # Homogeneous coordinates
    verts_h = torch.cat([vertices, torch.ones_like(vertices[:, :1])], dim=1) # (N, 4)
    verts_cam = camera_matrix @ verts_h.T # (4, N)
    verts_screen = verts_cam[:3, :] / verts_cam[3, :] # (3, N)
    
    # 2. For each pixel, identify covering triangles and compute barycentric coords
    # Note: Exact implementation of differentiable rasterization requires handling
    # the discrete selection step. This pseudocode assumes a differentiable rasterizer
    # is available or approximated, as the core analytical derivatives apply to
    # the continuous interpolation and shading steps [S1, unofficial].
    
    rendered_image = torch.zeros(H, W, 3)
    grad_vertices = torch.zeros_like(vertices)
    
    for face in faces:
        v0, v1, v2 = vertices[face[0]], vertices[face[1]], vertices[face[2]]
        
        # Compute screen space positions
        p0 = verts_screen[:, face[0]]
        p1 = verts_screen[:, face[1]]
        p2 = verts_screen[:, face[2]]
        
        # Compute barycentric weights for all pixels covered by this triangle
        # This step involves solving a linear system or using analytic formulas
        # that are differentiable with respect to vertex positions [S1, unofficial]
        weights = compute_barycentric_weights(p0, p1, p2) # (H, W, 3)
        
        # Interpolate attributes
        c0, c1, c2 = colors[face[0]], colors[face[1]], colors[face[2]]
        n0, n1, n2 = normals[face[0]], normals[face[1]], normals[face[2]]
        
        c_pixel = weights[:, :, 0:1] * c0 + weights[:, :, 1:2] * c1 + weights[:, :, 2:3] * c2
        n_pixel = weights[:, :, 0:1] * n0 + weights[:, :, 1:2] * n1 + weights[:, :, 2:3] * n2
        
        # Shading
        color = shading_model(c_pixel, n_pixel, light_params)
        
        # Accumulate gradients
        # Since the operation is linear in vertex attributes for color,
        # gradients w.r.t vertex colors are direct.
        # Gradients w.r.t vertex positions flow through barycentric weights
        # and shading model [S1, unofficial]
        grad_vertices[face[0]] += torch.sum(weights[:, :, 0:1] * (color - c_pixel) + ...) # Simplified
        # ... similar for v1, v2
        
    return rendered_image, grad_vertices
```

**Edge Cases and Pitfalls:**
*   **Zero Gradient for Non-Selected Triangles:** In standard rendering, exactly one triangle per pixel is selected. If a pixel's color is determined solely by a single triangle's vertex colors, the gradient of the pixel color with respect to vertices of *other* triangles is zero, preventing geometry optimization for occluded or background triangles [S1, unofficial].
*   **Zero Gradient for Vertex Position:** The derivative of pixel color with respect to vertex position is always zero for all pixels and vertices in the analytical formulation if the triangle selection is fixed, meaning it does not help optimize geometry when vertex movement changes which triangle is selected or affects neighboring pixels [S1, unofficial].

## Subtopics

### [DIB-R](6798-dib-r.md)
DIB-R is a differentiable rendering technique for meshes that computes exact gradients of the output image with respect to input parameters by analytically differentiating the rendering pipeline, specifically addressing the non-differentiability of standard rasterization. It sets itself apart from siblings like [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md) by explicitly separating the image into foreground and background regions to handle gradients differently, thereby avoiding the blurriness associated with soft rasterization while mitigating vanishing gradients in background areas.

### [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md)
Neural 3D Mesh Renderer (NMR) is a differentiable rendering method that approximates the non-differentiable rasterization step by replacing sudden visibility changes with linear interpolation, allowing gradients to flow from pixels to mesh vertices beyond the immediate boundary. It addresses the "localness" limitation of prior methods like OpenDR, where gradients only flowed from boundary pixels, by using non-local approximated gradients that are aware of the optimization objective.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles and pixels, assuming efficient rasterization. The analytical gradient computation adds a constant factor overhead per pixel/triangle interaction.
*   **Memory Complexity:** Linear with respect to the number of vertices and pixels to store gradients.
*   **Strengths:** Provides exact gradients for continuous operations (projection, interpolation, shading). No need to store the full computational graph for these steps, allowing gradients to be computed in a single backward pass or alongside the forward pass with constant memory overhead relative to the number of parameters [S5].
*   **Limitations:** Fails to provide gradients for the discrete triangle selection step. This leads to zero gradients for occluded triangles and no gradient flow for geometry changes that alter visibility [S1, unofficial].
*   **When to Use:** When the mesh topology is fixed and the optimization focuses on parameters that do not change visibility (e.g., texture colors, lighting, camera pose) or when combined with methods that approximate the rasterization step.
*   **When Not to Use:** When optimizing mesh geometry where vertex movements significantly change visibility or occlusion relationships, as the analytical derivatives alone are insufficient [S1, unofficial].

## Background
Analytical Derivative of Rendering (Mesh) originates from the broader field of differentiable rendering, which seeks to make the rendering pipeline differentiable to enable gradient-based optimization of 3D scenes [S1, unofficial]. The specific application to meshes leverages the differentiability of standard graphics operations (matrix transformations, barycentric interpolation, shading models) while acknowledging the non-differentiability of the rasterization visibility step [S1, unofficial]. It is used in applications such as 3D reconstruction, texture optimization, and camera pose estimation where exact gradients are beneficial for convergence.

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
