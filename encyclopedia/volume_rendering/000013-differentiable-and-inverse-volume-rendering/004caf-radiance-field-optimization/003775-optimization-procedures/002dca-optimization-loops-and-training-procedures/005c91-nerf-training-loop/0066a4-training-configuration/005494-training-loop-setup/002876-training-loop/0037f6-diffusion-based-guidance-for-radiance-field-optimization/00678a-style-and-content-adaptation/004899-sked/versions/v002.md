# SKED

Also called: Sketch-guided NeRF Editing

## Overview
SKED is a method for editing a Neural Radiance Field (NeRF) scene using guidance from 2D sketches provided from multiple views, in addition to an input text prompt [S1, S2]. It solves the problem of localized 3D editing by confining generative edits to sketched regions while preserving the unedited parts of the original scene [S1, S2]. It is distinguished from sibling methods like [Single-Image Textual Inversion](19221-single-image-textual-inversion.md) by its explicit use of multi-view 2D sketches as spatial priors to define the edit region, rather than relying solely on text prompts or single-view masks [S1, S2].

## How it works
**Inputs and outputs:**
*   **Input:** A pre-existing NeRF scene, a set of 2D sketches corresponding to specific camera views, and a text prompt describing the desired edit [S1].
*   **Output:** An edited NeRF model where sketched regions reflect the text prompt and the rest of the scene remains unchanged [S1].

**Data structures:**
*   **NeRF:** Standard MLP or grid-based representation mapping 3D coordinates and viewing directions to color and density [S1, S2].
*   **Sketches:** Binary masks $M_v$ for each view $v$, indicating the region to be edited [S1, S2].

**Numbered steps:**
The optimization follows the shared framework of the parent family, where a frozen diffusion model provides gradient signals to update radiance field parameters [Parent](19896-style-and-content-adaptation.md). The exact procedural steps are not detailed in the sources [S1, S2]. The method iteratively updates NeRF parameters by backpropagating gradients from a composite loss, conditioning the diffusion model on the text prompt and sketch masks [S1, S2].

**Formulas:**
The method optimizes NeRF parameters by minimizing a composite loss combining a diffusion-based guidance term with two spatial priors [S1, S2]. The exact mathematical formulation of the composite loss is not given in the sources [S1, S2]. The components are:
1. **SDS Loss:** Uses Score Distillation Sampling (SDS) loss [50, 89] to steer the rendered image towards the text prompt, providing the generative signal for new content [S1, S2].
2. **Silhouette Loss:** Facilitates the addition of new objects within the sketch region by enforcing geometric consistency with the provided sketches [S2].
3. **Preservation Loss:** Maintains fidelity to the original scene outside the sketched regions, penalizing unwanted changes to unedited areas [S1, S2].

**Parameters:**
*   Weights for silhouette and preservation losses: Not given in the sources.
*   Learning rate: Not given in the sources.

**Edge cases and pitfalls:**
*   **Sketch Accuracy:** The method relies on accurate multi-view sketches to guide the edit [S1, S2].
*   **Scene Blur:** Without sufficient preservation loss, the method may introduce blur or unintended texture and color changes to the entire scene [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Effective for localized object insertion and replacement tasks on synthetic objects, such as adding hats, flowers, or glasses [S1].
*   **Limitations:** Primarily demonstrated on synthetic objects; may struggle with complex real-world scenes if preservation is not strictly enforced [S1].
*   **When to use:** When precise spatial control over the edit region is required via 2D sketches, and the target is a synthetic object [S1].
*   **When not:** When sketches are unavailable or when editing real-world scenes with complex geometry and texture is required without additional constraints [S1].

## Background
SKED was introduced as a 3D editing method guided by sketches and text prompts [S1, S2]. It builds on the SDS loss framework [50, 89] and extends it with sketch-based spatial priors [S1, S2]. The method is used primarily for zero-shot object generation and blending in existing NeRF scenes, demonstrating applicability on object insertion and replacement tasks [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
