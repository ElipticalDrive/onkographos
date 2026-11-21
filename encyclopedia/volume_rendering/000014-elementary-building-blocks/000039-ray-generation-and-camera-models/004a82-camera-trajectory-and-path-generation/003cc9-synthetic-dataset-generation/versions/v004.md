# Synthetic Dataset Generation

Also called: RealEstate10K stratification, Stratified Scene Dataset Generation, Synthetic stratified dataset creation, Training Data Generation, Training dataset generation for DEH

## Overview
Synthetic dataset generation is the procedure of creating labeled training images by rendering 3D scenes with controlled variations in lighting, camera poses, and material properties [S3, S4]. It solves the problem of insufficient or biased real-world data for training neural rendering and computer vision models by providing ground-truth annotations (depth, normals, segmentation) and diverse appearance variations [S3, S4]. It differs from sibling methods like [Synthetic Camera Trajectory Generation](7942-synthetic-camera-trajectory-generation.md) by focusing on the full pipeline of scene composition, texture/lighting randomization, and dataset balancing, rather than just the geometric path of the camera.

## How it works
The method varies significantly by application domain (volume visualization, holography, SLAM, NeRF). The following steps cover the common patterns found in the sources.

### Inputs and Outputs
*   **Inputs:** 3D scene geometry (meshes/volumes), texture libraries, camera parameters (FOV, resolution), lighting models, and scene layout constraints.
*   **Outputs:** A dataset $D$ consisting of tuples $(I, A, P)$, where $I$ is the rendered image (RGB/Depth), $A$ are annotations (depth, normals, instance masks), and $P$ are camera poses [S2, S5].

### Data Structures
*   **Scene Graph:** Nested structure levels (e.g., exterior, interior, object) [S1].
*   **Dataset Sample:** For SDDF models, a sample is defined as $s_j = (p_j, v_j, f^*_j, i^*_j, s^*_j)$, where $p_j$ is ray origin, $v_j$ is direction, $f^*_j$ is signed distance, $i^*_j$ is intersection indicator, and $s^*_j$ is sign indicator [S5].
*   **View Space:** Defined as a hemisphere (radius 4) for Synthetic-NeRF or a 3D bounding box for Tanks & Temple [S6, unofficial].

### Steps
1.  **Scene Composition:**
    *   For stratified scenes, define nested levels (Level 0: exterior, Level 1: interior, Level 2: object) [S1].
    *   For indoor scenes, compose rooms using object meshes (e.g., from free3d.com) with scale/pose transformations [S7, unofficial].
    *   For holography, ensure distances between camera and objects are significantly longer than distances between different objects to symmetrically blur either side of the focal plane [S4].
2.  **Texture and Lighting Randomization:**
    *   Sample textures from CC0 libraries for training and benchmark datasets for evaluation [S4].
    *   Randomly sample light source colors, orientations, and intensities. Restrict maximum intensity (e.g., <1000W or specific bounds) to prevent overexposure [S4][S7, unofficial].
    *   For volume visualization, uniformly sample Phong reflection coefficients: Ambient=1, Diffuse $\in [0.25, 0.75]$, Specular $\in [0.5, 1]$, Shininess $\in [20, 100]$ [S3].
3.  **Camera Trajectory and Pose Sampling:**
    *   Define trajectories using Catmull-Rom splines on control points [S2] or Bézier curves [S6, unofficial].
    *   Sample poses from curved hemispheres centered on focal objects [S1].
    *   For active mapping, sample waypoints uniformly from the trajectory (e.g., 100 waypoints) and select a subset (e.g., 10) for information gain calculation [S6, unofficial].
4.  **Rendering:**
    *   Render color and depth images using a renderer (e.g., Blender, BlenderProc) [S2, S4].
    *   Apply noise to depth maps to simulate real sensor noise [S2].
    *   For holography, render varifocal images with symmetric blur consistent with eye optics [S4].
5.  **Dataset Augmentation (SDDF Specific):**
    *   Convert sensor poses and range measurements into dataset samples [S5].
    *   Generate negative samples to balance sign indicators: for each sample $(p_j, v_j, f^*_j, i^*_j, s^*_j)$, generate $(p', v_j, -\epsilon, 1, -1)$ where $p' = p_j + (f^*_j + \epsilon)v_j$ and $\epsilon > 0$ [S5].

### Formulas
*   **Ray Generation (SDDF):** $p_j = p_t$, $v_j = R_t v_i$, $f^*_j = r_{t,i}$, $i^*_j = 1$, $s^*_j = 1$ [S5].
*   **Negative Sample Origin:** $p' = p_j + (f^*_j + \epsilon)v_j$ [S5].
*   **Holographic Blur Condition:** Distance(Camera, Object) $\gg$ Distance(Object 1, Object 2) [S4].

### Parameters
*   **Resolution:** 256x256 [S3], 400x400 [S7, unofficial].
*   **FOV:** 50 degrees [S7, unofficial].
*   **Viewpoints:** $N=2,352$ for volume categories [S3]; 100 waypoints for NeRF [S6, unofficial].
*   **Split:** Typically 30/15/15 or 30/10/10 images per level [S1].

### Edge Cases and Pitfalls
*   **Overexposure:** If light intensity is too high, intensity sums vary across varifocal images, violating energy conservation and preventing single wave field construction [S4].
*   **Depth Overfitting:** Pixel-wise depth distribution must be almost uniform (except background) to prevent overfitting to specific depths [S4].
*   **Sign Imbalance:** Raw SDDF data has all $s^*_j=1$ and $f^*_j \ge 0$. Negative samples are required for training stability [S5].
*   **Dynamic Motion:** Real scenes must filter out clips with dynamic motion to ensure static scene assumption [S1].

## Subtopics

### Dataset Generation and Augmentation
A procedure to convert sensor poses and range measurements into a dataset $D$ suitable for training the SDDF model, including augmentation for sign balance. It is distinct by its focus on balancing signed distance function signs through negative sample generation. Choose this when training signed distance field networks that require balanced positive/negative samples. [S5]

### Synthetic-NeRF View Space and Sampling
Defines the camera view space (hemisphere or bounding box) and sampling strategy for active viewpoint generation and information gain calculation. It is distinct by its use of spherical coordinates constrained to point at the object center and sparse 3D point sampling for efficiency. Choose this for active NeRF mapping where viewpoint selection is critical. [S6, unofficial]

### DM-SR Dataset Generation
Synthetic dataset creation for evaluating geometry manipulation, using 8 types of complex indoor rooms. It is distinct by its focus on instance-level rendering and object manipulation within composed indoor scenes. Choose this for tasks requiring instance segmentation and geometry decomposition. [S7, unofficial]

## Operating characteristics
*   **Time Complexity:** Training time scales with dataset size and scene complexity. For example, Strata-NeRF takes 6h 20m for 150k iterations on an RTX 3090 for a specific scene, compared to 5h 30m for mipNeRF-360 [S1].
*   **Memory Complexity:** Depends on the number of rendered images and annotations. Volume categories can generate up to $6.0 \times 10^5$ images [S3].
*   **Strengths:** Provides perfect ground truth; allows controlled variation of lighting and camera poses; avoids real-world sensor noise and occlusion ambiguities.
*   **Limitations:** Domain gap between synthetic and real data; computationally expensive to render large datasets; requires careful randomization to avoid overfitting to synthetic artifacts.
*   **When to Use:** When real data is scarce, expensive to collect, or lacks precise annotations.
*   **When Not to Use:** When the target application is highly sensitive to real-world sensor noise or complex lighting interactions not captured by the synthetic renderer.

## Background
Synthetic dataset generation originated from the need for labeled data in computer vision and volume visualization. Early work focused on volume rendering for medical imaging, using opacity and color transfer functions to generate training data for viewpoint estimation [S3]. Later, it expanded to neural rendering (NeRF) and SLAM, where synthetic scenes are used to train and evaluate reconstruction algorithms [S1, S2]. Recent developments include stratified scene generation for hierarchical modeling [S1] and holographic display training data with specific optical constraints [S4].

## Sources
[S1] Strata-NeRF : Neural Radiance Fields for Stratified Scenes, https://arxiv.org/abs/2308.10337v1, fetched 2026-10-26
[S2] Neural RGB-D Surface Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00619, fetched 2026-10-26
[S3] CNNs Based Viewpoint Estimation for Volume Visualization, https://doi.org/10.1145/3309993, fetched 2026-10-29
[S4] Diffraction-engineered holography: Beyond the depth representation limit of holographic displays, https://www.nature.com/articles/s41467-022-33728-5, fetched 2026-10-29
[S5] Learning Scene-Level Signed Directional Distance Function With Ellipsoidal Priors and Neural Residuals., https://doi.org/10.1109/tpami.2026.3688658, fetched 2026-10-26
[S6] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S7] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
