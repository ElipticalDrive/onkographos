# Atlas-Net
Also called: Atlas-based representations, Neural Surfaces (Atlas)

## Overview
Atlas-Net represents 3D surfaces as a collection of overlapping charts (surface patches), where each chart is parameterized by a continuous mapping from a 2D domain to 3D space learned via neural networks [S1]. It distinguishes itself from single-chart methods (e.g., FoldingNet) by using multiple independent or shared MLPs to cover complex topologies, trading increased memory cost for better geometric fidelity and the ability to handle arbitrary surface structures [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 2D query coordinates $\mathbf{s} \in D \subset \mathbb{R}^2$ belonging to a specific chart $i$, and the learnable weights $\theta_i$ of the corresponding MLP.
*   **Output:** A 3D position $\mathbf{p} \in \mathbb{R}^3$ on the surface.

**Data Structures:**
*   **Atlas:** A collection of $N$ charts $\{ f_{\theta_i} : D_i \subset \mathbb{R}^2 \rightarrow \mathbb{R}^3 \mid i = 1, \dots, N \}$.
*   **Chart MLP:** A Multilayer Perceptron (MLP) with weights $\theta_i$.
    *   If weights are shared: One set of weights $\theta$ for all $N$ charts.
    *   If weights are independent: $N$ distinct sets of weights $\{\theta_1, \dots, \theta_N\}$.
*   **Domain $D$:** A continuous 2D manifold, typically a unit square or a sphere [S1].

**Algorithm:**
1.  **Initialization:** Define the number of patches $N$. Initialize $N$ MLPs. Decide whether to share weights across all charts or assign independent weights to each [S1].
2.  **Parameterization:** For each chart $i$, define the mapping function:
    $$ f_{\theta_i}(\mathbf{s}) = \mathbf{p} $$
    where $\mathbf{s}$ is the 2D coordinate and $\mathbf{p}$ is the resulting 3D position [S1].
3.  **Surface Representation:** The complete surface is the union of the images of all charts:
    $$ \mathcal{S} = \bigcup_{i=1}^{N} \text{Image}(f_{\theta_i}) $$
    [S1].
4.  **Rendering/Querying:** To evaluate a point on the surface, identify the chart $i$ containing the query $\mathbf{s}$, and compute $\mathbf{p} = f_{\theta_i}(\mathbf{s})$ [S1].

**Parameters:**
*   **Number of patches ($N$):** Controls the resolution and complexity of the representation. Memory cost scales linearly with $N$ [S1].
*   **MLP Architecture:** Depth and width of the MLPs. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Inconsistencies:** Since each chart is defined independently, overlapping regions between charts may exhibit geometric inconsistencies or distortions [S1].
*   **Memory Cost:** Using independent MLPs for each chart increases the number of parameters linearly with $N$, which is a significant limitation for real-time applications or high-resolution meshes [S1].
*   **Topology:** While capable of arbitrary topology, the method requires a valid atlas (covering the surface with charts). Single-chart methods fail on complex topologies without tearing or folding [S1].

## Subtopics

### Per-face texture parametrization
A texture mapping scheme used in NeRFMeshing for unbounded scenes where each triangular face of the mesh is assigned a dedicated region in a feature texture image [S2]. It maps each face to a right-angled triangle of side 4 pixels in a 2048x2048 texture image [S2]. Each pixel stores a feature vector $\hat{\mathbf{f}}$ predicted by the SSAN at the corresponding 3D position [S2]. This approach retains high image resolution for per-face features but risks memory overhead for high-resolution meshes with many faces [S2].

## Operating characteristics
*   **Memory Complexity:** Linear with respect to the number of patches $N$ if independent MLPs are used. If weights are shared, memory is constant with respect to $N$ (only the domain mapping changes) [S1].
*   **Strengths:** Can represent surfaces of arbitrary topologies in a continuous and differentiable manner [S1]. Independent MLPs offer a better fit to 3D shapes than shared weights [S1].
*   **Limitations:** Inconsistencies and distortions across overlapping regions due to independent chart definitions [S1]. High memory cost with independent MLPs limits scalability [S1].
*   **When to use:** When representing complex 3D shapes with arbitrary topology is required, and single-chart methods (like FoldingNet) are insufficient [S1].
*   **When not to use:** When memory is strictly constrained and the surface topology is simple enough for a single chart or voxel-based representation [S1].

## Background
Atlas-Net was introduced by Groueix et al. in [GFK∗ 18b] [S1]. It builds on the idea of representing surfaces as a collection of charts parameterized by continuous mappings from a 2D domain to 3D space [S1]. Subsequent works have proposed strategies to address inconsistencies, such as enforcing consistency across overlapping charts [BPG∗ 20], using local shape priors [BGKS20], or using sophisticated architectures to model relationships between charts [LRPK21] [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
