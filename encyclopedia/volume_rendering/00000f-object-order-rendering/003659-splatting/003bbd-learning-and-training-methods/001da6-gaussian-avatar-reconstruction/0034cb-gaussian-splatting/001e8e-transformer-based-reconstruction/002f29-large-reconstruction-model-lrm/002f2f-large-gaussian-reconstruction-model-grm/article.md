# Large Gaussian Reconstruction Model (GRM)
Also called: GRM (Large Gaussian Reconstruction Model)

## Overview
The Large Gaussian Reconstruction Model (GRM) is a feedforward neural network that performs single-image (or multi-view) to 3D reconstruction by directly predicting a set of 3D Gaussians [S1, unofficial]. It addresses the limitations of iterative optimization methods (like NeRF or standard 3DGS) by providing immediate, differentiable 3D representations from input images [S1, unofficial]. It builds on [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md) by replacing the output format (e.g., NeRF fields or meshes) with Gaussian maps, supporting both single and multi-view inputs [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: One or more source images (single-view or multi-view).
- Output: A set of 3D Gaussian parameters (position, covariance, opacity, spherical harmonics coefficients) representing the scene [S1, unofficial].

**Data Structures:**
The network outputs a "Gaussian map," which is a structured representation of Gaussian primitives. The exact field layout (e.g., number of Gaussians, dimensionality of SH coefficients) is not specified in the provided sources [S1, unofficial].

**Steps:**
1. Encode the input image(s) using a vision backbone (architecture details inherited from or similar to LRM, but specific encoder details are not provided in the notes) [S1, unofficial].
2. Decode the encoded features into a Gaussian map [S1, unofficial].
3. Render the predicted Gaussians to verify or compute loss (standard 3DGS rasterization) [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
- Multi-view support: Enabled [S1, unofficial].
- Specific hyperparameters (e.g., number of Gaussians, SH degree for the base GRM) are not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- Input image quality: As with other feedforward models, performance depends on the quality and consistency of input images [S1, unofficial].
- Multi-view consistency: When using multi-view inputs, ensuring geometric consistency across views is critical [S1, unofficial].

## Subtopics

### GS-LRM Baseline Modifications
A modified baseline of GRM/GS-LRM that enhances speed and training objectives. It replaces the CNN deconvolution layer with an MLP unpatchify layer (as proposed in [21]) and is implemented in JAX [S2]. It maintains the same network parameters and training loss weights as the original paper [S2]. An LPIPS loss is added to the training objective, which improves LPIPS and SSIM scores but causes a slight decrease in PSNR [S2]. Spherical harmonics degree is set to 0 for RealEstate10K and 1 for DL3DV [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** The GS-LRM baseline with LPIPS loss shows improved LPIPS and SSIM scores, with a slight decrease in PSNR [S2].
- **Strengths:** Feedforward inference (fast), supports multi-view inputs [S1, unofficial].
- **Limitations:** Specific quantitative metrics for the base GRM are not provided in the sources [S1, unofficial].

## Background
GRM was introduced as an extension of the Large Reconstruction Model (LRM) [S1, unofficial]. While LRM uses a feedforward model for single image to 3D reconstruction, GRM specifically uses Gaussian maps as the network output result [S1, unofficial]. It is designed for efficient 3D reconstruction and generation [S1, unofficial]. The concept is discussed in the context of comparing feedforward prediction methods (LRM, GRM, iNVS) against iterative methods like NeRF [S1, unofficial].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
