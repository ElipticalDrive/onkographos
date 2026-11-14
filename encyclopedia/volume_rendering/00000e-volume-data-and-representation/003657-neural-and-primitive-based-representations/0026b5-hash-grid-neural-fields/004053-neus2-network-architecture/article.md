# NeuS2 Network Architecture
Also called: NeuS2 components
## Overview
NeuS2 is a neural implicit surface reconstruction method that uses a multi-resolution hash grid to accelerate the evaluation of signed distance fields (SDF) and color, enabling fast learning from multi-view images [S1]. It sets itself apart from its sibling [Neural Volumes with Multi-Resolution Hash Encoding](1497-neural-volumes-with-multi-resolution-hash-encoding.md) by specifically optimizing for surface reconstruction via SDF regularization and unbiased volume rendering, rather than general volumetric density estimation. It builds on [Hash Grid Neural Fields](3855-hash-grid-neural-fields.md) by utilizing the hash encoding mechanism for spatial feature storage.
## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial coordinates $\mathbf{x} \in \mathbb{R}^3$.
*   **Output:** A signed distance value $f(\mathbf{x})$ and an RGB color vector $\mathbf{c}$.

**Data Structures:**
1.  **Multi-resolution Hash Grid:** Contains 14 levels of resolution. The resolutions range from 16 to 2048 [S1]. Features are stored in hash tables at each level.
2.  **SDF Network:** A Multi-Layer Perceptron (MLP) with 1 hidden layer and 64 hidden units [S1].
3.  **RGB Network:** A Multi-Layer Perceptron (MLP) with 2 hidden layers and 64 hidden units [S1].

**Steps:**
1.  **Hash Encoding:** For a query point $\mathbf{x}$, features are retrieved from the multi-resolution hash grid. The features from all 14 levels are concatenated [G].
2.  **SDF Prediction:** The concatenated features are passed through the SDF network (1-layer MLP, 64 units) to predict the signed distance $f(\mathbf{x})$ [S1].
3.  **Color Prediction:** The concatenated features are passed through the RGB network (2-layer MLP, 64 units) to predict the color $\mathbf{c}$ [S1].
4.  **Volume Rendering:** To render an image, the SDF is transformed into a volume density field using a logistic distribution [S1]. The density $\sigma$ is computed as:
    $$ \sigma = \phi_s(f(\mathbf{x})) $$
    where $\phi_s(x) = \frac{s e^{-sx}}{(1 + e^{-sx})^2}$ is the logistic density distribution, $s$ is a learnable parameter, and $\Phi_s(x) = \frac{1}{1 + e^{-sx}}$ is the Sigmoid function [S1].
5.  **Unbiased Weighting:** Unbiased weights are constructed for the volume rendering equation along camera rays [S1].

**Parameters:**
*   **Hash Grid Levels:** 14 [S1].
*   **Hash Grid Resolutions:** Range from 16 to 2048 [S1].
*   **SDF MLP:** 1 hidden layer, 64 units [S1].
*   **RGB MLP:** 2 hidden layers, 64 units [S1].
*   **Learnable Parameter $s$:** Controls the sharpness of the density transition [S1].

**Edge Cases and Pitfalls:**
*   The parameter $s$ must be learned; improper initialization may lead to poor surface convergence [S1].
*   Training on unbounded backgrounds requires foreground masks to be included in the supervision [S3].

## Subtopics

### STSR-INR
Extends the CoordNet architecture (related to NeuS2's INR approach) with variable-specific latent embeddings via a variational auto-decoder and a modulated dual-network architecture to handle multivariate and ensemble data [S2, unofficial]. It supports non-integer upscaling factors, such as 1.75× spatial and 2.5× temporal, unifying temporal SR, spatial SR, view synthesis, and ambient occlusion prediction within a single compact architecture (~5.7 MB) [S2, unofficial].

## Operating characteristics
*   **Training Time:** Training on real scenes takes approximately 3 hours, while synthetic scenes take about 1 hour [S1].
*   **Hardware:** Implemented on a single Nvidia RTX3090-24G GPU [S3].
*   **Batch Size:** 512 rays per batch [S3].
*   **Optimizer:** Adam with an initial learning rate of 0.0005 and a cosine annealing scheduler with 5000 warm-up steps [S3].
*   **Strengths:** Fast learning compared to previous neural implicit methods; efficient multi-resolution hash grid allows for detailed reconstruction [S1].
*   **Limitations:** Training time on real scenes is longer than on synthetic scenes [S1].

## Background
NeuS2 was introduced to address the slow training speeds of previous neural implicit surface reconstruction methods like NeuS [S1]. It leverages multi-resolution hash encoding to accelerate feature lookup, allowing for faster convergence [S1]. The method was presented at ICCV 2023 [S1]. A related distillation and fine-tuning scheme was later applied in NeuMesh, which uses a teacher model (NeuS) to supervise the optimization of codes and decoders [S3].

## Sources
[S1] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S3] NeuMesh: Learning Disentangled Neural Mesh-Based Implicit Field for Geometry and Texture Editing, https://doi.org/10.1007/978-3-031-19787-1_34, fetched 2026-10-26
