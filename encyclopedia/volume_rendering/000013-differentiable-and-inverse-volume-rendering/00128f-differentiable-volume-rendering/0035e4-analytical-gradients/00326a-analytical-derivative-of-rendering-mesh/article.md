# Analytical Derivative of Rendering (Mesh)

Also called: Analytical Differentiable Mesh Rendering

## Overview
Analytical Derivative of Rendering (Mesh) is a differentiable rendering technique that computes exact gradients of the output image with respect to input parameters (shape, camera, material, lighting) by analytically differentiating the continuous steps of the rendering pipeline, excluding the discrete triangle selection step [S1, unofficial]. It sets itself apart from siblings like [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md) and [Differentiable Volume Rendering with Analytical Gradients](5983-differentiable-volume-rendering-with-analytical-gradients.md) by operating on explicit polygonal meshes rather than volumetric fields, relying on barycentric interpolation and standard rasterization logic [S1, unofficial].

## How it works
The method computes gradients for the continuous portion of the rendering pipeline. The discrete triangle selection is excluded from the analytical differentiation [S1, unofficial].

**Inputs:**
- Mesh vertices $V = \{v_1, \dots, v_N\}$ in world space.
- Camera extrinsic matrix $E$ and intrinsic matrix $K$.
- Lighting parameters and material properties defined per vertex or triangle.

**Outputs:**
- Gradient of pixel color $C$ with respect to vertex positions, camera parameters, and material/lighting parameters.

**Data Structures:**
- Triangle defined by indices $(i, j, k)$ mapping to vertices $v_i, v_j, v_k$.
- Barycentric coordinates $(w_1, w_2, w_3)$ for a pixel $p$.
- Vertex attributes: position $v$, normal $n$, color $c$.

**Steps:**

1.  **World to Camera Space:**
    Transform vertex positions $v$ from world space to camera space using the extrinsic matrix $E$:
    $$ v_{cam} = E \cdot v_{world} $$
    This operation is differentiable via matrix multiplication [S1, unofficial].

2.  **Camera to Screen Space:**
    Project $v_{cam}$ to screen space using the intrinsic matrix $K$:
    $$ v_{screen} = K \cdot v_{cam} $$
    This is also differentiable via matrix multiplication [S1, unofficial].

3.  **Triangle Selection (Non-Differentiable):**
    For each pixel, select exactly one triangle that covers it. This step is discrete and not differentiable with respect to all parameters [S1, unofficial].

4.  **Barycentric Coordinate Computation:**
    Compute barycentric weights $w_1, w_2, w_3$ for the pixel $p$ relative to the selected triangle's vertices. This is solved as a differentiable linear program [S1, unofficial]. The weights satisfy:
    $$ p = w_1 v_1 + w_2 v_2 + w_3 v_3 $$
    $$ w_1 + w_2 + w_3 = 1 $$
    where $v_1, v_2, v_3$ are the screen-space positions of the triangle vertices [S1, unofficial].

5.  **Depth Interpolation:**
    Compute pixel depth $z_p$ by interpolating the depth of the three vertices in camera space:
    $$ z_p = w_1 z_1 + w_2 z_2 + w_3 z_3 $$
    where $z_i$ is the camera-space depth of vertex $i$ [S1, unofficial].

6.  **Attribute Interpolation:**
    Express material and normal vectors at the pixel as a weighted sum of the vertex attributes:
    $$ n_p = w_1 n_1 + w_2 n_2 + w_3 n_3 $$
    $$ c_p = w_1 c_1 + w_2 c_2 + w_3 c_3 $$
    where $n_i$ and $c_i$ are the normal and color at vertex $i$ [S1, unofficial].

7.  **Color Computation:**
    Compute pixel color using a differentiable local illumination reflection model (e.g., Phong, Lambertian, Spherical Harmonics) given the interpolated normal $n_p$, material, and lighting [S1, unofficial]. Popular models like Phong [47], Lambertian [48], and Spherical Harmonics [49] are differentiable [S1, unofficial].

8.  **Gradient Calculation:**
    Compute the derivatives of pixel depth and color with respect to input parameters analytically [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def analytical_gradient_mesh_rendering(vertices_world, camera_extrinsic, camera_intrinsic, 
                                       vertex_normals, vertex_colors, lighting, material_params):
    # Step 1: World to Camera Space
    vertices_cam = apply_matrix(camera_extrinsic, vertices_world)
    
    # Step 2: Camera to Screen Space
    vertices_screen = apply_matrix(camera_intrinsic, vertices_cam)
    
    # Step 3: Triangle Selection (Discrete, Non-Differentiable)
    # For each pixel, determine the covering triangle. 
    # This step is skipped for gradient computation regarding triangle selection.
    triangles = get_covering_triangles(vertices_screen, image_resolution)
    
    gradients = {}
    
    for pixel, triangle in triangles.items():
        v1, v2, v3 = vertices_screen[triangle]
        
        # Step 4: Barycentric Coordinates (Differentiable Linear Program)
        w1, w2, w3 = solve_barycentric(pixel, v1, v2, v3)
        
        # Step 5: Depth Interpolation
        z1, z2, z3 = vertices_cam[triangle].z
        z_pixel = w1 * z1 + w2 * z2 + w3 * z3
        
        # Step 6: Attribute Interpolation
        n1, n2, n3 = vertex_normals[triangle]
        n_pixel = w1 * n1 + w2 * n2 + w3 * n3
        
        c1, c2, c3 = vertex_colors[triangle]
        c_pixel = w1 * c1 + w2 * c2 + w3 * c3
        
        # Step 7: Color Computation (Differentiable Model)
        # Example: Lambertian
        light_dir = normalize(lighting.direction)
        diffuse = max(0, dot(n_pixel, light_dir))
        color_pixel = c_pixel * diffuse * lighting.intensity
        
        # Step 8: Analytical Derivatives
        # Compute d(color_pixel)/d(vertices_world), d(color_pixel)/d(camera_params), etc.
        # using chain rule through w, z, n, c.
        # Note: Gradients w.r.t. vertices of non-selected triangles are zero.
        pass
        
    return gradients
```

**Edge Cases and Pitfalls:**
- **Zero Gradient for Non-Selected Triangles:** If a pixel is covered by only one triangle, the gradient of the pixel color with respect to the vertices of *other* triangles is zero [S1, unofficial].
- **Geometry Optimization Failure:** The derivative of pixel color with respect to vertex position is zero for all pixels and vertices in the analytical formulation if the triangle selection does not change. This prevents geometry optimization when vertex movement changes which triangle is selected or affects neighboring pixels [S1, unofficial].

## Subtopics

### [DIB-R](6798-dib-r.md)
A differentiable renderer that focuses independently on foreground pixels (covered by faces) and background pixels (uncovered) to avoid blurriness and vanishing gradients [6798-dib-r].

### [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md)
A differentiable rendering method that approximates the non-differentiable rasterization step by replacing sudden changes with linear interpolation changes, allowing gradients to flow to vertices from pixels beyond the boundary [7042-neural-3d-mesh-renderer-nmr].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels and triangles for the differentiable steps. The triangle selection step is typically $O(P \cdot T)$ or optimized with spatial data structures, but is non-differentiable.
- **Memory Complexity:** Linear with respect to the number of vertices and pixels for storing attributes and gradients.
- **Strengths:** Provides exact gradients for the continuous parts of the pipeline (projection, interpolation, shading). Computationally efficient as it reuses standard rasterization logic.
- **Limitations:** Fails to provide gradients for geometry optimization when vertex movement changes the set of covered triangles (the "zero gradient" problem). Cannot optimize occluded or background triangles [S1, unofficial].
- **When to Use:** When optimizing camera parameters, material properties, or lighting where the mesh topology and coverage remain stable.
- **When Not to Use:** When optimizing mesh geometry where vertex positions significantly alter the silhouette or occlusion relationships, as gradients will be zero for non-selected triangles [S1, unofficial].

## Background
The method is described in "Differentiable Rendering: A Survey" [S1, unofficial]. It builds on the observation that standard rendering operations (projection, interpolation, shading) are differentiable, while the discrete triangle selection is not [S1, unofficial]. The survey cites earlier works on analytical derivatives [16, 17, 18] and approximated gradients [19, 9, 20, 14, 22, 23, 21] that address the limitations of the pure analytical approach.

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
