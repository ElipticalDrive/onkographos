# Global NTS

Also called: Global Neural Texture Stack

## Overview
Global NTS is a Neural Texture Stack (NTS) variant that achieves novel view synthesis by directly conditioning texture generation on the global camera pose $\boldsymbol{\theta}$, replacing the local texture embedding used in the base Neural Textures method [S1]. Unlike Local NTS, which relies on local geometric correspondence and deformation fields, Global NTS models the entire texture variation globally, allowing it to generate new topological spaces (e.g., reconstructing occluded objects like a belt on pants) but suffering from ambiguous textures due to poor information reuse across different observation spaces [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Global camera pose $\boldsymbol{\theta}$, viewing direction $\boldsymbol{d}$, and part label vector $\boldsymbol{k}$.
*   **Output:** View-dependent color $\boldsymbol{c}$.

**Data Structures:**
The architecture mirrors the color MLP $M_c$ of the parent Neural Textures method [S1]. The key structural difference is the input layer:
*   **Standard NTS:** Input includes local texture embedding $\boldsymbol{e}_k(\boldsymbol{r})$.
*   **Global NTS:** Input replaces $\boldsymbol{e}_k(\boldsymbol{r})$ with the global pose vector $\boldsymbol{\theta}$ [S1].

**Algorithm:**
1.  **Pose Encoding:** The global pose $\boldsymbol{\theta}$ is encoded. In the source architecture, this replaces the texture embedding step.
2.  **Concatenation:** The pose representation is concatenated with the viewing direction $\boldsymbol{d}$ and the part label vector $\boldsymbol{k}$ [S1].
3.  **Color Decoding:** The concatenated vector is passed through a Multi-Layer Perceptron (MLP) to predict the RGB color $\boldsymbol{c}$ for the ray $\boldsymbol{r}$ [S1].

**Pitfalls:**
*   **Ambiguous Textures:** Because Global NTS directly conditions on global pose without local mapping restrictions, it struggles to reuse information across different observation spaces. This leads to visual artifacts such as unclear outlines or merged features (e.g., the number '9' on a shirt appearing connected and resembling an '8', or looking like a red stain) [S1].
*   **Topological Rigidity vs. Flexibility:** While it can generate new topological space (unlike Local NTS), the lack of local constraints makes high-frequency detail reconstruction difficult [S1].

## Subtopics

### [Local NTS](18022-local-nts.md)
Local NTS is a Neural Texture Stack variant that generates view-dependent textures by transforming observation-space UV coordinates into canonical UV coordinates using a pose-conditioned deformation field. Unlike Global NTS, which conditions texture generation directly on global pose $\boldsymbol{\theta}$, Local NTS relies on local geometric correspondence, making it computationally efficient but topologically rigid. It is distinguished from its sibling Spatial NTS by its inability to generate new topological space, limiting its capacity to model occlusion-induced texture changes (e.g., belts appearing or disappearing).

### Hyper NTS
Hyper NTS is a Neural Texture Stack variant that adds an ambient MLP to the Local NTS model to parameterize the deformation field, yielding warped UV coordinates and a coordinate $\boldsymbol{w}$ in an ambient space [S1]. The ambient MLP has the same architecture as the deformation MLP (three fully-connected layers of 256 channels) and outputs warped UV coordinates $(\hat{U}_k^t(\boldsymbol{r}), \hat{V}_k^t(\boldsymbol{r}))$ and the ambient coordinate $\boldsymbol{w}$ [S1]. These outputs are concatenated with the part label vector $\boldsymbol{k}$ and viewing direction $\boldsymbol{d}$ to produce view-dependent colors via a subsequent MLP (five fully-connected layers of 256 channels and one of 128) [S1]. Tuning the dimension of coordinate $\boldsymbol{w}$ is difficult: if too high, Hyper NTS behaves like Global NTS; if too low, it behaves like Local NTS [S1]. Performance on the CMU dataset: Novel PSNR 30.08, View SSIM 0.966, Synthesis LPIPS 0.059 [S1].

## Operating characteristics
**Performance:**
On the CMU dataset, Global NTS achieves a Novel View PSNR of 29.86, View SSIM of 0.966, and Synthesis LPIPS of 0.062 [S1]. It outperforms Local NTS (Novel PSNR 29.82) but is outperformed by Hyper NTS (Novel PSNR 30.08) and Spatial NTS (Novel PSNR 30.38) [S1].

**Strengths and Limitations:**
*   **Strengths:** Global NTS can successfully reconstruct topologically varying textures that Local NTS cannot, such as a belt on pants that becomes visible or hidden due to pose changes [S1].
*   **Limitations:** The method suffers from ambiguous textures and unclear outlines due to the difficulty of reusing information across different observation spaces without local mapping constraints [S1]. It is less effective at capturing high-frequency details compared to CNN-based Spatial NTS [S1].

## Background
Global NTS was introduced as an ablation study within the "UV Volumes for Real-time Rendering of Editable Free-view Human Performance" paper to evaluate different types of Neural Texture Stacks [S1]. It serves as a baseline to demonstrate the trade-offs between global pose conditioning and local texture embedding in neural rendering architectures [S1].

## Sources
[S1] UV Volumes for Real-time Rendering of Editable Free-view Human Performance, https://doi.org/10.1109/cvpr52729.2023.01595, fetched 2026-10-26
