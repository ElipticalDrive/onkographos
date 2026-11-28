# Rasterization and Hardware Primitives


## Overview
This family renders isosurfaces by projecting geometric primitives (tetrahedra, voxels, or points) onto the screen and using fixed-function or early programmable graphics hardware features (alpha testing, stencil buffers, z-testing) to determine surface visibility and shading, rather than explicit ray-marching or mesh generation [S1, unofficial][S2, unofficial]. It differs from sibling methods like [Volumetric Isosurface Rendering with Intersection Refinement](958-volumetric-isosurface-rendering-with-intersection-refinement.md) by avoiding per-pixel ray traversal in favor of hardware-accelerated primitive rasterization, and from [Multiple Flat Shaded Opaque Isosurfaces](1498-multiple-flat-shaded-opaque-isosurfaces.md) by potentially using stencil logic for visibility rather than texture-encoded ordering [G].

## Shared principles
All methods in this family project volume elements or point primitives onto the screen and rely on fixed-function graphics hardware features to resolve visibility and shading without explicit ray-marching or mesh generation [S1, unofficial][S2, unofficial]. Common mechanisms include using alpha testing to discard fragments that do not meet an isovalue threshold, z-testing for depth occlusion, and stencil buffers for front/back face intersection parity [S2, unofficial]. Lighting is typically computed per-vertex or via pre-computed normal maps to avoid per-pixel gradient evaluation [S1, unofficial][S2, unofficial].

## Subtopics

### [Multiple Flat Shaded Opaque Isosurfaces](1498-multiple-flat-shaded-opaque-isosurfaces.md)
This method renders multiple flat-shaded opaque isosurfaces from an unstructured tetrahedral mesh without requiring explicit visibility sorting of the cells. It sets itself apart by encoding visibility ordering directly into the scalar field values relative to the isovalue and combining a texture map lookup with OpenGL alpha and z-tests. Choose this approach when rendering multiple isovalue surfaces from tetrahedral data without explicit sorting overhead.

### [Upson-Keller Polygon Method](2041-upson-keller-polygon-method.md)
The Upson-Keller Polygon Method is an early image-order isosurface rendering technique that uses fixed-function rasterization hardware, specifically alpha-testing and stencil buffers, to extract and render isosurfaces without programmable shaders. It sets itself apart by explicitly using the stencil buffer to resolve front/back face intersections via XOR logic, allowing correct visibility handling of complex topology in a two-pass process. Choose this method when targeting legacy fixed-function hardware or requiring robust visibility handling for complex topology.

### [OpenGL Point Rendering for Isosurfaces](8433-opengl-point-rendering-for-isosurfaces.md)
This method renders isosurfaces by generating point primitives (`GL_POINTS`) at the surface location and rendering them as discs, rather than constructing polygonal meshes. It sets itself apart by bypassing the geometry pipeline's vertex processing overhead and avoiding backface culling, though it suffers from specific dilation artifacts at silhouettes. Choose this approach when minimizing geometry generation costs for dynamic datasets is prioritized over silhouette accuracy.

### [Parallax Occlusion Mapping](9926-parallax-occlusion-mapping.md)
This technique reveals Signed Distance Field (SDF) iso-surfaces within rasterized brick bounding boxes using displacement maps, providing depth detail without the computational cost of full raymarching. It sets itself apart by physically altering the silhouette and geometry to avoid the illusion breakdown at sharp edges that plagues texture-based parallax techniques. Choose this method when adding voxel-scale depth detail to low-poly meshes in real-time without converting the scene into a voxel grid.

### [Differentiable Bitmap Splatting](13624-differentiable-bitmap-splatting.md)
This method optimizes the position, rotation, scale, color, and opacity of thousands of bitmap primitives using custom CUDA kernels for forward and backward passes. It sets itself apart by supporting diverse, semi-transparent compositions with specialized parallelism, unlike prior bitmap methods that lacked parallelism and were limited to repetitive, opaque patterns. Choose this approach when differentiable optimization of raster bitmap compositions is required.

### Hardware Rasterization
This variant replaces the CUDA software rasterizer with a GPU hardware rasterizer to separate pipeline overhead from primitive efficiency [S1, unofficial]. It operates by porting the rendering pipeline to Vulkan, sorting surfels globally instead of per tile, and drawing each primitive as a quad through the graphics pipeline [S1, unofficial]. Quality-wise, the GPU hardware rasterizer matches the CUDA software rasterizer on all 13 tested scenes, with at most 0.05 dB and 0.0007 LPIPS deviation [S1, unofficial]. Mean frame rates rise to 2698, 3016, and 4413 FPS on Mip-NeRF 360, Tanks & Temples, and Deep Blending, respectively, representing a 1.8× speedup over the CUDA implementation [S1, unofficial]. Additionally, 77% of fragments generated by the hardware rasterizer survive support and opacity tests on average (53–86% across 13 scenes), compared to 17% for software tile evaluation [S1, unofficial].

### Analytic Wavelet Rasterization (Haar)
This method analytically calculates antialiased rasterization of oriented triangle meshes using wavelets, specifically the Haar basis, which is equivalent to performing a box-filter on the rasterized image [S2, unofficial]. It calculates wavelet coefficients by using the divergence theorem to relate the integral over the triangle mesh to the integral over the boundary of the mesh, because wavelet coefficients are non-zero only where the boundary intersects the basis function [S2, unofficial]. The indicator function for object interior $M$ is defined as $\chi_M(p) = 1$ if $p \in M$, else $0$ [S2, unofficial]. The Haar scaling function is $\phi(t) = 1$ if $0 \leq t < 1$, else $0$ [S2, unofficial]. The Haar basis function is $\psi(t) = \phi(2t) - \phi(2t - 1)$ [S2, unofficial]. Integrating over the domain of a triangle mesh is difficult, so the method handles this by using the divergence theorem to convert surface integrals to boundary integrals [S2, unofficial].

## Background
The Upson-Keller method originated in the early days of 3D graphics when fixed-function hardware lacked programmable shaders, necessitating clever use of stencil buffers and alpha testing for isosurface extraction [S2, unofficial]. The use of point primitives for isosurfaces emerged as a way to avoid the computational cost of generating and sorting polygonal meshes (like Marching Cubes) for dynamic data [S1, unofficial]. The application of Haar wavelets for analytic rasterization was developed to provide antialiased rendering of triangle meshes by leveraging the properties of the Haar basis functions [S2, unofficial]. Recent applications include replacing software rasterizers with hardware equivalents in neural rendering pipelines to achieve significant frame rate improvements [S1, unofficial].

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
[S2] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
