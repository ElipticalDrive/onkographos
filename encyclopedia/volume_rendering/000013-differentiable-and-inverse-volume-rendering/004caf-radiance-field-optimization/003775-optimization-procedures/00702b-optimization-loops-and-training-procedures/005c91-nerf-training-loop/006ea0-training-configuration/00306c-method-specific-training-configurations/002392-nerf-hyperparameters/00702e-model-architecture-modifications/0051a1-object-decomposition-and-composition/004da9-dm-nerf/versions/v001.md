# DM-NeRF


## Overview
DM-NeRF is a method for simultaneously reconstructing continuous 3D scene geometry, decomposing individual objects in 3D space, and supporting flexible object shape manipulation (translation, rotation, size adjustment, deformation) from 2D images only [S1, unofficial]. It sets itself apart from standard NeRF by introducing an **object field** that predicts a unique one-hot object code for every 3D point, enabling instance-level decomposition and manipulation without explicit 3D mesh extraction [S1, unofficial]. Unlike sibling methods that focus solely on rendering quality or static reconstruction, DM-NeRF explicitly addresses the challenges of object ordering, empty space supervision, and visual occlusions during manipulation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $L$ images for a static scene with known camera poses and intrinsics, denoted as $\{(I_1, \xi_1, K_1) \dots (I_L, \xi_L, K_L)\}$ [S1, unofficial].
*   **Output:** Novel view renderings, 3D object decomposition (object codes), and manipulated scene geometry [S1, unofficial].

**Data Structures:**
*   **Object Code:** A one-hot vector representing the object identity for a 3D point [S1, unofficial].
*   **2D Object Prediction Tensor:** $I_l \in \mathbb{R}^{U \times V \times (H+1)}$, where $U, V$ are spatial width and height, and $H$ is the predefined number of objects [S1, unofficial].
*   **2D Ground Truth Label Tensor:** $\bar{I}_l \in \mathbb{R}^{U \times V \times T}$, where $T$ is the number of ground truth objects [S1, unofficial].
*   **MLP Architecture:** The network consists of layers with dimensions: $128 \rightarrow 256 \rightarrow 256 \rightarrow 256 \rightarrow 256 \rightarrow 256 \rightarrow 256 \rightarrow 256 \rightarrow 256 \rightarrow 256 \rightarrow 128$ [S1, unofficial].

**Method Steps:**
1.  **Positional Encoding:** Compute positional encoding $PE(x,y,z)$ for the 3D location and $PE(\theta,\phi)$ for the viewing direction [S1, unofficial].
2.  **Network Forward Pass:** Feed the encoded location and direction into the MLP. The network outputs:
    *   Volume density $\sigma \in \mathbb{R}$ (function of location only) [S1, unofficial].
    *   Object code $o \in \mathbb{R}^{H+1}$ (function of location only) [S1, unofficial].
    *   Color $c \in \mathbb{R}^3$ (function of location and viewing direction) [S1, unofficial].
3.  **2D Projection and Matching:**
    *   Project the 3D object codes to 2D to generate prediction images $I_l$ [S1, unofficial].
    *   Reshape predictions $I_l$ to matrix $M \in \mathbb{R}^{N \times H}$ where $N = U \times V$ [S1, unofficial].
    *   Reshape ground truth labels $\bar{I}_l$ to matrix $\bar{M} \in \mathbb{R}^{N \times T}$ [S1, unofficial].
    *   Apply the Hungarian algorithm to associate every ground truth 2D object mask with a unique predicted 2D object mask based on Soft Intersection-over-Union (sIoU) and Cross-Entropy Score (CES) [S1, unofficial].
4.  **Loss Computation:** Optimize the network using loss functions supervised by color images and 2D object masks. Specifically, apply losses $\ell_{3d}^{empty}$ and $\ell_{3d}^{obj}$ to jointly learn correct codes for empty 3D points [S1, unofficial].
5.  **Object Manipulation:**
    *   Specify target objects, viewing angles, and manipulation settings (e.g., translation, rotation) [S1, unofficial].
    *   Execute the 'Tackle visual occlusions' step using a lightweight inverse query algorithm to automatically edit the scene geometry, addressing visual occlusions and collisions without evaluating dense 3D points from implicit fields explicitly [S1, unofficial].
    *   Render the novel view from the manipulated geometry [S1, unofficial].

**Formulas:**
*   **sIoU Cost:** The Soft Intersection-over-Union (sIoU) cost between the $h$-th predicted box and the $t$-th ground truth box in the $l$-th pair is defined as:
    $$
    C^{sIoU}_{h,t} = \frac{\sum_{n=1}^{N} M_{n,h} \bar{M}_{n,t}}{\sum_{n=1}^{N} M_{n,h} + \sum_{n=1}^{N} \bar{M}_{n,t} - \sum_{n=1}^{N} M_{n,h} \bar{M}_{n,t}}
    $$
    *(Note: The exact summation formula is truncated in the source text, but the definition relies on sIoU and CES as described in [S1, unofficial].)*

**Parameters:**
*   $H$: Predefined number of objects, determines the channel dimension of 2D object prediction tensors [S1, unofficial].
*   $T$: Number of ground truth objects, determines the channel dimension of 2D ground truth label tensors [S1, unofficial].
*   $L$: Number of generated 2D images used for 2D object matching and supervision [S1, unofficial].
*   $U, V$: Spatial width and height dimensions of the 2D object prediction and ground truth label images [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Space:** Non-occupied (empty) 3D space must be taken into account, but there are no labels for supervision [S1, unofficial]. This is addressed by $\ell_{3d}^{empty}$ and $\ell_{3d}^{obj}$ [S1, unofficial].
*   **Object Ordering:** The number of total objects is variable and there is no fixed order for objects [S1, unofficial].
*   **Occlusion Artifacts:** Manipulating originally occluded objects or parts may produce artifacts due to inaccuracies in the learned radiance fields [S1, unofficial].
*   **Lighting:** Lighting factors are not decomposed, causing illumination discrepancies on edited objects compared to non-manipulated scenes [S1, unofficial].

## Subtopics

### Object Field
A component that learns a unique one-hot object code for every 3D point, enabling instance-level decomposition. It differs from standard NeRF by adding an auxiliary branch for object identity. Choose when instance-level manipulation is required. [Link](12095-semantic-and-feature-branches.md)

### Inverse Query Algorithm
A lightweight method to edit scene geometry and handle occlusions without explicit dense 3D point evaluation. It differs from naive mesh extraction by being more efficient. Choose when real-time or efficient manipulation is needed. [Link](11313-model-architecture-modifications.md)

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Novel view rendering quality is clearly better than PointNeRF, though it decreases after manipulation compared to non-manipulation baselines [S1, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Simultaneously reconstructs, decomposes, manipulates, and renders complex 3D scenes in a single pipeline from 2D views. Robust and accurate object decomposition [S1, unofficial].
*   **Limitations:** Lighting factors are not decomposed, causing illumination discrepancies. Manipulation of occluded objects may produce artifacts [S1, unofficial].
*   **When to use:** When 3D scene geometry decomposition and manipulation from 2D images is required.
*   **When not to use:** When lighting decomposition or high-fidelity rendering of manipulated occluded objects is critical without post-processing [S1, unofficial].

## Background
DM-NeRF was published as a conference paper at ICLR 2023 [S1, unofficial]. It was developed to address the need for virtual manipulation of objects in 3D scenes (e.g., mixed reality applications) by decomposing scenes into individual objects from 2D views only [S1, unofficial]. It builds on the success of NeRF by adding an object field component [S1, unofficial].

## Sources
[S1] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
