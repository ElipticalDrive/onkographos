# NeuS2 Network Architecture

Also called: NeuS2 components

## Overview
NeuS2 is a neural implicit surface reconstruction method that combines a multi-resolution hash encoding with separate networks for signed distance fields (SDF) and color [S1]. It solves the problem of slow convergence and poor detail recovery in previous neural implicit methods by leveraging the efficiency of hash grids for spatial lookup [S1]. It differs from its sibling [Neural Volumes with Multi-Resolution Hash Encoding](1497-neural-volumes-with-multi-resolution-hash-encoding.md) by specifically decoupling the SDF and RGB prediction into distinct MLPs and employing a specific logistic density transformation for unbiased volume rendering [S1].

## How it works
**Inputs and Outputs:**
*   Input: A 3D spatial coordinate $\mathbf{x} \in \mathbb{R}^3$.
*   Outputs: A signed distance value $s \in \mathbb{R}$ and a color vector $\mathbf{c} \in \mathbb{R}^3$.

**Data Structures:**
1.  **Multi-resolution Hash Grid:** Contains 14 levels of resolution. The resolutions range from 16 to 2048 [S1]. Each level maps spatial coordinates to feature vectors via a hash function.
2.  **SDF Network:** A 1-layer Multi-Layer Perceptron (MLP) with 64 hidden units [S1].
3.  **RGB Network:** A 2-layer MLP with 64 hidden units [S1].

**Algorithm Steps:**
1.  **Feature Extraction:** The input coordinate $\mathbf{x}$ is hashed into the multi-resolution hash grid. Features from all 14 levels are aggregated (typically concatenated or summed, though the specific aggregation operation is not detailed in the provided text) to form a combined feature vector $\mathbf{f}$ [S1].
2.  **SDF Prediction:** The feature vector $\mathbf{f}$ is passed through the SDF network (1-layer MLP, 64 hidden units) to predict the signed distance $s$ [S1].
3.  **Density Transformation:** The signed distance $s$ is transformed into a volume density $\sigma$ using a logistic distribution. A learnable parameter $s_{scale}$ (denoted as $s$ in the formula context) controls the sharpness:
    $$ \sigma = \frac{e^{-s_{scale} \cdot s}}{(1 + e^{-s_{scale} \cdot s})^2} $$
    This function $\phi_s(x)$ is the derivative of the Sigmoid function $\Phi_s(x) = \frac{1}{1 + e^{-s_{scale} \cdot x}}$ [S1].
4.  **Color Prediction:** The feature vector $\mathbf{f}$ is passed through the RGB network (2-layer MLP, 64 hidden units) to predict the color $\mathbf{c}$ [S1].
5.  **Volume Rendering:** Unbiased volume rendering is applied. For each pixel, $n$ points $\{p(t_i) = o + t_i v \mid i = 0, 1, \dots, n-1\}$ are sampled along the camera ray, where $o$ is the camera center and $v$ is the ray direction [S1]. The final color is computed by integrating the density and color along the ray using the unbiased weight function derived from the density [S1].

**Parameters:**
*   Hash Grid Levels: 14 [S1].
*   Hash Grid Resolutions: Range from 16 to 2048 [S1].
*   SDF MLP Layers: 1 [S1].
*   SDF MLP Hidden Units: 64 [S1].
*   RGB MLP Layers: 2 [S1].
*   RGB MLP Hidden Units: 64 [S1].

**Edge Cases and Pitfalls:**
*   The parameter $s_{scale}$ is learnable; improper initialization or training dynamics may lead to unstable density values [S1].
*   The source notes that training time on real scenes is significantly longer (3 hours) than on synthetic scenes (1 hour) [S1].

## Subtopics

### STSR-INR
Extends the CoordNet architecture (related to hash grid neural fields) with variable-specific latent embeddings via a variational auto-decoder and a modulated dual-network architecture to handle multivariate and ensemble data [S2, unofficial]. It supports non-integer upscaling factors, such as 1.75× spatial and 2.5× temporal, unifying temporal super-resolution, spatial super-resolution, view synthesis, and ambient occlusion prediction within a single compact model (~5.7 MB) [S2, unofficial].

## Operating characteristics
*   **Performance:** The method achieves fast learning compared to previous neural implicit methods [S1].
*   **Training Time:** On real scenes, training takes approximately 3 hours, whereas on synthetic scenes it takes 1 hour [S1].
*   **Model Size:** The related STSR-INR variant (which shares architectural principles of compact neural fields) is ~5.7 MB, an order of magnitude smaller than CNN-based alternatives [S2, unofficial].
*   **Strengths:** Efficient spatial lookup via hash grids; decoupled SDF/RGB networks allow specialized optimization; unbiased rendering reduces artifacts [S1].
*   **Limitations:** Training time on real-world data is non-trivial [S1].

## Background
NeuS2 was introduced to improve upon the original NeuS method, which suffered from slow convergence and limited detail recovery [S1]. It utilizes multi-resolution hash encoding to accelerate training and improve reconstruction quality [S1]. The method is part of the broader family of hash grid neural fields [S1].

## Sources
[S1] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
