# Neural Pre-Integrated Lighting (Neural-PIL)
Also called: Neural-PIL

## Overview
Neural-PIL replaces the costly numerical integration of incoming light with a simple neural network query, predicting pre-integrated light $\tilde{L}_i(\omega_r, b_r)$ directly from an illumination embedding, view direction, and material roughness [S1, unofficial]. Unlike [Spherical Harmonics NeRF](14480-spherical-harmonics-nerf.md) which suffers from lack of fine details or high memory costs for high-frequency illumination, Neural-PIL uses an MLP to represent finer illumination details efficiently [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Illumination embedding $\mathbf{z}_l \in \mathbb{R}^{128}$, incoming light direction (mirrored view direction) $\omega_r \in \mathbb{R}^3$, and roughness $b_r$ (scalar) [S1, unofficial].
*   **Output:** Pre-integrated light $\tilde{L}_i(\omega_r, b_r)$ [S1, unofficial].

**Data Structures:**
The method employs a coordinate-based Multi-Layer Perceptron (MLP) architecture based on pi-GAN with FiLM-SIREN layers [S1, unofficial].
*   **FiLM-SIREN Layer:** A linear layer followed by a sine activation, modulated by parameters. The transformation is defined as:
    $$ \phi(y) = \sin(\lambda_0 \gamma \odot y + \beta) $$
    where $y$ is the output of the linear layer, $\odot$ denotes the Hadamard product, and $\lambda_0, \gamma, \beta$ are modulating parameters [S1, unofficial].
*   **Mapping Networks:** Two separate mapping networks predict the modulating parameters $(\gamma, \beta)$ for the FiLM-SIREN layers [S1, unofficial].
    1.  The first mapping network generates parameters for the first few layers from the illumination embedding $\mathbf{z}_l$ [S1, unofficial].
    2.  The second mapping network generates parameters for the penultimate layer from the roughness $b_r$ [S1, unofficial].

**Algorithm Steps:**
1.  Encode the environment map into a 128-dimensional illumination embedding $\mathbf{z}_l$ using a frozen Light-SMAE (Smooth Manifold Auto-encoder) [S1, unofficial].
2.  For a given surface point, obtain the mirrored view direction $\omega_r$ and material roughness $b_r$ [S1, unofficial].
3.  Pass $\mathbf{z}_l$ through the first mapping network to generate modulating parameters $(\gamma, \beta)$ for the initial FiLM-SIREN layers [S1, unofficial].
4.  Pass $\omega_r$ through the initial FiLM-SIREN layers (modulated by parameters from step 3) to decode illumination information for the given direction [S1, unofficial].
5.  Pass $b_r$ through the second mapping network to generate modulating parameters for the penultimate layer [S1, unofficial].
6.  Pass the output from step 4 through the remaining FiLM-SIREN layers (modulated by parameters from step 5) to perform light integration conditioned on roughness [S1, unofficial].
7.  Output the final pre-integrated light value $\tilde{L}_i(\omega_r, b_r)$ [S1, unofficial].

**Rendering Equation Integration:**
The predicted $\tilde{L}_i$ is used in the split-sum approximation of the rendering equation [S1, unofficial]:
$$ L_o(x, \omega_o) \approx (b_d/\pi)\tilde{L}_i(n, 1) + b_s (F_0(\omega_o, n)B_0(\omega_o \cdot n, b_r) + B_1(\omega_o \cdot n, b_r))\tilde{L}_i(\omega_r, b_r) $$
where:
*   $b_d, b_s$ are diffuse and specular albedo weights [S1, unofficial].
*   $F_0(\omega_o, n) = (1 - \omega_o \cdot h)^5$ with $h = (\omega_i + \omega_o)/\|\omega_i + \omega_o\|$ [S1, unofficial].
*   $B_0, B_1$ are 2D lookup textures indexed by $(\omega_o \cdot n)$ and roughness $b_r$ [S1, unofficial].
*   $\tilde{L}_i(n, 1)$ is queried for the diffuse part (parameterized by normal $n$ and diffuse roughness 1) [S1, unofficial].
*   $\tilde{L}_i(\omega_r, b_r)$ is queried for the specular part [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The illumination embedding $\mathbf{z}_l$ must be derived from a dataset of environment maps to capture regularities (e.g., brighter light from above, tinting) [S1, unofficial].
*   Training the Neural-PIL requires a frozen Light-SMAE to ease memory requirements; joint training is not described [S1, unofficial].

## Subtopics

### Split-Sum and PIL Approximation
A baseline approximation method that splits the specular term into two factors: one involving the light integral and one involving the BRDF terms ($D, G, F$), allowing the light integral to be pre-integrated into a neural network [S2, unofficial]. The specular term is split as:
$$ \int_{\Omega_+} L(\hat{x}, l)(\hat{n} \cdot l) dl \int_{\Omega_+} \frac{DGF}{4(\hat{n} \cdot v)(\hat{n} \cdot l)} (\hat{n} \cdot l) dl $$
The rendering equation becomes:
$$ \tilde{L}\hat{\mathbf{c}}_b + (\pi\tilde{L}) \int_{\Omega_+} \frac{DGF}{4(\hat{\mathbf{n}} \cdot \mathbf{v})(\hat{\mathbf{n}} \cdot \mathbf{l})} (\hat{\mathbf{n}} \cdot \mathbf{l}) d\mathbf{l} $$
where $\tilde{L}$ is the pre-integrated neural network representing the light integral factor divided by $\pi$ [S2, unofficial]. This approach is used as a baseline in [NDJIR](14523-ndjir.md) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Replaces costly numerical integration with a simple network query, making it faster than direct integration or convolution on environment maps [S1, unofficial].
*   **Memory Complexity:** Lower memory cost compared to high-band spherical harmonic or spherical Gaussian representations required for fine illumination details [S1, unofficial].
*   **Strengths:** Represents finer illumination details than SG or SH representations; efficient for differentiable rendering engines [S1, unofficial].
*   **Limitations:** Requires training a separate Light-SMAE; performance depends on the quality of the illumination embedding space [S1, unofficial].
*   **When to use:** When high-fidelity illumination details are needed and standard basis functions (SH/SG) are insufficient or too expensive [S1, unofficial].

## Background
Neural-PIL was introduced in "Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition" [S1, unofficial]. It addresses the impracticality of costly numerical accumulation or convolution on complete environment maps within differential rendering engines [S1, unofficial]. The method leverages a pi-GAN-like architecture with FiLM-SIREN layers [S1, unofficial]. It builds on the split-sum approximation technique [S2, unofficial].

## Sources
[S1] Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition, https://doi.org/10.48550/arxiv.2110.14373, fetched 2026-10-26 (unofficial)
[S2] NDJIR: Neural Direct and Joint Inverse Rendering for Geometry, Lights, and Materials of Real Object, https://doi.org/10.48550/arxiv.2302.00675, fetched 2026-10-26 (unofficial)
