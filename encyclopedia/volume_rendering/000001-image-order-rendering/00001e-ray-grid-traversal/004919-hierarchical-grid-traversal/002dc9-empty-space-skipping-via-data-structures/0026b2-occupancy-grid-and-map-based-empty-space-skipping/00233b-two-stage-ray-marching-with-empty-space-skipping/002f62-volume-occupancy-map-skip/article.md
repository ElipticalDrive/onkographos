# Volume Occupancy Map Skip

Also called: Empty Voxel Skip, 空体素跳过技术

## Overview
Volume Occupancy Map Skip is a rendering optimization that accelerates ray traversal in sparse volumetric data by bypassing empty regions. It builds on [Two-stage Ray Marching with Empty-Space Skipping](2985-two-stage-ray-marching-with-empty-space-skipping.md) by using a preprocessed occupancy map to jump directly from one non-empty voxel to the next, avoiding density evaluations in empty space. Unlike [PDF Light Weight Space Leaping](5848-pdf-light-weight-space-leaping.md), which typically uses hierarchical structures like octrees for space leaping, this method relies on a flat or sparse occupancy grid to identify empty voxels, offering a trade-off between memory overhead and traversal simplicity for specific sparse datasets.

## How it works
**Inputs and Outputs:**
*   Input: A 3D volumetric dataset (density field) and a set of rays.
*   Output: Rendered image or accumulated density/transmittance.
*   Auxiliary: A preprocessed Volume Occupancy Map (binary mask).

**Data Structures:**
*   **Volume Occupancy Map:** A binary volume (mask) of the same dimensions as the density field. Each voxel stores a single bit or byte indicating occupancy: `1` (occupied/non-empty) or `0` (empty). [S2] specifies this as a "binary volume." [S1, unofficial] refers to it generally as a "Volume Occupancy Map."
*   **Density Field:** The primary volumetric data containing density values.

**Algorithm Steps:**
1.  **Preprocessing (Offline/Initial Stage):**
    *   Analyze the volumetric dataset to determine which voxels contain non-zero density or significant content.
    *   Construct the Volume Occupancy Map. For each voxel $v$ in the grid, set $Map[v] = 1$ if the voxel is occupied, else $Map[v] = 0$. [S1, unofficial][S2]
    *   In the context of dynamic scenes (e.g., ViSNeRF), this mask may be updated based on empty voxels determined from multiple static scenes. [S2]

2.  **Ray Traversal (Runtime):**
    *   For each ray, perform standard ray-box intersection to find the entry point into the volume bounding box.
    *   Initialize ray parameters (position, direction, step size).
    *   **Loop:**
        a.  Check the current voxel's occupancy status in the Volume Occupancy Map.
        b.  **If Empty ($Map[v] == 0$):**
            *   Skip sampling the density function.
            *   Jump to the next non-empty voxel. The method of jumping depends on the implementation:
                *   *Naive:* Increment voxel coordinates until a non-empty voxel is found.
                *   *Optimized:* Use a precomputed list of occupied voxel indices or a sparse structure (like a linked list of occupied voxels) to jump directly. [S1, unofficial] implies "jumping directly to the next non-empty voxel," suggesting an efficient lookup rather than linear scan, though the exact data structure for the "jump" is not detailed in the sources. [G] A common implementation uses a sorted list of occupied voxel coordinates or a sparse hash map for $O(1)$ or $O(\log N)$ lookup.
        c.  **If Occupied ($Map[v] == 1$):**
            *   Sample the density function at the current ray position.
            *   Accumulate color and opacity (standard volume rendering integration).
            *   Advance the ray by a step size (e.g., 0.5 voxel spacing as noted in medical imaging contexts). [S1, unofficial]
            *   Update ray position.
        d.  **Termination:**
            *   Stop if the ray exits the volume bounding box.
            *   Stop if transmittance drops below a threshold (early ray termination).

**Formulas:**
*   Occupancy Check: $Occupancy(v) = \begin{cases} 1 & \text{if } \rho(v) > \epsilon \\ 0 & \text{otherwise} \end{cases}$ where $\rho(v)$ is the density at voxel $v$ and $\epsilon$ is a small threshold. [G]
*   Ray Advancement: $P_{next} = P_{current} + \Delta t \cdot \vec{d}$, where $\Delta t$ is the step size. [S1, unofficial]

**Parameters:**
*   **Step Size:** Typically set to 0.5 voxel spacing to balance detail and performance in medical imaging. [S1, unofficial]
*   **Threshold ($\epsilon$):** Not given in the sources. [G] A small value (e.g., $10^{-6}$) is typical to define "empty."

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point comparisons for occupancy should use a small epsilon to avoid missing thin structures due to numerical noise. [G]
*   **Boundaries:** Ensure the occupancy map covers the entire volume bounding box. Rays exiting the box must be handled correctly. [S1, unofficial]
*   **Dynamic Scenes:** For dynamic scenes, the occupancy map must be updated efficiently. [S2]
*   **Sparse Data:** If the data is extremely sparse, a flat occupancy map may consume significant memory. [S1, unofficial] mentions "sparse clouds" and "sparse voxel" structures in comparisons, implying that for very large sparse volumes, a sparse representation of the occupancy map (e.g., using a hash table or linked list) might be necessary, though the core method described uses a "Volume Occupancy Map."

## Subtopics

### Voxel skipping
An efficiency optimization used in training neural radiance fields (e.g., ViSNeRF) to skip sampling at empty voxels along valid rays. It creates a binary mask volume after an initial training stage to record empty voxels, improving subsequent training efficiency. For dynamic scenes, the mask is updated based on empty voxels from multiple static scenes. [S2]

## Operating characteristics
*   **Time Complexity:** Depends on the implementation of the "jump." If using a linear scan through the occupancy map, it is $O(N)$ in the worst case (where $N$ is the number of voxels along the ray). If using a precomputed list of occupied voxels, it is $O(K)$ where $K$ is the number of occupied voxels along the ray. [S1, unofficial] notes that 3D-DDA (which this builds on) has complexity $O(n)$ where $n$ is the number of voxels traversed, compared to $O(N^3)$ for brute force.
*   **Memory Complexity:** $O(V)$ where $V$ is the total number of voxels in the grid, for the occupancy map. [S1, unofficial] compares memory usage: "Low" for brute force, "High" for octrees, "Medium" for 3D-DDA, and "Extremely High" for SVO. The occupancy map adds a memory overhead proportional to the volume size.
*   **Performance:** In a medical imaging project, this technique increased rendering frame rate by 3.2x. [S1, unofficial]
*   **Strengths:** Simple to implement; effective for sparse data; reduces expensive density evaluations.
*   **Limitations:** Requires preprocessing; memory overhead for the occupancy map; less effective for dense data.
*   **When to use:** Sparse volumetric data (e.g., atmospheric clouds, medical CT with large empty regions). [S1, unofficial]
*   **When not to use:** Dense volumetric data where most voxels are occupied; memory-constrained environments.

## Background
The technique is described in the context of volume rendering optimizations, specifically for sparse data like atmospheric clouds or medical imaging. [S1, unofficial] cites a medical imaging project where it improved frame rates. [S2] applies a similar concept ("Voxel skipping") in the context of ViSNeRF for dynamic volumetric scenes, using a mask volume to skip empty voxels during training. The concept builds on earlier ray traversal algorithms like 3D-DDA. [S1, unofficial]

## Sources
[S1] 体渲染中的体素遍历技术:从理论到实践-百度开发者中心, https://developer.baidu.com/article/detail.html?id=6975905, fetched 2026-10-25 (unofficial)
[S2] ViSNeRF: Efficient Multidimensional Neural Radiance Field Representation for Visualization Synthesis of Dynamic Volumetric Scenes, https://doi.org/10.1109/pacificvis64226.2025.00029, fetched 2026-10-26
