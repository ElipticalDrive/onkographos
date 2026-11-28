# Multi-view correspondence loss

Also called: MV-Corr

## Overview
A training loss that enforces geometric consistency across multiple training views by minimizing the re-projection error of corresponding rays [S1]. It acts on both camera pose estimates and the learned neural field to prevent overfitting to sparse input views [S1]. Unlike photometric loss [7], which fails to register poses in sparse regimes, MV-Corr provides strong supervision on rendered depth, implicitly encouraging it to be close to the true surface and favoring a solution consistent across all training images [S1].

## How it works
**Inputs and outputs:**
*   **Inputs:** A set of $N$ training images $\{I_i\}_{i=1}^N$, their initial (potentially noisy) camera poses $P_i$, and a differentiable volume renderer that produces rendered depth $D_i$ and color $C_i$ for each view [S1].
*   **Outputs:** A scalar loss value $L_{MV}$ used for gradient backpropagation to update poses and network weights [S1].

**Data structures:**
*   **Correspondences:** A set of pixel coordinates representing corresponding points across views [S1]. Not given in the sources.
*   **Ray representation:** A ray is defined by an origin and direction [G]. Not given in the sources.

**Numbered steps:**
1.  **Ray Sampling:** For a reference view $i$ and a target view $k$, sample a pixel $(u, v)$ in view $i$ [G].
2.  **Back-projection:** Construct the ray for pixel $(u, v)$ in view $i$ using the current pose $P_i$ [G].
3.  **Depth Query:** Query the rendered depth $D_i(u, v)$ from the neural field along the ray [S1].
4.  **3D Point Estimation:** Compute the 3D point $X$ in the world frame using the ray origin, direction, and sampled depth [G].
5.  **Re-projection:** Transform $X$ into the coordinate system of the target view $k$ using the current pose $P_k$ and project it onto the image plane of view $k$ to obtain predicted pixel coordinates $(u', v')$ [G].
6.  **Depth Sampling:** Sample the rendered depth $D_k(u', v')$ from the neural field at the predicted pixel location in view $k$ [S1].
7.  **Consistency Check:** Compute the loss by penalizing the discrepancy between the depth implied by the correspondence and the actual rendered depth in the target view, minimizing the rays' intersection re-projection error [S1].

**Formulas:**
The core intuition is minimizing the intersection re-projection error [S1]. The exact mathematical formulation is not explicitly expanded in the provided text [S1]. In practice, this involves a symmetric loss between views $i$ and $k$ that penalizes the reprojection discrepancy [S1].

**Parameters:**
*   **Number of views:** The method is designed for sparse regimes, e.g., $N=3$ views [S1].
*   **Pose Noise:** Evaluated with 15% additive Gaussian noise on initial poses [S1].

**Edge cases and pitfalls:**
*   **Sparse Views:** Photometric loss alone fails in sparse regimes; MV-Corr is specifically required to register poses in these conditions [S1].
*   **Disconnected Solutions:** Without multi-view constraints, the NeRF/poses can converge to disconnected solutions for each training image [S1].
*   **Background:** DTU scenes have black backgrounds, which can impact PSNR values if color prediction is wrong, though MV-Corr focuses on geometry [S1].

## Subtopics

### Depth Consistency Module
A loss objective that directly enforces the learned scene to be consistent from any viewpoint, boosting performance when combined with multi-view correspondences [S1]. It works best in collaboration with multi-view correspondences loss (8) because the latter is needed to learn an accurate geometry first; on its own or without (8), it yields lower performance [S1]. On DTU with 3 views and fixed GT poses, adding DCons to the baseline yields PSNR 18.30 (21.01), SSIM 0.78 (0.87), LPIPS 0.21 (0.10), DE 0.08, compared to no DCons which yields PSNR 18.13 (20.81), SSIM 0.77 (0.87), LPIPS 0.22 (0.10), DE 0.10 [S1].

## Operating characteristics
**Time and memory complexity:**
Not given in the sources.

**Measured performance:**
On DTU with 3 views and noisy poses, combining Photo + MVCorr yields PSNR 16.0 (17.8), SSIM 0.68 (0.81), LPIPS 0.28 (0.14), DE 0.13, outperforming Photo alone which yields PSNR 10.7 (9.8), SSIM 0.43 (0.62), LPIPS 0.59 (0.36), DE 1.9 [S1].

**Scaling:**
Not given in the sources.

**Strengths and limitations:**
*   **Strengths:** Drastically outperforms photometric loss in terms of pose and learned geometry (depth error) in sparse regimes [S1]. Favors a NeRF/pose solution consistent across all training images [S1].
*   **Limitations:** Requires accurate geometry learning first to be effective when combined with other consistency modules [S1].

**When to use it and when not:**
Use it when training NeRFs from sparse views with noisy poses where photometric loss fails to register poses [S1]. It is particularly effective when combined with photometric loss for the best overall performance [S1].

## Background
Originated in SPARF: Neural Radiance Fields from Sparse and Noisy Poses [S1]. The method was developed to address the failure of existing regularization approaches in joint pose-NeRF training in the sparse regime [S1].

## Sources
[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
