# Transformer-based Reconstruction

Also called: Feed-forward 3DGS, Direct Gaussian Prediction, LRM-based 3DGS

## Overview
Transformer-based Reconstruction methods replace the iterative optimization loop of standard 3D Gaussian Splatting (3DGS) with a feed-forward neural network that directly predicts Gaussian parameters from input images [S1, unofficial]. This approach solves the computational bottleneck of per-scene optimization, enabling real-time reconstruction from single or sparse views. It differs from iterative siblings by eliminating the need for gradient descent on scene parameters, instead relying on learned priors within the transformer architecture to generalize across scenes [S1, unofficial].

## Shared principles
All methods in this family share a common architectural paradigm:
1.  **Tokenization**: Input data (images and/or camera poses) is encoded into a sequence of feature tokens. Images are typically processed by a Vision Transformer (ViT) backbone [S1, unofficial]. Camera parameters (intrinsics and extrinsics) are encoded into separate token embeddings [S1, unofficial].
2.  **Transformer Processing**: The concatenated token sequence is processed by a transformer encoder/decoder to model spatial and geometric relationships between views and the 3D scene structure [S1, unofficial].
3.  **Direct Parameter Prediction**: The network decodes the transformer output directly into 3D Gaussian attributes: mean position $\mu$, covariance $\Sigma$ (or rotation $R$ and scale $S$), color $c$, and opacity $\alpha$ [S1, unofficial].
4.  **Differentiable Rendering**: The predicted Gaussians are rendered using standard differentiable rasterization to produce an output image [S1, unofficial].
5.  **Training Objective**: The network is trained end-to-end using photometric losses (e.g., LPIPS, SSIM) between the rendered output and ground truth images [S1, unofficial].

## Subtopics

### [Flash3D](1481-flash3d.md)
Flash3D is a feed-forward method for monocular 3DGS reconstruction that estimates depth from a single view to generate Gaussians. It differs from GS-LRM and LRM by relying on monocular depth estimation rather than multi-view feature aggregation or tokenized multi-image inputs. It is chosen when only a single image is available and a depth prior is acceptable. See [Flash3D](1481-flash3d.md).

### [GS-LRM](1483-gs-lrm.md)
GS-LRM is a feed-forward transformer method that predicts 3DGS parameters directly from a small set of input images (typically 2-4). It distinguishes itself from LRM by specifically targeting Gaussian parameter prediction rather than mesh or NeRF fields, and from Flash3D by its explicit handling of multi-view camera pose tokens. It is chosen for sparse-view reconstruction where camera poses are known. See [GS-LRM](1483-gs-lrm.md).

### [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md)
LRM is a feed-forward neural network designed for single-image to 3D reconstruction, establishing the foundational transformer-based architecture for direct parameter prediction in this family. It sets itself apart from Flash3D and GS-LRM by its generalizable architecture that can be adapted to various 3D representations, though here it predicts Gaussians. It is chosen as a baseline for single-view direct reconstruction. See [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md).

### [Reconstruction-based Generation](11811-reconstruction-based-generation.md)
Reconstruction-based Generation decomposes 3D generation into two stages: multi-view image synthesis followed by 3D reconstruction. Unlike direct prediction methods, it leverages pre-trained diffusion models to generate posed multi-view images, which then serve as supervision for a Gaussian Splatting reconstructor. This approach reuses mature reconstruction pipelines, making system-level engineering straightforward, but inherits the consistency limitations of the multi-view generator. It is chosen when text-to-3D or single-image-to-3D generation is required without training a dedicated reconstruction network from scratch. See [Reconstruction-based Generation](11811-reconstruction-based-generation.md).

## Background
The family originates from the need to accelerate 3D Gaussian Splatting, which traditionally requires per-scene optimization [S1, unofficial]. Large Reconstruction Model (LRM) introduced the transformer-based feed-forward approach for 3D reconstruction [S1, unofficial]. GS-LRM and Flash3D adapted this paradigm specifically for 3DGS, with GS-LRM focusing on sparse multi-view inputs and Flash3D on monocular depth estimation [S1][S2, unofficial]. Reconstruction-based Generation extends this by integrating diffusion-based multi-view synthesis [S1, S2].

## Sources
[S1] LRM, GS-LRM, Flash3D, Reconstruction-based Generation notes (unofficial summaries provided in context).
