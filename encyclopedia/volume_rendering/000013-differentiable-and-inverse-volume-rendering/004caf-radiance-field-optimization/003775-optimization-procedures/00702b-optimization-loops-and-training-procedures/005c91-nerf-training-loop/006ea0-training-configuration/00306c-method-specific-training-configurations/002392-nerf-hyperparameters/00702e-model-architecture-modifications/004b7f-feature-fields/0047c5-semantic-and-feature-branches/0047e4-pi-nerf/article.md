# PI-NeRF

Also called: Partial-Invertible Neural Radiance Fields

## Overview
PI-NeRF is a framework that integrates NeRF with invertible neural networks (INN) to directly output the 6-DoF pose of a given image without pose initialization or iterative optimization, solving the inverse problem of view synthesis [S1]. It differs from existing NeRF-based pose estimation methods (like iNeRF and pNeRF) by directly outputting the pose without iterative optimization or initial pose, whereas existing methods rely on synthesizing images from a number of poses or iterative optimization, which is time-consuming and sensitive to initial poses [S1].

## How it works
PI-NeRF modifies the standard NeRF architecture to output high-dimensional features instead of RGB color, then uses an Invertible Neural Network Module (INM) to map between pixel features and ray representations. The method consists of three main components: NeRF-Feat, the INM, and a ResUNet-based feature extractor.

### Inputs and Outputs
*   **Input:** A single RGB image $I$ with unknown camera pose.
*   **Output:** The 6-DoF camera pose (rotation $\mathbf{R}$ and translation $\mathbf{t}$) relative to the scene.

### Data Structures
1.  **NeRF-Feat Model:** An MLP that maps a 3D point $\mathbf{x}$ and viewing direction $\mathbf{d}$ to a high-dimensional feature vector $\mathbf{f} \in \mathbb{R}^D$ and density $\sigma$. This replaces the standard RGB output $\mathbf{c} \in \mathbb{R}^3$.
2.  **Invertible Neural Network Module (INM):** Consists of two invertible neural networks (INNs) operating in forward and reverse directions.
    *   Forward path: Inputs are ray feature $\mathbf{f}_{ray}$ and ray representation $\mathbf{r}_{rep}$. Output is pixel feature $\mathbf{f}_{pix}$.
    *   Reverse path: Input is pixel feature $\mathbf{f}_{pix}$. Outputs are ray feature $\mathbf{f}_{ray}$ and ray representation $\mathbf{r}_{rep}$.
3.  **ResUNet:** A convolutional neural network used to extract pixel features $\mathbf{f}_{pix}$ from the input image $I$.

### Numbered Steps

1.  **NeRF-Feat Rendering:**
    For a ray $\mathbf{r}$ originating from the camera, sample $K$ points $\mathbf{x}_k$ along the ray. For each point, query the NeRF-Feat MLP to obtain the feature vector $\mathbf{f}(\mathbf{x}_k)$ and density $\sigma(\mathbf{x}_k)$. Compute the ray feature $\mathbf{f}_{ray}$ by aggregating these features using volume rendering:
    $$
    \mathbf{f}_{ray} = \sum_{k=1}^{K} \hat{T}_k \alpha(\sigma(\mathbf{x}_k)\delta_k) \mathbf{f}(\mathbf{x}_k)
    $$
    where $\hat{T}_k = \exp(-\sum_{j=1}^{k-1} \sigma(\mathbf{x}_j)\delta_j)$ is the accumulated transmittance, $\alpha(\cdot)$ is the opacity function, and $\delta_k$ is the distance between samples [S1]. This $\mathbf{f}_{ray}$ represents the high-dimensional feature for the ray.

2.  **Feature Extraction (ResUNet):**
    During training and inference, pass the input image $I$ through the ResUNet to extract a pixel feature map. Each pixel $p$ in the image corresponds to a ray $\mathbf{r}_p$ and has an associated pixel feature $\mathbf{f}_{pix}^{(p)}$ [S1].

3.  **Invertible Neural Network Module (INM) Forward Path (Training):**
    During training, for each pixel $p$, take the ray feature $\mathbf{f}_{ray}^{(p)}$ (computed from the NeRF-Feat model using the ground-truth camera pose) and the ray representation $\mathbf{r}_{rep}^{(p)}$ (derived from the ground-truth pose and geometry) as inputs to the forward path of the INM. The INM outputs a predicted pixel feature $\hat{\mathbf{f}}_{pix}^{(p)}$ [S1].

4.  **Barlow Twins Loss Computation (Training):**
    Compute the Barlow Twins (BT) loss between the predicted pixel feature $\hat{\mathbf{f}}_{pix}^{(p)}$ from the INM forward path and the actual pixel feature $\mathbf{f}_{pix}^{(p)}$ extracted by the ResUNet from the input image. This loss trains the ResUNet to generate features suitable for the reverse path of the INM and trains the INM to maintain invertibility [S1].

5.  **INM Reverse Path (Inference):**
    During inference, for each pixel $p$ in the input image:
    a. Extract the pixel feature $\mathbf{f}_{pix}^{(p)}$ using the trained ResUNet.
    b. Feed $\mathbf{f}_{pix}^{(p)}$ into the reverse path of the trained INM.
    c. The INM outputs the corresponding ray feature $\mathbf{f}_{ray}^{(p)}$ and ray representation $\mathbf{r}_{rep}^{(p)}$ [S1].

6.  **Pose Estimation via PnP:**
    The ray representations $\mathbf{r}_{rep}^{(p)}$ recovered by the INM correspond to the 3D rays in the scene. Use these rays and their corresponding 2D pixel locations to solve the Perspective-n-Point (PnP) problem, directly calculating the 6-DoF camera pose (rotation $\mathbf{R}$ and translation $\mathbf{t}$) without iterative optimization [S1].

### Parameters
*   **Feature Dimension $D$:** The dimension of the high-dimensional feature vector output by NeRF-Feat. Not given in the sources.
*   **Number of Samples $K$:** The number of points sampled along each ray for volume rendering. Not given in the sources.

### Edge Cases and Pitfalls
*   **Precision:** The method relies on the invertibility of the INM. If the INM is not perfectly invertible, the reverse path may produce inaccurate ray representations, leading to errors in the PnP solution [S1].
*   **Boundary Conditions:** The PnP algorithm requires a sufficient number of correspondences. If the image contains few distinct features or the INM fails to recover accurate ray representations for many pixels, the PnP solution may be unstable or fail [S1].
*   **Degenerate Input:** The method assumes the input image contains sufficient information to recover the camera pose. Images with low texture or ambiguous geometry may lead to poor feature extraction by ResUNet or inaccurate INM inversion [S1].

## Operating characteristics
*   **Time Complexity:** PI-NeRF is two orders of magnitude faster than existing NeRF-based methods (e.g., pNeRF takes more than 5 seconds, iNeRF is slower) [S1]. This is because it avoids iterative optimization and synthesizing images from multiple poses.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Experiments on synthetic and real-world datasets demonstrate competitive accuracy compared to iterative methods, while significantly reducing computation time [S1].
*   **Strengths:** Direct pose estimation without initial pose or iterative optimization; fast inference time; competitive accuracy.
*   **Limitations:** Relies on the accuracy of the INM and ResUNet; may struggle with degenerate scenes or low-texture images; requires training of multiple components (NeRF-Feat, INM, ResUNet).
*   **When to Use:** When fast, direct camera pose estimation from a single image is required, and iterative methods are too slow or sensitive to initialization.
*   **When Not to Use:** When high precision is required in complex scenes with limited texture, or when iterative optimization is acceptable and more robust.

## Background
PI-NeRF was proposed by Zhihao Li, Kexue Fu, Haoran Wang, and Manning Wang in 2023 [S1]. It addresses the limitations of existing NeRF-based pose estimation methods, which rely on synthesizing images from a number of poses to calculate the optimal pose, a process that is time-consuming and sensitive to initial poses [S1]. PI-NeRF is the first work to use NeRF for direct pose estimation by integrating it with invertible neural networks [S1].

## Sources
[S1] PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation, https://doi.org/10.1145/3581783.3612590, fetched 2026-11-08
