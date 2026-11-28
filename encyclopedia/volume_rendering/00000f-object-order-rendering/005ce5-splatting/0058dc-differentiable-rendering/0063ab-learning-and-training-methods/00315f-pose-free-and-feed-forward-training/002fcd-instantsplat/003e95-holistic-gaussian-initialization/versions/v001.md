# Holistic Gaussian Initialization


## Overview
Holistic Gaussian Initialization (HGI) is a strategy to generate dense and accurate initialized 3D Gaussians within seconds for endoscopic scenes, replacing the time-consuming and sparse Structure-from-Motion (SfM) pipeline (e.g., COLMAP) used in standard 3DGS [S1, unofficial]. It works for both binocular and monocular input sequences by leveraging depth estimation models to back-project pixels into world coordinates, creating a dense point cloud that serves as the initial state for Gaussian optimization [S1, unofficial]. It sets itself apart from standard 3DGS by eliminating the need for sparse SfM points, thereby accelerating initialization and improving density [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of input images $\{I_i\}_{i=1}^T$ (either binocular pairs $\{I^l_i, I^r_i\}$ or monocular $\{I_i\}$), camera intrinsic parameters $\mathbf{K}$, and extrinsic parameters $\mathbf{T}_i$ for each view. For binocular input, a stereo depth estimation model is required. For monocular input, a monocular depth estimation model is required.
*   **Outputs:** A set of initialized 3D Gaussians (positions, orientations, scales, opacities, colors) derived from the back-projected point clouds.

**Data Structures:**
*   **Depth Maps:** $\mathbf{D}_i$, a 2D array of depth values for each view $i$.
*   **Binary Mask:** $\mathbf{M}_i$, a 2D array of the same resolution as the image, where 1 indicates valid scene pixels and 0 indicates invalid pixels (e.g., surgical tools) [S1, unofficial].
*   **Point Cloud:** $\mathbf{P}_i$, a set of 3D points in world coordinates for view $i$.

**Steps:**

1.  **Depth Estimation:**
    *   If **binocular input**: Use a stereo depth estimation model to predict metric depth maps $\{\mathbf{D}_i\}_{i=1}^T$ for the left views [S1, unofficial].
    *   If **monocular input**: Use a monocular depth estimation model to predict relative depth maps $\{\mathbf{D}_i\}_{i=1}^T$ [S1, unofficial].

2.  **Re-projection to World Coordinates:**
    For each view $i$, re-project the pixels of the image $I_i$ into world coordinates to obtain the partial point cloud $\mathbf{P}_i$. The formula is:
    $$
    \mathbf{P}_i = \mathbf{K}^{-1}\mathbf{T}_i\mathbf{D}_i(\mathbf{I}_i \odot \mathbf{M}_i)
    $$
    where $\odot$ denotes element-wise product, $\mathbf{K}$ is the camera intrinsic matrix, $\mathbf{T}_i$ is the camera extrinsic matrix (world-to-camera or camera-to-world depending on convention, typically applied to homogeneous coordinates), and $\mathbf{D}_i$ is the depth map [S1, unofficial]. Note: The term $\mathbf{D}_i(\mathbf{I}_i \odot \mathbf{M}_i)$ implies using the depth value at each pixel location, masked by $\mathbf{M}_i$.

3.  **Combination:**
    Combine the partial point clouds $\{\mathbf{P}_i\}_{i=1}^T$ from all views into a single holistic point cloud [S1, unofficial]. The specific combination formula (Eq. 3 in source) is not explicitly detailed in the provided text, but it involves merging the points from all views [S1, unofficial].

4.  **Gaussian Initialization:**
    Initialize the Gaussians using the points from the combined point cloud.
    *   **Positions:** Set to the 3D coordinates from the point cloud.
    *   **Orientations:** Estimate normal directions from the initial point cloud (e.g., using [61] as cited in [S2]) and align Gaussian orientations $\mathbf{q}$ with these estimated normals [S2].
    *   **Scales:** Set one scaling axis smaller than the others to encourage disc-like surfels [S2].
    *   **Opacities/Colors:** Initialize based on the input image colors and standard opacity values.

**Parameters:**
*   **Number of initialization points:** Typically 1M back-projected points from the training dataset sensor depths [S2].
*   **Depth Models:** Stereo depth estimation model [11] (EndoNeRF) for binocular [S1, unofficial]; Monocular depth estimation model [25] for monocular [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Monocular Scale Ambiguity:** Relative depth maps from monocular models lose scale information. However, accurate reconstruction is still achieved because the Gaussian optimization process incorporates explicit geometric constraints from real poses and rendered images [S1, unofficial].
*   **Tool Occlusion:** Binary masks $\mathbf{M}_i$ are used to filter out surgical tool pixels, which are not part of the scene geometry [S1, unofficial].
*   **Limited Perspective:** A single image contains a limited perspective, so combining multiple views is necessary for a holistic point cloud [S1, unofficial].

## Subtopics

### Gaussian Initialization with Sensor Depth
This variant initializes the Gaussian scene using back-projected sensor depth readings for dense initialization, rather than sparse SfM points [S2]. It is chosen when sensor depth data is available. The steps are: 1. Back-project sensor depths into world coordinates. 2. Estimate normal directions from the initial point cloud using [61]. 3. Align Gaussian orientations $\mathbf{q}$ with estimated normals using Eq. (7). 4. Set one scaling axis smaller than the others [S2]. The number of initialization points is 1M back-projected points from the training dataset sensor depths [S2].

## Operating characteristics
*   **Time Complexity:** Initialization takes seconds, significantly faster than the minutes required for SfM pipelines [S1, unofficial].
*   **Memory Complexity:** Depends on the number of points (e.g., 1M points) and the resolution of depth maps [S2].
*   **Strengths:** Generates dense and accurate initialized points quickly; works for both binocular and monocular inputs; avoids sparse SfM limitations [S1, unofficial].
*   **Limitations:** Relies on the accuracy of depth estimation models; monocular input requires scale recovery during optimization [S1, unofficial].
*   **When to use:** When fast initialization is required, especially in endoscopic scenes where SfM may fail or be too slow [S1, unofficial].
*   **When not to use:** When high-precision metric depth is not available and monocular depth estimates are unreliable [S1, unofficial].

## Background
Holistic Gaussian Initialization was introduced in the EndoGaussian paper [S1, unofficial]. It addresses the limitation of standard 3DGS, which relies on sparse SfM points for initialization, leading to longer optimization times and potential inaccuracies in dynamic endoscopic scenes [S1, unofficial]. The method leverages recent advances in depth estimation models to provide a dense and accurate initial point cloud [S1, unofficial].

## Sources
[S1] EndoGaussian: Real-time Gaussian Splatting for Dynamic Endoscopic Scene Reconstruction, https://huggingface.co/papers/2401.12561, fetched 2026-10-30 (unofficial)
[S2] DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing, https://doi.org/10.1109/wacv61041.2025.00241, fetched 2026-10-26
