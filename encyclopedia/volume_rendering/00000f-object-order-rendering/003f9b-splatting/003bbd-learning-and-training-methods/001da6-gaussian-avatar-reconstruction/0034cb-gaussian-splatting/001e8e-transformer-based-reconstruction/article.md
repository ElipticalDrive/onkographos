# Transformer-based Reconstruction

Also called: Feed-forward 3D Gaussian Splatting, Single-view/Sparse-view 3DGS Reconstruction

## Overview
Transformer-based Reconstruction refers to a family of feed-forward neural network methods that directly predict 3D Gaussian Splatting (3DGS) parameters from 2D image inputs, bypassing the iterative optimization loops typical of traditional 3DGS [S1, unofficial]. It solves the latency bottleneck of scene reconstruction by replacing gradient-based optimization with a single inference pass [S1, unofficial]. It differs from its siblings by utilizing transformer architectures to map image features directly to Gaussian attributes (mean, covariance, color, opacity) rather than relying on volume rendering quadrature or iterative density editing [S1, unofficial].

## Shared principles
All methods in this family share the following mechanism:
1.  **Input Encoding:** 2D images are encoded into feature tokens using a vision backbone (e.g., ViT) [S1, unofficial]. Camera parameters (intrinsics/extrinsics) are encoded into token embeddings if multi-view inputs are used [S1, unofficial].
2.  **Transformer Processing:** A transformer encoder/decoder processes the concatenated sequence of image and camera tokens to model spatial and geometric relationships [S1, unofficial].
3.  **Direct Parameter Prediction:** The network outputs a set of 3D Gaussian parameters ($\mu, \Sigma, c, \alpha$) directly, without intermediate volume rendering or iterative refinement [S1, unofficial].
4.  **Training Objective:** Models are supervised using photometric losses (e.g., LPIPS, SSIM) between rendered outputs and ground truth images during training [S1, unofficial].

## Subtopics

### [Flash3D](1481-flash3d.md)
A feed-forward network for monocular 3DGS reconstruction. It estimates depth from a single image and predicts Gaussian parameters. Choose this for single-view inputs where depth estimation is feasible.

### [GS-LRM](1483-gs-lrm.md)
A feed-forward transformer for sparse-view (2-4 images) 3DGS reconstruction. It tokenizes image and camera data to predict Gaussians. Choose this for sparse multi-view scenarios requiring explicit pose conditioning.

### [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md)
A feed-forward model for single-image to 3D reconstruction. It predicts scene parameters in one step. Choose this for general single-view 3D generation tasks, potentially outputting NeRF fields or meshes depending on the variant.

## Background
The method originates from the need to accelerate 3D Gaussian Splatting reconstruction, which traditionally requires per-scene optimization [S1, unofficial]. Feed-forward approaches were developed to enable real-time reconstruction from single or sparse views by leveraging transformer architectures to generalize from large datasets of image-3D pairs [S1, unofficial].

## Sources
[S1] Feed-forward 3D Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2024-05-20
