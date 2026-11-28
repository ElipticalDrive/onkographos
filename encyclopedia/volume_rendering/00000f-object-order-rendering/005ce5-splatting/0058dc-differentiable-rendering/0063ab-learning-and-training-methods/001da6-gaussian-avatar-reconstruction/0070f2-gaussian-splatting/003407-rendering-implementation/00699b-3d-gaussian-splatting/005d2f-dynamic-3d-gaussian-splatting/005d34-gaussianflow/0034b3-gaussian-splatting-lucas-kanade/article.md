# Gaussian Splatting Lucas-Kanade


## Overview
Gaussian Splatting Lucas-Kanade is a method that directly applies optical flow estimation to the rendered Gaussians, using the flow to align dynamic objects across frames and improve temporal consistency [S1, unofficial]. It distinguishes itself from parent method [GaussianFlow](6149-gaussianflow.md), which uses an explicit flow field for temporal regularization, by operating directly on the rendered Gaussian primitives rather than an explicit geometric point cloud or a separate flow MLP [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of 2D image frames and a set of 3D Gaussians representing the scene.
*   **Output:** Temporally consistent 3D Gaussian parameters (position, covariance, opacity, color) aligned across frames.

**Data Structures:**
*   **Gaussian Set:** A collection of $N$ Gaussians. Each Gaussian $i$ is defined by position $\mathbf{p}_i$, covariance matrix $\mathbf{\Sigma}_i$, opacity $\alpha_i$, and spherical harmonics coefficients for color $\mathbf{c}_i$.
*   **Optical Flow Field:** A 2D vector field $\mathbf{u}(x, y)$ representing pixel displacement between frames.

**Steps:**
1.  **Render:** Render the current frame using the current set of 3D Gaussians via differentiable rasterization.
2.  **Estimate Flow:** Compute the optical flow $\mathbf{u}(x, y)$ between the rendered frame and the target frame (or previous frame) using a standard Lucas-Kanade optical flow algorithm or a learned equivalent.
3.  **Align Gaussians:** Map the 2D optical flow to 3D space. For each Gaussian, determine its projected 2D center $\mathbf{c}_i$. Apply the flow vector $\mathbf{u}(\mathbf{c}_i)$ to update the Gaussian's position or adjust its parameters to minimize the photometric error between the rendered and target frames.
4.  **Update:** Update the Gaussian parameters based on the flow alignment. This may involve translating the Gaussian centers or adjusting their covariances to account for motion blur or deformation.
5.  **Iterate:** Repeat for subsequent frames to maintain temporal consistency.

**Formulas:**
*   **Lucas-Kanade Optical Flow:** Solves for $\mathbf{u}$ by minimizing:
    $$ \sum_{i \in W} \left( I_1(x_i + u_x, y_i + u_y) - I_0(x_i, y_i) \right)^2 $$
    where $I_0, I_1$ are image intensities, $W$ is a window around the pixel, and $\mathbf{u} = (u_x, u_y)$.
*   **Gaussian Update:** The position $\mathbf{p}_i$ is updated by $\Delta \mathbf{p}_i$ derived from the flow.

**Parameters:**
*   **Window Size:** For Lucas-Kanade, typically $5 \times 5$ or $7 \times 7$ pixels.
*   **Regularization:** Not specified in sources.

**Edge Cases:**
*   **Occlusions:** Flow estimation may fail in occluded regions.
*   **Large Motions:** Standard Lucas-Kanade assumes small motions; large displacements may require pyramidal approaches.
*   **Degenerate Input:** Low-texture regions may yield unreliable flow.

## Subtopics

### [Scene Flow Prior](18519-scene-flow-prior.md)
Scene Flow Prior enhances the temporal consistency of 4D spatio-temporal representations by incorporating a flow Multi-Layer Perceptron (MLP) for motion estimation. It maps spatio-temporal coordinates from $\mathbb{R}^4$ to motion fields in $\mathbb{R}^3$, predicting displacement only between adjacent frames to avoid the difficulty of establishing long-term correspondences in deformable scenes. Unlike its parent method, Gaussian Splatting Lucas-Kanade, which applies optical flow to rendered Gaussians, this method operates on explicit geometric point clouds derived from LiDAR data to impose a motion prior and geometric supervision.

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians and the resolution of the flow estimation.
*   **Memory Complexity:** Linear in the number of Gaussians.
*   **Strengths:** Direct application to Gaussians; improves temporal consistency.
*   **Limitations:** Relies on accurate optical flow; may struggle with large motions or occlusions.
*   **When to Use:** Dynamic scenes requiring temporal consistency in Gaussian Splatting.
*   **When Not to Use:** Scenes with large, unstructured motions or significant occlusions where flow estimation is unreliable.

## Background
Gaussian Splatting Lucas-Kanade is a method that directly applies optical flow estimation to the rendered Gaussians, using the flow to align dynamic objects across frames and improve temporal consistency [S1, unofficial]. It builds on the principles of 3D Gaussian Splatting and optical flow estimation.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
