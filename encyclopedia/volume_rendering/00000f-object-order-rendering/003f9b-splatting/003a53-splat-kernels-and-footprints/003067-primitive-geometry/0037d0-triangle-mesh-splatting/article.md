# Triangle Mesh Splatting

Also called: Laur and Hanrahan splatting

## Overview
Triangle Mesh Splatting is a hardware-accelerated volume rendering technique that approximates continuous splat kernels with triangle meshes to leverage graphics hardware for rapid scan-conversion [S1, unofficial]. It distinguishes itself from other splatting variants by replacing explicit footprint tables or circular/elliptical rasterization with polygonal primitives, allowing the GPU's standard rasterizer to handle the projection and blending of the kernel's spatial influence [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of volumetric primitives (voxels or splats) defined by position, density, and a reconstruction kernel (typically Gaussian).
*   **Output:** A 2D image with accumulated color and opacity.

**Data Structures:**
*   **Primitive Representation:** Each splat is approximated by a triangle mesh. The specific topology of the mesh (e.g., a single quad split into two triangles, or a more complex polygon) depends on the desired approximation of the kernel's footprint.
*   **Attributes:** Vertices of the mesh carry screen-space coordinates and potentially texture coordinates or varying attributes for density/opacity if the kernel is non-uniform.

**Steps:**
1.  **Approximation:** Replace the continuous kernel function $h(\mathbf{x})$ with a discrete triangle mesh representation. The mesh vertices are positioned to approximate the shape and extent of the kernel's footprint in screen space [S1, unofficial].
2.  **Transformation:** Transform the vertices of the triangle mesh from world/volume space to clip space using the Model-View-Projection (MVP) matrix, consistent with standard graphics pipeline transformations [S1, unofficial].
3.  **Rasterization:** Use hardware-accelerated scan-conversion to rasterize the triangle meshes onto the screen. The GPU's rasterizer determines which pixels are covered by each triangle [S1, unofficial].
4.  **Fragment Processing:** For each covered pixel, compute the contribution of the splat. If the triangle mesh carries varying attributes (e.g., from a texture map representing the kernel profile), interpolate these values across the triangle. The intensity written to the pixel is modulated by the voxel density $\rho$ and the interpolated kernel weight [S1, unofficial].
5.  **Compositing:** Accumulate the contributions of all splats using alpha blending, typically in back-to-front order to ensure correct transparency handling [S1, unofficial].

**Formulas:**
The contribution of a splat to a pixel is determined by the rasterized footprint. If the triangle mesh approximates a kernel $h$, the weight $w$ at a pixel is derived from the rasterization process. The final pixel color $C$ is:
$$ C = \sum_{k} \alpha_k \cdot c_k \cdot \prod_{j=1}^{k-1} (1 - \alpha_j) $$
where $\alpha_k$ is the alpha value of the $k$-th splat at the pixel, and $c_k$ is its color. The alpha value $\alpha_k$ is derived from the density $\rho_k$ and the kernel footprint approximation [S1, unofficial].

**Parameters:**
*   **Mesh Resolution:** The number of triangles used to approximate the kernel. Higher resolution yields better fidelity but increases rendering cost. Not given in the sources.
*   **Culling:** Octree structures can be used to cull voxels that do not contribute to the final image [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Standard floating-point precision in the graphics pipeline is used.
*   **Boundaries:** The approximation error depends on the mesh resolution relative to the kernel width.
*   **Degenerate Input:** Triangles with zero area or collinear vertices should be handled by the rasterizer or pre-filtered.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Triangle Mesh Splatting
# Assumes a graphics pipeline is available

def render_triangle_mesh_splats(splats, camera, screen):
    # 1. Approximate each splat with a triangle mesh
    # This step is typically done during preprocessing or on the fly
    # depending on the implementation.
    meshes = []
    for splat in splats:
        # Approximate the kernel footprint with a triangle mesh
        # The exact method of approximation is not detailed in [S1, unofficial]
        mesh = approximate_kernel_with_mesh(splat.kernel, splat.position)
        meshes.append(mesh)

    # 2. Transform and Rasterize
    # Use hardware rasterization
    for mesh in meshes:
        # Transform vertices to clip space
        clip_vertices = transform_to_clip_space(mesh.vertices, camera.mvp_matrix)
        
        # Rasterize the mesh
        # The GPU handles scan-conversion
        fragments = rasterize(clip_vertices, screen.resolution)
        
        # 3. Fragment Processing
        for frag in fragments:
            # Interpolate attributes (e.g., density, color)
            # If the mesh carries varying attributes
            weight = interpolate_attribute(frag, mesh.attributes)
            alpha = compute_alpha(weight, splat.density)
            color = splat.color
            
            # 4. Compositing
            # Blend into the frame buffer
            blend_pixel(screen, frag.x, frag.y, color, alpha)

def approximate_kernel_with_mesh(kernel, position):
    # Not given in the sources.
    # This function would generate a triangle mesh that approximates
    # the shape of the kernel centered at position.
    pass

def interpolate_attribute(frag, attributes):
    # Standard barycentric interpolation
    pass

def compute_alpha(weight, density):
    # Alpha is typically 1 - exp(-density * weight)
    return 1.0 - math.exp(-density * weight)

def blend_pixel(screen, x, y, color, alpha):
    # Standard alpha blending
    existing_color = screen.get_pixel(x, y)
    existing_alpha = screen.get_alpha(x, y)
    
    new_color = color * alpha + existing_color * (1.0 - alpha)
    new_alpha = alpha + existing_alpha * (1.0 - alpha)
    
    screen.set_pixel(x, y, new_color, new_alpha)
```

## Subtopics

### [Differentiable Triangle Splatting](8761-differentiable-triangle-splatting.md)
Differentiable Triangle Splatting is a real-time differentiable renderer that represents a scene as a soup of unstructured, disconnected 3D triangles and optimizes them via end-to-end gradients by rendering each triangle as a differentiable splat in screen space. It merges the adaptability of Gaussian splatting with the efficiency of triangle primitives, surpassing 3D Gaussian Splatting (3DGS), 2D Gaussian Splatting (2DGS), and 3D Convex Splatting (3DCS) in visual fidelity, training speed, and rendering throughput. Unlike Gaussian methods that require complex depth-dependent covariance adjustments, Triangle Splatting uses a single smoothness parameter $\sigma$ for all triangles, ensuring consistent spatial influence regardless of depth or projected scale.

## Operating characteristics
*   **Time Complexity:** Depends on the number of splats and the complexity of the triangle meshes. Hardware rasterization provides efficient scan-conversion.
*   **Memory Complexity:** Depends on the number of splats and the resolution of the triangle meshes.
*   **Performance:** Fast due to hardware acceleration.
*   **Strengths:** Leverages existing graphics hardware; efficient scan-conversion.
*   **Limitations:** Approximation error from mesh discretization; potential artifacts at boundaries.
*   **When to use:** Real-time volume rendering where hardware acceleration is available and kernel approximation is acceptable.
*   **When not to use:** When high-fidelity kernel representation is required without approximation artifacts.

## Background
Triangle Mesh Splatting was introduced by Laur and Hanrahan as a method to improve rendering speed by approximating splats with triangle meshes and using graphics hardware for scan-conversion [S1, unofficial]. It is part of the broader splatting family of volume rendering techniques.

## Sources
[S1] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
