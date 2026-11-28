# Geometry and material MLP architecture

Also called: Geometry Prediction, NeuS-style MLP, SDF and Color Network Prediction, dual MLP network

## Overview
This architecture defines the neural network components for decoding signed distance functions (SDF) and material properties (diffuse albedo, specular coefficients) in neural implicit surface reconstruction [S1, S2]. It separates geometry and material decoding into two distinct multi-layer perceptrons (MLPs) [S1]. It is distinguished from single-MLP approaches by its explicit feature vector passing from the geometry network to the material network, allowing the material decoder to access high-level geometric context [S1].

## How it works

### Inputs and Outputs
*   **Geometry MLP Input:** 3D position $\mathbf{x} \in \mathbb{R}^3$ encoded with 6-frequency positional encoding [S2].
*   **Geometry MLP Output:** Scalar signed distance $d$ and a 256-dimensional feature vector $\mathbf{f} \in \mathbb{R}^{256}$ [S2].
*   **Material MLP Input:** Concatenation of the 3D position $\mathbf{x}$, the surface normal $\mathbf{n}$, and the feature vector $\mathbf{f}$ from the Geometry MLP [S2].
*   **Material MLP Output:** 3-channel diffuse albedo $\mathbf{c}_{diff}$ and 27 specular coefficients $\mathbf{c}_{spec}$ [S2].

### Data Structures and Layout

**Geometry MLP**
*   **Layers:** 8 fully connected layers [S2].
*   **Hidden Dimension:** 256 units per layer [S2].
*   **Activation:** Softplus [S2].
*   **Input Encoding:** Positional encoding with 6 frequencies [S2].

**Material MLP**
*   **Layers:** 4 fully connected layers [S2].
*   **Hidden Dimension:** 256 units per layer [S2].
*   **Activation:** ReLU for hidden layers [S2].
*   **Output Activation:** Softplus with $\beta = 100$ [S2].

### Steps

1.  **Positional Encoding:** Encode the input 3D position $\mathbf{x}$ using 6-frequency positional encoding $\gamma(\mathbf{x})$ [S2].
2.  **Geometry Decoding:** Pass $\gamma(\mathbf{x})$ through the 8-layer Geometry MLP to predict the signed distance and extract the feature vector [S2].
3.  **Normal Computation:** Compute the surface normal $\mathbf{n}$ as the normalized gradient of the signed distance function with respect to position: $\mathbf{n} = \frac{\nabla d}{\|\nabla d\|_2}$ [G].
4.  **Feature Concatenation:** Concatenate the original position $\mathbf{x}$, the computed normal $\mathbf{n}$, and the feature vector $\mathbf{f}$: $\mathbf{z} = [\mathbf{x}, \mathbf{n}, \mathbf{f}]$ [S2].
5.  **Material Decoding:** Pass $\mathbf{z}$ through the 4-layer Material MLP to obtain diffuse albedo and specular coefficients [S2].
6.  **Specular Color Calculation:** Compute the 3-channel specular color $\mathbf{c}_{spec\_color}$ by linearly combining 9 spherical Gaussian bases $\{B_i\}_{i=1}^9$ with different shininess using the 27 output coefficients $\mathbf{c}_{spec}$: $\mathbf{c}_{spec\_color} = \sum_{i=1}^{9} c_{spec, i} \cdot B_i$, where $\mathbf{c}_{spec}$ contains 9 coefficients per color channel (R, G, B) [S2].

### Parameters
*   **Positional Encoding Frequencies:** 6 [S2].
*   **Softplus Beta ($\beta$):** 100 for material output activation [S2].

### Edge Cases and Pitfalls
*   **Gradient Vanishing/Exploding:** The use of Softplus in the geometry MLP and ReLU in the material MLP requires careful initialization (e.g., geometric initialization) to ensure stable training [S3].
*   **SDF Sensitivity:** Deriving density or material properties directly from SDF without a dedicated branch can lead to inaccuracies due to SDF sensitivity to nearby objects, potentially generating false surfaces [S3].

## Subtopics

### SRDF Branch Architecture
A dedicated neural network branch that predicts the Ray-Specific Density Function (SRDF) directly, rather than deriving it from the SDF, to avoid inaccuracies caused by the SDF's sensitivity to nearby objects [S3]. This approach prevents the generation of false surfaces and imprecise density estimates that occur when SRDF is derived from SDF [S3].

*   **SRDF MLP:** 2 layers, intermediate output of 256 channels, Softplus activation [S3].
*   **Geometry MLP:** 8 layers, hidden dimension 256, Softplus activation [S3].
*   **Color MLP:** 2 layers, intermediate output 256 channels, ReLU activation [S3].
*   **Grid Representation:** 2 layers [S3].

## Operating characteristics
Not given in the sources.

## Background
This architecture builds on [Neural SDF and Material MLPs](10268-neural-sdf-and-material-mlps.md) and is based on the NeuS framework, which introduced the separation of geometry and material decoding using a feature vector [S1]. It has been extended in subsequent works such as ShadowNeuS, which incorporates shadow ray supervision, and Ray-Distance Volume Rendering, which introduces the SRDF branch to address SDF limitations [S2, S3].

## Sources
[S1] Sensing Surface Patches in Volume Rendering for Inferring Signed Distance Functions, https://doi.org/10.1609/aaai.v39i4.32426, fetched 2026-10-26
[S2] ShadowNeuS: Neural SDF Reconstruction by Shadow Ray Supervision, https://doi.org/10.1109/cvpr52729.2023.00025, fetched 2026-10-26
[S3] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
