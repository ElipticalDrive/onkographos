# Analytical Derivative of Rendering (Mesh)
Also called: Analytical Differentiable Mesh Rendering

## Overview
Analytical Derivative of Rendering (Mesh) computes exact gradients of the output image with respect to input parameters (shape, camera, material, lighting) by analytically differentiating the continuous steps of the rendering pipeline, excluding the discrete triangle selection step [S1, unofficial]. It operates on explicit triangle meshes rather than volumetric representations, distinguishing it from its volume rendering siblings, and relies on exact derivatives for continuous operations rather than approximating the rasterization visibility step [G].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Mesh geometry (vertex positions $\mathbf{v}_i$), material properties (vertex colors $\mathbf{c}_i$, normals $\mathbf{n}_i$), camera parameters (view/projection matrices), and lighting parameters [S1, unofficial].
*   **Outputs:** Rendered image color $\mathbf{C}$ and its analytical gradients $\frac{\partial \mathbf{C}}{\partial \mathbf{v}_i}, \frac{\partial \mathbf{C}}{\partial \mathbf{c}_i}, \dots$ with respect to input parameters [S1, unofficial].

**Data Structures:**
*   **Triangle:** Defined by three vertices $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$, where each vertex stores position $\mathbf{p} \in \mathbb{R}^3$, color $\mathbf{c} \in \mathbb{R}^3$, and normal $\mathbf{n} \in \mathbb{R}^3$ [S1, unofficial].
*   **Pixel:** Defined by screen coordinates $(u, v)$, depth $z$, and barycentric weights $(w_1, w_2, w_3)$ [S1, unofficial].

**Steps:**
1.  **Projection:** Project mesh triangles from world space to camera space using matrix products, then transform triangle positions from camera space to screen space using matrix products [S1, unofficial].
    $$ \mathbf{p}_{cam} = \mathbf{V} \mathbf{p}_{world}, \quad \mathbf{p}_{screen} = \mathbf{P} \mathbf{p}_{cam} $$
2.  **Barycentric Coordinate Computation:** Compute barycentric weights $w_i$ by solving differentiable linear programs [S1, unofficial].
    $$ \mathbf{p}_{pixel} = \sum_{i=1}^3 w_i \mathbf{p}_{i, screen}, \quad \sum_{i=1}^3 w_i = 1 $$
3.  **Depth Interpolation:** Compute pixel depth $z$ by interpolating the depth of the three vertices in camera space using the barycentric weights [S1, unofficial].
    $$ z = \sum_{i=1}^3 w_i z_i $$
4.  **Attribute Interpolation:** Express material and normal vectors at the pixel as a weighted sum of the material and normal vectors defined at the triangle's vertices [S1, unofficial].
    $$ \mathbf{c}_{pixel} = \sum_{i=1}^3 w_i \mathbf{c}_i, \quad \mathbf{n}_{pixel} = \sum_{i=1}^3 w_i \mathbf{n}_i $$
5.  **Shading:** Compute pixel color using a differentiable local illumination reflection model (e.g., Phong, Lambertian, Spherical Harmonics) given the interpolated material, lighting, and normal vectors [S1, unofficial].
    $$ \mathbf{C}_{pixel} = \text{Shading}(\mathbf{c}_{pixel}, \mathbf{n}_{pixel}, \mathbf{L}, \mathbf{M}) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def barycentric_weights(p0, p1, p2, px, py):
    """Computes barycentric coordinates for point (px, py) relative to triangle vertices."""
    denom = (p1[1] - p2[1]) * (p0[0] - p2[0]) + (p2[0] - p1[0]) * (p0[1] - p2[1])
    if abs(denom) < 1e-9:
        return 0.0, 0.0, 0.0
    w0 = ((p1[1] - p2[1]) * (px - p2[0]) + (p2[0] - p1[0]) * (py - p2[1])) / denom
    w1 = ((p2[1] - p0[1]) * (px - p0[0]) + (p0[0] - p2[0]) * (py - p0[1])) / denom
    w2 = 1.0 - w0 - w1
    return w0, w1, w2

def shade(color, normal, light_dir, light_color, ambient, diffuse, specular):
    """Differentiable Phong shading model."""
    diff = max(0.0, sum(n * l for n, l in zip(normal, light_dir)))
    spec = pow(max(0.0, sum(n * v for n, v in zip(normal, light_dir))), 32.0)
    return [c * (ambient + diffuse * diff + specular * spec) for c in color]

def analytical_gradient_mesh_rendering(vertices, faces, camera_matrix, light_params):
    """
    Compute rendered image and analytical gradients for a mesh.
    vertices: list of (x, y, z)
    faces: list of (i0, i1, i2)
    camera_matrix: 4x4 list of lists
    light_params: dict with light_dir, light_color, ambient, diffuse, specular
    Returns: rendered_image, grad_vertices
    """
    # 1. Project vertices to screen space
    verts_screen = []
    for v in vertices:
        v_h = v + (1.0,)
        v_cam = [sum(c * m for c, m in zip(v_h, row)) for row in camera_matrix]
        if v_cam[3] == 0.0:
            verts_screen.append((0.0, 0.0, 0.0))
        else:
            verts_screen.append((v_cam[0]/v_cam[3], v_cam[1]/v_cam[3], v_cam[2]/v_cam[3]))
            
    # 2. Initialize image and gradient accumulators
    H, W = 512, 512
    rendered_image = [[0.0, 0.0, 0.0] for _ in range(H * W)]
    grad_vertices = [[0.0, 0.0, 0.0] for _ in range(len(vertices))]
    
    # 3. Rasterize and shade each triangle
    for face in faces:
        v0, v1, v2 = vertices[face[0]], vertices[face[1]], vertices[face[2]]
        p0, p1, p2 = verts_screen[face[0]], verts_screen[face[1]], verts_screen[face[2]]
        
        # Compute bounding box
        min_x, max_x = int(min(p0[0], p1[0], p2[0])), int(max(p0[0], p1[0], p2[0]))
        min_y, max_y = int(min(p0[1], p1[1], p2[1])), int(max(p0[1], p1[1], p2[1]))
        
        for y in range(min_y, max_y + 1):
            for x in range(min_x, max_x + 1):
                w0, w1, w2 = barycentric_weights(p0, p1, p2, float(x), float(y))
                if w0 < 0 or w1 < 0 or w2 < 0:
                    continue
                    
                # Interpolate attributes
                c_pixel = [w0 * v0[i] + w1 * v1[i] + w2 * v2[i] for i in range(3)]
                n_pixel = [w0 * v0[i+3] + w1 * v1[i+3] + w2 * v2[i+3] for i in range(3)]
                
                # Shading
                color = shade(c_pixel, n_pixel, light_params['light_dir'], light_params['light_color'],
                              light_params['ambient'], light_params['diffuse'], light_params['specular'])
                
                # Accumulate color
                idx = y * W + x
                rendered_image[idx] = color
                
                # Accumulate gradients (chain rule over barycentric interpolation)
                for i, v_idx in enumerate(face):
                    grad_vertices[v_idx] = [grad_vertices[v_idx][j] + color[j] * w0 if i == 0 else 
                                            grad_vertices[v_idx][j] + color[j] * w1 if i == 1 else 
                                            grad_vertices[v_idx][j] + color[j] * w2 
                                            for j in range(3)]
                    
    return rendered_image, grad_vertices
```

**Edge Cases and Pitfalls:**
*   **Zero Gradient for Non-Selected Triangles:** In standard rendering, exactly one triangle per pixel is selected. If a pixel's color is determined solely by a single triangle's vertex colors, the gradient of the pixel color with respect to vertices of *other* triangles is zero, preventing geometry optimization for occluded or background triangles [S1, unofficial].
*   **Zero Gradient for Vertex Position:** The derivative of pixel color with respect to vertex position is always zero for all pixels and vertices in the analytical formulation if the triangle selection is fixed, meaning it does not help optimize geometry when vertex movement changes which triangle is selected or affects neighboring pixels [S1, unofficial].

## Subtopics

### [DIB-R](6798-dib-r.md)
DIB-R is a differentiable renderer that addresses the non-differentiability of standard rasterization by partitioning pixels into foreground (covered by mesh faces) and background (uncovered) regions [G]. It computes exact analytical gradients for foreground pixels using barycentric interpolation and prevents vanishing gradients for background pixels via distance-based aggregation [G]. Unlike its sibling [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md), which relies on soft rasterization approximations for all pixels, DIB-R maintains sharp foreground edges while providing a differentiable path for background pixels [G]. The method serves as the geometric differentiability foundation for DIB-R++, a hybrid renderer that extends this framework to support photorealistic shading, environmental lighting, and inverse graphics without ground-truth 3D supervision [G].

### [Neural 3D Mesh Renderer (NMR)](7042-neural-3d-mesh-renderer-nmr.md)
Neural 3D Mesh Renderer (NMR) is a differentiable rendering method for 3D polygon meshes that enables end-to-end training via back-propagation by approximating the non-differentiable rasterization step [G]. It solves the discontinuity problem of standard rasterization by replacing sudden visibility changes with linear interpolations, allowing gradients to flow from pixel colors to geometry vertices, textures, and lighting parameters [G]. Unlike local gradient approximations such as OpenDR, NMR provides non-local gradients that allow pixels beyond the boundary to influence vertices, avoiding poor local minima [G]. It is distinguished from siblings like SoftRas by retaining standard z-culling, which saves computation but prevents gradient flow to occluded triangles [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides exact gradients for continuous operations (projection, interpolation, shading) without storing the full computational graph [S1, unofficial].
*   **Limitations:** Fails to provide gradients for the discrete triangle selection step, leading to zero gradients for occluded triangles and no gradient flow for geometry changes that alter visibility [S1, unofficial].
*   **When to Use:** When the mesh topology is fixed and the optimization focuses on parameters that do not change visibility (e.g., texture colors, lighting, camera pose) or when combined with methods that approximate the rasterization step [S1, unofficial].
*   **When Not to Use:** When optimizing mesh geometry where vertex movements significantly change visibility or occlusion relationships, as the analytical derivatives alone are insufficient [S1, unofficial].

## Background
Analytical Derivative of Rendering (Mesh) originates from the broader field of differentiable rendering, which seeks to make the rendering pipeline differentiable to enable gradient-based optimization of 3D scenes [S1, unofficial]. The specific application to meshes leverages the differentiability of standard graphics operations (matrix transformations, barycentric interpolation, shading models) while acknowledging the non-differentiability of the rasterization visibility step [S1, unofficial]. Not given in the sources.

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
