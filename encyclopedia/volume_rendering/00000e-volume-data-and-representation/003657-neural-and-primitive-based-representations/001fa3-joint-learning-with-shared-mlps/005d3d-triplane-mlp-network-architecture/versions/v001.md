# Triplane-MLP Network Architecture

Also called: HUGS Triplane-MLP

## Overview
The Triplane-MLP Network Architecture is a method for representing 3D scenes (specifically human avatars) by decoding compact triplane features into Gaussian parameters using lightweight Multi-Layer Perceptrons (MLPs) [S1]. It solves the problem of efficient training and fast rendering by replacing explicit per-Gaussian storage with a shared neural decoder, allowing the triplane and MLP models to be discarded after predicting canonical avatar and deformation parameters to achieve 60 FPS rendering [S1]. It sets itself apart from standard Gaussian Splatting by using a neural representation for geometry and appearance rather than static attributes, and from implicit neural representations by utilizing triplane features for spatial locality [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Triplane features (initialized from a normal distribution with standard deviation 0.1) [S2].
*   **Outputs:** Appearance parameters (opacity, spherical harmonics), Geometry parameters (delta mean $\Delta\mu$, rotation $R$, scale $S$), and Deformation parameters (LBS weights $W$) [S1].

**Data Structures:**
The architecture consists of three distinct MLP decoders operating on shared triplane features. The specific layouts are:

1.  **Appearance Decoder ($D_A$):**
    *   Structure: 2-layer MLP.
    *   Activation: GELU [S1].
    *   Input: Triplane features.
    *   Output: Opacity and Spherical Harmonics (SH) parameters.
2.  **Geometry Decoder ($D_G$):**
    *   Structure: 2-layer MLP.
    *   Activation: GELU [S1].
    *   Input: Triplane features.
    *   Output: $\Delta\mu$, $R$, $S$.
3.  **Deformation Decoder ($D_D$):**
    *   Structure: 3-layer MLP.
    *   Activation: GELU [S1].
    *   Input: Triplane features.
    *   Output: LBS weights $W$.

**Numbered Steps:**
1.  **Initialization:** Initialize triplane features from a normal distribution with standard deviation 0.1 [S2].
2.  **Feature Sampling:** Sample triplane features corresponding to the 3D location of each Gaussian [S1].
3.  **Appearance Decoding:** Pass triplane features through $D_A$ to predict opacity and SH parameters.
    *   Apply Sigmoid activation to opacity to constrain it to $[0, 1]$ [S1].
    *   SH parameters are not activated but RGB values derived from them are clipped to $[0, 1]$ during rasterization [S1].
4.  **Geometry Decoding:** Pass triplane features through $D_G$ to predict $\Delta\mu$, $R$, and $S$.
    *   GELU activation ensures $S \ge 0$ [S1].
    *   Rotation $R$ is predicted using a 6D representation, which does not require a normalization activation [S1].
5.  **Deformation Decoding:** Pass triplane features through $D_D$ to predict LBS weights $W$.
    *   Apply a low-temperature softmax function with temperature $T=0.1$ to ensure $\sum_k W_{k,i} = 1$ and to predict unimodal distributions [S1].
6.  **Rendering:** Discard the triplane and MLP models once canonical avatar and deformation parameters are predicted to enable fast rendering [S1].

**Formulas:**
*   **Opacity Constraint:**
    $$ \alpha = \text{Sigmoid}(D_A(\text{features})) $$
*   **Scale Constraint (Implicit via GELU):**
    $$ S = D_G(\text{features}) \implies S \ge 0 $$
*   **LBS Weights (Low-Temperature Softmax):**
    $$ W_{k,i} = \frac{\exp(z_{k,i}/T)}{\sum_{k} \exp(z_{k,i}/T)} $$
    Where $T = 0.1$ [S1].

**Parameters:**
*   **Activation Function:** GELU (Gaussian Error Linear Units) is used in all MLPs ($D_A, D_G, D_D$) to lead to quicker convergence [S1].
*   **Deformation Softmax Temperature:** $T = 0.1$ [S1].
*   **Rotation Representation:** 6D representation [S1].

**Edge Cases and Pitfalls:**
*   **Rotation Normalization:** Unlike Kerbl et al. [15], this method does not require a normalization activation for rotation $R$ because it employs a 6D rotation representation [S1].
*   **RGB Clipping:** While SH parameters are not activated, derived RGB values must be clipped to $[0, 1]$ during rasterization [S1].
*   **Unimodal Distribution:** The low-temperature softmax ($T=0.1$) is critical to ensure most Gaussians are assigned to a single bone transformation, avoiding ambiguous deformations [S1].

## Subtopics

### Implicit Neural Representation via Shared MLP
A lightweight Multi-Layer Perceptron (MLP) that decodes triplane features into occupancy values for 3D points, using Fourier feature mapping [S2]. It is chosen when explicit Gaussian parameterization is not required and volumetric occupancy is the target. It differs from the HUGS Triplane-MLP by predicting scalar occupancy rather than Gaussian attributes.
*   **Input Processing:** 3D coordinates are processed by a Fourier feature mapping layer with a scale factor of 1 [S2].
*   **Network Structure:** The output of the Fourier mapping is passed through 3 fully connected layers [S2].
*   **Parameters:**
    *   Fully connected layer dimension: 128 [S2].
    *   Activation function: ReLU [S2].
    *   Fourier feature mapping scale factor: 1 [S2].

## Operating characteristics
*   **Time Complexity:** Training is efficient due to lightweight MLPs. Joint training on a subset of data (500 shapes, 500k occupancy points per object) takes approximately 1 day on a single RTX 2080 Ti [S2].
*   **Memory Complexity:** Reduces redundancy by sharing MLPs across scenes/Gaussians. Triplane features are initialized with $\sigma=0.1$ [S2].
*   **Performance:** Enables fast rendering at 60 FPS by discarding triplane/MLP models after predicting canonical avatar and deformation parameters [S1].
*   **Strengths:** Quick convergence (GELU), fast rendering, efficient training [S1].
*   **Limitations:** Requires two-stage training (joint then independent) for implicit variants [S2].

## Background
The Triplane-MLP architecture was introduced in the context of Human Gaussian Splats (HUGS) by Bouaziz et al. [S1]. It builds on the concept of triplane representations for neural radiance fields [S1]. The HUGS method was presented at CVPR 2024 [S1]. The variant using shared MLPs for implicit neural representation was presented at CVPR 2023 [S2].

## Sources
[S1] HUGS: Human Gaussian Splats, https://doi.org/10.1109/cvpr52733.2024.00055, fetched 2026-10-26
[S2] 3D Neural Field Generation Using Triplane Diffusion, https://doi.org/10.1109/cvpr52729.2023.02000, fetched 2026-10-26
