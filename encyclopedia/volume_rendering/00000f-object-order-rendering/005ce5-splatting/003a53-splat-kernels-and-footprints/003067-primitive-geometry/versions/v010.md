# Primitive Geometry

Also called: Splat Primitives, Geometric Splat Kernels

## Overview
Primitive Geometry defines the specific 2D or 3D shapes used to represent volumetric data points as splats during object-order rendering. It solves the problem of how to approximate continuous volumetric fields or discrete geometric structures using rasterizable primitives that can be efficiently projected and composited. It differs from sibling methods by focusing on the topological and geometric definition of the primitive itself (e.g., circle, capsule, tetrahedron, brane) rather than the kernel function's mathematical falloff or the projection matrix mechanics.

## Shared principles
The family shares the fundamental mechanism of mapping a volumetric entity to a screen-space footprint via projection. All methods assume a camera model defined by a Model-View-Projection (MVP) matrix or separate Model-View (MV) and Projection (P) matrices. The core workflow involves:
1.  **Transformation:** Converting object-space coordinates to screen space.
2.  **Footprint Definition:** Defining the spatial extent of the primitive in screen space (e.g., radius, bounding box, or parametric surface).
3.  **Rasterization/Compositing:** Projecting the footprint onto the image buffer and accumulating color/opacity.

Mathematically, for a primitive centered at $\mathbf{p}$, the screen-space contribution $C(u,v)$ at pixel $(u,v)$ is derived from a footprint function $F$ and a color/opacity field $I$:
$$ C(u,v) = I \cdot F(\mathbf{p}_{screen}, (u,v)) $$
where $F$ depends on the specific geometry (e.g., Gaussian for circles, distance-to-segment for capsules).

## Subtopics

### [Circle Splat Rendering](2981-circle-splat-rendering.md)
Renders voxels as 2D quads with a fixed offset derived from splat size and grid scale. Chosen for simple, fast voxel visualization where square footprints approximate circular coverage.

### [Line Capsule Primitive](3131-line-capsule-primitive.md)
Renders line segments as rotationally symmetric capsules in screen space. Chosen for line-based volumetric data (e.g., streamlines) to maintain visual continuity without gaps using deficit-rule joints.

### [Tetrahedral Splatting](3181-tetrahedral-splatting.md)
Maps unstructured tetrahedral mesh elements to spherical or ellipsoidal splats. Chosen for real-time rendering of large, static unstructured meshes, leveraging hierarchical culling.

### [Voxel Splat Rendering](3730-voxel-splat-rendering.md)
Projects voxels as 2D footprints (squares or circles) composited into the image buffer. Chosen for interactive rendering of large regular volumetric datasets, trading geometric precision for performance.

### [Decomposition](7254-decomposition.md)
Converts volume into discrete primitives (spheres/slices) mapped by data values. Chosen when explicit discretization is preferred over direct ray casting, preserving continuous data nature through scaling.

### [Triangle Mesh Splatting](8109-triangle-mesh-splatting.md)
Approximates analytic splat kernels with discrete triangle meshes. Chosen to leverage standard graphics hardware rasterization pipelines for rapid scan conversion, trading geometric precision for speed.

### [Resonant Brane Splatting](9845-resonant-brane-splatting.md)
Uses 'Branes' (Gaussian-Hermite modes) to emit spatially varying colors. Chosen for arbitrary-scale super-resolution, modeling local contrast and sharp edges with fewer primitives than standard Gaussians.

## Background
The concept of splatting originated in the early 1990s as a method for interactive volume rendering, replacing ray casting with object-order projection to improve performance . Early implementations focused on Gaussian kernels and voxel-based data. As rendering hardware evolved, the definition of "primitive" expanded from simple point sprites to complex geometric shapes like capsules for line data  and tetrahedra for unstructured meshes . Recent developments incorporate neural networks to define expressive primitives like Branes for super-resolution .

## Sources
[S1] Unofficial notes on Primitive Geometry family methods.
[S2] Unofficial notes on Voxel Splat Rendering parameters.
[S3] Unofficial notes on Voxel Splat Rendering overview and compositing.
