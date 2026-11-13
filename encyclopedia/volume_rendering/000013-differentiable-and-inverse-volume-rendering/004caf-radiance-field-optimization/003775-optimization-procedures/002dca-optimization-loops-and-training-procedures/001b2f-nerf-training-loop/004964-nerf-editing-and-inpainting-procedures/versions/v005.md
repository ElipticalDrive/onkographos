# NeRF Editing and Inpainting Procedures

Also called: NeRF Editing, NeRF Inpainting, 3D Content Editing

## Overview
This family of methods modifies a pre-trained Neural Radiance Field (NeRF) to remove unwanted objects, inpaint occluded regions, or alter semantic content without retraining the entire scene from scratch. They solve the problem of localized 3D editing, addressing view-dependent artifacts and geometric inconsistencies that arise when modifying specific regions. Unlike global retraining, these methods decouple editing logic from global parameter updates, using techniques such as feature field distillation, diffusion-guided pseudo-observations, or adaptive view selection to ensure multi-view consistency [S1, S2].

## Shared principles
All methods in this family operate on a pre-trained NeRF representation $F_\Theta$ mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $c$. The core mechanism involves intercepting the standard volume rendering pipeline to apply transformations based on external guidance (semantic masks, diffusion priors, or depth consistency).

1.  **Ray Marching Interception**: During rendering, points $\mathbf{x}_k$ along rays $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ are sampled. The standard query $F_\Theta(\mathbf{x}_k, \mathbf{d})$ is modified or weighted based on an edit mask or probability map.
2.  **Guidance Integration**: External information is integrated via:
    *   **Feature Distillation**: Aligning NeRF features with 2D teacher network outputs (e.g., LSeg, DINO) to enable semantic decomposition [S1].
    *   **Diffusion Priors**: Using 2D generative models (e.g., Stable Diffusion, Instruct-Pix2Pix) to generate color or depth priors for occluded or missing regions [S1, S2].
    *   **Geometric Consistency**: Enforcing multi-view consistency through depth warping and epipolar constraints to validate inpainted regions [S1].
3.  **Optimization Loop**: The NeRF parameters $\Theta$ are fine-tuned using a composite loss function. Common loss components include:
    *   **Photometric Loss**: Difference between rendered and target (original or inpainted) colors.
    *   **Depth Loss**: Consistency between rendered depth and guidance depth.
    *   **Semantic/Feature Loss**: Alignment with distilled features or text embeddings.
    *   **Regularization**: Transmittance and depth variance losses to prevent transparency or amorphous artifacts [S1].

## Subtopics

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
A method for reconstructing a panoramic NeRF from a single input panorama by iteratively expanding training viewpoint coverage. It solves geometry conflicts in invisible regions by detecting and erasing inconsistent regions from supervision. It uses Instant-NGP.

### [Deceptive-NeRF/3DGS Pipeline](11860-deceptive-nerf-3dgs-pipeline.md)
A sparse-view reconstruction method that enhances NeRF/3DGS by generating synthetic pseudo-observations via a diffusion model during training. It acquires uncertainty through correspondence matching and epipolar constraints, minimizing computation while enforcing consistency.

### [DFF (Decomposing NeRF for Editing Via Feature Field Distillation)](12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md)
Enables 3D semantic segmentation and editing by distilling features from a pre-trained 2D teacher (LSeg/DINO) into a NeRF. It produces multi-view consistent feature fields for selective editing (deletion, color change) without breaking unintended areas.

### [Blended-NeRF](12273-blended-nerf.md)
A framework for local 3D editing using text prompts and a 3D ROI bounding box. It trains a generator NeRF initialized with original weights and blends content volumetrically, operating on any region without learning a new feature space.

### [NeRF-In Inpainting Optimization](12372-nerf-in-inpainting-optimization.md)
Removes unwanted objects by optimizing NeRF parameters using color and depth guidance from 2D inpainted images. It combines color-guiding and depth-guiding losses to ensure visual plausibility and geometric consistency.

### [Adaptive View Selection for Inpainting](12405-adaptive-view-selection-for-inpainting.md)
A 3D scene de-occlusion method combining 2D appearance priors (LaMa) with multi-view consistency. It dynamically selects between 2D priors and 3D consistency based on local texture richness (average gradient).

### [NeRF Editing Procedure](12491-nerf-editing-procedure.md)
Enables localized 3D editing by integrating external semantic/geometric guidance into the volume rendering pipeline. It uses query-based selection (via DFF or masks) to selectively transform density and color during ray marching.

### Semantic Inpainting for Neural Fields
**Definition**: A 3D content editing capability allowing editing of neural fields through simple interfaces like free-form sketching or language instructions [S1].
**Steps**: Given a user-supplied edit in a single reference view, the encoding of a neural field automatically propagates this edit to all views. This propagation promotes consistency between all views thanks to the implicit geometrical and photometric priors embodied by the neural network, preserving the structure of the whole scene [S1].
**History**: Described in reference [19] [S1].

### Post-Refinement Strategy
**Definition**: A strategy employed after the NeRF has been trained on updated data to enhance consistency by generating a more accurate target for each view using the modified Instruct-Pix2Pix with averaging architecture [S2].
**Steps**: Use the rendered image as input while continuing to use the mixup image as a condition [S2].

## Background
NeRF editing and inpainting emerged as a response to the limitations of static 3D reconstructions, enabling dynamic content creation and correction. Early work focused on global retraining, but recent methods leverage 2D generative priors (e.g., DreamFusion, Stable Diffusion) and semantic feature distillation to achieve localized edits efficiently. Key developments include the use of Instant-NGP for fast training [S1], diffusion models for pseudo-observation generation [S1], and feature field distillation for semantic decomposition [S1]. The field is characterized by a shift from explicit physical modeling to data-driven inference, integrating 2D generative knowledge into 3D geometry [S1].

## Sources
[S1] Review of Key Technologies and Application Scenarios of Neural Rendering, https://doi.org/10.54254/2755-2721/2026.36116, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
