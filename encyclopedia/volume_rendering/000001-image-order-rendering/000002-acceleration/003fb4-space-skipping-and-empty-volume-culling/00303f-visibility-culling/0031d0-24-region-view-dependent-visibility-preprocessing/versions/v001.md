# 24-Region View-Dependent Visibility Preprocessing

Also called: 24-Region Decomposition, 24-Region Visibility Culling

## Overview
A preprocessing technique that decomposes the domain of viewing directions into 24 regions on a directional cube to extract a lower number of potentially visible voxels compared to the standard 6-region decomposition, improving extraction rate and rendering speed [S1, unofficial]. It sets itself apart from [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md) by using a directional decomposition strategy rather than orthogonal projection maps, and from [Adaptive Multi-Step Empty Voxel Skipping](6289-adaptive-multi-step-empty-voxel-skipping.md) by focusing on surface boundary extraction rather than empty space traversal.

## How it works
**Inputs:**
*   Volume dataset $V$ with dimensions $N_x \times N_x \times N_z$ (assuming square cross-section for simplicity, or general $N_x, N_y, N_z$).
*   Binary occupancy grid $b(i, j, k)$ indicating whether voxel $(i, j, k)$ is occupied (1) or empty (0).

**Outputs:**
*   A set of 24 precomputed visibility masks (or a single data structure indexed by region), where each mask contains the indices of potentially visible boundary voxels for that specific viewing direction region.

**Data Structures:**
*   **Visibility Mask:** A boolean array or list of voxel coordinates. For a specific region, it stores $v(i, j, k) = 1$ for potentially visible voxels.
*   **Occupancy Grid:** $b(i, j, k) \in \{0, 1\}$.

**Steps:**
1.  **Decomposition:** Divide the viewing direction domain into 24 regions. This is done by assuming the principal component of the viewing direction is the $z$-coordinate. The remaining $x$ and $y$ components define the region based on their signs. There are 4 sign combinations for $(x, y)$: $(+,+), (+,-), (-,+), (-,-)$. For each combination, there are 6 possible orientations of the principal axis relative to the grid (positive/negative z, and potentially x/y if the principal axis assumption is relaxed, but the source specifies assuming $z$ is principal and decomposing based on $x,y$ signs, resulting in 24 regions total, implying a more granular decomposition than just 4 quadrants, likely involving the 6 faces of a cube split into 4 triangles each, or similar directional partitioning [S1, unofficial]).
2.  **Visibility Function Evaluation:** For each region, compute the visibility function $v(i, j, k)$ for all voxels. The source provides the specific logic for the case where $x$ and $y$ components are positive:
    $$v(i, j, k) = \begin{cases} 1 & \text{if } k = 0 \text{ or} \\ \{ \exists l, m \mid i - 1 \leq l \leq i, \\ j - 1 \leq m \leq j, \\ v(l, m, k - 1) = 1, b(l, m, k - 1) = 0 \} \\ 0 & \text{otherwise.} \end{cases}$$
    This function checks if a voxel is on the boundary ($k=0$) or if it is occluded by a previously visible voxel in the "shadow" cast by the viewing direction. Specifically, it checks only four voxels $(l, m)$ in the plane $k-1$ that could hide the current voxel $(i, j, k)$ from the specific viewing direction. This is a stronger condition than the general 6-region case which checks nine voxels [S1, unofficial].
3.  **Preprocessing:** Repeat the visibility function evaluation for all 24 regions. Store the resulting sets of potentially visible voxels.
4.  **Rendering Selection:** During rendering, determine the current viewing direction's region. Select the precomputed visibility mask corresponding to that region. Use this mask to limit the voxel sampling to only the potentially visible voxels [S1, unofficial].

**Parameters:**
*   **Region Count:** 24.
*   **Principal Axis Assumption:** $z$-coordinate (can be extended to arbitrary principal directions [S1, unofficial]).

**Edge Cases and Pitfalls:**
*   **Preprocessing Time:** The preprocessing time increases significantly compared to the 6-region method because 24 masks must be computed [S1, unofficial].
*   **Storage Requirements:** Storage requirements increase proportionally to the number of regions (24x vs 6x the size of the visibility mask) [S1, unofficial].
*   **Directional Ambiguity:** If the viewing direction is nearly parallel to the $x$ or $y$ axis, the assumption that $z$ is the principal component may break down, requiring a different decomposition or principal axis selection [S1, unofficial].

## Subtopics

### View-Dependent Preprocessing
A lightweight preprocessing step performed by each rendering processor before local rendering to optimize the subsequent image compositing step and select the appropriate octree level for adaptive rendering [S2]. It involves quantization (32-bit to 8-bit), central differencing for gradient vectors, and one-side differencing for temporal domain enhancement [S2]. The octree level is chosen based on image resolution, data resolution, and a user-specified limit on elements projected per pixel [S2]. This is distinct from the 24-region method as it focuses on data precision and hierarchical level selection rather than directional visibility masking [S2].

## Operating characteristics
*   **Extraction Rate:** Decomposing into 24 regions extracts significantly fewer voxels than 6 regions. On a low-resolution dataset, it extracts 1.8% of voxels compared to 3.4% for 6 regions. On a high-resolution volume, it extracts 0.9% compared to 1.8% for 6 regions [S1, unofficial].
*   **Preprocessing Time:** Increases due to the computation of 24 visibility masks. For a $256 \times 256 \times 225$ volume, preprocessing took 55-56 seconds [S1, unofficial].
*   **Rendering Speed:** Significantly faster due to the lower extraction rate. Intermediate image generation time dropped from 213 msec (Phong, 6-region implied) to 18 msec (Lambert, 24-region) in some configurations, though direct comparison is complex due to shading model differences [S1, unofficial].
*   **Storage:** Higher storage requirements for the 24 precomputed masks [S1, unofficial].
*   **When to Use:** When rendering speed is critical and preprocessing time can be amortized or is acceptable. Particularly effective for high-resolution volumes where the reduction in extracted voxels is more pronounced [S1, unofficial].
*   **When Not to Use:** When memory is constrained or the viewing direction changes frequently, requiring frequent recomputation or large storage for all 24 masks [S1, unofficial].

## Background
Originated in the context of interactive volume rendering techniques for medical data visualization [S1, unofficial]. The method extends surface-rendering techniques to arbitrary viewing directions by decomposing the viewing direction domain. It was developed to improve extraction rates and rendering speeds by reducing the number of potentially visible voxels through a more granular directional decomposition [S1, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] I/O Strategies for Parallel Rendering of Large Time-Varying Volume Data, https://doi.org/10.2312/egpgv/egpgv04/031-040, fetched 2026-10-26
