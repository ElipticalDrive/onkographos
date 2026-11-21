# Hierarchical and Block-Based Data Structures

Also called: Hierarchical Grids

## Overview
Hierarchical and block-based data structures organize volumetric data into a multi-resolution hierarchy of uniform blocks (bricks) or spatial subdivisions (octrees) to accelerate ray traversal and rendering. This approach serves as a compromise between the memory overhead of dense grids and the complexity of unstructured meshes, preferred by many practitioners for its balance of performance and implementation simplicity [S1, unofficial]. It enables efficient empty-space skipping, early ray termination, and adaptive resolution sampling by allowing the renderer to skip large empty regions or sample coarse data for distant objects.

## Shared principles
All methods in this family share the following mechanisms:

1.  **Spatial Subdivision:** The volume is partitioned into smaller, manageable units. These units are either uniform blocks (bricks) of fixed size or spatial nodes in a tree structure (octree/quadtree).
2.  **Hierarchical Aggregation:** Parent nodes or blocks store aggregate metadata derived from their children. Common metadata includes:
    *   **Scalar Range:** Minimum and maximum values ($min, max$) of the scalar field within the subtree. This allows for empty-space skipping: if the isosurface threshold $T$ lies outside $[min, max]$, the entire subtree is pruned [G].
    *   **Occupancy/Visibility:** Flags indicating if the block contains non-empty or non-transparent data.
3.  **Traversal:** Ray traversal follows the hierarchy from root to leaf.
    *   If a node is determined to be empty or irrelevant (e.g., threshold out of range, fully transparent), its children are skipped.
    *   If a node is relevant, the ray is tested against its children.
4.  **Adaptive Resolution:** Higher resolution data is stored in smaller, deeper nodes (leaves), while coarser data is stored in larger, shallower nodes. This allows the renderer to select the appropriate resolution based on distance from the camera or visual importance [G].

## Subtopics

### [Ray-marching reference (min-max octree accelerated)](1554-ray-marching-reference-min-max-octree-accelerated.md)
[Ray-marching reference (min-max octree accelerated)](1554-ray-marching-reference-min-max-octree-accelerated.md) uses a min-max octree to accelerate direct isosurface visualization. It differs from siblings by guaranteeing continuous isosurfaces via binary search refinement within leaf nodes, avoiding the topological artifacts of mesh extraction. It is chosen when a baseline for general scalar volume isosurface rendering with low auxiliary memory overhead is required.

### [ExaBricks Rendering with RTX BVH](2577-exabricks-rendering-with-rtx-bvh.md)
[ExaBricks Rendering with RTX BVH](2577-exabricks-rendering-with-rtx-bvh.md) renders Adaptive Block-Refined (ABR) data using hardware-accelerated RTX BVH traversal. It distinguishes itself by replacing software-based tree traversal with dedicated RT Cores, avoiding complex control flow in shaders. It is chosen for interactive editing of transfer functions on large ABR datasets, provided the BVH can be rebuilt efficiently.

### [Brick Size and Octree Limit Tuning](4263-brick-size-and-octree-limit-tuning.md)
[Brick Size and Octree Limit Tuning](4263-brick-size-and-octree-limit-tuning.md) focuses on empirically selecting optimal brick dimensions and octree depth limits to mitigate hardware-specific bottlenecks like 3D texture sampling penalties. It differs from siblings by treating brick size and octree limit as orthogonal parameters to be tuned for specific GPU architectures. It is chosen when optimizing GPU-based volume rendering for specific hardware constraints.

### [Residency Octree](4950-residency-octree.md)
[Residency Octree](4950-residency-octree.md) is a hybrid structure for out-of-core DVR that decouples cache residency from spatial subdivision. Unlike standard octrees, each node represents a resolution-independent spatial region and references all resolution levels. It enables efficient mixing of channel resolutions and cache-miss compensation. It is chosen for multi-volume out-of-core rendering where flexible mixed-resolution access is critical.

### [Multiresolution with Octree Bricking](6106-multiresolution-with-octree-bricking.md)
[Multiresolution with Octree Bricking](6106-multiresolution-with-octree-bricking.md) encodes volumetric data at varying resolutions into a spatial octree, explicitly linking octree depth to resolution levels. It relies on software-managed out-of-core loading. It is chosen for large datasets exceeding texture memory where adaptive resolution based on viewer distance is needed.

### Block-based GPU Volume Rendering with Acceleration
An efficient GPU rendering algorithm for large CT datasets that subdivides the volume into uniform blocks and accelerates rendering by combining early ray termination, empty-space skipping, and visibility culling to render only visible parts [S2, unofficial]. It achieves 3-4 frames per second on a 512 × 304 × 1878 dimension dataset using a Pentium 4 2.4 GHz PC with an NVIDIA Geforce 6600 [S2, unofficial].

### Block-based Alpha Histogram Construction
Uses spatial coherence to enhance histograms and transfer function design. The standard block size is $8^3$ voxels, but the minimum recommended size is $6^3$ to obtain good effect from spatial coherence; features of the same scale as the block size are favored [S3]. Anisotropic blocks (e.g., 3-4 times larger in the z-direction) can be used if tissue has a larger extent in a certain dimension [S3].

## Background
Hierarchical grids are considered a good compromise preferred by many practitioners for volume rendering acceleration [S1, unofficial]. The concept of hierarchical bounding volumes, where children are contained within parents, has roots in general ray tracing algorithms, including recursive inorder traversal methods by Kaplan, Arvo, and Jansen [S1, unofficial]. The specific application to volume rendering involves subdividing the volume into uniform blocks to take advantage of early ray termination and empty-space skipping, as demonstrated in efficient GPU rendering algorithms for large CT datasets [S2, unofficial]. The use of block-based structures for alpha-histograms to enhance transfer function design was further explored in the context of spatial coherence [S3].

## Sources
[S1] PPT - Ray Tracing 2: Acceleration PowerPoint Presentation, free ..., https://www.slideserve.com/chaim/ray-tracing-2-acceleration, fetched 2026-10-26 (unofficial)
[S2] Academia | (PDF) FAST VOLUME RENDERING USING A SHEAR-WARP FACTORIZATION ..., https://www.academia.edu/6981092/FAST_VOLUME_RENDERING_USING_A_SHEAR_WARP_FACTORIZATION_OF_THE_VIEWING_TRANSFORMATION, fetched 2026-10-27 (unofficial)
[S3] The alpha -histogram: Using Spatial Coherence to Enhance Histograms and Transfer Function Design, https://doi.org/10.2312/vissym/eurovis06/227-234, fetched 2026-10-26
