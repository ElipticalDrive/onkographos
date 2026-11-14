# Neural RGB-D Surface Reconstruction

Also called: Hybrid Scene Representation Network, Neural RGB-D

## Overview
Neural RGB-D Surface Reconstruction is a method for high-quality 3D reconstruction of room-scale scenes from an RGB-D input sequence [S1]. It solves the problem of generating metric, clean triangle meshes from noisy depth data, addressing the limitation of standard NeRF which produces semitransparent artifacts and lacks hard geometric boundaries [S1]. It sets itself apart by replacing the density-based volumetric representation with a hybrid scene representation: a Truncated Signed Distance Field (TSDF) for geometry and a volumetric radiance field for appearance, optimized jointly with camera poses [S1]. Unlike density-based models that introduce noise in free space, this method predicts hard boundaries between occupied and free space [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $N$ color frames $I_i$, corresponding aligned depth frames $D_i$, and initial camera poses $T_i$.
*   **Outputs:** Optimized scene representation (MLP parameters), refined camera poses, and a triangle mesh extracted via Marching Cubes.

**Data Structures:**
1.  **Shape MLP:**
    *   Input: Sinusoidal positional encoding of 3D query point $\gamma(\mathbf{p})$.
    *   Output: Truncated signed distance value $D_i$.
2.  **Radiance MLP:**
    *   Inputs: Sinusoidal positional encoding of viewing direction $\gamma(\mathbf{d})$, intermediate feature from Shape MLP, and a per-frame latent corrective code vector.
    *   Output: Surface radiance $\mathbf{c}_i$.
3.  **Deformation Field:** A 6-layer ReLU MLP acting as a residual to pixel location before unprojecting into a 3D ray. Shared across all frames.
4.  **Camera Poses $T_i$:** Represented with Euler angles and a translation vector for every frame.

**Parameters:**
*   Truncation distance $tr = 5$ cm [S1].
*   Scene scaling: Truncation region maps to $[-1, 1]$ (positive in front of surface, negative behind) [S1].
*   Loss weights: $\lambda_1 = 0.1$ (RGB), $\lambda_2 = 10$ (Free-space), $\lambda_3 = 6 \times 10^3$ (Truncation) [S1].
*   Fine samples $S'_f = 16$ [S1].
*   Coarse sampling rate: One sample for every 1.5 cm of ray length [S1].
*   Ray length: Greater than the largest distance in the scene (typical range 4 to 8 meters) [S1].
*   Optimizer: ADAM with learning rate $5 \times 10^{-4}$ [S1].
*   Training iterations: $2 \times 10^5$ iterations, computing gradient w.r.t. $|P_b| = 1024$ randomly chosen rays per iteration [S1].

**Algorithm:**

1.  **Initialization:** Initialize camera poses $T_i$ using BundleFusion [S1].
2.  **Ray Generation & Refinement:**
    *   For each pixel $p$ in a batch $P_b$, retrieve the 2D pixel coordinate.
    *   Apply the deformation field MLP to the pixel coordinate to get a 2D shift vector.
    *   Shift the pixel coordinate by this vector.
    *   Unproject the shifted pixel into a 3D ray using the camera pose $T_i$ and intrinsics.
3.  **Sampling:**
    *   **Coarse Step:** Generate $S'_c$ sample points on the ray using stratified sampling. The rate is set such that there is on average one sample for every 1.5 cm of ray length [S1].
    *   Evaluate the Shape MLP on these $S'_c$ points to get signed distances.
    *   Search for the zero-crossing in the predicted signed distance values to estimate the ray depth [S1].
    *   **Fine Step:** Generate $S'_f$ sample points around the identified zero-crossing [S1].
    *   Evaluate the Shape and Radiance MLPs on all $S'_c + S'_f$ samples.
4.  **Weight Calculation:**
    *   For each sample $i$ with signed distance $D_i$ and truncation distance $tr$, compute weight $w_i$:
        $$w_i = \sigma \left( \frac{D_i}{tr} \right) \cdot \sigma \left( -\frac{D_i}{tr} \right)$$
        where $\sigma$ is the sigmoid function [S1].
    *   **Pitfall:** If the coarse sampling rate is too low, samples may miss the truncation region, causing the zero-crossing to be missed [S1].
    *   **Multiple Intersections:** Set weights of samples beyond the first truncation region to zero to account for multiple intersections [S1].
5.  **Color Integration:**
    *   Compute the integrated color $\mathbf{C}$ for the ray:
        $$\mathbf{C} = \frac{1}{\sum_{i=0}^{K-1} w_i} \sum_{i=0}^{K-1} w_i \cdot \mathbf{c}_i$$
        where $K = S'_c + S'_f$, $w_i$ are weights, and $\mathbf{c}_i$ are radiance values [S1].
6.  **Loss Computation:**
    *   Compute the global objective function $\mathcal{L}(\mathcal{P})$ for the batch, where $\mathcal{P}$ includes network parameters $\Theta$ and camera poses $T_i$:
        $$\mathcal{L}(\mathcal{P}) = \sum_{b=0}^{B-1} \lambda_1 \mathcal{L}_{rgb}^b(\mathcal{P}) + \lambda_2 \mathcal{L}_{fs}^b(\mathcal{P}) + \lambda_3 \mathcal{L}_{tr}^b(\mathcal{P})$$
    *   **RGB Loss:**
        $$\mathcal{L}_{rgb}^b(\mathcal{P}) = \frac{1}{|P_b|} \sum_{p \in P_b} (\mathbf{C}_p - \hat{\mathbf{C}}_p)^2$$
        where $\mathbf{C}_p$ is predicted color, $\hat{\mathbf{C}}_p$ is observed color, and $P_b$ is the set of pixels in batch $b$ [S1].
    *   **Free-Space Loss:** Forces samples $s$ between the camera origin and the truncation region ($\mathcal{S}_p^{fs}$) to predict $tr$:
        $$\mathcal{L}_{fs}^b(\mathcal{P}) = \frac{1}{|P_b|} \sum_{p \in P_b} \frac{1}{|\mathcal{S}_p^{fs}|} \sum_{s \in \mathcal{S}_p^{fs}} (D_s - tr)^2$$
        where $D_s$ is predicted signed distance [S1].
    *   **Truncation Loss:** Forces samples $s$ within the truncation region ($S_p^{tr}$) to match observed signed distance $\hat{D}_s$ from the depth sensor:
        $$\mathcal{L}_{tr}^b(\mathcal{P}) = \frac{1}{|P_b|} \sum_{p \in P_b} \frac{1}{|S_p^{tr}|} \sum_{s \in S_p^{tr}} (D_s - \hat{D}_s)^2$$
        where $\hat{D}_s$ is the observed signed distance along the optical axis [S1].
7.  **Optimization:** Update $\Theta$, $T_i$, and per-frame latent corrective codes using ADAM [S1].
8.  **Mesh Extraction:** At evaluation time, use Marching Cubes with a spatial resolution of 1 cm to extract a triangle mesh from the optimized implicit scene representation [S1].

## Subtopics

### [InFusionSurf](12834-infusionsurf.md)
A NeRF-style RGB-D 3D surface reconstruction framework that refines depth information to enhance reconstruction quality with minimal impact on optimization time, using per-frame intrinsic refinement and TSDF Fusion prior learning. It addresses camera motion-induced blurs in depth frames by correcting the camera intrinsic matrix per frame and accelerates training by pre-training the geometric prior using classical TSDF Fusion. It builds on Neural RGB-D Surface Reconstruction by adding specific intrinsic refinement and a two-stage initialization process.

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On 10 synthetic scenes, the method achieved a Chamfer $\ell_1$ distance of 0.044, IoU of 0.747, Normal Consistency of 0.918, and F-score of 0.924 [S1].
*   **Strengths:** Produces high-quality, metrical 3D reconstructions with hard boundaries between occupied and free space, avoiding the semitransparent artifacts common in density-based models like NeRF [S1].
*   **Limitations:** Requires RGB-D input data. Performance depends on the quality of initial camera poses (initialized via BundleFusion) and the accuracy of the depth sensor [S1].
*   **When to use:** When high-quality, metric 3D mesh reconstruction is required from RGB-D sequences, particularly for room-scale scenes in AR/VR applications [S1].
*   **When not to use:** When only color images are available (no depth) or when novel view synthesis without explicit mesh extraction is the primary goal [S1].

## Background
Neural RGB-D Surface Reconstruction was proposed by Dejan Azinovic, Ricardo Martin-Brualla, Dan B Goldman, Matthias Nießner, and Justus Thies [S1]. It was published in 2022 [S1]. The method builds on the canonical NeRF architecture but reformulates the neural radiance field definition to use a differentiable rendering formulation based on signed distance fields specifically tailored to geometry reconstruction [S1]. It addresses the limitations of volume-based view synthesis methods that reproduce appearance but fail to reconstruct metric and clean meshes [S1].

## Sources
[S1] Neural RGB-D Surface Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00619, fetched 2026-10-26
