# Sorting Algorithms


## Overview
Sorting Algorithms in volume rendering splatting reorder volumetric primitives (typically 3D Gaussian splats) by depth to ensure correct front-to-back alpha blending [G]. This solves the order-dependent transparency problem inherent in splatting, where incorrect draw order leads to visual artifacts [G]. Unlike siblings that may use partial ordering or hardware-specific tricks, this family focuses on generating a strict depth-sorted index buffer for the rasterizer [G].

## Shared principles
Builds on the splatting family's shared mechanism of treating volumetric data as transformed primitives rasterized onto the image plane [Parent](8126-splat-sorting-and-composition.md).

## Subtopics

### [CPU-Based Splat Sorting](2097-cpu-based-splat-sorting.md)
Reorders 3D Gaussian splats by depth on the CPU to enable correct front-to-back alpha blending. Offloads sorting from the GPU to avoid compute stalls, making it suitable when GPU compute shaders are unavailable or underutilized.

### [Cascaded 2-way Merge for Global Splat Sorting](7689-cascaded-2-way-merge-for-global-splat-sorting.md)
Merges pre-sorted splat lists from multiple independent renderers into a single globally depth-sorted order buffer using a cascaded binary merge tree. Chosen for unified multi-scene rendering to prevent incorrect depth ordering and excessive overdraw.

### [Half-Angle Slicing with Odd-Even Merge Sort](14210-half-angle-slicing-with-odd-even-merge-sort.md)
Sorts particles along a view-dependent half-angle axis using an odd-even merge sort, then renders them in slices. Sets itself apart by decoupling the sorting axis from the view direction to allow efficient accumulation of opacity from the light's perspective for volumetric shadows.

### [Bitonic Sort](14217-bitonic-sort.md)
A data-independent network sorting algorithm used on the GPU to achieve exact depth ordering of splatted voxels per frame. Chosen over partial-ordering methods because volume rendering requires strict front-to-back or back-to-front ordering for correct transparency compositing.

### [Index Sorting](15067-index-sorting.md)
Produces a permutation of indices rather than reordering data values, maintaining data association during sorting. Chosen when voxel attributes (density, color) must remain linked to their spatial keys without packing keys and indices into a single register.

### Back-to-front Splat Sorting
A compositing technique for splats (oriented ellipses) that sorts them back-to-front relative to a camera viewpoint to ensure correct translucent blending [S1, unofficial]. When `cameraPositionProvider` is null, the `SplatNode` automatically sorts its splats for the scene camera's current position [S1, unofficial]. If `cameraPositionProvider` is provided, the node sorts splats for the viewpoint returned by that provider instead of the scene camera [S1, unofficial]. The sorting is performed on a background thread whenever the camera moves [S1, unofficial]. Without a `cameraPositionProvider` (and thus without sorting), the compositing order remains as loaded, causing visual popping artifacts when orbiting the camera [S1, unofficial].

### Record Sorting
A sorting method that shuffles full records (data) rather than just indices, used to increase cache coherency with frequently-accessed sorted data [S2, unofficial]. It generates a simple spatial index for each input value, performs index sorting on the key-index pair array, and then performs the full shuffling step for record sorting if needed [S2, unofficial]. Record sizes tested are 32, 64, and 128 bits, breaking a regular array of records into a structure of arrays with typical length types [S2, unofficial]. The final shuffle step involves accessing data in arbitrary-like order from the original array while outputting sorted data sequentially, leading to non-coalesced reading which is a major bottleneck [S2, unofficial]. Using texture fetching provides a factor of three improvement in shuffling performance over typical mapping using global memory for uniformly distributed inputs [S2, unofficial].

## Background
Originating from the need to resolve order-dependent transparency in splat-based volume rendering, these algorithms evolved alongside 3D Gaussian Splatting [G]. Early implementations relied on CPU sorting due to GPU limitations, while modern approaches leverage GPU subgroups and merge networks for real-time global sorting [G].

## Sources
[S1] feat(sceneview): sort splats for the scene camera by default and add …, https://github.com/sceneview/sceneview/commit/5f2a4dd1f683ddb444669be54b78ce7913b55b70, fetched 2026-10-30 (unofficial)
[S2] Fast 4-way parallel radix sorting on GPUs, https://www.sci.utah.edu/~csilva/papers/cgf.pdf, fetched 2026-10-29 (unofficial)
