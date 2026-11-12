# VS-Splat Voxel Selection

Also called: Ours, VS-Splat selection

## Overview
VS-Splat Voxel Selection is a learnable method for selecting active voxels in feed-forward Gaussian splatting by predicting confidence scores via Gumbel-Sigmoid and using them to refine opacity values [S1, unofficial]. It solves the problem of unreliable object-centric voxel selection inherent in static opacity-based thresholding (OBT) and the excessive sparsity of straight-through estimator (STE) approaches [S1, unofficial]. It sets itself apart from OBT by being fully learnable and from STE by using a differentiable stochastic relaxation that prevents the collapse to an extremely limited number of voxels [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D voxel grid where each voxel contains a set of fine-grained 3D Gaussians.
- Output: A binary mask of selected voxels and refined opacity values for the Gaussians within those voxels.

**Data Structures:**
- **Voxel Grid:** A 3D array of resolution $H \times W \times D$. In the reference implementation, $H=W=D=48$ [S1, unofficial].
- **Confidence Score:** A scalar value $c_v \in [0, 1]$ predicted for each voxel $v$ by a Selective Feature Refinement (SFR) module [S1, unofficial].
- **Gumbel-Sigmoid Variable:** A stochastic variable $g_v$ used to approximate the binary selection decision [S1, unofficial].

**Parameters:**
- **Temperature $\tau$:** Controls the sharpness of the Gumbel-Sigmoid distribution. Linearly annealed from 1.0 to 0.1 over the first 50,000 training iterations, then fixed at 0.1 for the remainder of training and inference [S1, unofficial].

**Steps:**
1. **Feature Extraction:** Extract 2D feature maps from input images using a pre-trained encoder (e.g., DINOv1) [S1, unofficial].
2. **3D Compression:** Compress features into the 3D voxel grid using a 3D convolutional compressor, reducing channel dimension from $2C$ to $\tilde{C}$ (e.g., 1536 to 128) [S1, unofficial].
3. **Confidence Prediction:** Pass the compressed features through the SFR module (containing $R$ residual convolutional blocks, e.g., $R=2$) to predict the confidence score $c_v$ for each voxel [S1, unofficial].
4. **Stochastic Selection:** Sample a Gumbel noise term and apply the Gumbel-Sigmoid function to the confidence score $c_v$ and temperature $\tau$ to generate the selection variable $g_v$:
   $$ g_v = \sigma\left(\frac{\log(c_v) - \log(1 - c_v) + \epsilon_1 - \epsilon_2}{\tau}\right) $$
   where $\epsilon_1, \epsilon_2$ are independent Gumbel(0,1) noise samples and $\sigma$ is the logistic sigmoid function [S1, unofficial].
5. **Opacity Refinement:** Use $g_v$ to modulate the opacity of the fine-grained Gaussians within voxel $v$. Voxels with $g_v \approx 1$ are selected, and their Gaussians retain or are refined with opacity; voxels with $g_v \approx 0$ are effectively pruned [S1, unofficial].
6. **Rendering:** Render the scene using only the selected Gaussians [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Deterministic Sigmoid Failure:** Replacing Gumbel-Sigmoid with a deterministic Sigmoid function fails to select object-centric voxels properly, leading to significant rendering quality drops (e.g., PSNR 18.01 vs 21.31 on CO3D) and excessive sparsity (2,223 voxels vs 19,587) [S1, unofficial].
- **STE Collapse:** Using a Straight-Through Estimator (STE) on confidence scores results in selecting an extremely limited number of voxels (121 on CO3D), severely degrading quality (PSNR 14.71) [S1, unofficial].
- **OBT Unreliability:** Static opacity thresholding (e.g., threshold 0.005) is unreliable for object-centric selection and generates excessive fine-grained primitives in background regions, degrading quality [S1, unofficial].

## Subtopics

### Deterministic Sigmoid selection
A variant that replaces the Gumbel-Sigmoid with the deterministic Sigmoid function for voxel selection [S1, unofficial]. It fails to properly select object-centric voxels, leading to significant rendering quality drops and excessive sparsity compared to the stochastic Gumbel-Sigmoid approach [S1, unofficial]. On the CO3D dataset, it yields PSNR 18.01, SSIM 0.835, LPIPS 0.288, and uses 2,223 voxels [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the SFR module inference and Gumbel-Sigmoid sampling per voxel. The reference implementation uses a $48^3$ voxel grid [S1, unofficial].
- **Memory Complexity:** Proportional to the number of selected voxels times the number of fine-grained primitives per voxel ($M=16$ in reference) [S1, unofficial].
- **Measured Performance:** On the CO3D dataset, VS-Splat yields PSNR 21.31, SSIM 0.863, LPIPS 0.213, and uses 19,587 voxels [S1, unofficial].
- **Comparison:** VS-Splat uses approximately 15% fewer voxels than OBT (19,587 vs 22,867) while outperforming it across all metrics (PSNR 21.31 vs 21.21, SSIM 0.863 vs 0.859, LPIPS 0.213 vs 0.223) [S1, unofficial].
- **Strengths:** Learnable, object-centric selection; better than OBT and STE [S1, unofficial].
- **Limitations:** Requires training of the SFR module; sensitive to temperature annealing schedule [S1, unofficial].

## Background
VS-Splat was introduced in the paper "VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views" [S1, unofficial]. It addresses limitations in opacity-based thresholding and STE-based selection methods for voxel-wise Gaussian splatting [S1, unofficial].

## Sources
[S1] VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views, https://doi.org/10.48550/arxiv.2609.12343, fetched 2026-10-26 (unofficial)
