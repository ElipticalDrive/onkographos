# Splat Extent and Projection

Also called: Splat Footprint Projection, Splat Clipping

## Overview
Splat Extent and Projection determines the screen-space position, shape, and boundaries of a splat (the footprint of a primitive) to ensure correct rasterization and compositing. It solves the problem of mapping 3D primitives (points, voxels, or Gaussians) to 2D screen space while handling perspective distortion, view-frustum clipping, and kernel integration. It differs from its siblings by focusing on the geometric transformation and extent calculation of the splat itself, rather than the kernel shape (Polynomial and Spline Kernels) or the compositing order (Footprint Evaluation).

## Shared principles
The family shares the goal of computing the 2D screen-space representation of a 3D primitive. The core mechanism involves:
1.  **Projection:** Mapping the primitive's center from object/world space to screen space using the view and projection matrices.
2.  **Extent Calculation:** Determining the region of the image plane affected by the primitive's kernel. This often involves approximating the kernel's support (e.g., radius) in screen space, which varies with depth in perspective projection.
3.  **Clipping:** Ensuring the splat is correctly handled when it intersects the view frustum or clipping planes, preventing artifacts from primitives partially outside the view.

## Subtopics

### [Perspective-Correct Point Splatting Vertex Shader](1082-perspective-correct-point-splatting-vertex-shader.md)
This method renders point clouds as splats with size attenuation based on perspective projection. It sets itself apart by computing the point size directly in the vertex shader using the projection matrix and view-space depth, ensuring visual size consistency with world-space radius. Choose this for point-based rendering where hardware point sprites are used.

### [Footprint Evaluation](1180-footprint-evaluation.md)
Footprint Evaluation is a forward-mapping algorithm that projects each voxel onto the 2D image plane, distributing its contribution as a "splat" or footprint. It sets itself apart by pre-integrating the 3D kernel into a 2D footprint table to avoid on-the-fly integration. Choose this for efficient volume rendering where cache coherency is prioritized.

### [Centralized Conics Approach](1243-centralized-conics-approach.md)
The Centralized Conics Approach is an efficient approximation method for computing the screen-space position and extents of a splat. It sets itself apart by using the inverse of a 2D-to-2D projective mapping from the splat's parameter space to screen space. Choose this when computational speed is critical and extreme perspective distortion is not a primary concern.

### [Exact Half-Space Clipping](3395-exact-half-space-clipping.md)
Exact Half-Space Clipping computes the exact retained ray integral of a 3D Gaussian primitive when intersected by a clipping plane. It sets itself apart by providing a closed-form backward pass for clip-aware training without auxiliary networks, avoiding the aliasing and popping artifacts of binary culling. Choose this for differentiable rendering and training of Gaussian splats with clipping planes.

### [Mueller's Splatting Approach](5752-mueller-s-splatting-approach.md)
Mueller's Splatting Approach is an object-order direct volume rendering technique that aligns sheet buffers parallel to the image plane. It sets itself apart by eliminating "popping" artifacts inherent in axis-aligned sheet approaches. Choose this for direct volume rendering of gray-scale volumes where sharp boundary preservation is less critical than speed.

### Splat Extent Calculation (Polygon Bounding)
#### definition
- [S1, unofficial] A method to compute screen-space position and extents of a splat by bounding it with a polygon in parameter space, transforming to clip space, clipping against the view-frustum, and performing perspective division.

#### step
- [S1, unofficial] 1. Bound the splat by a polygon in its parameter space. 2. Transform the polygon to clip space. 3. Clip the polygon by the view-frustum using the Sutherland-Hodgman algorithm. 4. Perform perspective division on each vertex to yield the clipped polygon in screen-space. 5. Compute desired quantities (position/extents) from the screen-space polygon.

#### pitfall
- [S1, unofficial] The method is not exact but produces a correct upper bound.

## Background
The concept of splatting was introduced by Levoy [S1, unofficial] and later refined by many authors. The specific technique of bounding a splat by a polygon in parameter space and clipping it in clip space was devised by Sebastian Lippner as part of his surface splatting implementation [S1, unofficial]. This approach addresses the limitations of previous methods that produced artifacts, particularly for large splats near the near plane. The method is implemented in the `sebastianlipponer/surface_splatting` repository [S1, unofficial].

## Sources
[S1] sebastianlipponer/surface_splatting, https://github.com/sebastianlipponer/surface_splatting, fetched 2026-10-28 (unofficial)
