# NeRF Editing and Inpainting Procedures

Also called: NeRF Editing, NeRF Inpainting, 3D Scene Editing

## Overview
NeRF Editing and Inpainting Procedures modify the appearance or geometry of a pre-trained Neural Radiance Field (NeRF) without retraining the network parameters $\Theta$ from scratch, or by optimizing $\Theta$ using external guidance to remove or replace objects. This family solves the problem of 3D scene manipulation, such as object removal, insertion, or semantic editing, which standard NeRF training does not support. Unlike [NeRF Training Loop](970-nerf-training-loop.md) which learns a scene from scratch, these methods either intercept the rendering equation at inference time to apply direct transformations [12491-nerf-editing-procedure.md] or optimize the existing model using 2D priors (inpainting, diffusion) to fill occluded or masked regions [12372-nerf-in-inpainting-optimization.md].

## Shared principles
The family shares the goal of modifying the radiance field $F_\Theta(\mathbf{x}, \mathbf{v}) \to (\sigma, \mathbf{c})$ to achieve a specific visual edit. The core mechanisms fall into two categories:
1.  **Inference-Time Interception:** The rendering process is modified by querying an external feature field or segmentation model to identify regions of interest (ROI). For points in the ROI, density $\sigma$ and color $\mathbf{c}$ are transformed (e.g., set to 0 for deletion, or blended) before volume rendering [12491-nerf-editing-procedure.md].
2.  **Optimization-Based Inpainting:** The NeRF parameters $\Theta$ are updated via gradient descent using a loss function derived from 2D guidance images (inpainting results, depth completion) or generative priors (diffusion models). This ensures multi-view consistency by enforcing that the edited NeRF renders images that match the 2D guidance [12372-nerf-in-inpainting-optimization.md], [11860-deceptive-nerf-3dgs-pipeline.md].

## Subtopics

### [NeRF Editing Procedure](12491-nerf-editing-procedure.md)
Modifies $\sigma$ and $\mathbf{c}$ at inference time using feature probabilities. Real-time, no backpropagation.

### [NeRF-In Inpainting Optimization](12372-nerf-in-inpainting-optimization.md)
Optimizes $\Theta$ using RGB-D guidance from 2D inpainting. Removes objects without retraining.

### [Blended-NeRF](12273-blended-nerf.md)
Trains a generator NeRF for a 3D ROI and blends it with the original. Text-prompt driven.

### [Adaptive View Selection for Inpainting](12405-adaptive-view-selection-for-inpainting.md)
Uses 2D inpainting priors only when multi-view consistency fails. De-occlusion focused.

### [Deceptive-NeRF/3DGS Pipeline](11860-deceptive-nerf-3dgs-pipeline.md)
Generates pseudo-observations via diffusion to densify sparse views.

### [DFF (Decomposing NeRF for Editing Via Feature Field Distillation)](12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md)
Distills 2D semantic features into NeRF for segmentation-based editing.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Iteratively inpaints and erases conflicting geometry for single-view panorama reconstruction.

### Semantic Inpainting for Neural Fields
*   **Definition:** A 3D content editing capability allowing editing of neural fields through simple interfaces like free-form sketching or language instructions [S1].
*   **Step:** Given a user-supplied edit in a single reference view, the encoding of a neural field automatically propagates this edit to all views. This propagation promotes consistency between all views thanks to the implicit geometrical and photometric priors embodied by the neural network, preserving the structure of the whole scene [S1].
*   **History:** Described in reference [19] [S1].

### Post-Refinement Strategy
*   **Definition:** A strategy employed after the NeRF has been trained on updated data to enhance consistency by generating a more accurate target for each view using the modified Instruct-Pix2Pix with averaging architecture [S2].
*   **Step:** Use the rendered image as input while continuing to use the mixup image as a condition [S2].

## Background
NeRF editing emerged as a natural extension of the NeRF framework, which initially focused on novel view synthesis. Early methods relied on optimization-based approaches to remove objects or inpaint missing regions by optimizing the NeRF parameters against 2D guidance [12372-nerf-in-inpainting-optimization.md]. Later, inference-time editing methods were developed to allow real-time manipulation by intercepting the rendering pipeline [12491-nerf-editing-procedure.md]. The integration of 2D generative models (like Stable Diffusion) and large language models has further expanded the capabilities to include text-driven editing and semantic decomposition [12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md], [12273-blended-nerf.md].

## Sources
[S1] Review of Key Technologies and Application Scenarios of Neural Rendering, https://doi.org/10.54254/2755-2721/2026.36116, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
