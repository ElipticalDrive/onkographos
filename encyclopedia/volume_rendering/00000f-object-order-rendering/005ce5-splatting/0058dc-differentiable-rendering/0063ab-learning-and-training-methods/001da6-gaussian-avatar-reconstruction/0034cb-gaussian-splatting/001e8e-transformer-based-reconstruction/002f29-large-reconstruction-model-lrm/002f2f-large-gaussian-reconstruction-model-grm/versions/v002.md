# Large Gaussian Reconstruction Model (GRM)
Also called: GRM (Large Gaussian Reconstruction Model)

## Overview
The Large Gaussian Reconstruction Model (GRM) is a feedforward neural network for single-image or multi-view to 3D reconstruction that directly predicts Gaussian maps [S1, unofficial]. It builds on [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md) by replacing its output format with Gaussian primitives, supporting both single and multi-view inputs [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: One or more source images [S1, unofficial].
- Output: A set of 3D Gaussian parameters representing the scene [S1, unofficial].

**Data Structures:**
The network outputs a "Gaussian map," which is a structured representation of Gaussian primitives [S1, unofficial]. The exact field layout is not given in the sources [S1, unofficial].

**Steps:**
1. Process the input image(s) through a feedforward architecture [S1, unofficial].
2. Directly output the Gaussian maps as the network result [S1, unofficial].
3. Render the predicted Gaussians for training or inference [S1, unofficial].

**Formulas:**
Not given in the sources [S1, unofficial][S2].

**Parameters:**
Multi-view input support is enabled [S1, unofficial]. Specific hyperparameters are not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
Not given in the sources [S1, unofficial][S2].

## Subtopics

### GS-LRM Baseline Modifications
A modified baseline of GRM that enhances speed and training objectives [S2]. It replaces the CNN deconvolution layer with an MLP unpatchify layer as proposed in [21], and is implemented in JAX [S2]. It maintains the same network parameters and training loss weights as the original paper [S2]. An LPIPS loss is added to the training objective, which improves LPIPS and SSIM scores but causes a slight decrease in PSNR [S2]. Spherical harmonics degree is set to 0 for RealEstate10K and 1 for DL3DV [S2].

## Operating characteristics
- Time complexity is not given in the sources [S1, unofficial][S2].
- Memory complexity is not given in the sources [S1, unofficial][S2].
- The GS-LRM baseline with LPIPS loss shows improved LPIPS and SSIM scores, with a slight decrease in PSNR [S2].
- Feedforward inference is fast and supports multi-view inputs [S1, unofficial].
- Specific quantitative metrics for the base GRM are not provided in the sources [S1, unofficial].

## Background
GRM was introduced as an extension of the Large Reconstruction Model (LRM) [S1, unofficial]. While LRM uses a feedforward model for single image to 3D reconstruction, GRM specifically uses Gaussian maps as the network output result [S1, unofficial]. It is designed for efficient 3D reconstruction and generation [S1, unofficial]. The concept is discussed in the context of comparing feedforward prediction methods against iterative methods like NeRF, which suffer from lighting inconsistency, inaccurate camera poses, and sparse input views [S1, unofficial].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
