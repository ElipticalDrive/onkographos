# NeRF-SR Training Configuration

Also called: NeRF-Supersampling, Ours-SS, Ours-Refine

## Overview
NeRF-SR is a training configuration that enhances the resolution of Neural Radiance Fields (NeRF) by integrating a supersampling strategy directly into the volume rendering process, allowing the network to learn high-frequency details from low-resolution inputs without requiring high-resolution ground truth [S1]. It distinguishes itself from standard NeRF by explicitly modeling the camera's point spread function (PSF) during training to prevent aliasing, and from post-hoc upsampling methods (like NeRF-Bi) by maintaining 3D consistency through differentiable rendering at multiple scales [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Low-resolution (LR) images of a scene, camera intrinsics ($K$) and extrinsics ($T$), and estimated depth maps ($d$) for the input views.
*   **Outputs:** High-resolution (HR) novel view images rendered from the trained NeRF model.

**Data Structures:**
The method relies on the standard NeRF MLP architecture mapping 5D coordinates $(x, y, z, \theta, \phi)$ to density $\sigma$ and color $c$. No new data structures are introduced for the core supersampling mechanism; it modifies the sampling and loss computation steps.

**Numbered Steps:**

1.  **Initialization:** Initialize a standard NeRF MLP. Set the optimizer to Adam with $\beta_1 = 0.9, \beta_2 = 0.999$. Set the batch size to 2048 rays. Initialize the learning rate to $5 \cdot 10^{-4}$.
2.  **Ray Sampling:** For each batch, sample 2048 rays. For each ray, perform hierarchical sampling:
    *   Generate 64 coarse samples along the ray using a uniform distribution.
    *   Query the coarse MLP to obtain densities.
    *   Use the densities to compute a probability distribution for sampling 64 fine samples [S1].
3.  **Supersampling Loss Computation:** Unlike standard NeRF which minimizes pixel-wise MSE between rendered and ground truth images, NeRF-SR computes the loss by simulating the camera's integration process.
    *   For each pixel in the *input* LR image, render the color $C_{rendered}$ using the fine samples.
    *   The ground truth color $C_{GT}$ for that pixel is the average of the high-resolution pixels that map to it.
    *   The loss is the Mean Squared Error (MSE) between $C_{rendered}$ and $C_{GT}$ [S1].
    *   *Note:* The sources describe the method as "supersampling" where the network is trained to predict values at a higher resolution grid, effectively learning the continuous function that, when integrated over the pixel footprint, matches the LR observation. This prevents the network from overfitting to high-frequency noise present in LR inputs [S1].
4.  **Optimization:** Update the MLP weights using the computed loss. Decay the learning rate exponentially from $5 \cdot 10^{-4}$ to $5 \cdot 10^{-6}$ over the course of training [S1].
5.  **Inference (Standard):** To render an HR image, sample rays through the HR pixel grid and query the trained NeRF. The output is denoted as "Ours-SS" [S1].

**Refinement Network (Ours-Refine):**
For real-world datasets (LLFF) where supersampling alone yields blurry textures, a refinement network is applied post-rendering [S1].

1.  **Patch Extraction:** Divide the input HR image into non-overlapping patches of size $64 \times 64$.
2.  **Reference Patch Selection:** Select $K=8$ reference patches from the training set (typically the first image in the dataset) [S1].
3.  **Patch Refinement:** For each patch $P_{i,j}$ in the rendered image:
    *   Compute the 3D world coordinate of the patch center using the estimated depth $d_{i,j}$ and camera parameters $K_n, T_n$ of the current view $I_n$.
    *   Backproject this 3D coordinate to the reference view $I_{REF}$ using $K_{REF}$ and $T_{REF}$ to find the corresponding location.
    *   Extract the corresponding patch from $I_{REF}$ as $P^{REF}$.
    *   The refinement network takes the rendered patch $P_{i,j}$ and the set of reference patches $\{P^{REF}_k\}_{k=1}^K$ as input to predict the refined texture details.
    *   *Formula:* The transformation involves computing the 3D world coordinate from depth and camera matrices, then backprojecting to the reference image plane. Points falling outside $I_{REF}$ are discarded [S1].
4.  **Stitching:** Stitch the refined patches together to form the final HR image [S1].

**Parameters:**
*   **Optimizer:** Adam [S1].
*   **Batch Size:** 2048 rays [S1].
*   **Learning Rate:** Exponential decay from $5 \cdot 10^{-4}$ to $5 \cdot 10^{-6}$ [S1].
*   **Samples:** 64 coarse, 64 fine [S1].
*   **Epochs:** 20 for Blender dataset, 30 for LLFF dataset [S1].
*   **Refinement Patches:** $K=8$, size $64 \times 64$ [S1].

**Edge Cases and Pitfalls:**
*   **Depth Estimation:** The refinement step relies on accurate depth estimates. Poor depth estimation leads to incorrect backprojection and artifacts [S1].
*   **Arbitrary Scale:** NeRF-SR does not support arbitrary-scale super-resolution; it is trained for specific scale factors (e.g., $\times 2, \times 4$) [S1].
*   **Real-world Blur:** Supersampling alone is insufficient for real-world images with complex structures; the refinement network is necessary for LLFF but adds inference time [S1].

## Subtopics

### [Super-NeRF](16427-super-nerf.md)
Super-NeRF is a method for NeRF super-resolution that generates plausible, 3D-consistent high-resolution (HR) details using only low-resolution (LR) observations, without requiring HR ground truth. It distinguishes itself from NeRF-SR by coupling the NeRF with a GAN-based 2D super-resolution backbone and per-view latent codes to explicitly search the latent space of a powerful SR prior, rather than relying on simple linear formulations or HR refinement via per-patch warping. The method employs a Consistency Controlling Super Resolution (CCSR) module and mutual learning between a fixed LR NeRF and a learnable HR NeRF to ensure view consistency.

## Operating characteristics
**Time and Memory Complexity:**
*   **Training Time:** Comparable to training a standard HR NeRF. The supersampling strategy introduces negligible additional computational overhead during training compared to standard NeRF [S1].
*   **Inference Time:** The base NeRF rendering time dominates. For a $1008 \times 756$ image, NeRF's MLP takes ~48 seconds on a single 1080Ti. The refinement stage adds ~1.3 seconds [S1].

**Measured Performance:**
*   **Hardware:** NVIDIA 1080Ti [S1].
*   **Datasets:** Blender (synthetic), LLFF (real-world) [S1].
*   **Metrics:** PSNR, SSIM, LPIPS [S1].
*   **LLFF Results (Scale $\times 4$):**
    *   Ours-SS: PSNR 25.13, SSIM 0.730, LPIPS 0.244 [S1].
    *   Ours-Refine: PSNR 25.59, SSIM 0.759, LPIPS 0.165 [S1].
*   **LLFF Results (Scale $\times 2$):**
    *   Ours-SS: PSNR 26.81, SSIM 0.823, LPIPS 0.145 [S1].
    *   Ours-Refine: PSNR 27.34, SSIM 0.842, LPIPS 0.103 [S1].

**Strengths and Limitations:**
*   **Strengths:** Achieves photorealistic HR novel view synthesis from LR inputs without external HR data. Maintains 3D consistency. Improves LPIPS significantly, indicating better perceptual quality [S1].
*   **Limitations:** Does not support arbitrary-scale super-resolution. Introduces extra computation during inference if refinement is used. Requires accurate depth estimation for the refinement step [S1].

## Background
NeRF-SR was introduced by Wang et al. in 2021 as the first pipeline for high-resolution novel view synthesis using mostly low-resolution inputs [S1]. It addresses the limitation of standard NeRF, which often produces blurry results when trained on LR images due to the inability to resolve high-frequency details. The method builds on the differentiable volume rendering framework of NeRF [S1].

## Sources
[S1] NeRF-SR: High-Quality Neural Radiance Fields using Supersampling, https://arxiv.org/abs/2112.01759v3, fetched 2026-10-26
