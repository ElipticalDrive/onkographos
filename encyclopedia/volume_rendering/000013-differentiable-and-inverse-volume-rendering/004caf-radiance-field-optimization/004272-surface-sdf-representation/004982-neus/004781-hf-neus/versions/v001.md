# HF-NeuS

Also called: High-Frequency NeuS

## Overview
HF-NeuS is an improvement on NeuS that separates low-frequency details into a base SDF and high-frequency details into a displacement function to increase reconstruction quality [S1]. It builds on [NeuS](6060-neus.md) by addressing the smoothness limitations of standard implicit SDF representations, which often fail to capture sharp geometric features [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Calibrated multi-view images (intrinsics $\{K_i\}$, extrinsics $\{T_i\}$), camera poses.
*   **Outputs:** Reconstructed 3D surface (SDF) and appearance (light field).

**Data Structures:**
The method decouples geometry and appearance. Geometry is represented by an implicit Signed Distance Function (SDF) $\Phi$, and appearance by a light field encoded via a color network $c: \mathbb{R}^3 \times \mathbb{S}^2 \rightarrow \mathbb{R}^3$ [S2]. The specific high-frequency displacement mechanism is described as a separation of low-frequency base SDF and high-frequency displacement, but the exact architectural layout of the displacement network is not detailed in the provided sources [S1].

**Numbered Steps:**
1.  **Ray Definition:** For a given pixel, define the ray $r(s) = o + sv$ for $s \ge 0$, where $o$ is the camera center and $v$ is the unit direction vector [S2].
2.  **Sampling:** Uniformly sample points along the ray within the bounding volume $\Omega$ [S2]. Additionally, uniformly sample 500 points per iteration in the bounding volume $\Omega$ for the off-surface loss calculation [S2].
3.  **Density and Color Evaluation:** Evaluate the density $\sigma(r(t))$ and color $c(r(t), v)$ at sampled points $r(t)$ [S2]. The density is formulated to be an unbiased and occlusion-aware function, as in NeuS [S2].
4.  **Volume Rendering:** Integrate color along the ray using the accumulated transmittance $T(t)$:
    $$ \hat{C}(r) = \int_{s_n}^{s_f} T(t) \sigma(r(t)) c(r(t), v) dt $$
    where $s_n$ and $s_f$ are the near and far bounds of the ray [S2].
5.  **Transmittance Calculation:** Compute accumulated transmittance:
    $$ T(t) = \exp\left(-\int_{s_n}^{t} \sigma(r(u)) du\right) $$
    [S2].
6.  **Mask Estimation:** Compute the predicted mask $\hat{M}(r)$ as the sum of transmittance weights along the ray [S2].
7.  **Loss Optimization:** Optimize network parameters by minimizing the total loss $L$ [S2]:
    $$ L = L_{color} + \alpha L_{eikonal} + \beta L_{mask} + \gamma L_{off} $$
    [S2].

**Formulas:**
*   **Color Loss ($L_{color}$):**
    $$ L_{color} = \frac{1}{\#R} \sum_{r \in R} \| M(r) (\hat{C}(r) - C(r)) \|_1 $$
    where $R$ is the ray set constructed from all images, $C(r)$ is ground truth color, and $M(r)$ is the ground truth object mask [S2].
*   **Eikonal Loss ($L_{eikonal}$):**
    $$ L_{eikonal} = \frac{1}{\#X} \sum_{p \in X} (\|\nabla_x \Phi(p)\|^2 - 1)^2 $$
    where $X$ is the set of sample points on rays [S2].
*   **Mask Loss ($L_{mask}$):**
    $$ L_{mask} = \frac{1}{\#R} \sum_{r \in R} BCE(\hat{M}(r), M(r)) $$
    where $BCE$ is binary cross entropy [S2].
*   **Off-Surface Loss ($L_{off}$):**
    $$ L_{off} = \frac{1}{\#\Omega} \sum_{x \in \Omega} \psi(\Phi(x), \beta_0) $$
    where $\Omega$ is the bounding volume, $\beta_0 = 100$, and $\psi$ is a penalty function encouraging off-surface points to have non-zero values [S2].

**Parameters:**
*   $\alpha, \beta, \gamma$: Weights for the respective loss terms. Values are not given in the sources.
*   $\beta_0$: Parameter for the off-surface loss, set to 100 [S2].
*   Sampling points: 500 points per iteration in $\Omega$ for $L_{off}$ [S2].

**Edge Cases and Pitfalls:**
*   The sources note that baseline geometry representations (like NGP) can lead to non-smooth reconstruction results with poor gradient continuity [S2]. HF-NeuS aims to mitigate this via high-frequency separation, though specific numerical stability guards for the displacement function are not provided [S1].
*   The mask term $L_{mask}$ is optional [S2].

## Subtopics

### Neural Multi-view Image Reconstruction
A framework that decouples geometry (SDF) and appearance (light field) from calibrated multi-view images, using volume rendering to generate images and minimizing the difference between rendered and input images [S2]. It utilizes the NeuS volume rendering framework to render 2D images based on the neural implicit SDF and light field [S2].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
HF-NeuS outperformed the baseline NeRF model and provided theoretical and experimental justifications for its SDF-based scene density implementation [S1]. It greatly increased reconstruction quality compared to standard NeuS [S1].

**Strengths and Limitations:**
*   **Strengths:** Improves geometric reconstruction quality by separating low-frequency and high-frequency details [S1].
*   **Limitations:** The specific computational overhead of the displacement function is not quantified in the sources.

**When to use it:**
Use when high-frequency geometric details are required and standard NeuS reconstruction quality is insufficient [S1].

## Background
HF-NeuS was introduced in June 2022 [S1]. It is an improvement on NeuS, which was designed to improve geometric reconstruction quality through volume rendering [S1]. Concurrently, other methods like Geo-NeuS and SparseNeus were introduced to address multi-view constraints and sparse-view reconstruction, respectively [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Multi-scale hash encoding based neural geometry representation, https://doi.org/10.1007/s41095-023-0340-x, fetched 2026-10-26
