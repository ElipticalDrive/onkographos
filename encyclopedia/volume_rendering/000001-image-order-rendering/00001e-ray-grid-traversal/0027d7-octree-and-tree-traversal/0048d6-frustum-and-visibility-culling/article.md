# Frustum and Visibility Culling

Also called: Frustum Culling, Visibility Culling, Frustum-Occlusion Culling

## Overview
Frustum and Visibility Culling methods optimize volume rendering by identifying and discarding geometry that does not contribute to the final image, thereby reducing computational load. This family distinguishes itself from sibling traversal methods by decoupling visibility determination from the core sampling loop, often serving as a pre-pass or integrated filter to limit the working set of voxels, splats, or bricks [S1, unofficial]. Unlike [Octree and Tree Traversal](4134-octree-and-tree-traversal.md) which focuses on hierarchical descent, these methods focus on spatial containment relative to the camera.

## Shared principles
All methods in this family share the core mechanism of hierarchical descent and ascent. The traversal begins at the root of the octree and determines which of the eight children (octants) the ray intersects. This is typically done by comparing the ray's position relative to the center of the current node's bounding box [S1, unofficial][S2, unofficial][S6, S8, unofficial].

1.  **Child Selection:** For a node with center $\mathbf{c}$ and half-extent $s$, the ray position $\mathbf{p}$ determines the child index $k \in \{0, \dots, 7\}$:
    $$
    k = 4 \cdot \mathbb{I}(p_x \ge c_x) + 2 \cdot \mathbb{I}(p_y \ge c_y) + 1 \cdot \mathbb{I}(p_z \ge c_z)
    $$
    where $\mathbb{I}(\cdot)$ is the indicator function [S1, unofficial][S2, unofficial][S6, S8, unofficial].

2.  **Empty Space Skipping:** If the selected child is marked as empty (or uniform), the traversal skips the subtree. This is achieved by advancing the ray parameter $t$ to the exit of the current node or using neighbor pointers to jump to the next potentially occupied node [S1, S6, S8, unofficial][G].

3.  **Leaf Handling:** Upon reaching a leaf node, the method typically switches to a fixed-step traversal (like DDA) within the leaf's voxel grid to sample the volume data [S1, S6, S8, unofficial][G].

4.  **Backtracking:** Because octrees are sparse and irregular, traversal often requires backtracking to the parent node when a ray exits a subtree or encounters an empty branch. This is implemented using an explicit stack (on GPU) or recursion (on CPU) [S4, S6, S8, unofficial].

## Subtopics

### [Bounding-Sphere Tree Traversal with Frame-Rate Control](7227-bounding-sphere-tree-traversal-with-frame-rate-control.md)
This method renders hierarchical shell and solid volumetric structures by traversing a bounding-sphere tree and adaptively controlling the frame rate through dynamic adjustment of minimum splat sizes. It distinguishes itself from standard octree traversal by replacing fixed-resolution voxel sampling with screen-space splat rendering, where the granularity of the visual output is directly tied to performance feedback. Unlike sibling methods that rely on geometric bounding boxes for culling, this approach uses bounding spheres and hierarchical normal cones to optimize visibility tests for point-based primitives. See [Bounding-Sphere Tree Traversal with Frame-Rate Control](7227-bounding-sphere-tree-traversal-with-frame-rate-control.md).

### [Frustum Culling Integration](8798-frustum-culling-integration.md)
Frustum Culling Integration is a visibility optimization technique applied during the progressive loading and rendering of 3D Gaussian Splatting scenes. It filters out splats that lie outside the user's current viewing frustum to prioritize the transmission and rendering of visible geometry. It differs from its sibling [Bounding-Sphere Tree Traversal with Frame-Rate Control](7227-bounding-sphere-tree-traversal-with-frame-rate-control.md) by focusing on spatial visibility rather than frame-rate stability, and from [Frustum-Occlusion Culling for Octree Traversal](12324-frustum-occlusion-culling-for-octree-traversal.md) by typically operating on a pre-ordered list of splats rather than a hierarchical tree structure during the culling phase. See [Frustum Culling Integration](8798-frustum-culling-integration.md).

### [Frustum-Occlusion Culling for Octree Traversal](12324-frustum-occlusion-culling-for-octree-traversal.md)
This method identifies a set of visible voxels by traversing an octree hierarchy and culling nodes that do not intersect the camera's view frustum or whose projected screen area is smaller than a single pixel. It serves as a visibility pre-pass for voxel rasterization, automatically determining the Level of Detail (LOD) for rendering. Unlike standard ray-marching which samples volume density, this approach outputs a list of 3D bounding boxes (voxels) to be rasterized as geometry using hardware instancing. It is distinct from [Bounding-Sphere Tree Traversal with Frame-Rate Control](7227-bounding-sphere-tree-traversal-with-frame-rate-control.md) by using explicit frustum intersection tests rather than frame-rate driven depth limits, and from [Frustum Culling Integration](8798-frustum-culling-integration.md) by specifically targeting octree structures for voxel-based rendering pipelines. See [Frustum-Occlusion Culling for Octree Traversal](12324-frustum-occlusion-culling-for-octree-traversal.md).

### Ray-Guided Volume Rendering
#### step
- [S1, unofficial] Visibility is determined during ray traversal, providing implicit view frustum culling without extra steps.
- [S1, unofficial] Provides implicit occlusion culling without extra steps or occlusion buffers.
- [S1, unofficial] Rays determine the working set directly by writing out a list of bricks they require (intersect) in front-to-back order.
- [S1, unofficial] Uses modern OpenGL extensions such as GL_ARB_shader_storage_buffer_object.
#### history
- [S1, unofficial] Introduced by Crassin et al. in 2009.

## Background
The concept of culling visible geometry is fundamental to computer graphics. In the context of volume rendering, early approaches relied on explicit pre-passes to determine the "working set" of bricks or voxels. As hardware capabilities evolved, particularly with the advent of shader storage buffer objects, methods like Ray-Guided Volume Rendering integrated culling directly into the rendering pipeline [S1, unofficial]. The specific techniques described in the subtopics represent specialized applications of these principles to different data structures (octrees, splats, bounding-sphere trees) and rendering paradigms (rasterization, ray-casting).

## Sources
[S1] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
