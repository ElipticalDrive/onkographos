# Mask+SDS with Background Prompt

Also called: Mask+SDS+BGT

## Overview
Mask+SDS with Background Prompt is a differentiable volume rendering optimization method that uses a frozen Stable Diffusion model to edit specific regions of a scene while explicitly guiding the generation of surrounding content [S1]. It solves the problem of "black holes" (unsupervised, often black or undefined regions) that occur when masking out parts of a NeRF for editing, by appending a background description to the diffusion model's text prompt [S1]. It sets itself apart from standard Mask+SDS by providing explicit semantic guidance for the occluded areas, and from Mask+SDS+CLIP by relying on the diffusion model's generative prior for the background rather than a separate CLIP-based texture fill [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF representation; a binary mask $M$ defining the region to be edited (object); a text prompt $P_{obj}$ describing the target object; a text prompt $P_{bg}$ describing the desired background content.
*   **Output:** An optimized NeRF representation where the masked region reflects $P_{obj}$ and the previously occluded/unmasked regions reflect $P_{bg}$.

**Data Structures:**
*   **Mask ($M$):** A 2D tensor of the same resolution as the rendered image, where $M_{ij}=1$ if pixel $(i,j)$ is in the masked region, and $0$ otherwise.
*   **Text Prompts:** Strings $P_{obj}$ and $P_{bg}$.

**Steps:**
1.  **Prompt Construction:** Combine the object and background prompts into a single text prompt $P_{total}$ for the diffusion model. The standard format is concatenation:
    $$ P_{total} = P_{obj} + \text{" in "} + P_{bg} $$
    Example: If $P_{obj} = \text{"a lego man head"}$ and $P_{bg} = \text{"leaves"}$, then $P_{total} = \text{"a lego man head in leaves"}$ [S1].
2.  **Rendering:** Render the current NeRF view to obtain an image $I$.
3.  **Diffusion Guidance (SDS):**
    *   Pass the full image $I$ (without masking the input to the diffusion model) and $P_{total}$ to the frozen Stable Diffusion model [S1].
    *   Compute the SDS loss $\mathcal{L}_{SDS}$ using the predicted noise $\epsilon_\theta$ and the actual noise $\epsilon$ [S1].
    *   The loss strongly controls the unmasked part to avoid unwanted content changes outside the target region, typically via a weighting scheme or loss function (referenced as Eq. 10 in [S1], though the exact formula is not provided in the source text, the intent is to constrain the unmasked region).
4.  **Optimization:** Backpropagate the gradient from $\mathcal{L}_{SDS}$ through the NeRF rendering process to update the NeRF parameters.
5.  **Iteration:** Repeat steps 2-4 for the training loop.

**Parameters:**
*   **Mask Weight ($\lambda_{unmask}$):** A parameter controlling the strength of supervision on the unmasked region to prevent unwanted content generation. The specific value is not given, but its effect is compared in [S1].
*   **Diffusion Model:** Stable Diffusion [S1].

**Pitfalls:**
*   **Black Holes:** If only the object prompt is used (Mask+SDS without background prompt), the previously occluded or covered regions develop into "black holes" due to a lack of explicit supervision for those areas [S1].
*   **Precision:** The diffusion model receives full-resolution images without any mask applied to the input image itself, relying on the text prompt and loss weighting for spatial control [S1].

## Subtopics

### Mask+SDS+CLIP
A variant that removes the background prompt from the diffusion model's text prompt and instead uses CLIP with a background prompt to fill occluded regions [S1]. It provides a view of the unmasked region to CLIP along with a background prompt to supervise and fill the black hole [S1]. While CLIP fills the black hole with reasonable texture, the generated shapes retain artifacts from the original occluded geometry (e.g., petal shapes) [S1].

### Mask+SDS+BGT+CLIP
The final combined editing configuration that uses SDS with a full prompt (object + background) and CLIP with a background prompt on unmasked views [S1]. It sets the diffusion model text prompt to include both object and background (e.g., "object in background") and simultaneously feeds unmasked views to CLIP with the background prompt to supervise the surroundings [S1]. This produces normal shapes and fills black holes with appropriate background material (e.g., green leaves around a blue rose) [S1].

## Operating characteristics
*   **Time Complexity:** Determined by the number of SDS gradient steps per iteration and the cost of the Stable Diffusion model inference. Not given in the sources.
*   **Memory Complexity:** Requires storing the NeRF parameters and the intermediate gradients. Not given in the sources.
*   **Strengths:** Explicitly guides the generation of background content, reducing the "black hole" artifact common in masked editing [S1].
*   **Limitations:** The diffusion model receives the full image without mask input, relying on text prompts for spatial guidance, which may lead to less precise spatial control compared to mask-aware diffusion methods [S1].
*   **When to use:** When editing a specific object in a scene and ensuring the surrounding context is semantically consistent and free of black holes.
*   **When not to use:** When precise spatial control of the background is required without relying on text prompts, or when computational resources for Stable Diffusion are limited.

## Background
Originated in the paper "RePaint-NeRF: NeRF Editing via Semantic Masks and Diffusion Models" [S1]. The method builds on the concept of using diffusion models for NeRF editing [S1]. It was developed to address the limitations of standard masked SDS editing, where occluded regions become undefined [S1]. The approach draws inspiration from prior work on using CLIP for filling occluded regions [S1].

## Sources
[S1] RePaint-NeRF: NeRF Editting via Semantic Masks and Diffusion Models, https://doi.org/10.24963/ijcai.2023/201, fetched 2026-10-26
