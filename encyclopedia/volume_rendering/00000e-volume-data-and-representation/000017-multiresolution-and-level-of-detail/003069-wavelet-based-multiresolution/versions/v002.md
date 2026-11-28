# Wavelet-based Multiresolution
## Overview
This family encompasses techniques that decompose volumetric data into a hierarchy of spatial-frequency components using wavelet transforms, enabling efficient Level-of-Detail (LOD) rendering and compression [S1, unofficial]. It differs from sibling methods like [VoxelGrid Down Sample](1014-voxelgrid-down-sample.md) or [Mipmap Generation](2602-mipmap-generation.md) by operating in the frequency domain to preserve high-frequency details at coarse resolutions, rather than relying solely on spatial averaging or subsampling [G].
## Shared principles
Builds on [Multiresolution and level of detail](7-multiresolution-and-level-of-detail.md). The family shares the parent's core mechanism of generating a hierarchy of data representations at different spatial resolutions, using scaling factors, hierarchical structures, interpolation or filtering kernels, and dynamic selection or culling [S2, unofficial].
## Subtopics

### [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md)
Renders multiresolution data by projecting detail coefficients and approximations directly to screen-space quads of decreasing resolution. It differs from tree-based methods by avoiding explicit tree traversal for ray casting, instead treating each level as a separate geometric primitive.

### [Wavelet Tree Overview Map](5354-wavelet-tree-overview-map.md)
A visualization tool that maps the wavelet tree structure into a 2D triangular frame to display error distribution and LOD cuts. It complements the raw tree structure by providing an intuitive spatial overview of data quality.

### [Wavelet Tree](5390-wavelet-tree.md)
The foundational data structure that organizes volume blocks into a strict parent-child hierarchy. Internal nodes store low-resolution approximations and compressed detail coefficients, enabling efficient reconstruction and LOD selection.

### Multiresolution Analysis (MRA) for Meshes
A technique using wavelets to guide a subdivision process, allowing a polygonal model to be interpolated smoothly between different LODs [S1, unofficial]. It creates a mesh by growing Voronoi-like regions across triangles, with a Delaunay triangulation of those regions forming the base mesh [S1, unofficial].

## Background
Not given in the sources. These methods are primarily used in scientific visualization for large-scale datasets where memory bandwidth and rendering performance are critical constraints [G].
## Sources
[S1] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
