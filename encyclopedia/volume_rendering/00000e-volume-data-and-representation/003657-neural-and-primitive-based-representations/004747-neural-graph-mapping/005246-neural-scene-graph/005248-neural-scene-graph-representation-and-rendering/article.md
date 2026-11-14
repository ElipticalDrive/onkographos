# Neural Scene Graph Representation and Rendering

Also called: Neural Scene Graphs for Dynamic Scenes

## Overview
Neural Scene Graph Representation and Rendering is a hierarchical scene representation that decomposes dynamic scenes into static and dynamic components without additional supervision, representing each object as an individual node [S1]. It solves the temporal inconsistency and ghosting artifacts inherent in single-representation methods like SRNs and NeRF by reconstructing objects individually, thereby improving temporal consistency and enabling novel view synthesis and scene manipulation [S1]. Unlike SRNs that optimize to a single static representation for all frames, or NeRF which suffers from ghosting due to camera pose changes, this method maintains distinct nodes for dynamic entities [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of images (and optionally LiDAR point clouds) from a dynamic scene. For supervised learning, manual annotations of object bounding boxes/labels are used; for unsupervised/noisy learning, outputs from tracking methods (e.g., PointRCNN + AB3DMOT or CenterTrack) are used [S1].
*   **Outputs:** A learned neural scene graph where nodes represent static background, dynamic objects, and the camera, and edges define their hierarchical relationships and transformations [S1]. Rendered images from isolated subgraphs (static-only, dynamic-only) or composed graphs [S1].

**Data Structures:**
The scene graph is a directed acyclic graph $S = (W, C, F, L, E)$ [Parent].
*   $W$: Background node (static).
*   $C$: Camera leaf node.
*   $F$: Representation leaf nodes for static and dynamic objects.
*   $L$: Latent object code leaf nodes.
*   $E$: Edges defining the hierarchy and transformations between nodes [Parent].

**Algorithm Steps:**
1.  **Scene Graph Construction:**
    *   If using manual annotations, assign each detected object to a dynamic node and the remainder to the static background node [S1].
    *   If using noisy trackers, combine a detector (e.g., PointRCNN for LiDAR) with a tracker (e.g., AB3DMOT) or use a camera-only tracker (e.g., CenterTrack) to generate node assignments [S1].
2.  **Learning:**
    *   Optimize the neural representations of each node to minimize the difference between the rendered image and the ground truth image for each frame in the sequence [S1].
    *   The model learns global scene illumination, adjusting shadows and specular highlights on vehicles consistent with the learned lighting [S1].
3.  **Rendering Isolated Components:**
    *   To render isolated static components: Remove all dynamic nodes from the graph. Render the image from the scene graph containing only the camera node $C$ and the background node $W$ [S1].
    *   To render isolated dynamic components: Exclude the static background node $W$. Render the remaining graph containing the camera and dynamic object nodes [S1].
    *   **Shadow Handling:** The shadow of each object is assigned to its dynamic representation. This ensures shadows are visible when rendering dynamic-only components and are correctly occluded or integrated in full scene renders [S1].
4.  **Novel View Synthesis:**
    *   To synthesize novel views, modify the camera node's pose (e.g., translate the ego-vehicle forward by $\approx 2$ meters) while keeping other nodes fixed. The method handles occlusions from static objects like traffic lights and signs [S1].

**Parameters and Typical Values:**
*   **Dataset:** KITTI dynamic tracking sequences [S1].
*   **Sequence Length:** Up to 90 time steps or 9 seconds duration [S1].
*   **Resolution:** 1242 × 375 pixels [S1].
*   **Camera Setup:** Stereo camera setup, two perspectives [S1].
*   **Object Count:** Up to 12 unique dynamic objects per scene [S1].
*   **Novel View Translation:** Approximately 2 meters forward [S1].

**Edge Cases and Pitfalls:**
*   **Long-distance Tracking:** Monocular camera-only tracking (e.g., CenterTrack) degrades for objects at long distances where the detection and tracking stack cannot provide accurate depth information [S1].
*   **Ghosting in Baselines:** Standard NeRF suffers from severe ghosting with small camera pose changes because viewing directions for spatial points differ across frames [S1].
*   **Static Optimization Failure:** Vanilla SRNs fail to reconstruct dynamic scenes because they optimize to a single static representation for all frames [S1].

## Subtopics

### [Scene Graph Manipulation and Illumination Preservation](14707-scene-graph-manipulation-and-illumination-preservation.md)
This variant demonstrates the flexibility of the representation by manipulating edges and nodes. Changing edge transformations preserves global light transport components such as reflections and shadows, allowing for consistent illumination changes across the scene.

### [Novel Scene Composition and View Synthesis](14710-novel-scene-composition-and-view-synthesis.md)
This variant involves training on subsequences of a scene (leaving out single frames) to test generalization. It enables the composition of novel scenes by sampling nodes and edges, and synthesizing views where the ego-vehicle moves into the scene (e.g., 2m forward), accurately handling occlusions from static elements like traffic lights and signs.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
*   Evaluated on KITTI dataset sequences with up to 90 time steps, 9 seconds duration, 1242×375 resolution images from two perspectives, and up to 12 unique dynamic objects [S1].
*   The method outperforms baseline methods (vanilla SRN, NeRF) on all temporal metrics [S1].
*   NeRF adjusted for temporal changes improves quality but lacks detail and suffers from blurry, uncertain predictions [S1].

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Decomposes scenes into dynamic and static components without additional supervision; improves temporal consistency; allows for novel view synthesis and scene manipulation; handles occlusions well [S1].
*   **Limitations:** Reliance on tracking methods can degrade at long distances for monocular setups; manual annotation is costly (though trackers can mitigate this) [S1].

**When to use it and when not:**
Use for dynamic scenes requiring temporal consistency, novel view synthesis, or component isolation (static/dynamic). Not suitable for static scenes where simpler representations suffice, or for real-time applications if the optimization cost is prohibitive (not explicitly stated, but implied by the complexity of graph learning) [S1].

## Background
**Origin:**
The method was introduced in the paper "Neural Scene Graphs for Dynamic Scenes" [S1].

**History and Development:**
The work addresses limitations in existing neural rendering methods. SRNs optimize to a single static representation, failing for dynamic scenes. NeRF suffers from ghosting due to camera pose changes. This method was developed to reconstruct each object individually to improve temporal consistency and enable scene manipulation [S1].

**Where it is used:**
Evaluated on dynamic tracking sequences from the KITTI dataset [S1].

## Sources
[S1] Neural Scene Graphs for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.00288, fetched 2026-10-26
