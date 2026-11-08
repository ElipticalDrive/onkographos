# Primitive Geometry


## Overview
Primitive Geometry is the foundational family of splatting methods that decompose volumetric data into explicit geometric or parametric primitives (such as voxels, tetrahedra, or line segments) which are transformed and rasterized in screen space [S1, S3, S4, unofficial]. It differs from ray-casting by computing contributions per primitive rather than per pixel, leveraging hardware-accelerated rasterization for real-time performance [S1, S2, S3, unofficial].

## Shared principles
The splatting family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [S1, S3, S4, unofficial].

1.  **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware [S1, S3, S4, unofficial]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1, unofficial][S6][S10, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2, S3, S4, unofficial]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [S6][S10, unofficial].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending [S1, S2, S3, S4, unofficial]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2, unofficial][S6][S10, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
    where $\text{weight}(x,y) = F(x-x_D, y-y_D)$ [S10, unofficial].
4.  **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, S2, S3, unofficial][S7, book]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [S10, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, S2, S3, unofficial].

## Subtopics

### [Circle Splat Rendering](2981-circle-splat-rendering.md)
Projects volumetric primitives as 2D quads in screen space, explicitly constructing quad geometry with size derived from voxel grid scale and splat parameter for consistent coverage and efficient frustum culling.

### [Line Capsule Primitive](3131-line-capsule-primitive.md)
Renders line-based volumetric data (streamlines/fibers) as rotationally symmetric capsules in screen space, solving visual continuity at joints via deficit-rule composition.

### [Tetrahedral Splatting](3181-tetrahedral-splatting.md)
Renders unstructured tetrahedral meshes by approximating each tetrahedron with a single spherical or ellipsoidal splat, operating directly on unstructured geometric primitives.

### [Voxel Splat Rendering](3730-voxel-splat-rendering.md)
Projects 3D voxels onto the 2D image plane as geometric primitives (quads/circles) for interactive visualization, distinct from ray-casting by computing contributions per voxel.

### [Decomposition](7254-decomposition.md)
Converts volumetric data into explicit geometric primitives (spheres/slices) placed and colored by transfer functions, suitable for geometric processing like segmentation.

### [Triangle Mesh Splatting](8109-triangle-mesh-splatting.md)
Approximates continuous splat kernels with triangle meshes to leverage standard GPU rasterization for rapid scan-conversion of kernel footprints.

### [Resonant Brane Splatting](9845-resonant-brane-splatting.md)
Replaces standard 2D Gaussian splats with "Branes" that emit spatially varying colors using Gaussian-Hermite modes, enabling super-resolution without dense overlapping splats.

## Background
Not given in the sources.

## Sources
