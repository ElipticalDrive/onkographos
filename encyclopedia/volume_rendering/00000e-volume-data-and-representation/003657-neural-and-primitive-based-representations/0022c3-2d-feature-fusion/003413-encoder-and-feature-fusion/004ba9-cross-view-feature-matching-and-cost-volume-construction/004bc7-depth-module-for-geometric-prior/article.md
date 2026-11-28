# Depth Module for Geometric Prior

Also called: Depth Map Regression Module, Geometric Prior Depth Estimation

## Overview
The Depth Module for Geometric Prior is a component in feed-forward 3D Gaussian Splatting architectures that regresses a dense per-pixel depth map from monocular features and a cost volume [S1]. This depth map serves as a geometric prior for lifting 2D image features into 3D space, enabling the construction of 3D point clouds and voxel-based features for volumetric reasoning [S1]. It distinguishes itself from sibling methods like Low-cost Cross-View Aggregation by explicitly providing the depth geometry required for the subsequent 3D lifting stage, rather than just aggregating features [S1].

## How it works
**Inputs:**
*   Monocular features $F_{mono} \in \mathbb{R}^{H \times W \times C}$ extracted from a context view [S1].
*   A cost volume $C_i$ constructed from cross-view feature matching [S1].

**Outputs:**
*   A dense per-pixel depth map $D_i \in \mathbb{R}^{H \times W}$ [S1].

**Data Structures:**
*   $F_{mono}$: Tensor of height $H$, width $W$, and channel dimension $C$.
*   $C_i$: Cost volume tensor aligned with the spatial dimensions of $F_{mono}$, representing similarity scores across disparity/depth hypotheses.
*   $D_i$: 2D tensor of height $H$ and width $W$, containing the predicted depth value for each pixel.

**Steps:**
1.  **Feature Fusion:** The module $f$ fuses the monocular features $F_{mono}$ with the cost volume $C_i$ [S1].
2.  **Depth Regression:** The fused representation is processed to regress a dense depth map $D_i$ [S1].
3.  **3D Lifting (Downstream):** The predicted depth map $D_i$ and camera parameters are used to unproject each pixel $(u, v)$ to a 3D point in the camera coordinate frame using camera intrinsics [S1].
4.  **Coordinate Transformation:** The 3D points are transformed into the global world coordinate system using the corresponding extrinsic parameters (rotation matrix $R_i$ and translation $t_i$) [S1].
5.  **Aggregation:** Features from different views are aggregated by transforming their respective point clouds into the global coordinate system [S1].

**Formulas:**
*   Depth map prediction: $D_i = f(F_{mono}, C_i)$ [S1].
*   Unprojection to camera frame: $\mathbf{X}_{cam} = K^{-1} \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} \cdot D_i$ (implicit in "unprojected... using camera intrinsics" [S1]).
*   World coordinate transformation: $\mathbf{X}_{world} = R_i \mathbf{X}_{cam} + t_i$ (implicit in "transformed... via extrinsic parameters" [S1]).

**Parameters:**
*   $H, W$: Height and width of the input image/features. Not given in the sources.
*   $C$: Channel dimension of monocular features. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The accuracy of the 3D lifting depends directly on the precision of the regressed depth map $D_i$ [S1].
*   **Camera Calibration:** The unprojection step requires accurate camera intrinsics [S1].
*   **Coordinate Consistency:** The extrinsic parameters must correctly align the camera frame with the global world frame for proper feature aggregation [S1].

## Subtopics

### Geometry-Guided Residual Encoding
A feature encoding strategy used in iterative refinement (e.g., DeltaSplat) that resolves spatial ambiguity in 3D positional updates by conditioning on 3D geometry (Plücker rays and rendered depth) alongside 2D image residuals [S2, unofficial]. It differs from the basic depth module by explicitly incorporating geometric context to regulate the direction and scale of predicted corrections [S2, unofficial]. Choose this variant when pose-free iterative refinement is required to disambiguate depth from appearance [S2, unofficial].

*   **Definition:** Inferring 3D positional updates from 2D image-space residuals is inherently underdetermined because displacing a Gaussian along its viewing ray yields an identical 2D projection, leaving the residual unchanged [S2, unofficial]. This method conditions the refinement on the 3D geometry of the current prediction [S2, unofficial].
*   **Input Construction:** For each context view, a 13-channel feature map is constructed by concatenating the image residual $r_i$, the current rendering $\hat{I}_i$, the Plücker-ray map $\ell_i$, and the depth map $d_i$ rendered alongside $\hat{I}_i$ by the same rasterizer [S2, unofficial].
*   **Plücker Ray Formula:** The entry $\ell_{i,u}$ of $\ell_i$ at pixel $u = (x, y)$ is given by:
    $$ \ell_{i,u} = [d_{i,u}, o_i \times d_{i,u}] \in \mathbb{R}^6 $$
    where $o_i \in \mathbb{R}^3$ is the center of camera $p_i$ and $d_{i,u} \in \mathbb{R}^3$ is the unit direction of the ray from $o_i$ through pixel $u$ [S2, unofficial].
*   **Pitfalls:** In pose-free settings, errors in estimated cameras displace both the residual and the geometric channels together, keeping them mutually consistent and ensuring refinement remains well-defined [S2, unofficial]. The geometric context acts as a soft prior that exposes the geometric structure of the ambiguity directly in the input [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** VolSplat achieves competitive results on the DL3DV testset (e.g., 24.53 PSNR with 6 views, 22.93 PSNR with 12 views, 22.17 PSNR with 24 views) [S2, unofficial]. DeltaSplat, using iterative refinement with geometric priors, achieves higher performance (e.g., 25.69 PSNR with 6 views, 24.34 PSNR with 12 views, 23.66 PSNR with 24 views) [S2, unofficial].
*   **Strengths:** Provides robust, multi-view consistent depth estimates [S1]. Enables explicit 3D lifting and volumetric reasoning [S1]. The geometric conditioning in variants like Geometry-Guided Residual Encoding resolves spatial ambiguity inherent in 2D residuals [S2, unofficial].
*   **Limitations:** Requires accurate camera parameters for lifting [S1]. The underdetermined nature of 2D residuals necessitates additional geometric constraints for precise 3D updates [S2, unofficial].
*   **When to use:** When 3D feature aggregation or volumetric reasoning is required from 2D features [S1]. When pose-free iterative refinement is needed to handle geometric ambiguities [S2, unofficial].
*   **When not to use:** When only 2D feature aggregation is sufficient and 3D lifting is not required [S1].

## Background
The Depth Module for Geometric Prior is a key component in feed-forward 3D Gaussian Splatting methods such as VolSplat [S1] and DeltaSplat [S2, unofficial]. VolSplat rethinks feed-forward 3D Gaussian Splatting by using voxel-aligned prediction and depth priors for volumetric reasoning [S1]. DeltaSplat introduces iterative Gaussian refinement for pose-free feed-forward 3D Gaussian Splatting, utilizing geometry-guided residual encoding to handle spatial ambiguity [S2, unofficial]. These methods build upon earlier work in iterative depth estimation, such as Teed & Deng (2020) [S2, unofficial].

## Sources
[S1] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
[S2] DeltaSplat: Iterative Gaussian Refinement for Pose-Free Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2610.09853, fetched 2026-10-29 (unofficial)
