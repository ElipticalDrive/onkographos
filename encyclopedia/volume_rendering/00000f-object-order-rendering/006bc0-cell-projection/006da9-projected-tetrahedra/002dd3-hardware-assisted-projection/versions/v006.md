# Hardware-Assisted Projection
## Overview
Hardware-Assisted Projection is a family of volume rendering techniques that project 3D geometric primitives onto the 2D screen space to leverage GPU rasterization pipelines for scan conversion and compositing [S1, unofficial]. It extends the projected tetrahedra algorithm to enable hardware-accelerated and accurate rendering of volumetric data [S1, unofficial].
## Shared principles
The family shares a common object-order approach where 3D cells are projected to 2D screen space, and the volume integral is approximated per pixel or fragment using hardware acceleration [S1, unofficial].
1. **Projection**: 3D vertices of the unstructured grid cells are transformed into screen coordinates using view-projection matrices [S1, unofficial].
2. **Scan Conversion**: The projected cells are rasterized to determine ray-cell intersections and attribute contributions for each pixel or fragment [S1, unofficial].
3. **Compositing**: Depth-sorted segments or accumulated values are combined to produce the final image [S1, unofficial].
## Subtopics

### [Hardware-Assisted Polyhedron Projection](1453-hardware-assisted-polyhedron-projection.md)
Projects arbitrary convex polyhedral cells to 2D, performs scan conversion to generate depth-sorted ray segments, and composites them using hardware acceleration. It differs from raycasting by operating in image space with object-order projection, allowing flexible data distribution on distributed-memory systems and leveraging standard polygon pipelines for scan conversion. Unlike [Projected Tetrahedra](877-projected-tetrahedra.md), which requires explicit triangle generation and sorting, this method projects cells directly as polygons and relies on hardware compositing or global sorting for depth ordering. Choose it for arbitrary convex polyhedral meshes requiring explicit depth sorting.

### [Hardware-Assisted Projected Tetrahedra](1729-hardware-assisted-projected-tetrahedra.md)
Projects tetrahedra onto the screen and rasterizes them as triangles, evaluating the volume ray integral per fragment in the fragment shader. It performs the entire rendering pipeline on the GPU in a single pass. Unlike its sibling [Projected Tetrahedra](877-projected-tetrahedra.md), which requires CPU-side sorting and triangle generation, HAPT streams sorted tetrahedra as point primitives and uses the GPU geometry shader to decompose them into triangles. It differs from [Tetrahedral Projection with Partial Pre-integration](1861-tetrahedral-projection-with-partial-pre-integration.md) (PTINT) by eliminating the need to read data back from the GPU, allowing rendering of volumes with several million tetrahedra that would exceed GPU memory limits. Choose it for unstructured tetrahedral grids requiring full GPU pipeline execution.

### [Absorption-Only Radiography via Unsorted Hexahedron Projection](10255-absorption-only-radiography-via-unsorted-hexahedron-projecti.md)
A hardware-accelerated technique to simulate radiographs in the absorption-only regime (emissivity $J=0$) for curvilinear hexahedral meshes, avoiding the need for a global back-to-front visibility sort by accumulating optical thickness additively. Unlike its siblings [Hardware-Assisted Polyhedron Projection](1453-hardware-assisted-polyhedron-projection.md) and [Hardware-Assisted Projected Tetrahedra](1729-hardware-assisted-projected-tetrahedra.md), which generally require depth sorting to correctly composite transparent volumes using alpha blending, this method exploits the commutative property of addition in the absorption-only recurrence to render cells in any order. Choose it for curvilinear hexahedral meshes in the absorption-only regime where sorting is prohibitive.
#### Röttger 3D Texture Mapping
- **Definition**: An extension of the projected tetrahedra algorithm for hardware-accelerated rendering of volumetric data using 3D texture mapping [S1, unofficial].
- **Steps**: Linearly approximate parts of the volume integral depending on segment length by modulating vertex colors. Tabulate remaining parts that depend only on textural coordinates in a 2D texture map [S1, unofficial].

## Background
The concept of hardware-assisted projection builds upon the projected tetrahedra algorithm [ST90], extending it for hardware-accelerated and accurate rendering of volumetric data using 3D texture mapping [S1, unofficial]. Röttger et al. [RKE00] were able to extend the projected tetrahedra algorithm for hardware-accelerated rendering [S1, unofficial].
## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
