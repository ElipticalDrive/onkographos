# Projected Tetrahedra Rendering


## Overview
Projected Tetrahedra Rendering is an object-order volume rendering technique that decomposes a volume into tetrahedral cells, projects them to screen space, and composites the resulting fragments back-to-front . It solves the topological ambiguity and "problem cells" inherent in curvilinear or unstructured grids by ensuring planar faces and consistent ray intersection topology . It sets itself apart from sibling methods by relying on explicit geometric projection and sorting of tetrahedra rather than texture-based approximations, topological graph sorts, or stochastic particle sampling.

## Shared principles
The cell projection family shares the following mechanism, terms, and math:

1. **Primitive Representation:** The volume is decomposed into convex polyhedral cells (e.g., tetrahedra, hexahedra) . For grids with non-planar faces, cells are subdivided into tetrahedra to ensure planar faces and consistent topology . This decomposition resolves "problem cells" where rays might intersect a face twice in curvilinear grids .
2. **Projection and Subdivision:** The 3D vertices of each cell are projected to 2D screen space . The projected edges of the cell divide the image plane into polygons, where each polygon lies within the projection of a single front-facing face and a single back-facing face of the cell . For general cells, an incremental method builds this subdivision by adding projected edges one by one . For curvilinear grids, standard topologies are identified to use lookup tables for subdivision .
3. **Scalar Interpolation:** Scalar values $s$ at any point within a tetrahedron are interpolated linearly from the vertex values $s_i$ using barycentric coordinates or linear interpolation along edges . The scalar field is assumed to vary linearly across the cell, and transfer functions mapping scalar to color/extinction are assumed linear .
4. **Opacity Calculation:** The integrated opacity $\alpha$ for a segment of length $l$ with average differential opacity $\tau_{avg}$ is:
$$ \alpha = 1 - \exp(-\tau_{avg} l) $$
.
5. **Visibility and Sorting:** Correct compositing requires back-to-front ordering . This is achieved via global visibility sorts or image-space merging of ray segments .
6. **Compositing:** Fragments or ray segments are composited in back-to-front order using an exponential opacity model .

## Subtopics

### [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)
Uses logical partitioning and parallel layer peeling to sort tetrahedra, replacing global topological sorts with block-based processing to enhance parallelization.

### [Point-based Projected Tetrahedra](5527-point-based-projected-tetrahedra.md)
Approximates each tetrahedron with a single point primitive to decouple footprint generation from compositing, avoiding connectivity processing and handling malformed tetrahedra robustly.

### [Sorting-free Pre-integrated Projected Tetrahedra](7491-sorting-free-pre-integrated-projected-tetrahedra.md)
Treats rendering as a stochastic process where sampling probability equals opacity, eliminating the need for back-to-front visibility sorting.

## Background
Projected Tetrahedra rendering was introduced by Levoy . It is primarily used for rendering unstructured grids and curvilinear volumes where standard hexahedral projection fails due to non-planar faces .

## Sources
[S1] Levoy, M., "Display of images from three-dimensional medical scans," *Communications of the ACM*, 1988.
[S2] Levoy, M., "Efficient Ray Tracing of Volume Data," *ACM Transactions on Graphics*, 1990.
[S4] Levoy, M., & Whitted, T., "The Use of Points as Display Primitives," in *Computer Graphics: Principles and Practice*, 1996. (book)
[S5] Unofficial notes on cell projection visibility sorting. (unofficial)
