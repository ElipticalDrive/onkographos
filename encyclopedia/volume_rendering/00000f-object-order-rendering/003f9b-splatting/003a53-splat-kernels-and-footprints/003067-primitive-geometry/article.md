# Primitive Geometry

Also called: Splat Primitives, Geometric Splatting Primitives

## Overview
Primitive Geometry defines the specific geometric shapes used to approximate volumetric data during the splatting process, translating 3D data elements into 2D screen-space primitives for hardware rasterization [S1, S3, S4, unofficial]. It solves the problem of efficiently mapping volumetric density fields to the GPU pipeline by replacing complex ray-marching calculations with simple, rasterizable shapes like quads, capsules, or spheres [S1, S2, S3, unofficial]. This family is distinguished from ray-casting methods by its object-order approach (processing primitives rather than pixels) and from pure point-splatting by its explicit construction of extended geometric footprints (e.g., quads for voxels, capsules for lines) to ensure consistent coverage and correct blending [S1, S2, S3, unofficial].

## Shared principles
The Primitive Geometry family operates on the mechanism of decomposing volumetric data into explicit geometric primitives that are transformed and rasterized [S1, S3, S4, unofficial]. While the parent [Splat Kernels and Footprints](8724-splat-kernels-and-footprints.md) defines the mathematical kernel $h$ and the footprint integral $F(x,y)$, the Primitive Geometry articles define the *mesh* or *shape* that carries this kernel.

1.  **Primitive Definition:** Each data element (voxel, line segment, tetrahedron) is mapped to a specific 2D or 3D geometric primitive. For voxels, this is typically a quad or circle [S1, S2, S3, unofficial]. For lines, it is a capsule [S1, unofficial]. For tetrahedra, it is a sphere or ellipsoid [S1, unofficial].
2.  **Screen-Space Projection:** The primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix [S1, S2, S3, S4, unofficial]. The geometry of the primitive is expanded in view space to account for the desired footprint size, which is often derived from the voxel grid scale and a configurable splat parameter [S1, unofficial].
3.  **Rasterization and Blending:** The transformed primitives are rasterized as triangles or points. The fragment shader evaluates the kernel intensity based on the fragment's distance from the primitive's center or axis [S1, S2, S3, unofficial]. Contributions are accumulated using alpha blending [S1, S2, S3, unofficial].

## Subtopics

### [Circle Splat Rendering](2981-circle-splat-rendering.md)
Circle Splat Rendering projects volumetric primitives as 2D quads in screen space, explicitly constructing geometry with a size derived from the voxel grid scale and a configurable splat parameter. It differs from standard point-splatting by ensuring consistent coverage and enabling efficient frustum culling in clip space by calculating a culling border based on the splat's half-edge length. Choose this method when hardware-accelerated rasterization of voxel data is required with precise control over primitive size and culling boundaries.

### [Line Capsule Primitive](3131-line-capsule-primitive.md)
The Line Capsule Primitive visualizes line-based volumetric data by approximating the 3D Gaussian footprint of a segment with a 2D rotationally symmetric capsule profile in screen space. It is distinguished from [EWA Splatting](1069-ewa-splatting.md) by its focus on line segments rather than point clouds, and from [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) by its use of a simplified 2D geometric profile instead of full 3D covariance matrices. It is chosen for high-density line structures to avoid the performance bottleneck of exact volumetric primitives, using a "deficit rule" at joints to prevent double brightness or gaps.

### [Tetrahedral Splatting](3181-tetrahedral-splatting.md)
Tetrahedral Splatting (TetSplat) renders large, static tetrahedral meshes by approximating each tetrahedron with a single spherical or ellipsoidal splat. Unlike [Voxel Splat Rendering](3730-voxel-splat-rendering.md), which relies on regular grids, TetSplat operates on unstructured elements but requires preprocessing to remove degenerate tetrahedra and partition the volume into shell and solid parts. It is chosen for real-time visualization of unstructured grids where ray-casting is too slow, though it may introduce gaps in small meshes.

### [Voxel Splat Rendering](3730-voxel-splat-rendering.md)
Voxel Splat Rendering projects 3D voxels onto the 2D image plane as geometric primitives (quads or circles) for rasterization and blending. It solves the problem of interactive visualization of large volumetric datasets by leveraging hardware pipelines, allowing for real-time parameter tuning. It is distinct from siblings like [Circle Splat Rendering](2981-circle-splat-rendering.md) in its broader applicability to general voxel data and support for multi-view weighting and normal-based shading.

### [Decomposition](7254-decomposition.md)
Decomposition converts the dataset into geometric primitives (e.g., spheres or slices) which are placed in space, scaled, and colored based on a mapping. This step is a prerequisite for many splatting methods, defining how the raw data is interpreted as geometry.

### [Triangle Mesh Splatting](8109-triangle-mesh-splatting.md)
Not given in the sources.

### [Resonant Brane Splatting](9845-resonant-brane-splatting.md)
Resonant Brane Splatting is a differentiable rendering primitive defined by a feature vector that predicts geometric anchoring, footprint/opacity, and internal mode colors, used for arbitrary-scale super-resolution (ASR).

## Background
The concept of splatting originated in the early 1990s as a method for volume visualization that leveraged existing graphics hardware [S1, S2, S3, unofficial]. Early implementations focused on point splats, but the need for better coverage and anti-aliasing led to the development of extended primitives like quads and ellipsoids [S1, S2, S3, unofficial]. The family has evolved to support various data types, including voxels, lines, and unstructured meshes, adapting the primitive geometry to the specific topological requirements of the data [S1, S2, S3, unofficial]. Modern implementations heavily rely on GPU fragment shaders to evaluate the kernel intensity for each pixel covered by the primitive [S1, S2, S3, unofficial].

## Sources
[S1] Circle Splat Rendering, https://example.com/circle-splat, 2023-10-01
[S2] Voxel Splat Rendering, https://example.com/voxel-splat, 2023-10-01
[S3] Splatting Overview, https://example.com/splatting-overview, 2023-10-01
[S4] Volume Rendering Principles, https://example.com/vol-rendering, 2023-10-01
[S6] EWA Splatting Kernel, https://example.com/ewa-kernel, 2023-10-01
[S7] Volume Rendering Textbook, https://example.com/textbook, 2023-10-01
[S10] Splatting Compositing, https://example.com/compositing, 2023-10-01
