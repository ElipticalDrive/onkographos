# Hardware-Assisted Projection

Also called: Hardware-Accelerated Volume Rendering, Texture-Mapped Volume Rendering

## Overview
Hardware-Assisted Projection is a family of volume rendering techniques that project 3D geometric primitives (cells or tetrahedra) onto the 2D image plane and utilize the graphics hardware's rasterization and texture mapping pipelines to perform the volume integral and compositing [S1][S2, unofficial]. It solves the problem of efficiently rendering unstructured grids by replacing irregular memory access patterns of raycasting with global back-to-front sorting and hardware-accelerated interpolation [S1][S4, unofficial]. Unlike sibling methods that may use local sorting or additive accumulation, this family generally requires explicit depth sorting of cells to ensure correct alpha blending, except in specific absorption-only variants [S1][S4, unofficial].

## Shared principles
The core mechanism involves projecting 3D cells into screen space, decomposing the projection into rasterizable regions, and using hardware interpolation to approximate the volume integral [S1].

1.  **Projection and Subdivision**: Each convex polyhedral cell is projected onto the image plane. The resulting 2D polygon is subdivided into regions where the optical thickness varies linearly [S1]. This is often managed using a winged-edge subdivision structure to handle edge intersections and create polygonal regions $Q$ [S1].
2.  **Thickness and Opacity Calculation**: For each region, the thickness $l$ (length of the viewing ray segment within the cell) is computed at the vertices [S1]. Thickness is zero at profile vertices (silhouettes) and computed from geometry at other vertices [S1]. The transparency $T$ is defined as:
    $$ T = \exp\left(-\int_0^l \tau(s) \, ds\right) $$
    where $\tau(s)$ is the extinction coefficient [S1]. If $\tau$ is constant, $T = \exp(-l \cdot \tau)$ [S1].
3.  **Hardware Compositing**: The method relies on the graphics pipeline to composite fragments. To avoid Mach bands caused by linear interpolation of transparency, texture mapping is used for the exponential lookup [S1].
    *   For constant $\tau$, the product $l \cdot \tau$ is used as a 1D texture coordinate [S1].
    *   For varying $\tau$, $l$ and $\tau$ are used as 2D texture coordinates [S1].
4.  **Sorting Requirement**: Correct compositing via alpha blending (Porter-Duff over operation) requires cells to be sorted in back-to-front order relative to the viewpoint [S1][S4, unofficial].

## Subtopics

### [Hardware-Assisted Polyhedron Projection](1453-hardware-assisted-polyhedron-projection.md)
Renders arbitrary convex polyhedral cells by projecting them, subdividing the projection, and using texture mapping for opacity. It requires global back-to-front sorting and explicit depth sorting of all cells.

### [Hardware-Assisted Projected Tetrahedra](1729-hardware-assisted-projected-tetrahedra.md)
A GPU-centric method for tetrahedral meshes that streams sorted tetrahedra as point primitives, decomposes them in a geometry shader, and uses partial pre-integration tables for fragment shading. It avoids auxiliary graph structures by processing tetrahedra independently.

### [Röttger 3D Texture Mapping](7896-r-ttger-3d-texture-mapping.md)
Extends projected tetrahedra by pre-computing the volume integral into a 3D texture map (approximated for 2D hardware) to support arbitrary non-linear transfer functions without Mach bands.

### [Absorption-Only Radiography via Unsorted Hexahedron Projection](10255-absorption-only-radiography-via-unsorted-hexahedron-projecti.md)
Simulates X-ray radiographs for curvilinear hexahedra by accumulating optical thickness additively, eliminating the need for global back-to-front sorting by exploiting the absorption-only regime.

## Background
The method originates from the need to render unstructured volume data efficiently using available graphics hardware [S1]. It builds on [Projected Tetrahedra](877-projected-tetrahedra.md) by leveraging hardware acceleration for the scan conversion and compositing steps [S1][S2, unofficial]. The approach was developed to handle complex, non-uniform meshes while avoiding the irregular memory access patterns associated with raycasting [S1][S4, unofficial].

## Sources
[S1] Hardware-Assisted Polyhedron Projection notes
[S2] Hardware-Assisted Projected Tetrahedra notes (unofficial)
[S3] Parallel Hardware-Assisted Polyhedron Projection notes
[S4] Hardware-Assisted Polyhedron Projection aliases/summary (unofficial)
