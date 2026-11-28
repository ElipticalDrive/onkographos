# SKED

Also called: Sketch-guided NeRF Editing

## Overview
SKED is a method for editing a Neural Radiance Field (NeRF) scene using guidance from 2D sketches provided from multiple views, in addition to an input text prompt [S1, S2]. It addresses the problem of localized 3D editing, specifically object insertion and replacement, by ensuring that edits are confined to sketched regions while preserving the unedited parts of the original scene [S1, S2]. It is distinguished from sibling methods like [Inpainting SDS Loss](11185-inpainting-sds-loss.md) by its explicit use of multi-view 2D sketches as spatial priors to define the edit region, rather than relying solely on text prompts or single-view masks [S1, S2].

## How it works
SKED optimizes a NeRF representation by minimizing a composite loss function that combines a diffusion-based guidance term with two regularization terms: a silhouette loss and a preservation loss [S1, S2].

**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model representing the original scene; a set of 2D sketches (binary masks) corresponding to specific camera views; a text prompt describing the desired edit.
*   **Output:** An edited NeRF model where the sketched regions reflect the text prompt and the rest of the scene remains unchanged.

**Data Structures:**
*   **NeRF:** Standard MLP or grid-based representation mapping 3D coordinates and viewing directions to color and density [S1, S2].
*   **Sketches:** Binary masks $M_v$ for each view $v$, indicating the region to be edited [S1, S2].

**Loss Function:**
The total loss $L$ is defined as:
$$ L = L_{SDS} + \lambda_s L_{sil} + \lambda_p L_{pres} $$
where $\lambda_s$ and $\lambda_p$ are weighting hyperparameters [S1, S2].

1.  **SDS Loss ($L_{SDS}$):** Uses the Score Distillation Sampling (SDS) loss from [50] (or [89]) to steer the rendered image towards the text prompt [S1, S2]. This provides the generative signal for the new content.
2.  **Silhouette Loss ($L_{sil}$):** Facilitates the addition of new objects within the sketch region. It ensures that the rendered silhouette of the edited region matches the provided sketch mask $M_v$ [S2].
3.  **Preservation Loss ($L_{pres}$):** Maintains fidelity to the original scene outside the sketched regions. It penalizes changes to the rendered image in areas not covered by the sketch masks [S1, S2].

**Algorithm Steps:**
1.  Initialize the NeRF parameters with the pre-trained scene [S1, S2].
2.  For each optimization step:
    a. Select a random camera view $v$ and a random text prompt [S1, S2].
    b. Render the current NeRF from view $v$ to obtain an image $I_{render}$ [S1, S2].
    c. Compute the SDS loss $L_{SDS}$ using $I_{render}$ and the text prompt, utilizing a frozen Stable Diffusion model [S1, S2].
    d. Compute the silhouette loss $L_{sil}$ by comparing the rendered silhouette of $I_{render}$ in the sketched region with the sketch mask $M_v$ [S2].
    e. Compute the preservation loss $L_{pres}$ by comparing the rendered image $I_{render}$ with the original rendered image $I_{orig}$ in the non-sketched regions (complement of $M_v$) [S1, S2].
    f. Compute the total loss $L = L_{SDS} + \lambda_s L_{sil} + \lambda_p L_{pres}$ [S1, S2].
    g. Backpropagate the gradient of $L$ through the NeRF parameters and update them [S1, S2].

**Parameters:**
*   $\lambda_s, \lambda_p$: Weights for silhouette and preservation losses. Not given in the sources.
*   Learning rate: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Sketch Accuracy:** The method relies on accurate multi-view sketches. Inaccurate or missing sketches may lead to incomplete edits or artifacts [S1, S2].
*   **Scene Blur:** Without sufficient preservation loss, the method may introduce blur or texture changes to the entire scene, as noted in comparisons with other methods [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Effective for localized object insertion and replacement tasks on synthetic objects, such as adding hats, flowers, or glasses [S1].
*   **Limitations:** Primarily demonstrated on synthetic objects; may struggle with complex real-world scenes if preservation is not strictly enforced [S1].
*   **When to use:** When precise spatial control over the edit region is required via 2D sketches, and the target is a synthetic object [S1].
*   **When not:** When sketches are unavailable or when editing real-world scenes with complex geometry and texture is required without additional constraints [S1].

## Background
SKED was introduced in the paper "Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields" [S1, S2]. It was developed to address limitations in previous NeRF editing methods that either failed to preserve the original scene or lacked precise spatial control [S1]. The method builds on the SDS loss framework [50, 89] and extends it with sketch-based priors [S1, S2]. It is used primarily for zero-shot object generation and blending in existing NeRF scenes [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
