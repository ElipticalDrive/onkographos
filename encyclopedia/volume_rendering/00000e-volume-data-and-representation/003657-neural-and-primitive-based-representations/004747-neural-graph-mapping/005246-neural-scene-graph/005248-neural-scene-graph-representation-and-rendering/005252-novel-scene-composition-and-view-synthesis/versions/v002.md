# Novel Scene Composition and View Synthesis

Also called: Novel View Synthesis with Scene Graphs, Dynamic Scene Composition

## Overview
This method enables the synthesis of novel views and scene compositions by leveraging a decomposed neural scene graph structure, allowing for the correct rendering of dynamic effects like shadows and reflections without altering the viewing direction. Unlike standard NeRF variants that fail to reason about scene dynamics, this approach handles occlusions from background and other objects by treating the scene as a collection of independent nodes and edges, facilitating the generation of physically plausible multi-object arrangements.

## How it works
The method operates on a learned neural scene graph where each object is an individual node. The core capability is synthesizing novel views and compositions by sampling nodes and edges from the dataset or applying new transformations (e.g., on road lanes) while ensuring no collisions occur [S1].

**Inputs and Outputs:**
*   **Input:** A learned neural scene graph representation of a dynamic scene (e.g., from KITTI data), including node features for objects and edges for relationships.
*   **Output:** Novel view images and composed scenes with new object arrangements or camera poses.

**Data Structures:**
The scene is represented as a graph $G = (V, E)$, where $V$ are object nodes and $E$ are edges. Each node contains neural representations of the object's geometry and appearance. The specific internal layout of the neural fields is defined in the parent article [Neural Scene Graph Representation and Rendering](14688-neural-scene-graph-representation-and-rendering.md).

**Steps:**
1.  **Scene Decomposition:** The scene is decomposed into static and dynamic components, with each object represented as an individual node in the graph [S1].
2.  **Composition/Sampling:** For novel scene composition, new scene graphs are sampled from existing nodes and edges in the dataset. New transformations are sampled on specific structures (e.g., road lanes) to generate new arrangements [S1].
3.  **Collision Avoidance:** The sampling process explicitly forbids collisions between objects during the arrangement phase [S1].
4.  **Rendering:** The novel view is synthesized by rendering the composed graph. The method leverages the decomposed structure to handle occlusions from the background and other objects [S1].
5.  **Dynamic Effect Synthesis:** The renderer correctly synthesizes shadows and reflections even when the viewing direction remains unchanged, a capability absent in standard NeRF variants [S1].

**Parameters:**
*   **Collision Constraint:** Hard constraint applied during composition to prevent object overlap [S1].
*   **Transformation Sampling:** Applied to driveable space/road lanes to generate novel trajectories [S1].

**Edge Cases:**
*   **Occlusion:** Handled by the graph structure, allowing the renderer to account for background and inter-object occlusions [S1].
*   **Unseen Arrangements:** The method generalizes to translations and arrangements not observed during training [S1].

## Subtopics

### [Probabilistic Scene Graph Generation](14712-probabilistic-scene-graph-generation.md)
Probabilistic Scene Graph Generation (CG3D) is a text-to-3D scene synthesis method that explicitly models semantic relationships and physical interactions between objects using a scene graph interpreted as a probabilistic graphical model. Unlike methods that generate individual objects or extend from a single reference image, CG3D addresses the challenge of multi-object composition by enforcing structural consistency and physical plausibility through a two-stage optimization process.

## Operating characteristics
*   **Strengths:** Capable of synthesizing shadows and reflections correctly without changing the viewing direction, which standard NeRF and time-aware NeRF cannot do [S1]. It handles occlusions from background and other objects effectively [S1]. It allows for novel scene compositions with new transformations (e.g., ego-vehicle motion) that were not seen during training [S1].
*   **Limitations:** Requires a pre-learned neural scene graph representation. The quality of novel compositions depends on the sampling of nodes and edges from the dataset [S1].
*   **Performance:** Evaluated on KITTI data set sequences [S1]. Qualitative results and evaluations are provided in the source paper (Fig. 10, Tab. 1) [S1].

## Background
This method is part of the "Neural Scene Graphs for Dynamic Scenes" framework, which learns scene graphs from manual annotations or noisy tracker outputs [S1]. It was applied to scenes from the KITTI data set, demonstrating the ability to handle dynamic scenes and novel compositions [S1]. The approach also extends to 3D object detection by using the scene graph as a forward model [S1].

## Sources
[S1] Neural Scene Graphs for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.00288, fetched 2026-10-26
