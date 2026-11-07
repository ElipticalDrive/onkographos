# StreetGaussians


## Overview
StreetGaussians is a method for driving scenes that models dynamic urban environments by associating each vehicle or pedestrian with a separate 3DGS sub-model that can be moved independently [S1, unofficial]. It solves the problem of rendering complex, dynamic traffic in autonomous driving simulations by separating static background geometry from dynamic actor geometry, allowing for explicit pose-based animation of actors without retraining the entire scene [S2, unofficial]. It sets itself apart from monolithic 3DGS approaches by enabling the insertion of rare-event vehicles into any scene with sensor realism [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of multi-view images of a driving scene; tracked trajectories and poses for dynamic actors (vehicles, pedestrians); a static background model.
*   **Outputs:** A rendered image of the dynamic scene; a scene representation consisting of a static 3DGS cloud and a list of dynamic actor sub-models with associated transformation matrices.

**Data Structures:**
1.  **Static Background:** Standard 3DGS representation: a list of $N_s$ Gaussians, each with position $\mathbf{p}_i$, rotation quaternion $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_{i,lm}$ [S2, unofficial].
2.  **Dynamic Actors:** A list of $K$ actors. Each actor $k$ has:
    *   A separate 3DGS sub-model: a list of $N_k$ Gaussians with parameters $\{\mathbf{p}_{k,j}, \mathbf{q}_{k,j}, \mathbf{s}_{k,j}, \alpha_{k,j}, \mathbf{c}_{k,j,lm}\}$ [S1, unofficial].
    *   A global transformation matrix $\mathbf{T}_k(t)$ representing the actor's pose (position, orientation, scale) at time $t$ [S2, unofficial].

**Steps:**
1.  **Scene Decomposition:** Separate the scene into static background regions and dynamic actor regions [S2, unofficial].
2.  **Static Background Training:** Train a standard 3DGS model on the static parts of the scene (buildings, roads, sky) using multi-view images [S2, unofficial].
3.  **Per-Actor Model Training:** For each dynamic actor (vehicle/pedestrian), train a separate 3DGS sub-model using images where the actor is visible [S1, unofficial].
4.  **Pose Association:** Associate each actor's sub-model with its tracked trajectory, providing explicit pose parameters $\mathbf{T}_k(t)$ for any time $t$ [S1, unofficial].
5.  **Rendering:**
    *   Render the static background using standard 3DGS rasterization.
    *   For each dynamic actor $k$, apply the transformation $\mathbf{T}_k(t)$ to the actor's Gaussian parameters:
        $$ \mathbf{p}'_{k,j} = \mathbf{T}_k(t) \cdot \mathbf{p}_{k,j} $$
        $$ \mathbf{q}'_{k,j} = \mathbf{q}_k(t) \otimes \mathbf{q}_{k,j} $$
        (where $\otimes$ is quaternion multiplication) [S2, unofficial].
    *   Rasterize the transformed dynamic Gaussians over the static background using alpha blending [S1, unofficial].

**Parameters:**
*   Number of actors $K$: Depends on scene density.
*   Actor sub-model resolution: Typically lower than background to save memory, as actors are smaller in the frame [S2, unofficial].

**Edge Cases:**
*   **Occlusion:** Actors occluding each other or the background must be sorted by depth after transformation [G].
*   **Pose Interpolation:** $\mathbf{T}_k(t)$ must be interpolated smoothly between tracked keyframes to avoid jitter [G].

## Subtopics

### DrivingGaussian
DrivingGaussian is a variant that composites surrounding views from multiple cameras into a single dynamic scene, aligning the Gaussian clouds of different vehicles with their tracked trajectories [S1, unofficial]. It is chosen when multi-camera setups are available and precise trajectory alignment is required for simulation consistency. It differs from StreetGaussians by explicitly focusing on the compositing and alignment of multi-view data for dynamic actors [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time is $O(N_s + \sum_{k=1}^K N_k)$, where $N_s$ is static Gaussians and $N_k$ is actor $k$'s Gaussians. Training is faster than monolithic 3DGS for dynamic scenes as actors can be trained independently [S1, unofficial].
*   **Memory Complexity:** $O(N_s + \sum_{k=1}^K N_k)$. Significantly reduces memory compared to modeling the entire dynamic scene with a single large 3DGS cloud, as background regions require fewer Gaussians [S1, unofficial].
*   **Strengths:** Enables insertion of rare-event vehicles into any scene; supports explicit pose control; reduces background Gaussian count [S2, unofficial].
*   **Limitations:** Requires accurate tracking and pose estimation for actors; complex occlusion handling between dynamic actors [G].
*   **When to Use:** Autonomous driving simulation, dynamic urban scene rendering where actors need independent control.
*   **When Not to Use:** Static scenes, or scenes with highly interdependent dynamic elements that are hard to segment [G].

## Background
StreetGaussians was introduced as part of the evolution of 3D Gaussian Splatting for autonomous driving and outdoor scenes [S2, unofficial]. It adapts the static/dynamic decomposition pattern (similar to EmerNeRF) to the Gaussian Splatting framework, becoming a dominant pattern for AV simulation in 2026 [S2, unofficial]. It allows for the placement of rare-event vehicles (e.g., school buses) into any scene with sensor realism [S2, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
