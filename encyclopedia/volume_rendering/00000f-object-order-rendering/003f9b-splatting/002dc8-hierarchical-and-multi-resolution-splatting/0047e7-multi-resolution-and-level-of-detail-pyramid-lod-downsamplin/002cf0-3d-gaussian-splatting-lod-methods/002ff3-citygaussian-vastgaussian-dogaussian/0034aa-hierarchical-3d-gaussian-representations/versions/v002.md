# Hierarchical 3D Gaussian Representations

Also called: Gaussian Pyramid, Multi-resolution Gaussian Pyramid

## Overview
Hierarchical 3D Gaussian Representations render very large datasets in real time by storing a multi-resolution pyramid of Gaussians, allowing the renderer to adapt to view distance and improve memory efficiency [S1, unofficial]. This approach sets itself apart from flat Gaussian Splatting by organizing primitives into a hierarchy where nodes summarize the radiance and opacity of their children, enabling efficient level-of-detail selection [S1, unofficial]. It serves as the foundational concept for specific implementations like Budgeted-GS, which uses a factoring tree to enable city-scale rendering on consumer hardware [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained set of 3D Gaussian primitives (position, covariance, color, opacity) representing a scene [S1, unofficial].
*   **Output:** A rendered image at a selected level of detail [S2, unofficial].

**Data Structures:**
The hierarchy is organized as an octree or pyramid where non-leaf nodes approximate their children and leaves correspond to original Gaussians [S3, unofficial]. In Budgeted-GS, this is implemented as a factoring tree: an octree of scale bands containing moment-matched equivalent planks [S2, unofficial].

**Steps:**
1.  **Hierarchy Construction:** Convert a flat trained model into a multi-resolution hierarchy by merging nearby Gaussians into parent nodes based on spatial proximity and scale [S3, unofficial]. In Budgeted-GS, this is a post-hoc conversion of raw planks into a factoring tree of leaf × scale-band nodes containing moment-matched equivalent planks [S2, unofficial].
2.  **Level Selection:** For each view, evaluate a cut condition to select the appropriate level of detail. Budgeted-GS uses a one-parameter pixel-footprint scheduler based on threshold $t = c \cdot d / f$ to select raw, fine, or aggregate content [S2, unofficial]. Gaussian Hierarchies use a distance-based cut condition evaluated during a breadth-first search (BFS) from the root [S3, unofficial].
3.  **Rendering:** Emit the selected stream of primitives to a standard splatting rasterizer [S2, unofficial].

**Formulas:**
*   Budgeted-GS scheduler threshold: $t = c \cdot \frac{d}{f}$, where $c$ is a constant, $d$ is camera distance, and $f$ is focal length [S2, unofficial].
*   Gaussian Hierarchies cut condition: $c_{\text{hier}}(i, \text{cam}) = \|\boldsymbol{\mu}_i - \mathbf{p}_{\text{cam}}\|_2 \geq m_d(i)$, where $\boldsymbol{\mu}_i$ is the Gaussian center, $\mathbf{p}_{\text{cam}}$ is the camera position, and $m_d(i) = \frac{T}{\max_j s_i^j}$ with $T$ as a global LoD threshold and $s_i^j$ as scale components [S3, unofficial].

**Parameters:**
*   $t$: The pixel-footprint threshold used by the scheduler [S2, unofficial].
*   $c$: A constant determining the threshold scale [S2, unofficial].
*   $T$: A global LoD threshold controlling the distance cutoff [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Constraints:** Without hierarchical selection, city-scale scenes with millions of primitives may exceed the memory capacity of consumer GPUs [S2, unofficial].
*   **Redundancy:** Naive training optimizes primitives later discarded during compression; hierarchical methods identify redundancy to avoid this [S2, unofficial].
*   **Precision and Boundaries:** Evaluating cut conditions at extreme distances or with degenerate scale components may cause floating-point instability or incorrect LoD selection [G].

## Subtopics

### Budgeted-GS
Budgeted-GS is a post-hoc method that converts any trained 3DGS model into a factoring tree (a multi-resolution hierarchy of moment-matched aggregates) to enable real-time rendering of city-scale scenes on consumer GPUs by selecting the level of detail that fits the target device's memory [S2, unofficial]. It also includes a training method (Budget-Centered Training) that measures the required primitive count based on a capacity floor derived from optimal transport and trains the model directly at that size, avoiding the optimization of primitives that would be discarded [S2, unofficial]. It is chosen when rendering large-scale urban environments on hardware with limited memory, offering a continuous memory-quality curve rather than a single fixed operating point [S2, unofficial]. It differs from standard hierarchical GS by explicitly using a factoring tree of scale bands and a one-parameter scheduler driven by optimal transport theory [S2, unofficial].

### Gaussian Hierarchies
Gaussian Hierarchies (H-3DGS) is a recursive tree structure where nearby Gaussians are merged; non-leaf nodes approximate children, and leaves are original Gaussians [S3, unofficial]. It provides a view-adaptive Level of Detail (LoD) representation via a 'cut' set determined by a distance-based condition evaluated in a breadth-first search [S3, unofficial]. It is chosen when explicit distance-based LoD traversal is preferred over footprint-based scheduling [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Hierarchy construction takes a few seconds [S2, unofficial]. Rendering time scales with the selected level of detail and the efficiency of the traversal or filtering structure [S1, unofficial].
*   **Memory Complexity:** Significantly reduced compared to flat Gaussian Splatting for large scenes, as only the necessary level of detail is loaded into memory [S1, unofficial][S2, unofficial].
*   **Strengths:** Enables real-time rendering of city-scale datasets on consumer hardware; adapts to view distance; allows a single model to serve GPUs with different memory capacities [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires additional post-processing or specialized training to build the hierarchy; standard implementations require tree traversal unless using parallel filtering structures [S1, unofficial][S2, unofficial].
*   **When to use:** Large-scale urban scenes, VR/AR applications with memory constraints, or streaming scenarios [S1, unofficial][S2, unofficial].
*   **When not to use:** Small-scale scenes where the overhead of hierarchy construction is not justified, or when flat splatting fits the target hardware [G].

## Background
The concept of hierarchical 3D Gaussian representations emerged as a solution to the scalability limits of standard 3D Gaussian Splatting [S1, unofficial][S2, unofficial]. Early works like Octree-GS built octrees over scenes to associate nodes with merged Gaussians [S1, unofficial]. Subsequent developments included Google's LODGE, which introduced a level-of-detail framework for large-scale 3D Gaussian Splatting [S1, unofficial], and Nvidia's HiGS, which proposed a hierarchical rendering architecture for real-time performance [S1, unofficial]. FilterGS further advanced this by proposing a traversal-free, parallel filtering structure [S1, unofficial]. Budgeted-GS formalized these ideas using optimal transport theory to define a capacity floor and license for redundancy, operationalizing them into a factoring tree and budget-centered training [S2, unofficial]. Gaussian Hierarchies introduced a recursive merge structure with a distance-based BFS cut condition [S3, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
[S3] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
