# Instruct 3D-to-3D


## Overview
Instruct 3D-to-3D is a method for performing 3D scene editing or conversion guided by natural language instructions [S1]. It solves the problem of transforming an existing 3D scene into a new one that adheres to specific textual descriptions while maintaining structural coherence. It distinguishes itself from text-to-3D generation methods like [DreamFusion](12087-dreamfusion.md) by conditioning the optimization on a source 3D scene rather than generating geometry from scratch, and from other 3D editing methods like [SKED](12087-dreamfusion.md) by leveraging InstructPix2Pix for instruction following [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A source 3D scene (represented as a NeRF or similar implicit representation) and a text instruction [S1].
*   **Output:** An optimized target 3D scene that reflects the text instruction [S1].

**Data Structures:**
*   **Source Scene:** The initial 3D representation (e.g., NeRF parameters) [S1].
*   **Text Instruction:** A string of natural language [S1].
*   **Optimization State:** The parameters of the target 3D scene, initialized from the source scene [S1].

**Steps:**
1.  **Conditioning:** Integrate the source 3D scene as a conditional input to the optimization process [S1]. This ensures the target scene remains grounded in the geometry and appearance of the source.
2.  **Loss Calculation:** Utilize the Score Distillation Sampling (SDS) loss [89] to guide the optimization [S1]. The SDS loss is computed using a text-to-image diffusion model (specifically InstructPix2Pix, as cited in [S1]) to evaluate how well the rendered images from the current 3D scene match the text instruction.
3.  **Geometry Modification:** Integrate dynamic scaling to effectively modify scene geometry [S1]. This mechanism allows the method to adjust the scale of geometric elements during optimization to better fit the textual description [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Instruct 3D-to-3D optimization loop
# Based on [S1]

def optimize_instruct_3d_to_3d(source_nerf, text_instruction, num_iterations):
    # Initialize target scene parameters from source
    target_params = copy(source_nerf.parameters)
    
    # Load InstructPix2Pix model for SDS loss computation [S1]
    # Note: InstructPix2Pix is a text-conditioned image editing model
    ip2p_model = load_instruct_pix2pix()
    
    for step in range(num_iterations):
        # 1. Render images from current target scene
        # Assume render function returns RGB images and potentially depth/normal
        rendered_images = render(target_params, camera_poses)
        
        # 2. Compute SDS Loss using InstructPix2Pix
        # The SDS loss guides the gradient based on the text instruction
        # InstructPix2Pix takes the rendered image and the text instruction
        # to predict the edited image, and the gradient is derived from the noise prediction
        sds_loss = compute_sds_loss(
            rendered_images=rendered_images,
            text_instruction=text_instruction,
            diffusion_model=ip2p_model
        )
        
        # 3. Integrate dynamic scaling for geometry modification [S1]
        # This step modifies the geometry parameters based on the gradient
        # and applies dynamic scaling to adjust scene structure
        gradients = compute_gradients(sds_loss, target_params)
        
        # Apply dynamic scaling factor to the geometry update
        # The exact formula for dynamic scaling is not detailed in [S1]
        # [G] Assuming a standard Adam optimizer with a scaling factor applied to geometry-related gradients
        target_params = update_parameters(target_params, gradients, scaling_factor=dynamic_scale(step))
        
    return target_params
```

**Parameters:**
*   **Text Instruction:** The natural language command describing the desired 3D modification [S1].
*   **Source Scene:** The initial 3D object or scene to be edited [S1].
*   **Dynamic Scaling:** A mechanism to adjust geometry; specific values or formulas are not provided in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The quality of the output depends heavily on the capability of the InstructPix2Pix model to interpret the text instruction in the context of the source scene [S1].
*   **Boundaries:** The method relies on the source scene as a conditional input; if the source scene is significantly different from the target concept, the optimization may struggle to converge [S1].

## Subtopics

### Pix2Vex Reconstructor
A network translating images to a 3D model, based on a pix2pix convolutional and deconvolutional ResNet, outputting vertex offsets from a base model [S2, unofficial]. It predicts offsets from a base model (uniform sphere with predefined topology) rather than absolute coordinates to improve convergence smoothness [S2, unofficial]. It optionally predicts symmetry by splitting the model into four quarters, predicting shifts for one quarter, mirroring it, and predicting shifts for the mirrored quarter until all four are predicted [S2, unofficial]. It also predicts uniform translation and scaling values to reduce offset magnitude and improve stability [S2, unofficial]. Regularization on the 3D mesh considers angles between normals, edge lengths, and curvature [S2, unofficial]. Input: Images. Internal State: Single representative image (from pix2pix component). Output: $3 \times \#vertices$ values from two fully connected layers, passed through a sigmoid layer, then linearly transformed to range $[-1, 1]$ [S2, unofficial].

### FaceCLIPNeRF
A text-based 3D facial alteration technique utilizing malleable NeRF [S3]. It manages facial distortion by instructing a scene manipulator to use hidden codes and adapt to depict different latent codes for scene manipulation using a Positional Conditional Anchor Combiner (PAC) [S3]. It successfully mirrors the visual characteristics of descriptive and emotionally charged texts, preserving the visual integrity and essence of 3D faces [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Leverages powerful text-to-image models (InstructPix2Pix) for instruction following [S1].
*   **Limitations:** Relies on the quality of the source scene and the diffusion model [S1].
*   **When to use:** When editing an existing 3D scene based on text instructions is required [S1].
*   **When not to use:** When generating 3D content from scratch without a source scene is required (use [DreamFusion](12087-dreamfusion.md) instead) [S1].

## Background
Instruct 3D-to-3D was introduced as a method for 3D editing following text instructions [S1]. It utilizes InstructPix2Pix for instruction following and SDS loss for optimization [S1]. It is part of a series of 3D editing works that utilize the SDS algorithm [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] Pix2Vex: Image-to-Geometry Reconstruction using a Smooth Differentiable Renderer, https://doi.org/10.48550/arxiv.1903.11149, fetched 2026-10-26 (unofficial)
[S3] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
