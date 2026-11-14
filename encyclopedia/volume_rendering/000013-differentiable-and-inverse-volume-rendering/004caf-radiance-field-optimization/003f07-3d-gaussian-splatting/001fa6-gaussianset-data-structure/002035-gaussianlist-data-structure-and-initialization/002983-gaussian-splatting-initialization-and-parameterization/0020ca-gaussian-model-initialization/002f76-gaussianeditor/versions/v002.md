# GaussianEditor


## Overview
GaussianEditor is a text-driven local editing method for 3D Gaussian Splatting (3DGS) that leverages 2D image-editing models to drive updates of 3D Gaussians [S1, S3]. It solves the problem of localized appearance and geometry modification by repeatedly rendering GS views, applying 2D edits, and back-propagating the differences to Gaussian parameters [S1]. It is distinguished from sibling methods like [Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md) or [DreamGaussian](5998-dreamgaussian.md) by its specific use of Hierarchical Gaussian Semantic Tracing (HGS) for adaptive generative guidance and its iterative refinement loop [S2, unofficial][S3].

## How it works
**Inputs and Outputs:**
- **Inputs:** A pre-trained 3DGS model, a text prompt describing the desired edit, and optionally a mask or region of interest (RoI) [S3].
- **Outputs:** An updated 3DGS model with modified Gaussian parameters [S1, S3].

**Data Structures:**
The method operates on the standard 3DGS data structure (position, covariance, opacity, spherical harmonics coefficients) [S1]. It introduces a semantic tracing mechanism to map text prompts to specific 3D Gaussian subsets [S3].

**Steps:**
1. **Region of Interest (RoI) Extraction:** Extract the target region from text instructions [S3].
2. **Semantic Tracing:** Align the text RoI with 3D Gaussians through the image space using Hierarchical Gaussian Semantic Tracing [S3]. This identifies which Gaussians correspond to the semantic region defined by the text [S2, unofficial][S3].
3. **2D Edit Application:** Render views of the current 3DGS. Apply a 2D image-editing model (e.g., diffusion-based editor) to the rendered images, guided by the traced RoI [S1, S3]. Two specific guidance models are implemented:
   - **GaussianEditor-iN2N:** Guided by IN2N [S3].
   - **GaussianEditor-DDS:** Guided by DDS [S3].
   GaussianEditor-iN2N is noted to achieve superior results [S3].
4. **Back-propagation:** Compute the difference between the edited 2D image and the original rendered image. Back-propagate this difference to update the 3D Gaussian parameters (position, scale, rotation, opacity, SH coefficients) [S1].
5. **Iterative Refinement:** Repeat steps 3-4 to refine the edit and improve multi-view consistency [S1, S3].

**Parameters:**
- **Guidance Model:** IN2N or DDS [S3].
- **Iteration Count:** Not given in the sources.

**Edge Cases and Pitfalls:**
- **Unintended Deformations:** Due to weak 3D geometry priors, the method can suffer from unintended deformations [S1].
- **Computational Cost:** The pipeline is computationally heavy due to repeated rendering and back-propagation [S1].

## Subtopics

### GaussianEditor-iN2N
A version guided by IN2N, achieving superior results compared to the DDS variant [S3].

### GaussianEditor-DDS
A version guided by DDS [S3].

## Operating characteristics
- **Time Complexity:** Computationally heavy due to iterative rendering and back-propagation [S1].
- **Memory Complexity:** Standard 3DGS memory usage plus overhead for semantic tracing and 2D model inference [S1, S3].
- **Strengths:** Intuitive, compatible with powerful 2D priors, enables text- or mask-driven appearance and geometry changes [S1].
- **Limitations:** Can suffer from unintended deformations due to weak 3D geometry priors [S1].
- **When to use:** When text-driven local editing is required and computational resources are available [S1, S3].
- **When not to use:** When real-time performance is critical or when strong 3D geometry consistency is required without iterative refinement [S1].

## Background
GaussianEditor was introduced by Chen et al. in CVPR'24 [S2, unofficial]. It is part of the family of methods that leverage 2D image-editing models to drive updates of 3D Gaussians [S1]. Follow-up works have improved spatial localization and multi-view consistency via attention, view correspondence, or iterative refinement [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
