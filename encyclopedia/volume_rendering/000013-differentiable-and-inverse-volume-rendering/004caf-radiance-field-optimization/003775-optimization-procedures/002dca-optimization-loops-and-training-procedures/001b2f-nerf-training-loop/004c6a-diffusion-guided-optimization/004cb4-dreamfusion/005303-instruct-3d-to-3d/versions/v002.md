# Instruct 3D-to-3D

Also called: Instruct 3D-to-3D conversion

## Overview
Instruct 3D-to-3D is a method for performing 3D-to-3D conversions guided by text instructions [S1]. It solves the problem of editing an existing 3D scene according to natural language prompts by integrating the source 3D scene as a conditional input and optimizing the target scene using the Score Distillation Sampling (SDS) loss [S1]. It is distinguished from its parent [DreamFusion](12087-dreamfusion.md) by its conditional nature: rather than generating a scene from noise, it modifies an existing geometry and appearance while preserving structural fidelity through dynamic scaling [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A source 3D scene (represented as a NeRF or similar radiance field) and a text instruction prompt.
*   **Output:** An optimized target 3D scene that reflects the text instruction while maintaining consistency with the source structure.

**Data Structures:**
*   **Source Scene:** The initial radiance field parameters (e.g., MLP weights for NeRF or Gaussian parameters for 3DGS) [S1].
*   **Conditional Input:** The source scene is encoded or concatenated as a condition for the optimization process [S1].

**Algorithm Steps:**
1.  **Conditioning:** Integrate the source 3D scene as a conditional input to the optimization loop [S1]. This ensures the output remains grounded in the original geometry.
2.  **Optimization Loop:**
    a. Render images from the current target scene estimate using standard volume rendering or rasterization [S1].
    b. Compute the SDS loss [89] using a pre-trained text-to-image diffusion model (e.g., InstructPix2Pix) conditioned on the text prompt and the rendered images [S1].
    c. Backpropagate the SDS loss to update the target scene parameters [S1].
3.  **Dynamic Scaling:** Integrate dynamic scaling mechanisms to effectively modify scene geometry during optimization [S1]. This allows the method to adjust the scale of specific features or the entire object to match the semantic changes requested by the text prompt without distorting the underlying structure arbitrarily.

**Parameters:**
*   **Learning Rate:** Not given in the sources.
*   **Number of Iterations:** Not given in the sources.
*   **Diffusion Model:** Typically InstructPix2Pix or similar instruction-following models [S1].

**Edge Cases and Pitfalls:**
*   **Geometry Distortion:** Without dynamic scaling, text-guided edits may lead to unnatural geometric distortions [S1].
*   **Fidelity Loss:** Failure to properly condition on the source scene may result in outputs that deviate significantly from the original object's identity [S1].

## Subtopics

### Focaldreamer
Focaldreamer is a variant focused on localized 3D editing [S1]. It initiates with a base shape and editable ellipsoids in focal regions, utilizing rendered normal maps as shape encoding for the text-to-image (T2I) model to guide geometry optimization [S1]. The shape is rendered with both base and editable textures, unified by a pixelwise discriminative mask [S1]. In addition to the SDS loss [89], it incorporates several regularizations to improve editing quality [S1]. It is chosen when precise, localized edits are required without affecting the entire scene.

### Pix2Vex Reconstructor
Pix2Vex Reconstructor is a network-based method translating images to a 3D model [S2, unofficial]. It is based on a pix2pix convolutional and deconvolutional ResNet architecture [S2, unofficial]. The output of the pix2pix component is passed through two fully connected layers to produce $3 \times \#vertices$ values, followed by a sigmoid layer and a linear transformation to map values to the range $[-1, 1]$ [S2, unofficial]. These values are treated as vectors and added to the vertices of a base model (a uniform sphere with predefined topology) [S2, unofficial]. It predicts offsets rather than absolute coordinates to improve convergence smoothness [S2, unofficial]. Optionally, it predicts symmetry by splitting the model into four quarters, predicting shifts for one, mirroring it, and repeating until all quarters are predicted [S2, unofficial]. It also predicts uniform translation and scaling values to reduce offset magnitude and improve stability [S2, unofficial]. Regularization is applied on the 3D mesh considering angles between normals, edge lengths, and curvature [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Enables text-guided editing of existing 3D assets; preserves source structure via conditioning [S1].
*   **Limitations:** Relies on the quality of the diffusion model and SDS loss; may struggle with complex geometric changes if dynamic scaling is insufficient [S1].
*   **When to use:** When the goal is to modify an existing 3D scene based on text instructions.
*   **When not to use:** When generating a 3D scene from scratch without a source object (use [DreamFusion](12087-dreamfusion.md) instead).

## Background
Instruct 3D-to-3D was introduced as a method for realizing 3D-to-3D conversions following text instructions [S1]. It builds upon the SDS loss framework [89] commonly used in differentiable rendering optimization [S1]. The method leverages instruction-following models like InstructPix2Pix to guide the editing process [S1]. It is part of a broader series of 3D editing works that utilize SDS to optimize NeRF and 3DGS representations [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] Pix2Vex: Image-to-Geometry Reconstruction using a Smooth Differentiable Renderer, https://doi.org/10.48550/arxiv.1903.11149, fetched 2026-10-26 (unofficial)
