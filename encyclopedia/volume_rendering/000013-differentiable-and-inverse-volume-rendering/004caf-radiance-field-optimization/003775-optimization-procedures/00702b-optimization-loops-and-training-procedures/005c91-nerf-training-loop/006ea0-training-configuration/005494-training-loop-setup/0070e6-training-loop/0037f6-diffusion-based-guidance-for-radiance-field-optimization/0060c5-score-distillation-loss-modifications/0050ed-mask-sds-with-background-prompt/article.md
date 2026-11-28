# Mask+SDS with Background Prompt

Also called: Mask+SDS+BGT

## Overview
Mask+SDS with Background Prompt is a score distillation loss modification for NeRF editing that explicitly conditions the diffusion model on the surrounding environment to prevent semantic inconsistencies in occluded regions [S1]. It solves the "black hole" artifact problem inherent in standard masked SDS, where previously occluded geometry collapses or generates undefined content due to lack of supervision [S1]. It distinguishes itself from siblings like [Inpainting SDS Loss](11185-inpainting-sds-loss.md) by integrating the background description directly into the diffusion model's text prompt rather than relying solely on geometric masking or external texture synthesis [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A radiance field $\Theta$ (or latent representation), a camera pose, a semantic mask $M$ identifying the region to be edited, an object text prompt $P_{obj}$, and a background text prompt $P_{bg}$ [S1].
*   **Output:** Gradients $\nabla_\Theta L$ for updating the radiance field parameters [S1].

**Data Structures:**
*   **Mask $M$:** A binary image of the same resolution as the rendered view, where $M_{ij}=1$ indicates the region to be edited (masked) and $M_{ij}=0$ indicates the background region [S1].
*   **Text Prompts:** Strings $P_{obj}$ and $P_{bg}$ [S1].

**Steps:**
1.  **Render:** Render an image $I$ from the radiance field $\Theta$ using the current camera pose [S1].
2.  **Combine Prompt:** Construct the full text prompt $P_{full}$ by concatenating the object prompt and the background prompt. For example, if $P_{obj}$ is "a lego man head" and $P_{bg}$ is "leaves", then $P_{full}$ is "a lego man head in leaves" [S1].
3.  **Diffusion Conditioning:** Pass the rendered image $I$ and the full prompt $P_{full}$ to the frozen diffusion model $\epsilon_\theta$ [S1].
4.  **Noise Prediction:** The diffusion model predicts the noise $\hat{\epsilon}$ for the perturbed image $x_t$ at timestep $t$, conditioned on $P_{full}$ [S1].
5.  **Gradient Computation:** Compute the SDS gradient based on the difference between the predicted noise and the actual noise, backpropagating through the diffusion model and the differentiable renderer [S1].
6.  **Optimization:** Update $\Theta$ using the computed gradient [S1].

**Pitfalls:**
*   **Black Holes:** If only the background prompt is added to the diffusion model without additional supervision for the unmasked regions, the previously occluded or covered areas may develop into "black holes" (undefined or black regions) due to the lack of explicit supervision for those specific pixels [S1].

## Subtopics

### Mask+SDS+CLIP
A variant that removes the background prompt from the diffusion model's text prompt and instead uses CLIP with a background prompt to fill occluded regions [S1]. It provides a view of the unmasked region to CLIP along with a background prompt to supervise and fill the black hole [S1]. While CLIP fills the black hole with reasonable texture, the generated shapes retain artifacts from the original occluded geometry (e.g., petal shapes) [S1].

### Mask+SDS+BGT+CLIP
The final combined editing configuration that uses SDS with a full prompt (object + background) and CLIP with a background prompt on unmasked views [S1]. It sets the diffusion model text prompt to include both object and background (e.g., "object in background") and simultaneously feeds unmasked views to CLIP with the background prompt to supervise the surroundings [S1]. This produces normal shapes and fills black holes with appropriate background material (e.g., green leaves around a blue rose) [S1].

## Operating characteristics
*   **Time Complexity:** Standard SDS complexity, plus the cost of CLIP inference if the combined variant is used [S1].
*   **Memory Complexity:** Standard SDS memory usage [S1].
*   **Strengths:** Explicitly guides the diffusion model to generate semantically consistent background content, reducing semantic artifacts in the edited region [S1].
*   **Limitations:** Does not fully solve the black hole problem on its own; requires additional CLIP supervision for unmasked regions to prevent undefined content in previously occluded areas [S1].
*   **When to use:** When editing a specific object in a scene and ensuring the surrounding environment remains semantically coherent is critical [S1].
*   **When not to use:** When computational resources are limited and the combined Mask+SDS+BGT+CLIP variant is too expensive, or when the background is simple and does not require explicit prompting [S1].

## Background
Originated in the RePaint-NeRF paper, which introduced NeRF editing via semantic masks and diffusion models [S1]. The method was developed to address the limitations of standard SDS in editing tasks, where occluded regions often fail to generate plausible content [S1]. It builds on the idea that providing explicit context to the diffusion model improves generation quality [S1].

## Sources
[S1] RePaint-NeRF: NeRF Editting via Semantic Masks and Diffusion Models, https://doi.org/10.24963/ijcai.2023/201, fetched 2026-10-26
