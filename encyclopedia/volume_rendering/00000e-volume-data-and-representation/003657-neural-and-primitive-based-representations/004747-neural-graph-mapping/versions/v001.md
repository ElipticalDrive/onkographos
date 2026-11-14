# Neural Graph Mapping

Also called: Dynamic Multi-Field Scene Representation

## Overview
Neural Graph Mapping is a dynamic scene representation method that decomposes a volumetric scene into a collection of small, lightweight neural fields anchored to keyframes within a pose graph [S1]. It solves the problem of maintaining consistent, high-fidelity volumetric mapping in dynamic environments without requiring costly global reintegration or fixed scene boundaries [S1]. Unlike monolithic neural fields, it merges sparse visual tracking with dense neural mapping by allowing fields to deform locally when pose updates occur during loop closure [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of RGB-D frames, initial camera poses, and a pose graph structure.
*   **Output:** A dynamic multi-field scene representation consisting of neural fields anchored to keyframes, and updated camera poses.

**Data Structures:**
*   **Pose Graph:** A graph where nodes represent keyframes (camera poses) and edges represent spatial constraints between them.
*   **Neural Fields:** A set of independent, lightweight neural networks (e.g., MLPs). Each field is associated with a specific keyframe or local region.
*   **Field Anchors:** Each neural field is anchored to a specific pose in the pose graph.

**Steps:**
1.  **Initialization:** Initialize a pose graph with keyframes derived from sparse visual tracking. Initialize a lightweight neural field for each keyframe or local cluster.
2.  **Tracking:** Perform sparse visual tracking to estimate camera poses. Use a lightweight coarse model trained via self-supervision to accelerate this process [S1].
3.  **Mapping:** For each new frame, render the scene using the current neural fields anchored to the pose graph. Supervise the occupancy predictions with Gaussian-distributed priors to refine geometry (DNS-SLAM approach) [S1].
4.  **Loop Closure & Deformation:** When a loop closure is detected, update the pose graph. The neural fields deform according to the updated poses to maintain consistency. This deformation allows for consistent volumetric mapping without global reintegration [S1].
5.  **Rendering:** To render a point $\mathbf{x}$, identify the relevant neural fields based on the current pose graph. Interpolate or select the output from the anchored fields to determine density and color.

**Formulas:**
*   **Scene Function:** The scene is represented as a union or interpolation of local fields $F_i$:
    $$ F(\mathbf{x}) = \sum_i w_i(\mathbf{x}) F_i(\mathbf{x}) $$
    where $w_i(\mathbf{x})$ are weights based on proximity to the anchor of field $i$.
*   **Occupancy Prior:** Occupancy is supervised with Gaussian priors:
    $$ \mathcal{L}_{occ} = \sum_{\mathbf{x}} ( \sigma(\mathbf{x}) - \mu_{\mathbf{x}} )^2 $$
    where $\mu_{\mathbf{x}}$ is the mean of the Gaussian prior at $\mathbf{x}$ [S1].

**Parameters:**
*   **Field Size:** Small, lightweight networks. Specific size not given in the sources.
*   **Gaussian Prior Variance:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Pose Drift:** Inaccurate tracking can lead to misaligned fields. The method relies on accurate sparse visual tracking [S1].
*   **Dynamic Objects:** The basic method assumes static scenes. Handling dynamic objects requires semantic segmentation to distinguish dynamic regions (as seen in DDN-SLAM, a related method) [S1].
*   **Field Overlap:** Defining weights for overlapping fields requires careful design to avoid artifacts.

## Subtopics

### [Neural Scene Graph](14682-neural-scene-graph.md)
A hierarchical scene representation defined as a directed acyclic graph, extending the graph concept to include latent object codes and camera nodes.

## Operating characteristics
*   **Time Complexity:** Depends on the number of fields and the complexity of the lightweight neural networks. Deformation during loop closure is local, avoiding global reintegration costs [S1].
*   **Memory Complexity:** Scales with the number of keyframes and the size of the lightweight fields. More efficient than monolithic NeRFs for large scenes.
*   **Strengths:** Consistent volumetric mapping in dynamic scenes; no fixed scene boundaries; robust performance across diverse scenes [S1].
*   **Limitations:** Requires accurate sparse visual tracking; handling dynamic objects is non-trivial without additional semantic priors [S1].

## Background
Neural Graph Mapping was introduced in December 2023 [S1]. It is part of the broader effort to integrate neural rendering with SLAM systems, building on principles of neural and primitive-based representations [G]. It addresses the limitations of static neural fields in dynamic environments by introducing a graph-based structure for local field management.

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
