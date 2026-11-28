# Instruct 3D-to-3D
Also called: Instruct 3D-to-3D conversion
## Overview
Instruct 3D-to-3D is a method for performing 3D-to-3D conversions guided by text instructions [S1]. It solves the problem of editing an existing 3D scene according to natural language prompts, building on [DreamFusion](12087-dreamfusion.md) by conditioning the optimization on a source scene rather than generating from noise [S1]. It sets itself apart by explicitly integrating the source 3D scene as a conditional input and employing dynamic scaling to modify geometry, whereas standard text-to-3D methods lack this conditional grounding [S1].
## How it works
**Inputs and Outputs:**
*   Input: A source 3D scene representation (e.g., NeRF or 3DGS), a text instruction prompt, and a pre-trained text-to-image diffusion model (specifically InstructPix2Pix) [S1].
*   Output: An optimized target 3D scene representation that satisfies the text instruction while preserving the structural context of the source [S1].

**Data Structures:**
*   Source Scene: The existing 3D representation (fields and layout depend on the underlying representation, e.g., NeRF MLP weights or 3DGS parameters) [S1].
*   Target Scene: The 3D representation being optimized [S1].
*   Diffusion Model: InstructPix2Pix, conditioned on the source scene and text prompt [S1].

**Steps:**
1.  **Conditional Integration:** The source 3D scene is integrated as a conditional input to the diffusion model [S1]. This ensures that the editing process is grounded in the existing geometry and appearance.
2.  **Optimization Loop:** The target scene is optimized using the Score Distillation Sampling (SDS) loss [S1]. The SDS loss is computed by rendering images from the target scene and passing them through the diffusion model along with the source condition and text prompt [S1].
3.  **Dynamic Scaling:** Dynamic scaling is applied to effectively modify the scene geometry during optimization [S1]. This allows the method to adjust the scale of edited regions to match the semantic changes requested by the text instruction.

**Formulas:**
*   SDS Loss: The method utilizes the SDS loss [89] as defined in [S1]. The exact formula is not provided in the source text, but it is standardly defined as the gradient of the diffusion model's score function with respect to the rendered image, scaled by a noise prediction term [S1].
*   Dynamic Scaling: The specific formula for dynamic scaling is not provided in the source text [S1].

**Parameters:**
*   Text Instruction: The natural language prompt guiding the edit [S1].
*   Source Scene: The 3D object to be edited [S1].
*   Diffusion Model: InstructPix2Pix [S1].

**Edge Cases and Pitfalls:**
*   Not given in the sources.

## Subtopics

### Focaldreamer
Focaldreamer is a variant that initiates with a base shape and editable ellipsoids in focal regions [S1]. It utilizes rendered normal maps as shape encoding for the text-to-image (T2I) models to guide geometry optimization [S1]. The shape is rendered with both base and editable textures, unified by a pixelwise discriminative mask [S1]. In addition to the SDS loss [89], it incorporates several regularizations to improve editing quality [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   Not given in the sources.

**Measured Performance:**
*   Not given in the sources.

**Strengths and Limitations:**
*   Strengths: Enables 3D-to-3D conversion following text instructions; integrates source scene as conditional input; uses dynamic scaling for geometry modification [S1].
*   Limitations: Not given in the sources.

**When to Use:**
*   Use when you need to edit an existing 3D scene based on text instructions [S1].
*   Do not use when you need to generate 3D content from scratch without a source scene (use [DreamFusion](12087-dreamfusion.md) instead) [S1].

## Background
Instruct 3D-to-3D was introduced as a method for realizing 3D-to-3D conversions following text instructions [S1]. It utilizes InstructPix2Pix for the text-to-image guidance [S1]. The method is part of a series of 3D editing works that utilize the SDS algorithm [89] to optimize NeRF and 3DGS representations [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
