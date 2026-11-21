# Triangle Mesh Splatting
Also called: Laur and Hanrahan splatting
## Overview
Triangle Mesh Splatting is an object-order rendering technique that approximates volumetric splats with triangle meshes to leverage graphics hardware for rapid scan conversion [S1, unofficial]. It distinguishes itself from sibling methods like Circle Splat Rendering by replacing explicit footprint functions with rasterized geometry, enabling efficient hardware-accelerated rendering of complex splat shapes [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A set of splats, each defined by a center position, orientation, and shape parameters (e.g., covariance matrix for elliptical splats).
- Output: A 2D image buffer with accumulated color and opacity.

**Data Structures:**
- Splat Representation: Each splat is approximated by a triangle mesh. The mesh geometry is derived from the splat's footprint in screen space.
- Triangle Mesh: A list of vertices and indices defining the triangles. The vertices are positioned to cover the splat's footprint.

**Steps:**
1. **Transformation:** Transform the splat center and orientation from object space to screen space using the Model-View-Projection (MVP) matrix.
2. **Footprint Approximation:** Approximate the splat's footprint (e.g., an ellipse) with a triangle mesh. The mesh should cover the footprint area. For an elliptical splat, this might involve generating a set of triangles that approximate the ellipse shape.
3. **Rasterization:** Use graphics hardware to rasterize the triangle mesh. This step converts the geometric representation into a set of fragments (pixels) covered by the mesh.
4. **Fragment Processing:** For each fragment, compute the splat's contribution (color and opacity) based on the fragment's position relative to the splat center. This often involves evaluating the splat kernel function (e.g., Gaussian) at the fragment's screen-space coordinates.
5. **Compositing:** Accumulate the fragment's color and opacity into the image buffer using alpha compositing. The order of compositing should follow the depth order of the splats to ensure correct blending.

**Formulas:**
- Screen-space position: $\mathbf{p}_{screen} = P \cdot V \cdot M \cdot \mathbf{p}_{object}$
- Splat kernel value: $K(\mathbf{u}) = \exp(-\frac{1}{2} \mathbf{u}^T \Sigma^{-1} \mathbf{u})$, where $\mathbf{u}$ is the vector from the splat center to the fragment, and $\Sigma$ is the covariance matrix.
- Color contribution: $C_{frag} = C_{splat} \cdot K(\mathbf{u}) \cdot \alpha_{splat}$

**Parameters:**
- Triangle mesh resolution: The number of triangles used to approximate the footprint. Higher resolution yields better accuracy but increases rendering cost.
- Splat kernel width: Controls the spread of the splat.

**Edge Cases and Pitfalls:**
- Precision: Floating-point precision issues may arise when transforming coordinates, especially for large scenes.
- Boundaries: Ensure that the triangle mesh fully covers the splat's footprint to avoid artifacts.
- Degenerate Input: Handle splats with zero size or invalid orientations.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def triangle_mesh_splatting(splats, image_buffer, mvp_matrix):
    for splat in splats:
        # Transform splat to screen space
        p_screen = mvp_matrix @ splat.center
        
        # Approximate footprint with triangle mesh
        triangles = approximate_footprint(splat)
        
        # Rasterize triangles
        fragments = rasterize(triangles, image_buffer.resolution)
        
        # Process fragments
        for frag in fragments:
            # Compute distance from splat center
            u = frag.position - p_screen
            
            # Evaluate kernel
            k = evaluate_kernel(u, splat.covariance)
            
            # Compute color contribution
            c_frag = splat.color * k * splat.opacity
            
            # Composite into image buffer
            image_buffer.composite(frag.position, c_frag, k * splat.opacity)

def approximate_footprint(splat):
    # Generate triangles to approximate the splat's footprint
    # This is a placeholder for the actual mesh generation logic
    return []

def evaluate_kernel(u, covariance):
    # Evaluate Gaussian kernel
    return exp(-0.5 * u.T @ inv(covariance) @ u)
```
## Subtopics

### [Differentiable Triangle Splatting](8761-differentiable-triangle-splatting.md)
Differentiable Triangle Splatting is a real-time differentiable renderer that represents a scene as a soup of unstructured, disconnected 3D triangles and optimizes them via end-to-end gradients by rendering each triangle as a differentiable splat in screen space. It merges the adaptability of Gaussian splatting with the efficiency of triangle primitives, surpassing 3D Gaussian Splatting (3DGS), 2D Gaussian Splatting (2DGS), and 3D Convex Splatting (3DCS) in visual fidelity, training speed, and rendering throughput.

## Operating characteristics
- **Time Complexity:** Depends on the number of splats and the resolution of the triangle mesh approximation. Rasterization is generally efficient on modern GPUs.
- **Memory Complexity:** Proportional to the number of splats and the size of the triangle meshes.
- **Performance:** Fast rendering due to hardware acceleration. Performance scales well with the number of splats.
- **Strengths:** Efficient use of graphics hardware, flexible footprint shapes.
- **Limitations:** Approximation errors from triangle mesh representation, potential artifacts at low mesh resolutions.
- **When to Use:** When real-time rendering of complex splat shapes is required and hardware acceleration is available.
- **When Not to Use:** When high precision is required and approximation errors are unacceptable.
## Background
Triangle Mesh Splatting was introduced by Laur and Hanrahan as a method to improve rendering speed by approximating splats with triangle meshes and using graphics hardware for scan conversion [S1, unofficial]. It builds on earlier splatting techniques that used explicit footprint functions.
## Sources
[S1] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
