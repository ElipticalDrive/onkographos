# TraGraph-GS

Also called: Trajectory Graph-based Gaussian Splatting

## Overview
TraGraph-GS is a high-quality rendering method for arbitrary large-scale scenes that employs graph-based adaptive spatial partitioning, regularization constraints, and adaptive region selection [S1]. It addresses the limitations of standard 3D Gaussian Splatting (3DGS) in handling vast scales and textureless regions by decoupling the scene into manageable local regions and a coarse global representation [S1]. Unlike methods that rely on fixed spatial grids or simple clustering, TraGraph-GS constructs a camera connectivity graph to ensure balanced, connected subgraphs, and uses a progressive rendering strategy for novel viewpoints to eliminate boundary artifacts [S1].

## How it works

### Inputs and Outputs
**Inputs:** A set of disordered scene photographs (images) and their associated camera poses.
**Outputs:** A set of 3D Gaussians (local and global) and a rendering pipeline capable of generating novel views.

### Data Structures
1.  **Camera Connectivity Graph:** Defined as $G = (V, E, w)$ [S1].
    *   $V = \{v_1, v_2, ..., v_n\}$: Set of vertices, where each vertex represents an input image.
    *   $E = \{(v_i, v_j) | v_i, v_j \in V\}$: Set of edges connecting pairs of vertices.
    *   $w: E \rightarrow \mathbb{R}$: Weight function assigning a real number to each edge. The weight is determined by the count of feature matches between image pairs; higher weights indicate larger co-visible regions [S1].
2.  **Regions:** A partition of $V$ into $k$ disjoint sets $\{P_i\}_{i=1}^k$, where each $P_i$ contains nodes (images) that are connected and balanced in size [S1].
3.  **Gaussians:** Each region $P_i$ maintains its own set of 3D Gaussians. A separate coarse global Gaussian set is trained for novel view synthesis [S1].

### Algorithm Steps

#### 1. Graph-Based Adaptive Spatial Partitioning
1.  **Construct Graph:** For all input images, compute feature matches. Build $G(V, E, w)$ where edge weights $w_{ij}$ equal the number of matches between image $i$ and image $j$ [S1].
2.  **BFS-Based Segmentation:** Divide $G$ into $k$ regions ensuring connectivity and balanced node distribution [S1].
    *   Initialize unassigned nodes $U = V$ [S1].
    *   Identify node $n$ with the highest degree in $U$ as the seed node [S1].
    *   Perform Breadth-First Search (BFS) from the seed, adding nodes to the current region until the region contains $|V|/k$ nodes. These nodes form a region [S1].
    *   For each remaining unassigned node in $U$, calculate a score $s_i$ relative to each existing region. Assign the node to the region that maintains balance and connectivity [S1].
    *   Repeat until all nodes are assigned [S1].

#### 2. Local Training with Regularization Constraints
For each region $P_i$, train local 3D Gaussians independently to optimize GPU memory [S1]. The total loss $L$ is defined as [S1]:
$$ L = (1 - \lambda)L_1 + \lambda L_{SSIM} + L_{mv} $$
where $L_1$ and $L_{SSIM}$ are standard image reconstruction losses [S1].

*   **Multi-View Constraint ($L_{mv}$):** To improve geometry in textureless regions, use patch matching [S1].
    *   Render depth and normal maps for each image in the region using alpha blending [S1].
    *   For a pixel $p$ in a reference view, establish a spatial plane defined by normal $n \in \mathbb{R}^{1 \times 3}$ and distance $d$ from the camera origin [S1].
    *   Project $p$ to a corresponding pixel $p'$ in a neighborhood view using homography warping [S1].
    *   Compute Normalized Cross Correlation (NCC) between patches at $p$ and $p'$ [S1].
    *   Calculate loss:
        $$ L_{mv} = \sum_{p \in P_i} (1 - \text{NCC}(p, p')) $$
        [S1]

*   **Multi-Scale Constraint:** Apply position-aware point adaptive control to adjust Gaussian scales based on viewer distance [S1].
    *   Define a scaling factor $\gamma(\mu)$ based on the distance of Gaussian center $\mu$ from camera center $c$ and scene radius $r$:
        $$ \gamma(\mu) = \begin{cases} 1 & \text{if } \|\mu - c\|_2 < 2r \\ \|\mu - c\|_2 / r - 1 & \text{if } \|\mu - c\|_2 \ge 2r \end{cases} $$
        [S1]
    *   **Cloning/Splitting Protocol:**
        *   If the maximum scale of the Gaussian ellipsoid $\max(s) < g \cdot \gamma(\mu)$, perform a cloning operation [S1].
        *   Otherwise, perform a splitting procedure [S1].
        *   Eliminate Gaussoids where the maximum scale criterion is not met (pruning) [S1].

#### 3. Adaptive Region Selection for Novel Viewpoints
To render novel viewpoints without boundary artifacts, TraGraph-GS uses a progressive rendering strategy combining local and global Gaussians [S1].

1.  **Global Representation:** Train a coarse global Gaussian representation by downsampling point clouds and images. Avoid densification (cloning/splitting) during this phase to prevent excessive memory consumption [S1].
2.  **Matching Score Calculation:** Render the novel viewpoint using local Gaussians ($G_i$) and global Gaussians separately. Calculate a matching score $S$ for each local-global pair [S1]:
    $$ S = (1 - \lambda)L_1 + \lambda L_{SSIM} $$
    Higher scores indicate higher likelihood of visibility [S1].
3.  **Sorting:** Sort local Gaussians in descending order of matching scores: $G_1, G_2, ..., G_k$ [S1].
4.  **Opacity Calculation:** Compute projection opacity $o_i(p)$ for each pixel $p$ for the sorted Gaussians [S1]:
    $$ o_i(p) = \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $\alpha_i$ is the opacity of Gaussian $G_i$ [S1].
5.  **Progressive Rendering:** Compute final pixel color $c(p)$ using weights $w_i$ based on an overlapped opacity threshold $\beta$ [S1]:
    $$ w_i = \begin{cases} o_i(p) & \text{if } \sum_{j=1}^{i-1} o_j(p) < \beta \\ 0 & \text{if } \sum_{j=1}^{i-1} o_j(p) \ge \beta \end{cases} $$
    [S1]
    $$ c(p) = \sum_{i=1}^k w_i c_i(p) $$
    where $c_i(p)$ is the color of pixel $p$ rendered by local Gaussian $G_i$ [S1].

### Parameters
*   $k$: Number of segmented regions.
*   $\lambda$: Weight for SSIM loss (typical value not specified in sources, standard 3DGS uses 0.2).
*   $r$: Scene radius.
*   $c$: Center of cameras.
*   $g$: Threshold multiplier for cloning/splitting (value not specified).
*   $\beta$: Overlapped opacity threshold for progressive rendering (value not specified).

### Edge Cases and Pitfalls
*   **Memory Overflow:** Naive partitioning can lead to unbalanced node distribution, causing memory overflow. The BFS method ensures balance [S1].
*   **Boundary Artifacts:** Combining local Gaussians naively causes floating artifacts or holes. The progressive rendering strategy with matching scores mitigates this [S1].
*   **Textureless Regions:** Standard 3DGS struggles with geometry here. The multi-view constraint using patch matching addresses this [S1].
*   **Distant Objects:** Standard representation fails for distant objects in large scenes. The multi-scale constraint $\gamma(\mu)$ adapts Gaussian sizes based on distance [S1].

## Operating characteristics
*   **Time Complexity:** Not explicitly given. Graph construction depends on feature matching. Segmentation is BFS-based ($O(V+E)$). Training is parallelized per region. Novel view rendering involves sorting and progressive accumulation.
*   **Memory Complexity:** Reduced for large scenes by partitioning into local regions and using a coarse global representation for novel views, avoiding densification in the global phase [S1].
*   **Performance:** Evaluated on large-scale aerial (Mill-19, UrbanScene3D) and ground-level datasets (SmallCity, Campus, Road, General) with thousands of images [S1]. Maintains real-time performance across vastly different scales [S1].
*   **Strengths:** Handles arbitrary large-scale scenes, improves texture/geometry in textureless regions, eliminates boundary artifacts in novel views.
*   **Limitations:** Requires feature matching for graph construction. Relies on heuristic parameters ($k, g, \beta$).

## Background
TraGraph-GS was proposed to address the challenges of rendering arbitrary large-scale scenes with high quality while maintaining real-time performance [S1]. It builds upon 3D Gaussian Splatting [11] and addresses issues found in previous methods like VastGaussian [14] regarding progressive partitioning and lighting variations. The method was published in IEEE Transactions on Pattern Analysis and Machine Intelligence in 2026 [S1].

## Sources
[S1] TraGraph-GS: Trajectory Graph-based Gaussian Splatting for Arbitrary Large-Scale Scene Rendering., https://doi.org/10.1109/tpami.2026.3709368, fetched 2026-10-26
