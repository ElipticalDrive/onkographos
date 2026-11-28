# SKED

Also called: Sketch-guided NeRF Editing

## Overview
SKED edits a NeRF scene using multi-view 2D sketches and a text prompt [S1, S2]. It is distinguished by its explicit use of multi-view sketches as spatial priors to confine edits, unlike text-only or single-view mask methods [S1, S2].

## How it works
**Inputs and outputs:**
*   **Input:** A pre-existing NeRF scene, a set of 2D sketches corresponding to specific camera views, and a text prompt describing the desired edit [S1, S2].
*   **Output:** An edited NeRF model where sketched regions reflect the text prompt and the rest of the scene remains unchanged [S1, S2].

**Data structures:**
*   **NeRF:** Standard MLP or grid-based representation mapping 3D coordinates and viewing directions to color and density [S1, S2].
*   **Sketches:** Binary masks $M_v$ for each view $v$, indicating the region to be edited [S1, S2].

**Numbered steps:**
1. Render an image from the current NeRF state for each sketch view [S1, S2].
2. Pass the rendered image and sketch mask through a frozen diffusion model conditioned on the text prompt to compute gradient signals [Parent](19896-style-and-content-adaptation.md), [S1, S2].
3. Compute the composite loss combining SDS guidance, silhouette priors, and preservation priors [S1, S2].
4. Backpropagate the loss to update NeRF parameters [S1, S2].
The exact procedural steps are not detailed in the sources [S1, S2].

**Formulas:**
The sources do not provide explicit mathematical formulations [S1, S2]. The composite loss is described as a combination of three terms [S1, S2]:
1. **SDS Loss:** Score Distillation Sampling loss [50, 89] steers the rendered image toward the text prompt [S1, S2].
2. **Silhouette Loss:** Enforces geometric consistency with the sketch masks to facilitate object addition within the sketched region [S2].
3. **Preservation Loss:** Penalizes changes outside the sketched regions to maintain fidelity to the original scene [S1, S2].

**Parameters:**
*   Weights for silhouette and preservation losses: Not given in the sources [S1, S2].
*   Learning rate: Not given in the sources [S1, S2].

**Edge cases and pitfalls:**
*   **Sketch Accuracy:** The method relies on accurate multi-view sketches to guide the edit [S1, S2].
*   **Scene Blur:** Without sufficient preservation loss, the method may introduce blur or unintended texture and color changes to the entire scene [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources [S1, S2].
*   **Measured Performance:** Not given in the sources [S1, S2].
*   **Strengths:** Effective for localized object insertion and replacement tasks on synthetic objects, such as adding hats, flowers, or glasses [S1].
*   **Limitations:** Primarily demonstrated on synthetic objects; may struggle with complex real-world scenes if preservation is not strictly enforced [S1].
*   **When to use:** When precise spatial control over the edit region is required via 2D sketches, and the target is a synthetic object [S1].
*   **When not:** When sketches are unavailable or when editing real-world scenes with complex geometry and texture is required without additional constraints [S1].

## Background
SKED was introduced as a 3D editing method guided by sketches and text prompts [S1, S2]. It builds on the SDS loss framework [50, 89] and extends it with sketch-based spatial priors [S1, S2]. The method is used primarily for zero-shot object generation and blending in existing NeRF scenes, demonstrating applicability on object insertion and replacement tasks [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
