# TraGraph-GS

Also called: Trajectory Graph-based Gaussian Splatting

## Overview
TraGraph-GS is a high-quality rendering method for arbitrary large-scale scenes that employs graph-based adaptive spatial partitioning, regularization constraints, and adaptive region selection to maintain real-time performance and visual fidelity across vastly different scales [S1]. It distinguishes itself from standard 3D Gaussian Splatting (3DGS) and its siblings by decoupling the scene into connected subgraphs based on camera feature matching, enabling independent local training and a progressive rendering strategy for novel viewpoints that avoids boundary artifacts [S1]. Unlike [FlexGS](16110-flexgs.md) which focuses on dynamic adaptivity, or [Mipmap-GS](16893-mipmap-gs.md) which handles scale via mipmaps, TraGraph-GS explicitly models the structural connectivity of input images to partition the scene and uses a coarse global representation combined with sorted local contributions for novel view synthesis [S1].

## How it works
**Inputs:** A set of disordered scene photographs (images) and their associated camera poses.
**Outputs:** A set of 3D Gaussians (local and global) and a rendering pipeline capable of generating novel views.

**Data Structures:**
1.  **Camera Connectivity Graph:** $G = (V, E, w)$, where $V = \{v_1, ..., v_n\}$ are image vertices, $E$ are edges, and $w: E \rightarrow \mathbb{R}$ are weights representing feature matching counts (covisibility) [S1].
2.  **Local Gaussians:** A set of 3D Gaussians $\{G_i\}$ trained independently for each segmented region $P_i$. Each Gaussian $G_i$ is defined by mean $\mathbf{\mu}_i$, covariance $\mathbf{\Sigma}_i$ (from scale $\mathbf{s}_i$ and rotation $\mathbf{q}_i$), opacity $\alpha_i$, and spherical harmonics coefficients $\mathbf{c}_i$ [S1].
3.  **Global Gaussians:** A coarse set of 3D Gaussians trained on downsampled data to capture overall scene structure [S1].

**Algorithm Steps:**

1.  **Graph Construction:** Construct the camera connectivity graph $G$. For every pair of images, count the number of matched features. Assign this count as the edge weight $w_{ij}$ [S1].
2.  **BFS-Based Graph Segmentation:** Divide $G$ into $k$ regions $\{P_i\}_{i=1}^k$ ensuring connectivity and balanced node distribution [S1]:
    *   Initialize unassigned nodes $U = V$.
    *   Identify the node $n$ with the highest degree in $U$ as a seed node.
    *   Perform Breadth-First Search (BFS) from the seed, adding nodes to the current region until it contains $|V|/k$ nodes.
    *   For remaining unassigned nodes, calculate a score $s_i$ relative to existing regions and assign them to maintain balance [S1].
3.  **Local Training with Regularization:** Train local 3D Gaussians independently for each region $P_i$ using the following loss function [S1]:
    $$ L = (1 - \lambda)L_1 + \lambda L_{SSIM} + L_{mv} $$
    where $L_1$ and $L_{SSIM}$ are standard reconstruction losses [S1].
    *   **Multi-View Constraint ($L_{mv}$):** For textureless regions, render depth and normal maps. For a pixel $p$ in a reference view, establish a spatial plane $(d, \mathbf{n})$. Use homography warping to project $p$ to a corresponding pixel $p'$ in a neighborhood view. Minimize the difference using Normalized Cross Correlation (NCC) [S1]:
        $$ L_{mv} = \sum_{p \in P_i} (1 - \text{NCC}(p, p')) $$
    *   **Multi-Scale Constraint:** Apply position-aware point adaptive control to adjust Gaussian scales based on viewer distance. Define a scale factor $\gamma(\mathbf{\mu})$ where $r$ is the scene radius and $\mathbf{c}$ is the camera center [S1]:
        $$ \gamma(\mathbf{\mu}) = \begin{cases} 1 & \text{if } \|\mathbf{\mu} - \mathbf{c}\|_2 < 2r \\ \|\mathbf{\mu} - \mathbf{c}\|_2 / r - 1 & \text{if } \|\mathbf{\mu} - \mathbf{c}\|_2 \ge 2r \end{cases} $$
        If the directional gradient exceeds a threshold:
        *   Clone if $\max(\mathbf{s}) < g \cdot \gamma(\mathbf{\mu})$.
        *   Split if the condition is not met.
        *   Prune Gaussians with low opacity [S1].
4.  **Global Representation Training:** For novel viewpoints, train a coarse global Gaussian representation by downsampling point clouds and images. Avoid the densification process to prevent excessive memory consumption [S1].
5.  **Adaptive Region Selection (Novel View Rendering):**
    *   Render the novel viewpoint using local and global Gaussians separately.
    *   Calculate a matching score $S$ for each local-global Gaussian pair [S1]:
        $$ S = (1 - \lambda)L_1 + \lambda L_{SSIM} $$
        Higher scores indicate higher visibility likelihood [S1].
    *   Sort local Gaussians in descending order of $S$: $G_1, G_2, ..., G_k$.
    *   Compute projection opacity $o_i(p)$ for each pixel $p$ [S1]:
        $$ o_i(p) = \sum_{i=1}^{N} \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    *   Compute final pixel color $c(p)$ using a progressive rendering strategy with weights $w_i$ based on an overlapped opacity threshold $\beta$ [S1]:
        $$ w_i = \begin{cases} o_i(p) & \text{if } \sum_{j=1}^{i-1} o_j(p) < \beta \\ 0 & \text{if } \sum_{j=1}^{i-1} o_j(p) \ge \beta \end{cases} $$
        $$ c(p) = \sum_{i=1}^{k} w_i c_i(p) $$
        where $c_i(p)$ is the color rendered by local Gaussian $G_i$ [S1].

**Parameters:**
*   $k$: Number of segmented regions (Not given in the sources).
*   $\lambda$: Weight for SSIM loss (Not given in the sources).
*   $r$: Scene radius (Not given in the sources).
*   $\beta$: Overlapped opacity threshold for progressive rendering (Not given in the sources).
*   $g$: Scaling factor for cloning criterion (Not given in the sources).

**Edge Cases and Pitfalls:**
*   **Memory Overflow:** Unbalanced node distribution in segmentation can lead to memory overflow [S1]. The BFS method is chosen specifically to prevent this [S1].
*   **Boundary Artifacts:** Naively combining local Gaussians can cause floating artifacts or holes at region boundaries [S1]. The progressive rendering strategy with matching scores mitigates this [S1].
*   **Textureless Regions:** Standard 3DGS struggles with geometry in textureless areas; the multi-view constraint is required to address this [S1].

## Operating characteristics
*   **Time Complexity:** Training is parallelized across regions, reducing per-region memory and potentially time [S1]. Novel view rendering involves sorting local Gaussians and progressive blending, adding overhead compared to standard alpha-blending [S1].
*   **Memory Complexity:** By partitioning the scene and using a coarse global representation for novel views, memory usage is controlled. Global training avoids densification to prevent excessive consumption [S1].
*   **Performance:** Evaluated on large-scale aerial (up to 3,019 images) and ground datasets (e.g., SmallCity, Campus) [S1]. Maintains real-time performance across vastly different scales [S1].
*   **Strengths:** Handles arbitrary large-scale scenes; improves texture details and geometry in textureless regions; avoids boundary artifacts in novel views.
*   **Limitations:** Requires feature matching for graph construction; performance depends on the quality of the graph segmentation [S1].

## Background
TraGraph-GS was proposed to address the limitations of existing methods in rendering large-scale scenes, such as memory overflow and boundary artifacts. It builds on the principles of 3D Gaussian Splatting [11] and extends them with graph-based partitioning and regularization constraints. The method was introduced in a paper published in IEEE Transactions on Pattern Analysis and Machine Intelligence in 2026 [S1].

## Sources
[S1] TraGraph-GS: Trajectory Graph-based Gaussian Splatting for Arbitrary Large-Scale Scene Rendering., https://doi.org/10.1109/tpami.2026.3709368, fetched 2026-10-26
