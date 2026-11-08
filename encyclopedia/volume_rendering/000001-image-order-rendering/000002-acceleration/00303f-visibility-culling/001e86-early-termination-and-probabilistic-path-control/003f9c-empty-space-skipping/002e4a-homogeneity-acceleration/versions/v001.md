# Homogeneity Acceleration

Also called: Denskin-Hanrahan acceleration, Range pyramid traversal

## Overview
Homogeneity acceleration is an image-order volume rendering technique that extends empty-space skipping by approximating the contribution of uniform, non-empty regions instead of sampling them voxel-by-voxel [S1, S2, unofficial]. It distinguishes itself from its sibling [Empty-space skipping](1002-empty-space-skipping.md) by handling regions that contain data but lack internal variation, thereby reducing computational load where presence acceleration alone would still require per-sample evaluation [S1, S2, unofficial].

## How it works
**Inputs:** A scalar volume dataset $V$ and a set of rays cast from the image plane into the volume.
**Outputs:** The final color and opacity for each pixel.

**Data Structures:**
The method relies on a **Range Pyramid**, a hierarchical data structure built over the volume. Unlike the binary pyramids used in presence acceleration, each node in the range pyramid stores two values:
1.  $min$: The minimum scalar value found in the subvolume covered by the node.
2.  $max$: The maximum scalar value found in the subvolume covered by the node.

The pyramid is constructed such that each level contains the min/max values of the subvolumes at the level immediately below it [S1, S2, unofficial].

**Algorithm Steps:**
1.  **Pyramid Construction:** Construct the range pyramid for the volume. For the leaf level (the voxel grid), $min$ and $max$ are simply the scalar value of each voxel. For higher levels, compute $min = \min(min_{children})$ and $max = \max(max_{children})$ [S1, S2, unofficial].
2.  **Ray Traversal:** For each ray, traverse the range pyramid from the root to the leaves.
3.  **Homogeneity Check:** At each node visited, compare the stored $min$ and $max$ values.
    *   If $max - min < \epsilon$ (where $\epsilon$ is a small threshold), the region is considered **homogeneous**.
    *   If $max - min \ge \epsilon$, the region is **heterogeneous** (or contains empty space, handled by the underlying empty-space skipping logic).
4.  **Approximate Evaluation:** If a node is identified as homogeneous:
    *   Do not descend to its children.
    *   Perform an approximate evaluation of the entire subvolume covered by this node. This typically involves treating the region as having a constant scalar value (e.g., the average of $min$ and $max$, or just $min$) and computing the accumulated color and opacity for the ray segment passing through this subvolume in one step [S1, S2, unofficial].
5.  **Detailed Sampling:** If a node is heterogeneous, descend to its children and repeat the check. If a leaf node is reached and is homogeneous, sample the single voxel. If it is heterogeneous (which implies non-uniformity within a single voxel, or potentially empty space if the underlying volume has empty voxels), sample the voxel directly [S1, S2, unofficial].

**Parameters:**
*   $\epsilon$: The homogeneity threshold. Not given in the sources. Typical values depend on the data range and transfer function sensitivity.

**Edge Cases and Pitfalls:**
*   **Precision:** Comparing floating-point $min$ and $max$ values requires care. A fixed $\epsilon$ may not scale well across different data ranges. [G] A relative threshold or normalized threshold based on the data range is often recommended in practice, though not explicitly detailed in the source notes.
*   **Approximation Error:** The approximate evaluation assumes uniformity. If the transfer function is highly non-linear, even small variations within the "homogeneous" range can cause significant color changes. The approximation may introduce visual artifacts if $\epsilon$ is too large. [G] The source notes do not specify the exact formula for the approximate evaluation (e.g., whether to use $min$, $max$, or $(min+max)/2$). [G] Implementers must choose a representative value for the homogeneous region.

## Subtopics

### Presence Acceleration
The predecessor technique that uses binary pyramids to skip only completely empty regions. Homogeneity acceleration builds on this by adding range information to handle uniform non-empty regions. See [Empty-space skipping](1002-empty-space-skipping.md).

### Range Pyramid
The specific data structure used. It stores min/max values instead of binary flags. This allows for the detection of homogeneity. See [Empty-space skipping](1002-empty-space-skipping.md) for the general concept of hierarchical acceleration structures.

## Operating characteristics
**Time Complexity:** Reduces the number of ray-marching steps in homogeneous regions from $O(N)$ to $O(1)$ per region, where $N$ is the number of voxels in the region. The construction time is $O(V)$ where $V$ is the total number of voxels.
**Memory Complexity:** Requires storing two scalar values per node in the pyramid. The size is proportional to the number of nodes in the pyramid, which is $O(V)$ for a full pyramid.
**Measured Performance:** Denskin and Hanrahan reported improvements over presence acceleration, particularly in volumes with large uniform regions. Specific speedup factors are not given in the sources.
**Strengths:** Effective for volumes with large homogeneous areas (e.g., background, uniform tissues). Reduces redundant sampling.
**Limitations:** Approximation may reduce visual fidelity if the transfer function is sensitive to small value changes. Requires additional memory for the range pyramid.
**When to use:** When the volume contains significant homogeneous regions and the transfer function is relatively smooth or linear in those regions.
**When not to use:** When the volume is highly heterogeneous or the transfer function is highly non-linear, as the approximation error may be visually unacceptable.

## Background
Homogeneity acceleration was introduced by Denskin and Hanrahan in 1992 [S2, unofficial]. It was an improvement over presence acceleration, which only skipped empty regions. The technique is named after its authors and is also referred to as Denskin-Hanrahan acceleration or range pyramid traversal [S1, S2, unofficial]. It is primarily used in interactive volume rendering for medical data visualization, where large uniform regions are common [S1, S2, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
