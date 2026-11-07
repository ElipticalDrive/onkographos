# Bundle Parallelisation

Also called: GPU Integration Optimization, Ray Compaction and Sorting Optimization

## Overview
Bundle parallelisation is a parallelization scheme for coherent grid traversal where bundles of rays are processed in parallel [S1, unofficial]. It distributes the processing of ray bundles across multiple processing elements to accelerate volume rendering [S3]. Unlike slice parallelisation, which requires complex index tables to map bundles to slices, bundle parallelisation processes bundles directly, though it may result in redundant block fetches from neighbouring bundles [S1, unofficial].

## How it works
**Inputs:** Ray bundles, depth map, shadow map, and image dimensions [S3].
**Outputs:** Rendered image pixels with accumulated color and opacity [G].

**Data Structures:**
*   **Bundle:** A collection of rays sharing a common traversal path or depth range [G].
*   **Shadow Map:** Depth buffer used for visibility prefiltering [S3].
*   **Partial Sum Tree:** Stores accumulated transmittance and color for a slice [S3].
*   **Thread Blocks:** Allocated per slice [S3].

**Steps:**
1.  **Rectification:** Perform a data-parallel operation on the depth and shadow maps to prepare data for visibility queries. One thread is assigned per output pixel [S3].
2.  **Sorting and Prefiltering:** Combine visibility prefiltering and sorting into a single kernel. This step coalesces groups of 4 rays based on their depth information from the shadow map. A modified GPU radix sort is used to coalesce rays with similar depths as it walks over the input shadow map [S3].
3.  **Integration:** Allocate one thread block of 32 threads per slice [S3]. Assign one thread per slice for the integration phase because partial sum trees are too large to fit into shared memory [S3]. Store the partial sum trees in global memory [S3]. Execute the inner loop using the `while-if-if` strategy (UPDATE-TREE and QUERY-TREE) to minimize instruction divergence, ensuring at least half the threads in a warp perform useful work [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bundle_parallelisation(bundles, shadow_map, image_width, image_height):
    # Step 1: Rectification
    # Data-parallel operation on depth and shadow maps
    # One thread per output pixel
    rectified_shadow = [[shadow_map[y][x] for x in range(image_width)] for y in range(image_height)]

    # Step 2: Sorting and Prefiltering
    # Combine visibility prefiltering and sorting into a single kernel
    # Coalesce groups of 4 rays based on depth from shadow map
    sorted_bundles = []
    for bundle in bundles:
        coalesced_groups = []
        for i in range(0, len(bundle), 4):
            group = bundle[i:i+4]
            # Modified GPU radix sort: coalesce rays with similar depths
            group.sort(key=lambda ray: rectified_shadow[ray.y][ray.x])
            coalesced_groups.append(group)
        sorted_bundles.append(coalesced_groups)

    # Step 3: Integration
    # Allocate one block of 32 threads per slice
    # Assign one thread per slice
    # Store trees in global memory
    num_slices = image_height // 32
    global_memory_trees = [None] * num_slices
    rendered_image = [[0.0, 0.0, 0.0, 0.0] for _ in range(image_height * image_width)]

    for slice_idx in range(num_slices):
        # One thread per slice handles this slice
        tree = global_memory_trees[slice_idx]
        for bundle in sorted_bundles:
            # while-if-if strategy to reduce instruction divergence
            # UPDATE-TREE and QUERY-TREE operations
            while len(bundle) > 0:
                # UPDATE-TREE: accumulate transmittance and color
                # (Implementation details not given in sources)
                pass
                # QUERY-TREE: sample and composite
                # (Implementation details not given in sources)
                pass
    return rendered_image
```

**Parameters:**
*   **Block Size:** 32 threads per slice [S3].
*   **Ray Grouping:** Groups of 4 rays coalesced during sorting [S3].

**Edge Cases and Pitfalls:**
*   **Redundant Fetches:** Some blocks may be fetched multiple times from neighbouring bundles [S1, unofficial].
*   **Memory Constraints:** Partial sum trees are too large for shared memory on current hardware, necessitating global memory storage [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Global memory usage for partial sum trees due to size constraints [S3].
*   **Performance:** The method is described as "trivial" to implement on CPU with OpenMP (1 line of code) [S1, unofficial]. On GPU, it uses specific optimizations like the `while-if-if` strategy to ensure at least half the threads in a warp do useful work [S3].
*   **Strengths:** Efficient parallelisation of ray bundles; reduced instruction divergence in integration [S3].
*   **Limitations:** Potential for redundant block fetches from neighbouring bundles [S1, unofficial].

## Background
Bundle parallelisation is a technique used in coherent grid traversal for volume rendering [S1, unofficial]. It was implemented as part of a hierarchical volumetric shadow algorithm for single scattering [S3]. The approach was also noted in presentations on coherent grid traversal algorithms [S1, unofficial].

## Sources
[S1] A Coherent Grid Traversal Algorithm for Volume Rendering PowerPoint Presentation, https://slideserve.com/kineta/a-coherent-grid-traversal-algorithm-for-volume-rendering-powerpoint-ppt-presentation, fetched 2026-10-24 (unofficial)
[S2] A Coherent Grid Traversal Algorithm for Volume Rendering, https://www.slideserve.com/iliana-house/a-coherent-grid-traversal-algorithm-for-volume-rendering, fetched 2026-10-26 (unofficial)
[S3] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
