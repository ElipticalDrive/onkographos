# Mask+SDS with Background Prompt

Also called: Mask+SDS+BGT

## Overview
Mask+SDS with Background Prompt is a differentiable rendering optimization method that addresses the "black hole" artifact in masked NeRF editing, where occluded regions lack supervision and collapse to empty space [S1]. It distinguishes itself from standard Masked SDS by explicitly appending a background description to the diffusion model's text prompt, guiding the generation of surrounding content while the mask restricts changes to the target object [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A radiance field $\Theta$ (NeRF parameters), a binary mask $M$ identifying the editable region, an object text prompt $P_{obj}$, and a background text prompt $P_{bg}$ [S1].
*   **Output:** Optimized radiance field $\Theta'$ [S1].

**Data Structures:**
*   **Mask $M$:** A 2D tensor of the same resolution as the rendered image, where $M_{ij}=1$ if pixel $(i,j)$ is in the editable region, and $0$ otherwise.
*   **Text Prompts:** Strings $P_{obj}$ and $P_{bg}$ [S1].

**Steps:**
1.  **Render:** Render an image $I$ from the current radiance field $\Theta$ using the current camera pose [S1].
2.  **Combine Prompt:** Construct the full text prompt $P_{full}$ by concatenating the object and background prompts. For example, if $P_{obj}$ is "a lego man head" and $P_{bg}$ is "leaves", then $P_{full}$ is "a lego man head in leaves" [S1].
3.  **Noise:** Add noise $\epsilon$ to the rendered image $I$ to obtain the noisy sample $x_t$ at timestep $t$ [S1].
4.  **Diffusion Query:** Query the frozen diffusion model $\epsilon_\theta$ with $x_t$ and $P_{full}$ to predict the noise $\hat{\epsilon}$ [S1].
5.  **Compute Loss:** Calculate the SDS loss. To prevent unwanted content generation in the unmasked region, a strong weighting is applied to the unmasked pixels. The loss is defined as:
    $$ L = \mathbb{E}_{t,\epsilon} \left[ \omega(t) \cdot (\hat{\epsilon}_\theta(x_t, P_{full}) - \epsilon) \frac{\partial x_t}{\partial \Theta} \right] $$
    where the gradient is masked such that only the unmasked region contributes significantly to the loss to avoid altering the background, or conversely, the mask is used to restrict the object deformation while the background prompt guides the surroundings [S1]. *Note: The source indicates Eq. 10 is used to strongly control the unmasked part to avoid unwanted content [S1].*
6.  **Backpropagate:** Backpropagate the gradient through the differentiable renderer to update $\Theta$ [S1].

**Parameters:**
*   **Mask Weight ($\lambda_{unmask}$):** A hyperparameter controlling the strength of the constraint on the unmasked region. The source compares different values of $\lambda_{unmask}$ in Eq. 14 [S1].
*   **Text Prompts:** $P_{obj}$ and $P_{bg}$ are user-defined [S1].

**Edge Cases and Pitfalls:**
*   **Black Holes:** Previously occluded or covered regions may develop into black holes (empty space) due to the lack of explicit supervision for those areas when only the object prompt is used or when the background prompt is insufficiently integrated [S1].

## Subtopics

### Mask+SDS+CLIP
A variant that removes the background prompt from the diffusion model's text prompt and instead uses CLIP with a background prompt to fill occluded regions [S1]. It provides a view of the unmasked region to CLIP along with a background prompt to supervise and fill the black hole [S1]. While CLIP fills the black hole with reasonable texture, the generated shapes retain artifacts from the original occluded geometry (e.g., petal shapes) [S1].

### Mask+SDS+BGT+CLIP
The final combined editing configuration that uses SDS with a full prompt (object + background) and CLIP with a background prompt on unmasked views [S1]. It sets the diffusion model text prompt to include both object and background (e.g., "object in background") and simultaneously feeds unmasked views to CLIP with the background prompt to supervise the surroundings [S1]. This produces normal shapes and fills black holes with appropriate background material (e.g., green leaves around a blue rose) [S1].

## Operating characteristics
*   **Strengths:** Explicitly guides the generation of surrounding content, resulting in more reasonable shapes for the edited object compared to using only an object prompt [S1].
*   **Limitations:** Without additional mechanisms (like CLIP), the previously covered parts of the object can become "black holes" [S1].
*   **Performance:** Not given in the sources.

## Background
Originated in the paper "RePaint-NeRF: NeRF Editing via Semantic Masks and Diffusion Models" [S1]. The method was developed to improve NeRF editing by leveraging semantic masks and diffusion models, specifically addressing the issue of generating coherent backgrounds and avoiding artifacts in masked regions [S1].

## Sources
[S1] RePaint-NeRF: NeRF Editting via Semantic Masks and Diffusion Models, https://doi.org/10.24963/ijcai.2023/201, fetched 2026-10-26
