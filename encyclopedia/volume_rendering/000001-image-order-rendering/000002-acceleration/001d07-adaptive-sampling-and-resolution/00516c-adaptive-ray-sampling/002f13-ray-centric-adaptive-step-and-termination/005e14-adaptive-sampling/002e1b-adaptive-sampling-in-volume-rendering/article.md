# Adaptive Sampling in Volume Rendering

Also called: Adaptive Sampling Strategy, Adaptive Sampling with Bricking

## Overview
Adaptive sampling is a technique where the fragment shader enumerates non-empty leaves in front-to-back order and chooses a step size matching local voxel density, adapting sampling density to brick resolution [S1]. It sets itself apart from its siblings by explicitly linking the discrete step size to octree levels to enable compact 2D transfer functions, whereas [Adaptive Sampling (Sparse Grid)](1656-adaptive-sampling-sparse-grid.md) uses sparse data structures and [Adaptive Sampling with Relative Standard Error](4545-adaptive-sampling-with-relative-standard-error.md) uses statistical error bounds.

## How it works
**Inputs and Outputs:**
*   Input: Ray origin, ray direction, octree structure of the volume, isovalue (for isosurface) or transfer function parameters (for direct volume rendering).
*   Output: Final color and depth for the fragment.

**Data Structures:**
*   **Octree:** Hierarchical structure where each node represents a brick. Leaves contain voxel data.
*   **Transfer Function:** A compact precomputed 2D texture. The first dimension maps to scalar value; the second dimension maps to the octree level.

**Steps:**
1.  **Traversal:** Perform stackless traversal to enumerate all non-empty leaves pierced by the ray in front-to-back order [S1].
2.  **Entry/Exit Determination:** Determine entry and exit points of the ray within the current brick during the octree traversal process [S1].
3.  **Step Size Selection:** Upon entering a leaf with data, choose a step size matching the local voxel density. The stepping is done by a discrete number of intervals directly associated with octree levels [S1].
4.  **Accumulation:**
    *   **For Isosurface Rendering:** Look for intervals that bracket the selected isovalue [S1].
    *   **For Semitransparent Direct Volume Rendering:** Implement a Phong illumination model with boundary enhancement and view-dependent transparency [S1]. Accumulate color and opacity information depending on the active rendering mode [S1].
5.  **Transfer Function Lookup:** Use the compact precomputed 2D transfer function. The octree level is used for the second dimension, adjusting opacity and color weighting accordingly [S1].
6.  **Termination Check:** When the ray exits the block, check the current accumulated opacity. If it exceeds a threshold (0.99 in tests), terminate the ray and update the fragment depth [S1].

**Parameters:**
*   **Opacity Threshold:** 0.99 [S1].
*   **Transfer Function Dimensions:** 2D, with the second dimension corresponding to octree levels [S1].

**Edge Cases and Pitfalls:**
*   **Interactive Modification:** Both the isovalue and transfer function can be modified interactively, requiring efficient lookup updates [S1].
*   **Precision:** The method relies on the octree structure's ability to define discrete intervals; precision is limited by the finest brick resolution [S1].

## Subtopics

### Voxel-Space Coherency Sampling
A method using voxel-space coherency to start sampling at low frequency (large steps) and increasing density where value differences are large [S2, unofficial]. It starts sampling the volume with large steps between sample points [S2, unofficial]. If a large value difference is encountered between two adjacent samples, additional samples are taken between them to resolve ambiguities in these high-frequency regions [S2, unofficial]. It efficiently lowers the sampling rate in areas where only small contributions of opacities are made, or in regions where the volume is homogeneous [S2, unofficial]. This is achieved by employing a pyramid of volumes that decode the minimum and maximum voxel value in a small neighborhood, as well as the distance between these measures [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of non-empty leaves and the adaptive step size. Adaptive sampling reduces the number of samples in homogeneous or empty regions, improving performance.
*   **Memory Complexity:** Requires storage for the octree structure and the 2D transfer function. The transfer function size depends on the number of octree levels.
*   **Strengths:** Enables fully adaptive GPU ray casting with empty space skipping and adaptive sampling [S1]. Supports interactive modification of isovalue and transfer function [S1].
*   **Limitations:** Performance may degrade in highly heterogeneous volumes with frequent level transitions. The discrete interval stepping may miss thin features if the brick resolution is too coarse.
*   **When to Use:** Suitable for interactive out-of-core volume rendering where memory bandwidth is a bottleneck and the volume has hierarchical structure [S1].
*   **When Not to Use:** Not suitable for volumes without hierarchical structure or where extremely high precision is required regardless of performance.

## Background
Originated in the context of single-pass GPU ray casting frameworks for interactive out-of-core rendering [S1]. The technique leverages stackless traversal and octree structures to efficiently manage memory and computation [S1]. Voxel-space coherency sampling was previously used by Van Walsum et al. [S2, unofficial].

## Sources
[S1] PDF A single-pass GPU ray casting framework for interactive out-of-core ..., https://www.crs4.it/vic/data/papers/cgi2008-movr.pdf, fetched 2026-10-29
[S2] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
