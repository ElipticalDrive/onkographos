# Voxelization via Rasterization
## Overview
This family encompasses methods that generate a voxel representation of 3D geometry by leveraging standard 2D rasterization hardware or libraries. Instead of performing explicit geometric intersection tests against a 3D voxel grid, these methods project the input primitives onto 2D planes, rasterize them to determine coverage, and map the results back to 3D voxel coordinates. This approach distinguishes itself from siblings like [Binary Volume Rendering](1015-binary-volume-rendering-opacity-thresholding.md) and [Naive Voxel Rendering](1103-naive-voxel-rendering.md) by relying on the efficiency of established 2D rasterization pipelines rather than brute-force 3D traversal or simple thresholding.
## Shared principles
All methods in this family share the core mechanism of decoupling the rasterization step from the final 3D grid storage or utilizing 2D buffers as an intermediate representation. The general workflow involves:
1.  **Projection:** Mapping 3D input primitives (typically triangles) onto a 2D domain (either a projection plane or a UV space).
2.  **Rasterization:** Using a 2D rasterizer to determine which pixels/texels are covered by the primitives.
3.  **Mapping:** Converting the 2D coverage information into 3D voxel indices. This may involve evaluating depth at pixel centers, storing face IDs for later interpolation, or using union-based hash tables to resolve overlaps.
4.  **Aggregation:** Combining results from multiple passes (e.g., three axes) or chunks into a final voxel set.

The specific implementation details, such as whether to use 6-separating or 26-separating connectivity, and how to handle memory for large meshes, vary by child method.
## Subtopics

### [3-axis rasterization voxelization](2480-3-axis-rasterization-voxelization.md)
Produces a 6-separating voxelization by rasterizing primitives along three axis-aligned projection directions in three separate passes. It is designed to generate a sparser voxel set than 26-separating methods, which is sufficient and preferable when ray marching algorithms utilize 6-connected paths. It differs from its siblings by relying on simple point-depth evaluations at pixel centers rather than complex geometric intersection tests against volumetric targets. Choose this method when a sparse, 6-connected voxel boundary is required for efficient ray marching.

### [Chunked UV Rasterization with Face ID Storage](2598-chunked-uv-rasterization-with-face-id-storage.md)
Converts a triangle mesh into a voxelized representation by rasterizing triangles into a 2D texture buffer in UV space, storing the face ID in the alpha channel to enable per-texel face identification and subsequent 3D position interpolation. It builds on [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by decoupling the rasterization step from the 3D grid storage, allowing for memory-friendly chunked processing of large meshes. Unlike [3D Grid Rasterization](903-3d-grid-rasterization.md) which operates directly in 3D, this approach projects the mesh onto a 2D plane, leveraging standard 2D rasterization hardware or libraries to determine coverage and face identity before mapping results back to 3D voxel coordinates. Choose this method for large meshes where VRAM is limited and precise surface position interpolation is needed.

### [Triangle Projection Voxelization](5432-triangle-projection-voxelization.md)
Converts a triangle mesh into a voxel grid by rasterizing triangles into a union-based hash table of voxel centers and colors. It sets itself apart from sibling methods by explicitly using a union-based approach to resolve overlaps, likely combining multiple passes or render targets to ensure correct coverage. Choose this method when a direct GPU-based voxelization with overlap resolution is required.

## Background
The use of rasterization for voxelization leverages the maturity and speed of 2D graphics pipelines to solve 3D geometric problems. This approach avoids the computational overhead of explicit 3D intersection tests. The specific variants within this family have been developed to address limitations in memory usage (chunking), connectivity requirements (6-axis vs 26-axis), and overlap resolution (union-based hashing).
## Sources
