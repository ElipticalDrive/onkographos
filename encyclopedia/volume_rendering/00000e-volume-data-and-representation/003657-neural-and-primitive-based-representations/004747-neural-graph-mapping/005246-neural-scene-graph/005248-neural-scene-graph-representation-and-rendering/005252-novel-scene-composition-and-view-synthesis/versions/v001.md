# Novel Scene Composition and View Synthesis

Also called: Novel View Synthesis with Scene Graphs

## Overview
This method synthesizes novel views of dynamic scenes by leveraging a decomposed neural scene graph structure, where the scene is represented as a hierarchy of static and dynamic object nodes. It solves the problem of realistic view synthesis in complex environments by explicitly handling occlusions from background and other objects, and correctly rendering dynamic effects like shadows and reflections even when the viewing direction remains static. Unlike NeRF variants that cannot reason about scene dynamics, this approach allows for novel scene composition by sampling new transformations on road lanes or object trajectories without collisions [S1].

## How it works
The method operates on a neural scene graph where each object is an individual node, decomposing the scene into dynamic and static components.

**Inputs:**
- A sequence of images from a dynamic scene (e.g., KITTI dataset) [S1].
- A scene graph structure defining nodes (objects/background) and edges (relationships/occlusions) [S1].

**Outputs:**
- Novel view images of the scene from new camera poses or object configurations [S1].

**Data Structures:**
- **Scene Graph:** A directed graph where nodes represent objects or background regions and edges represent spatial or temporal relationships. The graph is sampled from existing nodes and edges in the data set, with new transformations applied to road lanes or object trajectories [S1].
- **Node Representation:** Each node contains parameters for its appearance and motion, allowing for independent manipulation [S1].

**Steps:**
1. **Scene Graph Construction:** Learn or construct a neural scene graph from input video frames. The graph decomposes the scene into distinct objects and background elements [S1].
2. **Novel Composition Sampling:** Sample new transformations for objects or road lanes from the scene graph. Ensure that no collisions occur between objects during this sampling process [S1].
3. **Occlusion Handling:** During rendering, handle occlusions from the background and other objects by leveraging the decomposed scene graph structure. This involves determining visibility based on the spatial relationships defined in the graph [S1].
4. **View Synthesis:** Render the novel view by combining the contributions of all nodes in the scene graph, respecting the occlusion relationships and dynamic properties [S1].

**Formulas:**
Not given in the sources.

**Parameters:**
- Collision constraints: Ensured during the sampling of new transformations [S1].

**Edge Cases and Pitfalls:**
- Collisions between objects must be explicitly prevented during the sampling of new transformations [S1].
- Occlusions from background and other objects must be correctly handled to avoid visual artifacts [S1].

## Subtopics

### Probabilistic Scene Graph Generation
A scene generation method using a textual scene graph interpreted as a probabilistic graphical model. The directed edge has a tail of the object node and a head of the interaction node. Scene generation becomes ancestor sampling by first generating the objects and then their interactions. The optimization is divided into two stages, where gravity and normal contact forces are introduced in the second stage [S2].

## Operating characteristics
- **Strengths:** The method is able to adequately synthesize shadows and reflections without changing viewing direction, which prior NeRF and time-aware NeRF methods cannot do [S1]. It allows for novel scene compositions by sampling new transformations on road lanes or object trajectories [S1].
- **Limitations:** Not given in the sources.
- **Performance:** Not given in the sources.

## Background
The method is part of the broader field of neural scene graphs for dynamic scenes, introduced in "Neural Scene Graphs for Dynamic Scenes" [S1]. It builds on the idea of decomposing scenes into dynamic and static components without additional supervision. The approach has been applied to problems such as 3D object detection, using the learned scene graphs as a forward model in a single-shot learning approach [S1].

## Sources
[S1] Neural Scene Graphs for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.00288, fetched 2026-10-26
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
