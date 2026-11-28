# Neural Graph Mapping


## Overview
Neural Graph Mapping is a dynamic scene representation method that decomposes a volumetric scene into a collection of small, lightweight neural fields anchored to keyframes within a pose graph [S1]. It solves the problem of maintaining consistent, high-fidelity volumetric mapping in dynamic environments without requiring costly global reintegration or fixed scene boundaries [S1]. Unlike monolithic neural fields, it merges sparse visual tracking with dense neural mapping by allowing fields to deform locally when pose updates occur during loop closure [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of RGB-D frames and initial camera poses [S1].
*   **Output:** A dynamic multi-field scene representation consisting of neural fields anchored to keyframes, and updated camera poses [S1].

**Data Structures:**
*   **Pose Graph:** A graph where nodes represent keyframes and edges represent spatial constraints between them [G]. Exact layout and edge weight formulation are not given in the sources [S1].
*   **Neural Fields:** A set of independent, lightweight neural networks associated with specific keyframes or local regions [S1]. Exact network architecture and bit-packing are not given in the sources [S1].
*   **Field Anchors:** Each neural field is anchored to a specific pose in the pose graph [S1].

**Steps:**
1.  **Initialization:** Initialize a pose graph with keyframes derived from sparse visual tracking [S1]. Initialize a lightweight neural field for each keyframe [S1].
2.  **Tracking:** Perform sparse visual tracking to estimate camera poses [S1]. A lightweight coarse model trained via self-supervision accelerates tracking [S1].
3.  **Mapping:** For each new frame, render the scene using the current neural fields anchored to the pose graph [S1].
4.  **Loop Closure & Deformation:** When a loop closure is detected, update the pose graph [S1]. The neural fields deform according to the updated poses to maintain consistency, enabling consistent volumetric mapping without global reintegration [S1].
5.  **Rendering:** To render a point $\mathbf{x}$, identify the relevant neural fields based on the current pose graph [G]. Interpolate or select the output from the anchored fields to determine density and color [G].

**Formulas:**
*   **Scene Aggregation:** The scene function combines local fields $F_i$ at query point $\mathbf{x}$: $F(\mathbf{x}) = \sum_i w_i(\mathbf{x}) F_i(\mathbf{x})$, where $w_i(\mathbf{x})$ are proximity-based weights [G]. The exact aggregation function and weight computation are not given in the sources [S1].

**Parameters:**
*   **Field Size:** Small, lightweight networks [S1]. Typical size is not given in the sources.

**Edge Cases and Pitfalls:**
*   **Pose Drift:** Inaccurate tracking leads to misaligned fields; the method relies on accurate sparse visual tracking [S1].
*   **Dynamic Objects:** The basic method assumes static scenes; handling dynamic objects is not explicitly addressed in the base formulation [G].
*   **Field Overlap:** Defining weights for overlapping fields requires careful design to avoid artifacts [G].

## Subtopics

### [Neural Scene Graph](14682-neural-scene-graph.md)
Neural Scene Graphs (NSG) represent dynamic scenes as a hierarchical directed acyclic graph that separates static background elements from dynamic foreground objects, enabling temporal consistency and accurate modeling of moving components. Unlike Single Representation Networks that optimize a single static representation for all frames, NSG reconstructs each object individually using shared weights by class, ensuring distinct appearance and shape modeling for rigidly connected components.

## Operating characteristics
*   **Time Complexity:** Depends on the number of fields and the complexity of the lightweight neural networks [G]. Deformation during loop closure is local, avoiding global reintegration costs [S1].
*   **Memory Complexity:** Scales with the number of keyframes and the size of the lightweight fields [S1].
*   **Strengths:** Consistent volumetric mapping in dynamic scenes; no fixed scene boundaries; robust performance across diverse scenes [S1].
*   **Limitations:** Requires accurate sparse visual tracking [S1]. Handling dynamic objects is not explicitly addressed in the base method [G].

## Background
Neural Graph Mapping was introduced in December 2023 [S1]. It is part of the broader effort to integrate neural rendering with SLAM systems, building on principles of neural and primitive-based representations [G]. It addresses the limitations of static neural fields in dynamic environments by introducing a graph-based structure for local field management [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
