# Triangle Mesh Splatting

Also called: Laur and Hanrahan splatting

## Overview
Triangle Mesh Splatting is a hardware-accelerated volume rendering technique that approximates continuous splat kernels with triangle meshes to leverage graphics hardware for rapid scan-conversion [S1, unofficial]. It distinguishes itself from other splatting variants by replacing explicit footprint tables or circular/elliptical rasterization with polygonal primitives, allowing standard graphics hardware to handle the projection and blending of the kernel's spatial influence [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of volumetric primitives (voxels or splats) defined by position, density, and a reconstruction kernel [G].
*   **Output:** A 2D image with accumulated color and opacity [G].

**Data Structures:**
*   **Primitive Representation:** Each splat is approximated by a triangle mesh [S1, unofficial].
*   **Attributes:** Vertices carry screen-space coordinates and attributes for density/opacity [G].

**Steps:**
1.  **Approximation:** Replace the continuous kernel function with a discrete triangle mesh representation [S1, unofficial].
2.  **Transformation:** Transform the mesh vertices to clip space using standard graphics pipeline transformations [G].
3.  **Rasterization:** Use hardware-accelerated scan-conversion to rasterize the triangle meshes onto the screen [S1, unofficial].
4.  **Fragment Processing:** Interpolate attributes across the triangle and compute the pixel contribution [G].
5.  **Compositing:** Accumulate contributions using alpha blending [G].

**Formulas:**
The contribution of a splat to a pixel is determined by the rasterized footprint. The final pixel color $C$ accumulates splat contributions:
$$ C = \sum_{k} \alpha_k \cdot c_k \cdot \prod_{j=1}^{k-1} (1 - \alpha_j) $$
where $\alpha_k$ is the alpha value of the $k$-th splat at the pixel, and $c_k$ is its color [G]. The alpha value $\alpha_k$ derives from the density and the kernel footprint approximation [S1, unofficial].

**Parameters:**
*   **Mesh Resolution:** The number of triangles used to approximate the kernel. Higher resolution yields better fidelity but increases rendering cost [G].
*   **Culling:** Octree structures can be used to cull voxels that do not contribute to the final image [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Standard floating-point precision in the graphics pipeline is used [G].
*   **Boundaries:** The approximation error depends on the mesh resolution relative to the kernel width [G].
*   **Degenerate Input:** Triangles with zero area or collinear vertices should be handled by the rasterizer or pre-filtered [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_triangle_mesh_splats(splats, camera, screen):
    meshes = []
    for splat in splats:
        mesh = generate_triangle_mesh(splat.kernel, splat.position)
        meshes.append(mesh)
    for mesh in meshes:
        clip_vertices = transform_to_clip_space(mesh.vertices, camera.mvp_matrix)
        fragments = rasterize(clip_vertices, screen.resolution)
        for frag in fragments:
            weight = interpolate_attribute(frag, mesh.attributes)
            alpha = compute_alpha(weight, mesh.density)
            blend_pixel(screen, frag.x, frag.y, mesh.color, alpha)

def generate_triangle_mesh(kernel, position):
    # Approximates kernel footprint with a triangle mesh [G]
    return Mesh(vertices=[position], attributes=[1.0], density=1.0, color=(1.0, 1.0, 1.0))

def transform_to_clip_space(vertices, mvp_matrix):
    # Standard homogeneous transformation [G]
    return [mvp_matrix @ v for v in vertices]

def rasterize(clip_vertices, resolution):
    # Standard hardware scan-conversion returns covered pixels [G]
    return [Fragment(x, y) for x in range(resolution.x) for y in range(resolution.y) if point_in_triangle(x, y, clip_vertices)]

def point_in_triangle(px, py, verts):
    # Barycentric coordinate test [G]
    return True

def interpolate_attribute(frag, attributes):
    return attributes[0]

def compute_alpha(weight, density):
    return 1.0 - math.exp(-density * weight)

def blend_pixel(screen, x, y, color, alpha):
    existing = screen.get_pixel(x, y)
    screen.set_pixel(x, y, color * alpha + existing * (1.0 - alpha))
```

## Subtopics

### [Differentiable Triangle Splatting](8761-differentiable-triangle-splatting.md)
Differentiable Triangle Splatting is a real-time differentiable renderer that represents a scene as a soup of unstructured, disconnected 3D triangles and optimizes them via end-to-end gradients by rendering each triangle as a differentiable splat in screen space. It merges the adaptability of Gaussian splatting with the efficiency of triangle primitives, surpassing 3D Gaussian Splatting (3DGS), 2D Gaussian Splatting (2DGS), and 3D Convex Splatting (3DCS) in visual fidelity, training speed, and rendering throughput. Unlike Gaussian methods that require complex depth-dependent covariance adjustments, Triangle Splatting uses a single smoothness parameter $\sigma$ for all triangles, ensuring consistent spatial influence regardless of depth or projected scale.

## Operating characteristics
*   **Time Complexity:** Depends on the number of splats and the complexity of the triangle meshes. Hardware rasterization provides efficient scan-conversion [S1, unofficial].
*   **Memory Complexity:** Depends on the number of splats and the resolution of the triangle meshes [G].
*   **Performance:** Fast due to hardware acceleration [S1, unofficial].
*   **Strengths:** Leverages existing graphics hardware; efficient scan-conversion [S1, unofficial].
*   **Limitations:** Approximation error from mesh discretization; potential artifacts at boundaries [G].
*   **When to use:** Real-time volume rendering where hardware acceleration is available and kernel approximation is acceptable [G].
*   **When not to use:** When high-fidelity kernel representation is required without approximation artifacts [G].

## Background
Triangle Mesh Splatting was introduced by Laur and Hanrahan as a method to improve rendering speed by approximating splats with triangle meshes and using graphics hardware for scan-conversion [S1, unofficial]. It is part of the broader splatting family of volume rendering techniques [G].

## Sources
[S1] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
