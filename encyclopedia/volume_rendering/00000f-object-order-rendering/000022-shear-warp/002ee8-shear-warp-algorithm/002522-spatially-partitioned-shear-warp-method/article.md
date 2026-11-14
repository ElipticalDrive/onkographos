# Spatially Partitioned Shear-Warp Method

Also called: 空間分割シアーワープ法, Space-Split Shear-Warp Method, Partitioned Shear-Warp

## Overview
The Spatially Partitioned Shear-Warp Method is a parallel volume rendering technique that partitions a 3D volume dataset into axis-aligned sub-volumes, renders each independently into intermediate baseplanes using the shear-warp factorization, composites these baseplanes using a binary tree to resolve occlusion, and finally warps the result to the image plane [S1, unofficial][S2, unofficial]. It distinguishes itself from its sibling, the Distributed Data Volume Rendering via Shear-Warp Factorisation, by explicitly defining a hierarchical rendering tree with configurable branching factors and specific optimizations for network transfer (window-encoding, RLE) and memory storage (multi-order copies) to handle large datasets on distributed clusters [S1, unofficial]. Unlike standard shear-warp which processes the whole volume, this method allows view-independent partitioning, meaning the volume is partitioned once in object coordinates and can be rendered from any viewing direction without repartitioning [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A volume dataset defined in object coordinates $(u, v, w)$, a viewing direction (projection matrix $M$), and a partitioning scheme dividing the volume into axis-aligned sets of voxels [S1, unofficial].
*   **Output:** A final 2D image on the screen [S1, unofficial].

**Data Structures**
1.  **Volume Data:** Stored in memory according to object coordinates $(u, v, w)$ [S1, unofficial]. The origin is at one corner of the volume; unit distance equals voxel spacing [S1, unofficial].
2.  **Partition Map:** Defines axis-aligned sets of voxels [S1, unofficial]. Sets can have arbitrary dimensions (1D, 2D, or 3D) [S1, unofficial].
3.  **Intermediate Baseplane:** A 2D buffer corresponding to a specific voxel set [S1, unofficial]. Its size is determined by the viewing direction and set dimensions [S1, unofficial]. Coordinates $(x_b, y_b, z_b)$ where $z_b=0$ is co-planar with the permuted $xy$-face of the volume [S1, unofficial].
4.  **Compositing Tree:** A binary tree structure used to combine baseplanes [S2, unofficial]. Each node represents a compositing operation [S2, unofficial].

**Coordinate Systems**
1.  **Object Coordinates $(u, v, w)$:** The native storage format [S1, unofficial].
2.  **Permuted Coordinates $(x, y, z)$:** The origin is repositioned to the vertex nearest the image plane [S1, unofficial]. The $z$-axis is the edge most nearly parallel to the viewing direction [S1, unofficial]. $x$ and $y$ are selected so voxel traversal is always in the positive direction [S1, unofficial].
3.  **Baseplane Coordinates $(x_b, y_b, z_b)$:** $z_b=0$ is co-planar with the $xy$-face in permuted coordinates [S1, unofficial]. The baseplane extends from the origin to a maximum point based on volume size and view angle [S1, unofficial].
4.  **Image Coordinates $(x_i, y_i)$:** The final screen coordinates [S1, unofficial].

**Transformation Matrices**
The viewing transformation $M$ is decomposed into:
$$ M = W \cdot S \cdot P $$
Where:
*   $P$ is the permutation matrix (Object $\to$ Permuted) [S1, unofficial].
*   $S$ is the shear matrix (Permuted $\to$ Baseplane) [S1, unofficial].
*   $W$ is the warp matrix (Baseplane $\to$ Image) [S1, unofficial].

**Algorithm Steps**
1.  **Partitioning:** Divide the volume data set, stored in object coordinates, into a plurality of axis-aligned sets of voxels [S1, unofficial]. The sets can have arbitrary dimensions [S1, unofficial].
2.  **View Selection:** Select an arbitrary viewing direction [S1, unofficial]. This can be done after partitioning, allowing different orientations without repartitioning [S1, unofficial].
3.  **Shear Rendering (Parallel):** For each sub-volume assigned to a processing node:
    *   Initialize the local intermediate baseplane [S1, unofficial]. If traversing back-to-front, initialize with a background color; if front-to-back, initialize with a foreground color [S1, unofficial].
    *   Iterate through voxels $(i, j, k)$ in the sub-volume [S2, unofficial].
    *   Calculate the sheared position $(p, q, k)$ in the baseplane coordinate system [S2, unofficial].
    *   Resample the voxel value at $(i, j, k)$ to the sheared position $(p, q)$ using a convolution kernel [S2, unofficial].
    *   Composite the resampled pixel into the local baseplane at memory address $pmem = (\lfloor q \rfloor \cdot \text{width} + \lfloor p \rfloor) \cdot 4$ [S2, unofficial].
4.  **Compositing (Tree-based):** Combine the intermediate baseplanes using a binary compositing tree to handle occlusion and priority [S2, unofficial].
    *   Determine the priority order of sub-volumes using a Binary Space Partitioning (BSP) tree relative to the viewing direction [S2, unofficial].
    *   Perform binary-swap compositing. For two adjacent images $A$ (front/high priority) and $B$ (back/low priority), the composite $C$ is:
        $$ I_C = I_A + A_A \cdot I_B $$
        $$ A_C = A_A \cdot A_B $$
        Where $I$ is color/intensity and $A$ is opacity/transmittance [S2, unofficial].
    *   The compositing communication count lower bound is $N_c = N_p - 1$, where $N_p$ is the number of processing nodes [S2, unofficial].
5.  **Warping:** Apply the warp matrix $W$ to the combined baseplane to produce the final image [S1, unofficial]. This can be done either after all baseplanes are combined or incrementally after each baseplane is warped and then combined [S1, unofficial].

**Formulas**
*   **Baseplane Size:** $(0,0)$ is at the top-left-back [S1, unofficial].
    $$ \text{EndX} = \lfloor \text{StepX} \times \text{NumberOfSteps} + \text{SizeX} - 1 \rfloor $$
    $$ \text{EndY} = \lfloor \text{StepY} \times \text{NumberOfSteps} + \text{SizeY} - 1 \rfloor $$
    [S1, unofficial]
*   **Intermediate Baseplane Offset:** For set $(i, j, k)$, the location is $(i \cdot N_x, j \cdot N_y, k \cdot N_z)$, where $N_x, N_y, N_z$ are set dimensions [S1, unofficial]. If sets differ in size, replace with cumulative sums [S1, unofficial].
*   **Cursor Translation:** Cursor center $c_g(x,y,z)$ is specified in object coordinates relative to the supervolume origin [S1, unofficial]. To translate to set $(i,j,k)$, apply: $(x,y,z) - \text{Origin}_{\text{sub}}$, where $\text{Origin}_{\text{sub}} = (i \cdot (N_x - \text{overlap}), j \cdot (N_y - \text{overlap}), k \cdot (N_z - \text{overlap}))$ [S1, unofficial].

**Parameters**
*   **Object Coordinates:** Unit distance equals voxel spacing [S1, unofficial].
*   **Screen Size:** $768^2$ pixels in measured tests [S2, unofficial].
*   **Volume Resolution:** $256^3$ in measured tests [S2, unofficial].
*   **Baseplane Initialization:** Background color for back-to-front, foreground color for front-to-back [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Overlap Handling:** At the edges of volume partitions, rendering samples might be duplicated if overlapping sets of voxels are not handled correctly [S1, unofficial]. This is avoided by specific handling of overlapping sets, ensuring that samples on integer voxel positions in the overlap region are not counted twice [S1, unofficial].
*   **Baseplane Size:** The size of the baseplane depends on the viewing direction [S1, unofficial]. The maximum size occurs when the viewing direction is 45° to all faces [S1, unofficial]. The minimum size is the size of the face of the volume ($\text{ObjectSizeX} \times \text{ObjectSizeY}$) when the view is perpendicular to the baseplane [S1, unofficial].
*   **Cursor Handling:** If a 3D cursor is used, its center $c_g(x,y,z)$ is specified in object coordinates [S1, unofficial]. Cursor width and length are invariants with respect to a partitioned volume [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_subvolume(begin_i, begin_j, begin_k, last_i, last_j, last_k, bp_width, bp_height, volume_data, update_params):
    # Initialize baseplane with background color (back-to-front traversal) [S1, unofficial]
    baseplane = [[(0.0, 0.0, 0.0, 0.0) for _ in range(bp_width)] for _ in range(bp_height)]

    for k in range(begin_k, last_k + 1):
        update = update_params[k]
        q = update.begin_sheared_j
        for j in range(begin_j, last_j + 1):
            q += update.delta_j
            p = update.begin_sheared_i
            for i in range(begin_i, last_i + 1):
                p += update.delta_i
                ijk = (i, j, k)
                sijk = (p, q, k)

                # Resample voxel value using convolution kernel [S2, unofficial]
                pixel = resample_with_convolution(volume_data, ijk, sijk)

                # Calculate memory address [S2, unofficial]
                pmem = (int(q) * bp_width + int(p)) * 4

                # Composite into local baseplane [S2, unofficial]
                composite_baseplane(baseplane, pmem, pixel, bp_width)
    return baseplane

def composite_baseplane(baseplane, pmem, pixel, bp_width):
    # pixel is (r, g, b, a)
    # pmem is byte offset, assume 4 bytes per pixel
    idx = pmem // 4
    col = idx % bp_width
    row = idx // bp_width

    if 0 <= row < len(baseplane) and 0 <= col < len(baseplane[0]):
        r_old, g_old, b_old, a_old = baseplane[row][col]
        r_new, g_new, b_new, a_new = pixel

        # Composite formula from [S2, unofficial]: I_C = I_A + A_A * I_B, A_C = A_A * A_B
        # Side A is the existing baseplane pixel (front), Side B is the new pixel (back)
        r_new = r_old + a_old * r_new
        g_new = g_old + a_old * g_new
        b_new = b_old + a_old * b_new
        a_new = a_old * a_new

        baseplane[row][col] = (r_new, g_new, b_new, a_new)

def resample_with_convolution(volume_data, ijk, sijk):
    # Standard trilinear/bilinear resampling with convolution kernel [G]
    # Returns (r, g, b, a)
    return volume_data[ijk]  # Placeholder for kernel application [G]
```

## Subtopics

### [Distributed Data Volume Rendering via Shear-Warp Factorisation](7403-distributed-data-volume-rendering-via-shear-warp-factorisati.md)
A parallel rendering technique that divides a large 3D volume into sub-volumes, renders them independently using the shear-warp algorithm, and composites the results using the associative 'over' operator to produce an image identical to a single-pass renderer. It distinguishes itself from its sibling, the Spatially Partitioned Shear-Warp Method, by explicitly defining a hierarchical rendering tree with configurable branching factors and specific optimizations for network transfer (window-encoding, RLE) and memory storage (multi-order copies) to handle large datasets on distributed clusters.

## Operating characteristics
*   **Time Complexity:** The rendering complexity is $O(V^3)$ where $V$ is the side length of the volume [S2, unofficial]. This is dependent on the volume size, unlike Ray-casting which is $O(P^3)$ dependent on screen size $P$ [S2, unofficial]. When $P/V$ is constant (typical for high-resolution displays and high-resolution volumes), the Shear-Warp method is faster than Ray-casting [S2, unofficial].
*   **Memory Complexity:** Requires memory for the volume data and intermediate baseplanes [S1, unofficial]. The baseplane size varies with view angle but is generally smaller than the full image for oblique views [S1, unofficial].
*   **Measured Performance:** At a screen size of $768^2$ pixels and volume data resolution of $256^3$, the method is approximately 4 times faster than the Ray-casting method [S2, unofficial].
*   **Scalability:** The method exhibits good scalability within the measured range [S2, unofficial]. However, scalability analysis for very high parallelism is not sufficient; only good scalability within the measured range is observed [S2, unofficial]. The compositing communication count lower bound is $N_c = N_p - 1$ [S2, unofficial].
*   **Strengths:** View-independent partitioning (partition once, render from any angle); parallelizable; faster than Ray-casting for large volumes relative to screen size; handles arbitrary viewing directions [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires careful handling of overlaps at partition boundaries; compositing communication overhead increases with node count; scalability analysis for very high parallelism is limited [S1, unofficial][S2, unofficial].

## Background
The Spatially Partitioned Shear-Warp Method was proposed in 2001 by Ogata, Kajihara, Muraki, Shimokawa, Ma, and Doi in the paper "Parallel Volume Rendering System Using Multiple Shear Compositing Method" [S2, unofficial]. It was developed to enable high-speed, low-cost parallel volume rendering on PC clusters, addressing the limitations of dedicated hardware and the high cost of supercomputers [S2, unofficial]. It builds on the Shear-Warp Algorithm [4304-shear-warp-algorithm.md] by introducing spatial partitioning to enable parallel processing of sub-volumes and a binary tree compositing structure to resolve occlusion across partitions [S2, unofficial]. It differs from the Binary-Swap Compositing method by not requiring the number of partitions to be a power of two and by supporting perspective projections [S2, unofficial].

## Sources
[S1] Rendering a Shear-Warped Partitioned Volume Data Set, https://vcg.seas.harvard.edu/publications/rendering-a-shear-warped-partitioned-volume-data-set/paper, fetched 2026-10-29 (unofficial)
[S2] 空間分割シアーワープ法を用いた並列ボリュームレンダリング ..., https://www.jstage.jst.go.jp/article/artsci/1/1/1_1_1/_pdf, fetched 2026-10-30 (unofficial)
