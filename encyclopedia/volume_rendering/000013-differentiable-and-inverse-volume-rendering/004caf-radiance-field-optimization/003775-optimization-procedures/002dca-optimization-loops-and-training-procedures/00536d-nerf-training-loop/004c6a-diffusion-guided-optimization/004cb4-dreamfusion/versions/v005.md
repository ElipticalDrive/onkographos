# DreamFusion

Also called: Diffusion-guided NeRF Optimization, Score Distillation Sampling, Text-to-3D NeRF Optimization, Text-to-3D diffusion NeRF

## Overview
DreamFusion is a text-to-3D generation method that optimizes a Neural Radiance Field (NeRF) representation to match a given textual description or image without explicit 3D supervision [S2][S4]. It achieves this by backpropagating gradients from a frozen 2D diffusion model (specifically Imagen) through a differentiable renderer to guide the scene toward the diffusion prior [S1][S3, unofficial]. It is distinguished from its siblings by being the first method to introduce Score Distillation Sampling (SDS) for this purpose, effectively "distilling" 2D generative knowledge into 3D geometry [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A text prompt $c$ (optionally including view-specific keywords like "front view") [S1].
*   **Output:** A trained NeRF model $\theta$ representing the 3D scene [S1].

**Data Structures:**
*   **NeRF Model:** A mipNeRF 360-based architecture [S1].
    *   **Geometry:** Standard MLPs mapping 3D coordinates and viewing directions to density and color.
    *   **Color Parameterization:** A key modification is that surface color is parameterized by an MLP instead of standard radiance [S1].
*   **Diffusion Model:** A pretrained text-to-image diffusion model (Imagen) [S1][S2].
    *   **State:** Frozen during optimization [S3, unofficial].
    *   **Function:** $\epsilon_\phi(x_t, t, c)$ predicts the noise $\epsilon$ added to the image at timestep $t$, conditioned on the text $c$ [S3, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize a mipNeRF 360 model $\theta$ from scratch for the target object or scene [S1].
2.  **Camera Sampling:** Sample a random camera pose and viewing direction for the current iteration [G].
3.  **Differentiable Rendering:** Render an image $x$ from the current NeRF $\theta$ using volume rendering [S3, unofficial].
4.  **Noise Addition:** Sample a diffusion timestep $t$ uniformly from $[1, T]$ and add Gaussian noise $\epsilon \sim \mathcal{N}(0, I)$ to the rendered image $x$ to obtain the noisy latent $x_t = \sqrt{\bar{\alpha}_t} x + \sqrt{1-\bar{\alpha}_t} \epsilon$ [G].
5.  **Diffusion Prediction:** Pass $x_t$ and the text prompt $c$ into the frozen diffusion model to predict the noise $\epsilon_\phi(x_t, t, c)$ [S3, unofficial].
6.  **Gradient Computation:** Compute the SDS loss gradient with respect to the NeRF parameters $\theta$ [S3, unofficial]:
    $$ \nabla_\theta \mathcal{L}_{\text{SDS}} = \mathbb{E}_{t, \epsilon}\left[ w(t)\,(\epsilon_\phi(x_t, t, c) - \epsilon)\,\frac{\partial x}{\partial\theta} \right] $$
    Where:
    *   $w(t)$ is a timestep weighting function [S3, unofficial].
    *   $\frac{\partial x}{\partial\theta}$ is the gradient of the rendered image with respect to the NeRF parameters, obtained via backpropagation through the differentiable renderer [S3, unofficial].
7.  **Optimization Update:** Update $\theta$ using gradient descent (or Adam) with the computed gradient [S3, unofficial].
8.  **Iteration:** Repeat steps 2-7 for a fixed number of iterations or until convergence [S1].

**Parameters:**
*   **Diffusion Model:** Imagen is used as the parametric prior [S2].
*   **Resolution:** Imagen images were generated at 64x64 resolution [S1].

**Edge Cases and Pitfalls:**
*   **Resolution Limitation:** Due to the 64x64 resolution of the source diffusion images, the resulting NeRF lacks the capability to produce finer details [S1].
*   **Janus Problem:** The method suffers from the "Janus problem," where the generated 3D object exhibits multi-face artifacts (e.g., two faces on opposite sides of a head) [S3, unofficial].
*   **Color Saturation:** The optimization often produces saturated or oversaturated colors [S3, unofficial].
*   **Speed:** Optimization is slow, taking hours per scene [S3, unofficial].

## Subtopics

### Magic3D
A two-stage coarse-fine approach that builds on DreamFusion to address the low-resolution issue by refining a low-resolution NeRF into a high-resolution mesh [S1].

### Latent-NeRF
A method that trains a NeRF to output latent features operated on by Stable Diffusion, allowing for text and shape guidance [S1].

### VSD (Variational Score Distillation)
Used in ProlificDreamer, this replaces the noise target $\epsilon$ with a learned variational distribution to reduce mode collapse [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Optimization is slow, typically requiring hours per scene [S3, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Requires no 3D training data; leverages powerful 2D diffusion priors for photorealism and semantic fidelity [S2][S4].
*   **Limitations:** Suffers from the Janus problem (multi-face artifacts), saturated colors, and limited detail due to low-resolution diffusion outputs (64x64) [S1][S3, unofficial].
*   **When to use:** When generating 3D objects from text descriptions without access to 3D training data or multi-view images.
*   **When not to use:** When high-frequency detail is required, or when consistency across views (avoiding Janus artifacts) is critical.

## Background
DreamFusion was introduced in September 2022 [S1][S4]. It was proposed as a text-to-3D diffusion NeRF model that uses images from 2D diffusion models (specifically Imagen) to train a NeRF from scratch [S1]. The method sidesteps the lack of large-scale 3D training datasets by optimizing a NeRF representation so that its multiview renderings match a given textual description, constrained by a 2D diffusion model used as a parametric prior [S2]. It introduced the Score Distillation Sampling (SDS) technique for this purpose [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Review of Key Technologies and Application Scenarios of Neural Rendering, https://doi.org/10.54254/2755-2721/2026.36116, fetched 2026-10-26
[S3] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S4] Differentiable Rendering - 可微渲染：从2D 学习3D, https://www.ultralytics.com/zh/glossary/differentiable-rendering, fetched 2026-10-30
