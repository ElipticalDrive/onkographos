# Analytical Derivative of Rendering (Mesh)
Also called: Analytical Differentiable Mesh Rendering, Analytical Gradients for Mesh Rasterization

## Overview
This method computes exact gradients of the output image with respect to input parameters (shape, camera, material, lighting) by analytically differentiating the continuous steps of the mesh rendering pipeline, excluding the discrete triangle selection step [S1, unofficial]. It sets itself apart from siblings like [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md) by operating on explicit geometric primitives (triangles) rather than implicit volumetric fields, and from [Differentiable Volume Rendering with Analytical Gradients](5983-differentiable-volume-rendering-with-analytical-gradients.md) by relying on barycentric interpolation of vertex attributes rather than basis function derivatives. The primary limitation is that it yields zero gradients for vertices not currently selected for a pixel, failing to optimize occluded or background geometry without additional approximation mechanisms [S1, unofficial].

## How it works
The method differentiates the rendering pipeline by treating the rasterization selection as a discrete, non-differentiable gate, while all subsequent attribute interpolation and shading operations are treated as continuous, differentiable functions [S1, unofficial].

**Inputs and Outputs**
*   **Inputs:**
    *   Mesh geometry: Set of triangles $T$, where each triangle $t$ has three vertices $v_i \in \mathbb{R}^3$ (positions) and associated attributes (colors $\mathbf{c}_i$, normals $\mathbf{n}_i$).
    *   Camera parameters: Projection matrix $\mathbf{P}$ and view matrix $\mathbf{V}$.
    *   Lighting parameters: Light positions $\mathbf{l}_j$ and intensities.
    *   Material parameters: Coefficients for the reflection model (e.g., Phong, Lambertian).
*   **Outputs:**
    *   Gradient of pixel color $\frac{\partial C}{\partial \theta}$ with respect to input parameters $\theta$ (vertex positions, camera matrices, material coefficients).

**Data Structures**
*   **Triangle:** Struct containing three vertex indices.
*   **Vertex:** Struct containing 3D position $\mathbf{v}$, 3D normal $\mathbf{n}$, and color $\mathbf{c}$.
*   **Pixel:** Struct containing screen coordinates $(u, v)$, depth $z$, and computed color $C$.

**Numbered Steps**

1.  **World to Camera Space Transformation:**
    Transform vertex positions $\mathbf{v}_i$ and light positions $\mathbf{l}_j$ from world space to camera space using the view matrix $\mathbf{V}$:
    $$ \mathbf{v}_i^{cam} = \mathbf{V} \begin{bmatrix} \mathbf{v}_i \\ 1 \end{bmatrix}, \quad \mathbf{l}_j^{cam} = \mathbf{V} \begin{bmatrix} \mathbf{l}_j \\ 1 \end{bmatrix} $$
    Transform normal vectors $\mathbf{n}_i$ using the inverse-transpose of the upper-left $3 \times 3$ submatrix of $\mathbf{V}$ to preserve orthogonality:
    $$ \mathbf{n}_i^{cam} = (\mathbf{V}_{1:3,1:3}^{-1})^T \mathbf{n}_i $$
    These operations are differentiable with respect to $\mathbf{V}$ and vertex attributes [S1, unofficial].

2.  **Camera to Screen Space Projection:**
    Project camera-space coordinates to screen space using the projection matrix $\mathbf{P}$:
    $$ \begin{bmatrix} x_i^{scr} \\ y_i^{scr} \\ w_i^{scr} \end{bmatrix} = \mathbf{P} \begin{bmatrix} \mathbf{v}_i^{cam} \\ 1 \end{bmatrix} $$
    Perform perspective division to obtain normalized device coordinates (NDC) or screen coordinates:
    $$ u_i = \frac{x_i^{scr}}{w_i^{scr}}, \quad v_i = \frac{y_i^{scr}}{w_i^{scr}}, \quad z_i^{cam} = \frac{z_i^{scr}}{w_i^{scr}} $$
    The gradient with respect to $\mathbf{P}$ and $\mathbf{v}_i$ is computed via the chain rule through the matrix multiplication and division [S1, unofficial].

3.  **Triangle Selection (Non-Differentiable):**
    For each pixel, determine the visible triangle $t^*$ with the minimum depth $z$. This step is discrete and **not differentiable** [S1, unofficial]. Gradients are only computed for vertices of the selected triangle $t^*$.

4.  **Barycentric Coordinate Computation:**
    For a pixel $p$ inside triangle $t^*$ with vertices $v_1, v_2, v_3$, compute barycentric weights $\omega_1, \omega_2, \omega_3$ such that:
    $$ p = \omega_1 v_1 + \omega_2 v_2 + \omega_3 v_3, \quad \sum_{i=1}^3 \omega_i = 1 $$
    This is solved via a differentiable linear program or closed-form geometric solution. The weights $\omega_i$ are differentiable with respect to vertex positions $v_i$ and pixel coordinates $p$ [S1, unofficial].

5.  **Attribute Interpolation:**
    Interpolate vertex attributes (color $\mathbf{c}$, normal $\mathbf{n}$, depth $z$) at the pixel using the barycentric weights:
    $$ \mathbf{c}_p = \sum_{i=1}^3 \omega_i \mathbf{c}_i, \quad \mathbf{n}_p = \frac{\sum_{i=1}^3 \omega_i \mathbf{n}_i}{\| \sum_{i=1}^3 \omega_i \mathbf{n}_i \|}, \quad z_p = \sum_{i=1}^3 \omega_i z_i^{cam} $$
    Note: Normals are typically renormalized after interpolation. The gradient flows through the weighted sum [S1, unofficial].

6.  **Shading and Color Computation:**
    Compute pixel color $C_p$ using a differentiable local illumination model (e.g., Phong, Lambertian, Spherical Harmonics) given $\mathbf{c}_p$, $\mathbf{n}_p$, and light parameters:
    $$ C_p = f(\mathbf{c}_p, \mathbf{n}_p, \mathbf{l}_j, \text{material params}) $$
    Since models like Phong and Lambertian are analytic and differentiable, the gradient $\frac{\partial C_p}{\partial \theta}$ is computed directly [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_analytical_gradients(mesh, camera_matrix, light_positions, material_params):
    """
    Computes gradients for mesh rendering.
    Note: This is a simplified representation. The actual implementation requires
    handling the discrete selection step separately and only backpropagating
    through the selected triangle's vertices.
    """
    # 1. Transform vertices to camera space
    # v_cam = V * v_world
    # n_cam = (V_inv^T) * n_world
    v_cam, n_cam = transform_to_camera_space(mesh.vertices, camera_matrix)
    
    # 2. Project to screen space
    # u, v, w = P * v_cam
    # u_ndc = u/w, v_ndc = v/w
    u_ndc, v_ndc, z_cam = project_to_screen(v_cam, camera_matrix)
    
    # 3. Rasterization (Discrete Selection)
    # For each pixel, find triangle t* with min z_cam
    # This step is NOT differentiable. Gradients for non-selected triangles are 0.
    selected_triangles = rasterize(u_ndc, v_ndc, z_cam, image_resolution)
    
    gradients = {}
    
    for pixel in image_pixels:
        t = selected_triangles[pixel]
        if t is None:
            continue
            
        # 4. Compute Barycentric Weights
        # Solve for w1, w2, w3 such that pixel_pos = w1*v1 + w2*v2 + w3*v3
        weights = compute_barycentric_weights(pixel.position, t.vertices)
        
        # 5. Interpolate Attributes
        c_pixel = sum(weights[i] * t.vertices[i].color for i in range(3))
        n_pixel = normalize(sum(weights[i] * t.vertices[i].normal for i in range(3)))
        
        # 6. Compute Color and Gradient
        # C = shading_model(c_pixel, n_pixel, light_positions, material_params)
        C = compute_shading(c_pixel, n_pixel, light_positions, material_params)
        
        # Backpropagate through shading model
        dC_dC_pixel, dC_dn_pixel = gradient_shading(C, c_pixel, n_pixel, material_params)
        
        # Backpropagate through interpolation
        # dC/dv_i = dC/dC_pixel * dC_pixel/dv_i + dC/dn_pixel * dn_pixel/dv_i
        # dC_pixel/dv_i involves derivative of barycentric weights w.r.t vertex positions
        for i in range(3):
            dv_i = compute_vertex_gradient(weights, t.vertices[i], dC_dC_pixel, dC_dn_pixel)
            gradients[t.vertices[i]] = gradients.get(t.vertices[i], 0) + dv_i
            
    return gradients
```

**Parameters**
*   **Reflection Model:** Phong, Lambertian, or Spherical Harmonics. Typical values for Phong: specular exponent $k \in [1, 128]$, diffuse coefficient $k_d \in [0, 1]$, specular coefficient $k_s \in [0, 1]$ [S1, unofficial].
*   **Barycentric Solver:** Linear program or closed-form. No specific typical values, but precision matters for degenerate triangles [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Zero Gradient for Non-Selected Vertices:** If a pixel is covered by triangle $A$, the gradient of the pixel color with respect to vertices of triangle $B$ (occluded or background) is exactly zero. This prevents optimization of occluded geometry [S1, unofficial].
*   **Discrete Selection:** The derivative of the pixel color with respect to vertex position is zero for all pixels and vertices in the strict analytical formulation because the selection is discrete. Moving a vertex might change which triangle is selected, but the analytical derivative does not capture this "jump" [S1, unofficial].
*   **Degenerate Triangles:** If a triangle collapses to a line or point, barycentric coordinates become undefined or unstable. Guard against division by zero in area calculations [G].

## Subtopics

### [DIB-R](6798-dib-r.md)
DIB-R is a differentiable rendering technique for meshes that computes exact gradients of the output image with respect to input parameters by analytically differentiating the rendering pipeline, specifically addressing the non-differentiability of standard rasterization. It sets itself apart from siblings like [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md) by explicitly separating the image into foreground and background regions to handle gradients differently, thereby avoiding the blurriness associated with soft rasterization while mitigating vanishing gradients in background areas.

### [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md)
Neural 3D Mesh Renderer (NMR) is a differentiable rendering method that approximates the non-differentiable rasterization step by replacing sudden visibility changes with linear interpolation, allowing gradients to flow from pixels to mesh vertices beyond the immediate boundary. It addresses the "localness" limitation of prior methods like OpenDR, where gradients only flowed from boundary pixels, by using non-local approximated gradients that are aware of the optimization objective.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is the number of triangles per pixel (typically 1 for standard rasterization). The analytical gradient computation is linear in the number of attributes and shading model complexity.
*   **Memory Complexity:** $O(V + T)$ for storing mesh vertices and triangles, plus $O(N)$ for the image buffer. No significant overhead for gradient storage compared to standard backpropagation if computed on-the-fly [S1, unofficial].
*   **Strengths:** Exact gradients for selected pixels; efficient; compatible with standard graphics pipelines; differentiable with respect to camera, lighting, and material parameters [S1, unofficial].
*   **Limitations:** Zero gradients for occluded/background triangles; no gradient flow across triangle boundaries due to discrete selection; sensitive to degenerate geometry [S1, unofficial].
*   **When to Use:** Optimization of visible geometry, camera pose, or material parameters where occlusion is minimal or handled by other means.
*   **When Not to Use:** Optimization of occluded geometry or when global visibility changes are critical to the gradient signal.

## Background
The method originates from the survey "Differentiable Rendering: A Survey" [S1, unofficial], which categorizes analytical derivatives as a core technique in differentiable rendering. It builds on the principle that rasterization steps excluding triangle selection are differentiable [S1, unofficial]. The approach is foundational for methods like DIB-R and NMR, which extend it to handle the non-differentiable selection step via approximation [S1, unofficial][S2].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
