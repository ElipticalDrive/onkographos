# Neural Pre-Integrated Lighting (Neural-PIL)
Also called: Neural-PIL

## Overview
Neural-PIL replaces the costly numerical integration of incoming light in physically-based rendering with a direct neural network query, predicting pre-integrated light $\tilde{L}_i$ from illumination embeddings, view direction, and material roughness [S1, unofficial]. Unlike spherical harmonic or spherical Gaussian light representations which suffer from a lack of fine illumination details or high memory costs, Neural-PIL uses a coordinate-based MLP to represent finer details in illumination efficiently [S1, unofficial]. It builds on [Lighting and Appearance Models](14522-lighting-and-appearance-models.md) by decoupling the output space to predict intermediate physical quantities (pre-integrated radiance) rather than final color.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Illumination embedding $\mathbf{z}_l \in \mathbb{R}^{128}$, incoming light direction (mirrored view direction) $\omega_r \in \mathbb{R}^3$, and roughness scalar $b_r$ [S1, unofficial].
*   **Output:** Pre-integrated light $\tilde{L}_i(\omega_r, b_r)$ [S1, unofficial].

**Data Structures:**
The method employs a Multi-Layer Perceptron (MLP) architecture based on SIREN (Sinusoidal Representation Networks) enhanced with FiLM (Feature-wise Linear Modulation) layers [S1, unofficial].
*   **FiLM-SIREN Layer:** A layer that modulates the output of a linear transformation followed by a sine activation. It takes modulating parameters: a scalar $\lambda_0$ and vectors $\beta, \gamma$ [S1, unofficial].
*   **Mapping Networks:** Two separate networks predict the modulating parameters for the FiLM-SIREN layers.
    1.  **Light Mapping Network:** Takes $\mathbf{z}_l$ as input and generates parameters $(\gamma, \beta)$ for the first few layers to decode illumination information for the given direction [S1, unofficial].
    2.  **Roughness Mapping Network:** Takes $b_r$ as input and generates parameters for the penultimate layer to perform light integration conditioned on roughness [S1, unofficial].

**Mathematical Formulation:**
The rendering equation is approximated by splitting the specular term, allowing the light integral to be pre-integrated into the neural network [S2, unofficial]. The final radiance $L_o$ is computed as:

$$
L_o(\mathbf{x}, \omega_o) \approx \frac{b_d}{\pi}\tilde{L}_i(\mathbf{n}, 1) + b_s \left( F_0(\omega_o, \mathbf{n}) B_0(\omega_o \cdot \mathbf{n}, b_r) + B_1(\omega_o \cdot \mathbf{n}, b_r) \right) \tilde{L}_i(\omega_r, b_r)
$$

Where:
*   $b_d, b_s$ are diffuse and specular albedo components.
*   $\tilde{L}_i(\omega_r, b_r) = \int_{\Omega} D(b_r, \omega_i, \omega_r) L_i(\mathbf{x}, \omega_i) d\omega_i$ is the pre-integrated light, dependent on the microfacet distribution $D$ [S1, unofficial].
*   $F_0(\omega_o, \mathbf{n}) = (1 - \omega_o \cdot \mathbf{h})^5$ is the Fresnel term at normal incidence, with $\mathbf{h} = \frac{\omega_i + \omega_o}{\|\omega_i + \omega_o\|}$ [S1, unofficial].
*   $B_0, B_1$ are 2D lookup textures indexed by $(\omega_o \cdot \mathbf{n})$ and roughness $b_r$ [S1, unofficial].
*   Note: The diffuse part queries $\tilde{L}_i$ with normal $\mathbf{n}$ and fixed diffuse roughness 1, while the specular part queries $\tilde{L}_i$ with $\omega_r$ and material roughness $b_r$ [S1, unofficial].

**The Neural Network Activation:**
The core activation function in the FiLM-SIREN layers is defined as [S1, unofficial]:

$$
\phi(\mathbf{y}) = \sin(\lambda_0 \gamma \odot \mathbf{y} + \beta)
$$

Where $\odot$ denotes the Hadamard (element-wise) product, and $\mathbf{y}$ is the output of the preceding linear layer [S1, unofficial].

**Algorithm Steps:**
1.  **Encode Illumination:** Pass the illumination embedding $\mathbf{z}_l$ through the first mapping network to generate modulating parameters $(\gamma_{light}, \beta_{light})$ [S1, unofficial].
2.  **Encode Roughness:** Pass the roughness scalar $b_r$ through the second mapping network to generate modulating parameters $(\gamma_{rough}, \beta_{rough})$ for the penultimate layer [S1, unofficial].
3.  **Forward Pass:**
    a. Input the direction $\omega_r$ into the first layer of the MLP.
    b. For the first few layers, apply the FiLM modulation using $(\gamma_{light}, \beta_{light})$ and the sine activation $\phi(\mathbf{y})$ [S1, unofficial].
    c. Propagate through intermediate layers.
    d. For the penultimate layer, apply the FiLM modulation using $(\gamma_{rough}, \beta_{rough})$ [S1, unofficial].
    e. Output the final value $\tilde{L}_i(\omega_r, b_r)$ [S1, unofficial].
4.  **Rendering:** Use the predicted $\tilde{L}_i$ in the split-sum approximation formula to compute pixel color [S2, unofficial].

**Parameters:**
*   Illumination embedding dimension: 128 [S1, unofficial].
*   Direction dimension: 3 [S1, unofficial].
*   Roughness: Scalar [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Training Stability:** The method separates the training of the Light-SMAE (Smooth Manifold Autoencoder) and the Neural-PIL to ease memory requirements. The Light-SMAE is trained on a dataset of environment maps to map 2D environment maps to the 128-dimensional latent space $\mathbf{z}_l$, which is then frozen during Neural-PIL training [S1, unofficial].
*   **Differentiability:** The approach is designed for differentiable rendering engines, avoiding costly numerical accumulation or convolution on complete environment maps [S1, unofficial].

## Subtopics

### Split-Sum and PIL Approximation
A baseline approximation method that splits the specular term of the rendering equation into two factors: one involving the light integral and one involving the BRDF terms (D, G, F), allowing the light integral to be pre-integrated into a neural network [S2, unofficial]. The specular term is split as $\int_{\Omega_+} L(\hat{x}, l)(\hat{n} \cdot l) dl \int_{\Omega_+} \frac{DGF}{4(\hat{n} \cdot v)(\hat{n} \cdot l)} (\hat{n} \cdot l) dl$ [S2, unofficial]. The rendering equation becomes $\tilde{L}\hat{\mathbf{c}}_b + (\pi\tilde{L}) \int_{\Omega_+} \frac{DGF}{4(\hat{\mathbf{n}} \cdot \mathbf{v})(\hat{\mathbf{n}} \cdot \mathbf{l})} (\hat{\mathbf{n}} \cdot \mathbf{l}) d\mathbf{l}$, where $\tilde{L}$ is the pre-integrated neural network representing the light integral factor divided by $\pi$ [S2, unofficial]. This allows the forward rendering model to be expressed as $\widehat{Lc_b} + (\pi\widehat{L}) \int_{\Omega_+} \frac{DGF}{4(\hat{n} \cdot \mathbf{v})(\hat{n} \cdot \mathbf{l})} (\hat{n} \cdot \mathbf{l}) d\mathbf{l}$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Replaces numerical integration with a single MLP query, which is significantly faster than Monte Carlo integration or spherical harmonic evaluation for high-frequency details [S1, unofficial].
*   **Memory Complexity:** Lower than spherical harmonic representations which require high memory costs for fine details. The MLP size is determined by the number of FiLM-SIREN layers and mapping networks [S1, unofficial].
*   **Strengths:** Capable of representing finer details in illumination compared to SG or SH representations without prohibitive memory costs [S1, unofficial]. Suitable for differentiable rendering engines [S1, unofficial].
*   **Limitations:** Requires training of auxiliary networks (Light-SMAE, BRDF-SMAE) and careful handling of the computation graph to avoid large memory footprints from naive implementation of voxel grid features [S2, unofficial].
*   **When to use:** When high-fidelity illumination details are required in differentiable inverse rendering or neural radiance field optimization, and spherical harmonics are insufficient or too expensive [S1, unofficial].

## Background
Neural-PIL was introduced in the paper "Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition" [S1, unofficial]. It addresses the limitations of existing differentiable rendering frameworks that rely on spherical Gaussian (SG) or spherical harmonic (SH) light representations, which lack fine details or are computationally expensive [S1, unofficial]. The method leverages a pi-GAN-like architecture with FiLM-SIREN layers [S1, unofficial]. It is used in the context of reflectance decomposition and inverse rendering, where separating geometry, lighting, and material properties is required [S1, unofficial]. The approach is also referenced in the context of Neural Direct and Joint Inverse Rendering (NDJIR) as a baseline for split-sum and PIL approximation [S2, unofficial].

## Sources
[S1] Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition, https://doi.org/10.48550/arxiv.2110.14373, fetched 2026-10-26 (unofficial)
[S2] NDJIR: Neural Direct and Joint Inverse Rendering for Geometry, Lights, and Materials of Real Object, https://doi.org/10.48550/arxiv.2302.00675, fetched 2026-10-26 (unofficial)
