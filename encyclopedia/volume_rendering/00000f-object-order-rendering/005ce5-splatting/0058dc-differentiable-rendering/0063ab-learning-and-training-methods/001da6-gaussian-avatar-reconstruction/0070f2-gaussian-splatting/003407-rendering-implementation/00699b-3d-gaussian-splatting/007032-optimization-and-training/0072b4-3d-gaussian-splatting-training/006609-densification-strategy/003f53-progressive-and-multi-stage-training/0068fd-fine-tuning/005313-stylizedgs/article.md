# StylizedGS

Also called: StylizedGS

## Overview
StylizedGS is a controllable 3D stylization method for 3D Gaussian Splatting (3DGS) models that performs color matching with a reference style image while utilizing a 3DGS filter to remove artifacts [S1]. It distinguishes itself from other stylization approaches by explicitly optimizing for nearest neighbor feature matching and depth preservation, ensuring the stylized output retains the geometric structure of the original scene while adopting the artistic appearance of the style image [S1].

## How it works
**Inputs:** A pre-trained 3DGS model (representing the target scene) and a reference style image.
**Outputs:** A stylized 3DGS model with modified colors and features.

**Data Structures:**
The method operates on the standard 3DGS representation: a set of $N$ Gaussians, each defined by position $\mathbf{\mu}$, covariance $\mathbf{\Sigma}$ (or scale/rotation), opacity $\alpha$, and spherical harmonics coefficients for color/view-dependent appearance. StylizedGS modifies the color/feature attributes during optimization.

**Optimization Steps:**
1.  **Feature Extraction:** Extract features from the reference style image using a feature extractor (typically a CNN like VGG, though specific layers are not detailed in the summary [S1]).
2.  **Rendering:** Render the current 3DGS model to obtain color and depth maps for each training view.
3.  **Loss Computation:** Compute the total loss $\mathcal{L}$ as a weighted sum of the following components:
    *   **Nearest Neighbor Feature Match Loss ($\mathcal{L}_{feat}$):** Measures the discrepancy between the rendered features of the stylized 3DGS and the features of the style image. The method employs a nearest neighbor strategy to match features, ensuring semantic consistency with the style [S1].
    *   **Depth Preservation Loss ($\mathcal{L}_{depth}$):** Ensures that the geometric structure (depth) of the stylized result remains consistent with the original scene, preventing geometric distortion during stylization [S1].
    *   **3DGS Filter Loss:** A regularization term that utilizes the 3DGS filter to remove artifacts introduced by the stylization process [S1].
4.  **Optimization:** Update the 3DGS parameters (primarily color/feature attributes) to minimize $\mathcal{L}$.

**Formulas:**
The total loss is defined as:
$$ \mathcal{L} = \lambda_{feat} \mathcal{L}_{feat} + \lambda_{depth} \mathcal{L}_{depth} + \lambda_{filter} \mathcal{L}_{filter} $$
Where $\lambda_{feat}$, $\lambda_{depth}$, and $\lambda_{filter}$ are weighting hyperparameters. The exact mathematical formulation of $\mathcal{L}_{feat}$ using nearest neighbor matching is not provided in the sources [S1].

**Parameters:**
Typical values for $\lambda$ weights are not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Artifacts:** Without the 3DGS filter, stylization may introduce visual artifacts; the method specifically addresses this [S1].
*   **Geometric Drift:** If depth preservation is not enforced, the stylized scene may suffer from geometric inaccuracies [S1].

## Subtopics

### StyleGaussian
StyleGaussian is a method for real-time 3D style transfer with strict multi-view consistency [S1]. It differs from StylizedGS by embedding VGG features directly into 3D Gaussians and using an efficient rendering strategy: it initially renders lower-dimensional features and then maps them to high-dimensional ones [S1]. It aligns stylized features with a style image using an AdaIN-like mechanism and decodes features back to RGB using a KNN-based 3D CNN [S1].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.
**Measured Performance:**
Not given in the sources.
**Strengths and Limitations:**
StylizedGS provides controllable 3D stylization with artifact removal [S1]. Specific performance metrics or limitations compared to siblings are not detailed in the provided text [S1].

## Background
StylizedGS is part of the broader effort to apply artistic stylization to 3D Gaussian Splatting models, building on the efficiency of 3DGS while addressing challenges in visual fidelity and computational efficiency common in earlier NeRF-based methods [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
