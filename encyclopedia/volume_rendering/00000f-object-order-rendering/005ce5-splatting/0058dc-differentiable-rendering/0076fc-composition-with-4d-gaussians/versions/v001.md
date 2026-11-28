# Composition with 4D Gaussians
Also called: 4D Gaussian editing, differential rendering projection

## Overview
Composition with 4D Gaussians is a technique for editing dynamic scenes by predicting deformed 3D Gaussians in a canonical space and projecting them to viewpoints via differentiable rendering [S1]. It enables the manipulation of 4D Gaussian representations, allowing for the visualization of tracking objects and scene editing in both monocular and multi-view settings [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\mathcal{G}$ in a canonical state, a Gaussian deformation field network $\mathcal{F}$, and a target viewpoint/time.
*   **Output:** A rendered image $\hat{I}$ composed of the deformed Gaussians.

**Data Structures:**
*   **3D Gaussians $\mathcal{G}$:** Each Gaussian is defined by position, scale, rotation, opacity, and spherical harmonics coefficients. In the cited implementation, the storage for $\mathcal{G}$ is approximately 10 MB [S1].
*   **Gaussian Deformation Field Network $\mathcal{F}$:** A neural network that predicts the deformation parameters (position, scale, rotation changes) for each Gaussian based on its canonical position and time. The storage for $\mathcal{F}$ is approximately 8 MB [S1].

**Steps:**
1.  **Deformation Prediction:** For each 3D Gaussian in $\mathcal{G}$, query the deformation field network $\mathcal{F}$ using the Gaussian's canonical position and the target time $t$ as inputs [S1].
2.  **Parameter Update:** Apply the predicted deformation to the Gaussian's attributes (position, scale, rotation) to obtain the deformed Gaussian $\mathcal{G}'$ [S1].
3.  **Projection:** Project the deformed 3D Gaussians $\mathcal{G}'$ onto the 2D image plane of the target viewpoint [S1].
4.  **Rasterization and Blending:** Rasterize the projected 2D Gaussians and blend them using alpha compositing to produce the final image $\hat{I}$ [S1]. The rendering process is defined as $\hat{I} = \mathcal{S}(M, \mathcal{G}')$, where $\mathcal{S}$ is the splatting function and $M$ represents the view matrix [S1].

**Formulas:**
*   Deformation: $\mathcal{G}'_i = \mathcal{F}(\mathcal{G}_i^{canonical}, t)$ [S1]
*   Rendering: $\hat{I} = \mathcal{S}(M, \mathcal{G}')$ [S1]

**Parameters:**
*   **Warm-up Stage:** Training without a warm-up coarse stage leads to suffering rendering quality [S1].
*   **Resolution:** Performance tested at $800 \times 800$ resolution [S1].

**Edge Cases and Pitfalls:**
*   **Numeric Errors:** Care must be taken to avoid numeric errors when optimizing the Gaussian deformation network $\mathcal{F}$ to ensure training stability [S1].
*   **Scalability:** Handling urban-scale reconstruction requires a more compact algorithm due to the heavy querying of the Gaussian deformation field by huge numbers of 3D Gaussians [S1].

## Subtopics

### 4D Gaussian Splatting
Adopts 4D K-Planes and a tiny MLP to encode voxel features from 3D Gaussians to aggregate nearby spatial and temporal Gaussians that share similar spatial and temporal information, then uses a multi-head Gaussian deformation decoder to compute the change in position, scale, and rotation [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The method is storage-efficient, requiring approximately 10 MB for 3D Gaussians $\mathcal{G}$ and 8 MB for the Gaussian deformation field network $\mathcal{F}$ in monocular tracking settings [S1].
*   **Speed:** Rendering speed depends on the number of Gaussians. At $800 \times 800$ resolution, if the number of rendered Gaussians is fewer than 30,000, the speed can reach up to 90 FPS on a single RTX 3090 GPU [S1].

**Strengths and Limitations:**
*   **Strengths:** Can present tracking objects in monocular settings with low storage requirements [S1]. Allows for editing in 4D Gaussians [S1].
*   **Limitations:** Heavy querying of the deformation field by large numbers of Gaussians makes it challenging for urban-scale reconstruction without compact algorithms [S1]. Real-time rendering requires balancing resolution, number of Gaussians, network capacity, and hardware [S1].

**When to Use:**
*   Suitable for real-time dynamic scene rendering where storage efficiency and editing capabilities are prioritized, particularly for scenes with fewer than 30,000 Gaussians at $800 \times 800$ resolution [S1].

## Background
The method builds on differentiable rendering principles to enable editing of 4D Gaussian representations [S1]. It is related to Dynamic3DGS [33] but extends capabilities to monocular settings with low storage [S1]. The approach was presented in the paper "4D Gaussian Splatting for Real-Time Dynamic Scene Rendering" [S1].

## Sources
[S1] 4D Gaussian Splatting for Real-Time Dynamic Scene Rendering, https://doi.org/10.1109/cvpr52733.2024.01920, fetched 2026-10-26
[S2] Dynamic Scene Reconstruction: Recent Advance in Real-time Rendering and Streaming, https://doi.org/10.48550/arXiv.2503.08166, fetched 2026-10-29 (unofficial)
