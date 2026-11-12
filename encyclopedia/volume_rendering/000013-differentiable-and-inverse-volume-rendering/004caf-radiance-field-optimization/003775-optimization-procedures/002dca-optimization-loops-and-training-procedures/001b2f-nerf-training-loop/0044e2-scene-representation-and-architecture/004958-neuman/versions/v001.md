# NeuMan

Also called: Neural Human Radiance Field from a Single Video

## Overview
NeuMan is a method for novel view synthesis of a human subject interacting with a static scene, driven by a single video input [S1]. It jointly optimizes a background scene NeRF and a pose-driven human NeRF, allowing for the generation of novel views with novel human poses [S1]. It distinguishes itself from standard NeRF training by explicitly modeling the human as a deformable entity in a canonical space using SMPL parameters and an error-correction network, while simultaneously separating the human from the background to enable clean compositing [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single video sequence of a human in a scene.
*   **Output:** A trained scene NeRF ($F_S$), a human NeRF ($F_\Phi$), optimized SMPL parameters ($\theta_f$), and an error-correction network ($E$) for each frame $f$. The rendering output is a color image for a given camera pose and human pose.

**Data Structures:**
*   **SMPL Mesh ($\theta_f$):** Defines the pose and shape of the human for frame $f$.
*   **Error-Correction Network ($E$):** An MLP taking a 3D point $x_f$ and frame index $f$ as input, outputting a 3D displacement vector. Used only during training.
*   **Scene NeRF ($F_S$):** Standard MLP mapping 3D coordinates and viewing direction to color and density.
*   **Human NeRF ($F_\Phi$):** Standard MLP mapping canonical 3D coordinates and corrected viewing directions to color and density.
*   **Depth Maps:** Fused depth map $D_f^{use}$ combining MVS and monocular depth.
*   **Segmentation Masks:** Binary masks $Mask(\mathbf{r})$ indicating human presence.

**Numbered Steps:**

1.  **Preprocessing:**
    1.  Estimate camera poses, sparse scene model, and multi-view-stereo (MVS) depth maps ($D_{mvs}$) using COLMAP [S1].
    2.  Generate dense monocular depth maps ($D_{mono}$) using Miangoleh et al. [25].
    3.  Fuse $D_{mvs}$ and $D_{mono}$ into $D_f^{use}$ by finding a linear mapping between the two using pixels present in both, ensuring consistent scale [S1].
    4.  Retrieve human segmentation maps using Mask-RCNN [10]. Dilate these masks by 4% to ensure complete masking of the human [S1].
    5.  Estimate initial SMPL parameters ($\theta_f$) using ROMP [42].
    6.  Refine SMPL estimates by optimizing parameters against silhouettes [8, 48] and 2D joints [3, 4] [S1].
    7.  Align SMPL estimates to scene coordinates:
        *   Solve Perspective-n-Point (PnP) [14] between estimated 3D joints and projected 2D joints (using COLMAP intrinsics) to align orientation and position up to scale [S1].
        *   Determine the ground plane using RANSAC [S1].
        *   Solve for scale ambiguity by finding the scale factor that makes the SMPL feet meshes touch the ground plane, assuming the human stands on the ground in at least one frame [S1].

2.  **Scene NeRF Training:**
    *   Train the scene NeRF ($F_S$) using only background pixels (where $Mask(\mathbf{r}) = 0$) and the estimated camera poses [S1].
    *   Determine near and far planes for the scene NeRF using the estimated scene point cloud [S1].

3.  **Human NeRF Rendering Setup:**
    *   Determine near and far planes for the human NeRF using the estimated SMPL mesh [S1].
    *   For a ray $\mathbf{r}$ in observation space, sample points $x_f = \mathbf{r}(t)$ [S1].

4.  **Warping to Canonical Space:**
    *   For each sample point $x_f$ in observation space and corresponding SMPL mesh $\theta_f$, find the closest point on the mesh.
    *   Apply the rigid transformation $T_{\theta_f}$ of that closest point to warp $x_f$ to canonical space: $x'_f = T_{\theta_f}(x_f)$ [S1].
    *   Apply the error-correction network $E$ to correct warping errors:
        $$ \tilde{x}'_f = T_{\theta_f}(x_f) + E(x_f, f) $$
        where $E$ is an MLP [S1]. Note: $E$ is discarded during inference [S1].

5.  **Viewing Direction Correction:**
    *   Because straight lines in observation space curve in canonical space, recompute the viewing direction $d'_f$ for each sample $t_i$ based on the vector between consecutive warped samples:
        $$ d(t_i)'_f = \frac{\tilde{x}'_f(t_i) - \tilde{x}'_f(t_{i-1})}{\|\tilde{x}'_f(t_i) - \tilde{x}'_f(t_{i-1})\|} $$
        [S1]

6.  **Radiance Field Query:**
    *   Query the human NeRF with the corrected canonical coordinate and viewing direction:
        $$ c_h, \sigma_h = F_\Phi(\tilde{x}'_f, d'_f) $$
        [S1]

7.  **Compositing:**
    *   Shoot two rays: one for the human NeRF and one for the scene NeRF [S1].
    *   Evaluate colors and densities for both sets of samples.
    *   Sort all samples from both rays in ascending order based on depth [S1].
    *   Integrate using standard volume rendering equation (Eq. 2 in [S1]) to obtain the final pixel color [S1].

8.  **Joint Optimization:**
    *   Optimize SMPL parameters $\theta_f$, error-correction network $E$, and human NeRF parameters $\Phi$ jointly by minimizing the total loss $\mathcal{L}$ [S1].

**Formulas:**

*   **Empty Space Loss (Scene):** Encourages zero density between the camera and the estimated depth.
    $$ \mathcal{L}_{s,empty}(\mathbf{r}) = \int_{t_n}^{\alpha \hat{z}_\mathbf{r}} \sigma_s(\mathbf{r}(t)) dt $$
    where $\alpha = 0.8$ is a slack margin, $\hat{z}_\mathbf{r} = D_f^{use}(\mathbf{r})$ is the terminating depth, and $t_n$ is the near plane [S1].

*   **Total Scene Loss:**
    $$ \mathcal{L}_s = \mathcal{L}_{s,rgb}(\mathbf{r}) + \lambda_{empty}\mathcal{L}_{s,empty}(\mathbf{r}) $$
    where $\lambda_{empty} = 0.1$ [S1].

*   **Human RGB Loss:**
    $$ \mathcal{L}_{h,rgb}(\mathbf{r}) = Mask(\mathbf{r}) \|C_h(\mathbf{r}) - \hat{C}(\mathbf{r})\| $$
    where $C_h$ is rendered color and $\hat{C}$ is ground truth [S1].

*   **Mask Loss:** Encourages accumulated human alpha to match the binary mask.
    $$ \mathcal{L}_{mask}(\mathbf{r}) = Mask(\mathbf{r}) \|1 - \alpha_h(\mathbf{r})\| $$
    where $\alpha_h$ is accumulated alpha from the human NeRF [S1].

*   **SMPL Loss:** Enforces solid volume inside the canonical SMPL mesh and empty volume outside.
    $$ \mathcal{L}_{smpl}(\hat{x}'_f, \sigma_h) = \begin{cases} \|1 - \sigma_h\|, & \text{if } \hat{x}'_f \text{ inside SMPL mesh} \\ |\sigma_h|, & \text{otherwise} \end{cases} $$
    where $\hat{x}'_f$ is the canonical coordinate [S1].

*   **Hard Surface Loss:** Encourages sample weights to be binary (0 or 1) to reduce halos.
    $$ \mathcal{L}_{hard} = -\log(e^{-|w|} + e^{-|1-w|}) $$
    where $w$ is the transparency weight where the ray terminates [S1].

*   **Edge Loss:** Encourages sharp edges in canonical space by rendering a random straight ray.
    $$ \mathcal{L}_{edge} = -\log(e^{-|\alpha_c|} + e^{-|1-\alpha_c|}) $$
    where $\alpha_c$ is the accumulated alpha from a random straight ray in canonical space [S1].

*   **Total Human Loss:**
    $$ \mathcal{L} = \mathcal{L}_{h,rgb} + \lambda_{lpips}\mathcal{L}_{lpips} + \lambda_{mask}\mathcal{L}_{mask} + \lambda_{smpl}\mathcal{L}_{smpl} + \lambda_{hard}\mathcal{L}_{hard} + \lambda_{edge}\mathcal{L}_{edge} $$
    [S1]

**Parameters:**
*   $\lambda_{lpips} = 0.01$ [S1]
*   $\lambda_{mask} = 0.01$ (linearly decays to 0 through training) [S1]
*   $\lambda_{smpl} = 1.0$ [S1]
*   $\lambda_{hard} = 0.1$ [S1]
*   $\lambda_{edge} = 0.1$ [S1]
*   $\lambda_{empty} = 0.1$ [S1]
*   $\alpha = 0.8$ (slack margin) [S1]
*   LPIPS loss uses a $32 \times 32$ patch [S1]

**Edge Cases and Pitfalls:**
*   **SMPL Accuracy:** Initial SMPL estimates from ROMP are inaccurate; joint optimization is required to mitigate misalignment [S1].
*   **Scale Ambiguity:** The PnP solution is scale-ambiguous; the assumption that the human stands on a ground plane in at least one frame is critical for resolving scale [S1].
*   **Volume Collision:** Using the Da-pose (A-pose) canonical space avoids volume collision when warping legs, compared to T-pose [S1].
*   **Error Correction Overfitting:** The error-correction network $E$ overfits to each frame during training but is discarded for rendering, allowing the canonical volume to remain generalized [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   Seattle dataset: PSNR 24.71, SSIM 0.80, LPIPS 0.26 [S1].
    *   Citron dataset: PSNR 25.43, SSIM 0.79, LPIPS 0.31 [S1].
    *   Parking dataset: PSNR 23.98, SSIM 0.77, LPIPS 0.26 [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** First method to render a human together with a scene with novel human poses and novel views from a single video [S1]. Achieves best rendering quality across all scenes and metrics compared to NeRF-T and HyperNeRF [S1].
*   **Limitations:** Rendering quality depends on the estimated human pose [S1]. Requires accurate depth estimation and segmentation for preprocessing [S1].

## Background
NeuMan was introduced by W. Jiang et al. in the paper "NeuMan: Neural Human Radiance Field from a Single Video" [S1]. It builds on previous work in human NeRFs such as HumanNeRF [47] and THuman2.0 [19, 32], adapting them to the single-video setting by incorporating scene separation and robust preprocessing/alignment techniques [S1].

## Sources
[S1] NeuMan: Neural Human Radiance Field from a Single Video, https://doi.org/10.1007/978-3-031-19824-3_24, fetched 2026-10-29
