# Analytical Derivative of Rendering (Mesh)
Also called: Analytical Differentiable Mesh Rendering

## Overview
Analytical Derivative of Rendering (Mesh) is a differentiable rendering technique that computes exact gradients of the output image with respect to input parameters (shape, camera, material, lighting) by analytically differentiating the continuous steps of the rasterization pipeline, explicitly excluding the discrete triangle selection step [S1, unofficial]. It sets itself apart from siblings like [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md) and [Differentiable Volume Rendering with Analytical Gradients](5983-differentiable-volume-rendering-with-analytical-gradients.md) by operating on explicit mesh geometry rather than volumetric fields, and from [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md) by relying on exact analytical derivatives for the differentiable components rather than soft approximations of the visibility step [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Mesh vertex positions $\mathbf{v}_i$, vertex colors $\mathbf{c}_i$, vertex normals $\mathbf{n}_i$, camera extrinsic/intrinsic matrices, light positions/directions, and material parameters.
*   **Outputs:** Rendered image color $\hat{C}$ and its analytical gradients $\frac{\partial \hat{C}}{\partial \theta}$ for all input parameters $\theta$.

**Data Structures:**
*   **Triangle:** Defined by three vertices $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$. Each vertex has position $\mathbf{p} \in \mathbb{R}^3$, color $\mathbf{c} \in \mathbb{R}^3$, and normal $\mathbf{n} \in \mathbb{R}^3$.
*   **Pixel:** Defined by screen coordinates $(u, v)$, depth $z$, and computed color $\mathbf{C}$.

**Steps:**
1.  **World to Camera Space:** Project mesh triangles from world space to camera space using matrix multiplication. Let $\mathbf{M}_{wc}$ be the world-to-camera transformation matrix. For each vertex $\mathbf{v}_i$:
    $$ \mathbf{v}_i^{cam} = \mathbf{M}_{wc} \mathbf{v}_i $$
    This operation is differentiable with respect to vertex positions and camera matrices [S1, unofficial].

2.  **Camera to Screen Space:** Transform camera-space positions to screen space using the projection matrix $\mathbf{M}_{proj}$.
    $$ \mathbf{v}_i^{screen} = \mathbf{M}_{proj} \mathbf{v}_i^{cam} $$
    Perform perspective division to obtain normalized device coordinates (NDC) and subsequently screen coordinates $(u, v)$ [S1, unofficial].

3.  **Barycentric Coordinate Computation:** For a pixel $p$ covered by a triangle, compute barycentric weights $\mathbf{w} = [w_1, w_2, w_3]^T$. This is achieved by solving a differentiable linear program or using the inverse of the triangle's vertex matrix in screen space. The weights satisfy:
    $$ \mathbf{p}_{screen} = \sum_{i=1}^3 w_i \mathbf{v}_i^{screen}, \quad \sum_{i=1}^3 w_i = 1 $$
    The computation of $\mathbf{w}$ is differentiable with respect to vertex positions [S1, unofficial].

4.  **Depth Interpolation:** Compute the pixel depth $z_p$ by interpolating the depth of the three vertices in camera space:
    $$ z_p = \sum_{i=1}^3 w_i z_i^{cam} $$
    where $z_i^{cam}$ is the depth component of $\mathbf{v}_i^{cam}$ [S1, unofficial].

5.  **Attribute Interpolation:** Express material properties (e.g., color $\mathbf{c}_p$) and normal vectors $\mathbf{n}_p$ at the pixel as a weighted sum of the vertex attributes:
    $$ \mathbf{c}_p = \sum_{i=1}^3 w_i \mathbf{c}_i, \quad \mathbf{n}_p = \frac{\sum_{i=1}^3 w_i \mathbf{n}_i}{\|\sum_{i=1}^3 w_i \mathbf{n}_i\|} $$
    Normal interpolation requires renormalization, which is differentiable [S1, unofficial].

6.  **Shading:** Compute pixel color $\mathbf{C}_p$ using a differentiable local illumination reflection model (e.g., Phong, Lambertian, or Spherical Harmonics) given $\mathbf{c}_p$, $\mathbf{n}_p$, lighting parameters, and material parameters. Popular models like Phong [47], Lambertian [48], and Spherical Harmonics [49] are all differentiable [S1, unofficial].

7.  **Gradient Computation:** Compute the analytical derivatives of pixel depth and color with respect to input parameters using the chain rule on the steps above. Since steps 1-6 are composed of differentiable operations (matrix products, linear interpolation, differentiable shading models), the gradients $\frac{\partial \mathbf{C}_p}{\partial \theta}$ can be computed analytically [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_mesh_analytical(vertices, camera_matrix, projection_matrix, lights, material_params):
    """
    Renders a mesh and computes analytical gradients for differentiable parameters.
    vertices: List of vertex objects with pos, color, normal.
    camera_matrix: 4x4 world-to-camera matrix.
    projection_matrix: 4x4 projection matrix.
    lights: List of light sources.
    material_params: Material properties.
    Returns: Image tensor, gradients dict.
    """
    # Step 1 & 2: Transform vertices to screen space
    # v_cam = M_wc @ v_world
    # v_screen_ndc = M_proj @ v_cam
    # v_screen_uv = perspective_divide(v_screen_ndc)
    transformed_vertices = []
    for v in vertices:
        v_cam = camera_matrix @ v.pos
        v_ndc = projection_matrix @ v_cam
        # Perspective division
        v_uv = v_ndc[:3] / v_ndc[3]
        transformed_vertices.append({
            'uv': v_uv,
            'z_cam': v_cam[2], # Depth in camera space
            'color': v.color,
            'normal': v.normal
        })

    # Step 3-6: Rasterize and Shade (per pixel)
    image = zeros_like(screen_resolution)
    gradients = {} # Accumulate gradients

    for each pixel (u, v) in screen:
        # Find covering triangle(s) - Discrete step, gradient ignored here
        triangle = find_triangle_at_pixel(u, v, transformed_vertices)
        if triangle is None:
            continue

        # Step 3: Compute barycentric weights
        # Solve linear system: [v1_uv, v2_uv, v3_uv] * w = [pixel_uv, 1]
        # w is differentiable w.r.t vertex positions
        w = solve_barycentric(triangle.vertices_uv, (u, v))

        # Step 4: Interpolate depth
        z_p = sum(w[i] * triangle.vertices[i]['z_cam'] for i in range(3))

        # Step 5: Interpolate attributes
        c_p = sum(w[i] * triangle.vertices[i]['color'] for i in range(3))
        n_p_raw = sum(w[i] * triangle.vertices[i]['normal'] for i in range(3))
        n_p = n_p_raw / norm(n_p_raw) # Differentiable normalization

        # Step 6: Compute color using differentiable shading model
        # e.g., Phong or Lambertian
        color_p = shade(c_p, n_p, lights, material_params)

        image[u, v] = color_p

        # Step 7: Compute analytical gradients
        # d_color/d_params = d_color/d_c_p * d_c_p/d_w * d_w/d_v_pos + ...
        # This is computed via backpropagation through the differentiable ops above
        grad_c_p = compute_gradient_of_shading(color_p)
        grad_w = grad_c_p * (d_c_p/d_w) # Chain rule for color interpolation
        grad_v_pos = grad_w * (d_w/d_v_pos) # Chain rule for barycentric coords
        # Accumulate gradients for vertices, camera, lights, etc.
        accumulate_gradients(gradients, grad_v_pos, ...)

    return image, gradients
```

**Edge Cases and Pitfalls:**
*   **Zero Gradient for Occluded Triangles:** In standard rendering, exactly one triangle per pixel is selected. If a pixel's color is determined solely by a single triangle's vertex colors, the gradient of the pixel color with respect to vertices of *other* triangles is zero, preventing geometry optimization for occluded or background triangles [S1, unofficial].
*   **Zero Gradient for Vertex Position:** The derivative of pixel color with respect to vertex position is always zero for all pixels and vertices in the analytical formulation if the triangle selection remains constant. This means it does not help optimize geometry when vertex movement changes which triangle is selected or affects neighboring pixels [S1, unofficial].

## Subtopics

### [DIB-R](6798-dib-r.md)
DIB-R is a differentiable rendering technique for meshes that computes exact gradients of the output image with respect to input parameters by analytically differentiating the rendering pipeline, specifically addressing the non-differentiability of standard rasterization. It sets itself apart from siblings like [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md) by explicitly separating the image into foreground and background regions to handle gradients differently, thereby avoiding the blurriness associated with soft rasterization while mitigating vanishing gradients in background areas.

### [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md)
Neural 3D Mesh Renderer (NMR) is a differentiable rendering method that approximates the non-differentiable rasterization step by replacing sudden visibility changes with linear interpolation, allowing gradients to flow from pixels to mesh vertices beyond the immediate boundary. It addresses the "localness" limitation of prior methods like OpenDR, where gradients only flowed from boundary pixels, by using non-local approximated gradients that are aware of the optimization objective.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and triangles, similar to standard rasterization, plus the cost of solving linear programs for barycentric coordinates and backpropagating through the shading model.
*   **Memory Complexity:** Linear with respect to the number of vertices and pixels to store attributes and gradients.
*   **Strengths:** Provides exact gradients for the differentiable parts of the pipeline (projection, interpolation, shading), enabling precise optimization of shape, camera, and material parameters where the visibility topology is fixed.
*   **Limitations:** Fails to provide gradients for changes in visibility (topology changes), leading to zero gradients for occluded geometry and preventing global geometry optimization without additional approximations [S1, unofficial].
*   **When to use:** When the mesh topology is known and fixed, and optimization focuses on fine-tuning vertex positions, materials, or camera parameters within a stable visibility configuration.
*   **When not to use:** When optimizing coarse geometry where visibility changes frequently, or when occluded geometry needs to be optimized.

## Background
Analytical Derivative of Rendering (Mesh) originates from the broader field of differentiable rendering, which seeks to compute gradients of rendered images with respect to scene parameters. The specific approach of analytically differentiating the rasterization pipeline steps (excluding triangle selection) was detailed in surveys of differentiable rendering techniques [S1, unofficial]. It builds on the principle that while triangle selection is discrete and non-differentiable, the subsequent interpolation and shading steps are continuous and differentiable, allowing for exact gradient computation for those components [S1, unofficial].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
