# Interaction and Selection Techniques

Also called: Explicit Scene Exploration

## Overview
This family provides interaction mechanisms for isosurface rendering, allowing users to explore internal structures and select specific data elements without extracting explicit geometric meshes [G]. It solves the problem of limited visibility in dense volumetric data and the need for precise data selection for downstream tasks like segmentation [S1, unofficial]. It differs from standard clipping by supporting complex concave Boolean operations directly on the GPU [3592-interactive-concave-volume-clipping-via-gpu-boolean-operatio.md] and from cropping by allowing selective exposure of deeper layers via peeling [7679-surface-peeling.md] [G].

## Shared principles
The family relies on accurate ray-traversal and ray-isosurface intersection search to identify the exact intersecting voxel for each ray [S1, unofficial]. This precision enables the collection of metadata (such as voxel IDs or intersection counts) associated with the visible surface, which is then used for interaction logic [S1, unofficial]. The core mechanism builds on [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md), utilizing the computed intersection positions and normals for shading and selection [G].

## Subtopics

### [Interactive Concave Volume Clipping via GPU Boolean Operations](3592-interactive-concave-volume-clipping-via-gpu-boolean-operatio.md)
This method enables the interactive visualization of inner structures by performing clipping via Boolean operations (intersection, subtraction, union) directly on the GPU [3592-interactive-concave-volume-clipping-via-gpu-boolean-operatio.md]. It solves the problem of specifying complex transfer functions for revealing internal features by instead using geometric primitives to define visible regions [3592-interactive-concave-volume-clipping-via-gpu-boolean-operatio.md]. It differs from standard convex clipping by supporting concave volumes through the decomposition of 3D Boolean operations into 1D operations along the ray, avoiding the high video memory costs associated with storing complex analytical convex objects [3592-interactive-concave-volume-clipping-via-gpu-boolean-operatio.md].

### [Surface Peeling](7679-surface-peeling.md)
Surface peeling is an interaction technique in isosurface rendering that exposes internal structures by skipping the first $N$ intersection points within specific image plane regions, known as peeling windows [7679-surface-peeling.md]. It differs from cropping, which truncates the volume with a bounding box, by allowing selective exposure of deeper layers without altering the volume data or view frustum [7679-surface-peeling.md].

### Voxel Selection
#### definition
- [S1, unofficial] An interaction technique to collect voxel IDs of isosurface intersections for user selection, leveraging accurate ray-traversal that identifies the exact intersecting voxel.
#### step
- [S1, unofficial] Assign a unique sequential ID to each voxel in the volume.
- [S1, unofficial] During rendering, store the ID of the voxel containing the ray-isosurface intersection into the Voxel ID Buffer.
- [S1, unofficial] On user mouse click/drag, collect the corresponding voxel IDs from the buffer.
#### layout
- [S1, unofficial] Voxel ID Buffer: A buffer storing the unique sequential ID of the voxel containing the intersection point for each ray/pixel.

## Background
The concepts of surface peeling and voxel selection are described in the context of "explicit scene exploration" for efficient and effective volume visualization with enhanced isosurface rendering [S1, unofficial]. Voxel selection is specifically noted as an intuitive tool for seed selecting in isosurface segmentation [S1, unofficial].

## Sources
[S1] Efficient and Effective Volume Visualization with Enhanced Isosurface Rendering, https://doi.org/10.48550/arxiv.1202.5360, fetched 2026-10-26 (unofficial)
