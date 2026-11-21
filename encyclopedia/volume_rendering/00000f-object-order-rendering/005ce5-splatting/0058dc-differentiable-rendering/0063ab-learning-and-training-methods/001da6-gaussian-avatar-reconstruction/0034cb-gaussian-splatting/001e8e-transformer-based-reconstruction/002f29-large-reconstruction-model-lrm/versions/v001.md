# Large Reconstruction Model (LRM)
Also called: LRM (Large Reconstruction Model)
## Overview
LRM is a feedforward model for single-image to 3D reconstruction [S1, unofficial]. It sets itself apart from iterative NeRF-based methods by directly predicting 3D structure in a single inference step, avoiding the need for per-scene optimization [S1, unofficial].
## How it works
**Inputs and Outputs**: The input is a single 2D image. The output is a set of 3D Gaussian parameters (mean, covariance, color, opacity) [G].
**Data Structures**: The network outputs a Gaussian representation, typically structured as a set of points with associated attributes [S1, unofficial].
**Steps**:
1. Encode the input image using a vision backbone (e.g., ViT) [G].
2. Decode the features into 3D Gaussian parameters using a transformer decoder [G].
3. Render the predicted Gaussians using differentiable rasterization to compute photometric loss [G].
**Formulas**: Not given in the sources.
**Parameters**: Not given in the sources.
**Edge Cases**: Not given in the sources.
## Subtopics

### Large Gaussian Reconstruction Model (GRM)
GRM is a variant that uses Gaussians maps as the network output result [S1, unofficial]. It supports multi-view input, unlike the single-view LRM [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity**: Not given in the sources.
**Measured Performance**: Not given in the sources.
**Strengths and Limitations**: Feedforward nature allows for fast reconstruction [S1, unofficial].
## Background
LRM was introduced in "LRM: Large Reconstruction Model for Single Image to 3D" at ICLR 2024 [S1, unofficial]. It represents a shift from iterative optimization to direct feedforward prediction in 3D reconstruction [S1, unofficial].
## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
