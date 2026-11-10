# Hierarchical 3D Gaussian Representations

Also called: Gaussian Pyramid, Multi-resolution Gaussian Pyramid

## Overview
Hierarchical 3D Gaussian Representations render very large datasets in real time by storing a multi-resolution pyramid of Gaussians, allowing the renderer to adapt to view distance and improve memory efficiency [S1, unofficial]. This approach sets itself apart from flat Gaussian Splatting by organizing primitives into a hierarchy (such as an octree or pyramid) where nodes summarize the radiance and opacity of their children, enabling efficient level-of-detail (LoD) selection [S1, unofficial]. It serves as the foundational concept for specific implementations like Budgeted-GS, which uses a factoring tree to enable city-scale rendering on consumer hardware [S2, unofficial].

## How it works
The method involves storing Gaussians in a hierarchical structure that supports multi-resolution rendering. While specific traversal algorithms vary by implementation (e.g., octree traversal vs. parallel filtering), the core principle is the organization of primitives into a hierarchy where coarser levels represent aggregated information from finer levels.

**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussian primitives (position, covariance, color, opacity) representing a scene.
*   **Output:** A rendered image at a selected level of detail.

**Data Structures:**
The hierarchy is typically structured as an octree or a pyramid. Each node in the hierarchy contains a "merged Gaussian" or aggregate that summarizes the radiance and opacity of its children [S1, unofficial]. In specific realizations like Budgeted-GS, this is implemented as a "factoring tree," which is an octree of scale bands containing moment-matched equivalent planks [S2, unofficial].

**Steps:**
1.  **Hierarchy Construction:** Organize the flat set of trained Gaussian primitives into a multi-resolution hierarchy. This may involve merging primitives into parent nodes based on spatial proximity or scale [S1, unofficial]. In Budgeted-GS, this is a post-hoc conversion of a flat model into a factoring tree of leaf × scale-band nodes [S2, unofficial].
2.  **Level Selection:** For each view, select the appropriate level of detail. This is often driven by a scheduler that considers the pixel footprint of the primitives relative to the view distance and focal length [S2, unofficial].
3.  **Rendering:** Emit the selected stream of primitives (raw, fine, or aggregate) to a standard splatting rasterizer [S2, unofficial].

**Formulas:**
In Budgeted-GS, the selection of detail is governed by a one-parameter pixel-footprint threshold $t$:
$$ t = c \cdot \frac{d}{f} $$
Where:
*   $c$ is a constant [S2, unofficial].
*   $d$ is the distance from the camera [S2, unofficial].
*   $f$ is the focal length [S2, unofficial].

**Parameters:**
*   $t$: The pixel-footprint threshold used by the scheduler [S2, unofficial].
*   $c$: A constant determining the threshold scale [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Constraints:** Without hierarchical selection, city-scale scenes with millions of primitives may exceed the memory capacity of consumer GPUs [S2, unofficial].
*   **Redundancy:** Naive training of large scenes may optimize primitives that are later discarded during compression or pruning; hierarchical methods aim to identify and avoid this redundancy [S2, unofficial].

## Subtopics

### Budgeted-GS
Budgeted-GS is a post-hoc method that converts any trained 3DGS model into a factoring tree (a multi-resolution hierarchy of moment-matched aggregates) to enable real-time rendering of city-scale scenes on consumer GPUs by selecting the level of detail that fits the target device's memory [S2, unofficial]. It also includes a training method (Budget-Centered Training) that measures the required primitive count based on a capacity floor and trains the model directly at that size, avoiding the optimization of primitives that would be discarded [S2, unofficial]. It is chosen when rendering large-scale urban environments on hardware with limited memory, offering a continuous memory-quality curve rather than a single fixed operating point [S2, unofficial]. It differs from standard hierarchical GS by explicitly using a "factoring tree" of scale bands and a one-parameter scheduler driven by optimal transport theory [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Construction of the hierarchy (e.g., the factoring tree in Budgeted-GS) takes a few seconds [S2, unofficial]. Rendering time depends on the selected level of detail and the efficiency of the traversal or filtering structure (e.g., traversal-free parallel filtering in FilterGS) [S1, unofficial].
*   **Memory Complexity:** Significantly reduced compared to flat Gaussian Splatting for large scenes, as only the necessary level of detail is loaded into memory [S1, unofficial][S2, unofficial].
*   **Strengths:** Enables real-time rendering of city-scale datasets on consumer hardware; adapts to view distance; allows a single model to serve GPUs with different memory capacities [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires additional post-processing or specialized training to build the hierarchy; complexity of implementation compared to flat splatting [S2, unofficial].
*   **When to use:** Large-scale urban scenes, VR/AR applications with memory constraints, or streaming scenarios [S1, unofficial][S2, unofficial].
*   **When not to use:** Small-scale scenes where the overhead of hierarchy construction is not justified, or when maximum possible detail is required regardless of memory [S2, unofficial].

## Background
The concept of hierarchical 3D Gaussian representations emerged as a solution to the scalability limits of standard 3D Gaussian Splatting. Early works like Octree-GS built octrees over scenes to associate nodes with merged Gaussians [S1, unofficial]. Subsequent developments included Google's LODGE, which introduced a level-of-detail framework for large-scale 3DGS [S1, unofficial], and Nvidia's HiGS, which proposed a hierarchical rendering architecture for real-time performance [S1, unofficial]. FilterGS further advanced this by proposing a traversal-free, parallel filtering structure [S1, unofficial]. Budgeted-GS formalized these ideas using optimal transport theory to define a "capacity floor" and "license" for redundancy, operationalizing them into a factoring tree and budget-centered training [S2, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
