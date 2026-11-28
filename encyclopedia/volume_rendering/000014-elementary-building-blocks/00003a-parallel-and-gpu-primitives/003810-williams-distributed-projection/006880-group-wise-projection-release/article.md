# Group-wise Projection Release

Also called: Compact gradients with group-wise release

## Overview
Group-wise projection release is a memory optimization for 3D Gaussian Splatting training that reduces the working set by processing projection in 3x3 groups and releasing them immediately, constrained by a 128 MiB Leaf budget [S1, unofficial]. It sets itself apart from its parent, Williams Distributed Projection, by introducing fine-grained release of projection data to lower peak memory usage, at the cost of increased warm-pass time [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D Gaussian parameters (position, covariance, color, opacity), camera view matrices, and a Leaf memory budget.
*   **Output:** Gradients for Gaussian parameters and updated Gaussian attributes.

**Data Structures:**
*   **Leaf Budget:** A fixed memory limit of 128 MiB [S1, unofficial].
*   **Projection Groups:** Data is partitioned into 3x3 groups of pixels/tiles [S1, unofficial].
*   **Parameter Gradients:** Accumulated gradients for Gaussian parameters, stored densely [S1, unofficial].

**Steps:**
1.  **Partition Projection:** Divide the projection task into 3x3 groups [S1, unofficial].
2.  **Process Group:** Compute projection and gradients for the current 3x3 group [S1, unofficial].
3.  **Accumulate Gradients:** Add the computed gradients to the dense parameter gradient buffer [S1, unofficial].
4.  **Release Projection:** Immediately release the memory associated with the current 3x3 projection group, ensuring the working set does not exceed the 128 MiB Leaf budget [S1, unofficial].
5.  **Repeat:** Iterate through all projection groups [S1, unofficial].
6.  **View Update:** After all groups are processed, perform the view update using the accumulated parameter gradients [S1, unofficial].

**Formulas:**
*   Let $G$ be the set of 3x3 projection groups.
*   Let $M_{leaf}$ be the Leaf budget (128 MiB) [S1, unofficial].
*   Let $P_g$ be the memory footprint of projection data for group $g \in G$.
*   Let $G_{param}$ be the dense parameter gradient buffer.
*   For each group $g$:
    1.  Compute projection data $D_g$.
    2.  Compute gradients $\nabla_g$.
    3.  $G_{param} \leftarrow G_{param} + \nabla_g$.
    4.  Release $D_g$.
    5.  Check: $Memory(G_{param}) \le M_{leaf}$? (Note: The base memory footprint is formed by $G_{param}$ accumulating until view update) [S1, unofficial].

**Parameters:**
*   **Group Size:** 3x3 [S1, unofficial].
*   **Leaf Budget:** 128 MiB [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Accumulated Gradients:** Parameter gradients are accumulated until the view update, forming the base memory footprint. This can become the dominant memory consumer if not managed [S1, unofficial].
*   **Precision:** Not given in the sources.
*   **Boundaries:** The 128 MiB budget must be respected; exceeding it may cause OOM errors [S1, unofficial].

## Subtopics

### Candidate-indexed Projection
Candidate-indexed projection is a variant that uses candidate indexing to reduce memory footprint during the backward pass [S1, unofficial]. It applies candidate-indexed projection to lower the working set after Auto Leaf, lowering the working set to 3.27 GiB and making dense parameter gradients the main memory occupant [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Raises warm-pass time to 12.69 seconds [S1, unofficial].
*   **Memory Complexity:** Reduces working set to 1.31 GiB [S1, unofficial].
*   **Measured Performance:** On a Bicycle 30K SH3 model with ~6M Gaussians, this configuration cuts the working set by 83.3% relative to Original but increases time by 14.2% (from 11.35s to 12.69s) [S1, unofficial].
*   **Strengths:** Significant reduction in peak memory usage, enabling higher resolution training.
*   **Limitations:** Increased warm-pass time; parameter gradients become the main memory occupant [S1, unofficial].
*   **When to Use:** When memory is the primary constraint and higher resolution training is required.
*   **When Not to Use:** When training speed is more critical than memory usage.

## Background
Originating from BoundedGS: Memory-Efficient High-Resolution Training for 3D Gaussian Splatting [S1, unofficial], this method addresses the memory bottlenecks in high-resolution 3D Gaussian Splatting training by coordinating buffer lifetimes for rasterization, projection, and gradients [S1, unofficial].

## Sources
[S1] BoundedGS: Memory-Efficient High-Resolution Training for 3D Gaussian Splatting, https://arxiv.org/abs/2610.34233v1, fetched 2026-10-26 (unofficial)
