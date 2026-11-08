# Triangle Mesh Splatting

Also called: Laur and Hanrahan splatting

## Overview
Triangle Mesh Splatting is a volume rendering technique that approximates continuous splat kernels with discrete triangle meshes to leverage graphics hardware for rapid rasterization [S1, unofficial]. It distinguishes itself from other primitive geometry methods by replacing analytical footprint evaluation with hardware-accelerated scan conversion, trading geometric exactness for significant rendering speed improvements [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A set of volumetric data elements (e.g., voxels) mapped to 3D geometric primitives, typically represented as triangle meshes.
- **Output:** A 2D screen-space image with accumulated color and opacity.

**Data Structures:**
- **Primitive Representation:** Each splat is defined as a triangle mesh. The specific topology (e.g., number of triangles, vertex layout) is not detailed in the sources [G].
- **Transformation Matrices:** Standard Model-View-Projection (MVP) matrices are used to transform vertices from volume space to clip space [S1, unofficial].

**Steps:**
1. **Primitive Approximation:** Each volumetric element is approximated by a triangle mesh that encloses or represents the splat kernel's footprint [S1, unofficial].
2. **Transformation:** The vertices of the triangle mesh are transformed using the MVP matrix to project them into screen space [S1, unofficial].
3. **Rasterization:** The transformed triangle meshes are scan-converted (rasterized) by the graphics hardware [S1, unofficial].
4. **Fragment Processing:** For each fragment covered by a triangle, the fragment shader evaluates the splat kernel intensity. The intensity is typically based on the fragment's distance from the primitive's center or axis, consistent with the parent [Primitive Geometry](6264-primitive-geometry.md) principles [S1, unofficial].
5. **Blending:** The computed color and opacity contributions are accumulated into the frame buffer using alpha blending [S1, unofficial].

**Formulas:**
- The screen-space footprint $F(x,y)$ is evaluated implicitly through rasterization rather than explicit integration [S1, unofficial].
- Kernel intensity $h(r)$ is evaluated per fragment based on distance $r$ [S1, unofficial].

**Parameters:**
- **Mesh Resolution:** The number of triangles used to approximate the splat. Not given in the sources.
- **Splat Radius:** The effective radius of the splat, influencing the mesh size. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Rasterization artifacts may occur at boundaries if the mesh does not perfectly align with pixel centers [G].
- **Degenerate Input:** Overlapping triangles or zero-area triangles should be handled by the rasterizer [G].

## Subtopics

### [Differentiable Triangle Splatting](8761-differentiable-triangle-splatting.md)
A real-time differentiable renderer that represents scenes as unstructured 3D triangles, optimizing via end-to-end gradients. It uses a single smoothness parameter $\sigma$ for consistent spatial influence, surpassing Gaussian methods in fidelity and speed.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of triangles and fragments covered. Hardware rasterization provides constant-time per-fragment evaluation [S1, unofficial].
- **Memory Complexity:** Proportional to the number of triangles in the mesh representation [S1, unofficial].
- **Performance:** Significantly faster than analytical footprint evaluation due to hardware acceleration [S1, unofficial].
- **Strengths:** High rendering speed, compatibility with standard graphics pipelines [S1, unofficial].
- **Limitations:** Approximation error due to mesh discretization; less flexible than analytical kernels for complex footprint shapes [S1, unofficial].
- **When to Use:** Real-time rendering where speed is critical and geometric approximation is acceptable [S1, unofficial].
- **When Not to Use:** Applications requiring high geometric fidelity or complex, non-convex footprints [S1, unofficial].

## Background
Originated by Laur and Hanrahan, this method was proposed to improve rendering speed by mapping view-dependent footprints to triangle meshes for hardware scan conversion [S1, unofficial]. It is part of the broader splatting family, which includes Westover's circular/elliptical footprint tables and Crawfis and Max's texture mapping approaches [S1, unofficial].

## Sources
[S1] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
