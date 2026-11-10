# MS-NeuS
Also called: Multi-Space NeuS
## Overview
MS-NeuS integrates the Multi-Space (MS) module into the NeuS framework to address multi-view inconsistency, specifically improving novel view synthesis and the reconstruction of mirror surfaces [S1]. It distinguishes itself from its parent [D-NeuS](10217-d-neus.md) by focusing on view-dependent appearance consistency via sub-space decomposition rather than explicit geometric bias or feature consistency losses [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** Multi-view images of a scene containing reflective/mirror surfaces.
*   **Output:** A neural implicit surface representation (SDF) and a view-dependent radiance field for novel view synthesis.

**Data Structures:**
The method extends the NeuS architecture [S1]. The core addition is the MS module, which decomposes the feature space.
*   **Sub-spaces:** The feature space is divided into $K$ sub-spaces.
*   **Feature Dimension:** $d$ (dimension of features in each sub-space).
*   **Hidden Dimension:** $h$ (hidden dimension of the MLPs).
*   **Parameters:** The source specifies grid parameters and MLP parameters separately, denoted as "A|B" [S1].

**Algorithm:**
1.  **Initialization:** Initialize the NeuS backbone (SDF MLP and Color MLP) and the MS module parameters.
2.  **Sampling:** For each pixel, sample points along the camera ray.
3.  **Sub-space Decomposition:** Instead of a single feature vector, the representation utilizes $K$ sub-spaces. The specific mathematical formulation of how features are projected into or aggregated from these $K$ sub-spaces is not detailed in the provided sources [G].
4.  **Rendering:** Perform volumetric rendering using the decomposed features to compute color and density.
5.  **Optimization:** Minimize the standard NeuS losses (photometric, SDF, etc.) combined with the multi-view consistency enforced by the MS module.

**Parameters:**
*   $K = 4$: Number of sub-spaces [S1].
*   $d = 16$: Feature dimension [S1].
*   $h = 32$: Hidden dimension [S1].

**Edge Cases and Pitfalls:**
*   **Geometric Cleanliness:** MS-NeuS struggles to reconstruct clean geometry because the MS module is primarily designed for novel view synthesis and lacks explicit geometric constraints on the sub-spaces [S1].
*   **Mirror Surfaces:** While it improves mirror reconstruction compared to standard NeuS, artifacts may persist due to the lack of geometric regularization in the sub-spaces [S1].

## Subtopics

### Geo-NeuS
Geo-NeuS introduces multi-view constraints for SDF supervised by a NeRF, specifically a multi-view geometry constraint supervised by a sparse point cloud and a multi-view photometric consistency constraint [S2]. It is chosen when sparse geometric priors are available to enforce consistency, whereas MS-NeuS relies on sub-space decomposition for view-dependent appearance.

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** MS-NeuS has more parameters than NeuS. On the synthesized dataset, NeuS uses $0.16 \times 10^1$ M parameters, while MS-NeuS uses $0.37 \times 10^1$ M parameters [S1].
*   **Time:** The method performs multiple volumetric rendering operations per pixel. The bottleneck is network inference, and time consumption is related to the importance sampling strategy and backbone networks [S1].

**Measured Performance:**
On a synthesized dataset with circle paths [S1]:
*   **PSNR:** MS-NeuS achieves 28.65, compared to NeuS 25.86.
*   **SSIM:** MS-NeuS achieves 0.859, compared to NeuS 0.831.
*   **LPIPS:** MS-NeuS achieves 0.195, compared to NeuS 0.258.

**Strengths and Limitations:**
*   **Strengths:** Significantly improves novel view synthesis quality and reconstructs better geometry for mirror surfaces compared to standard NeuS [S1].
*   **Limitations:** Does not produce clean geometry due to the absence of geometric constraints on sub-spaces [S1].

## Background
MS-NeuS is a generalization of the MS-NeRF scheme [S1] applied to the NeuS framework [S1]. It was developed to solve multi-view inconsistency for novel view synthesis, particularly in scenes with reflective surfaces. The implementation is based on the NeuS code from [66] (cited in [S1]).
## Sources
[S1] MS-NeRF: Multi-Space Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3540074, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
