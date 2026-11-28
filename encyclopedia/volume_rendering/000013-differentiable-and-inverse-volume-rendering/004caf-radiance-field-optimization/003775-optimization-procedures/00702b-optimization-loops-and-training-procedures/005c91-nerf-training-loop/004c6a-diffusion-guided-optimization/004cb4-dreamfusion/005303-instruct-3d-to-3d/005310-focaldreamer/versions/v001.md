# Focaldreamer

Also called: GSEditor-DDS, GSEditor-iN2N

## Overview
Focaldreamer is a 3D Gaussian Splatting (3DGS) editing method that performs localized geometric and textural modifications guided by text prompts. It solves the problem of precise, focal-region editing in existing 3D scenes without degrading the background. It distinguishes itself from sibling methods like [I3DS](16419-i3ds.md) by explicitly utilizing rendered normal maps as shape encodings for the text-to-image (T2I) model and employing a pixelwise discriminative mask to unify base and editable textures, thereby ensuring geometric consistency during optimization.

## How it works
**Inputs:**
*   An existing 3D Gaussian Splatting scene (base shape).
*   A text prompt describing the desired edit.
*   A mask defining the focal region for editing.

**Outputs:**
*   An edited 3D Gaussian Splatting scene with modified geometry and textures in the focal region.

**Data Structures:**
*   **3DGS Scene:** Standard 3DGS representation containing Gaussian primitives with position, covariance, opacity, spherical harmonics coefficients, and scale.
*   **Editable Ellipsoids:** A set of Gaussian primitives initialized in the focal regions to allow for geometric deformation.
*   **Pixelwise Discriminative Mask:** A binary or soft mask $M$ of image resolution, where $M=1$ indicates the focal region and $M=0$ indicates the background.

**Steps:**
1.  **Initialization:** Initiate the editing process with the base 3DGS shape. Initialize editable ellipsoids (Gaussian primitives) specifically within the focal regions defined by the user or prompt.
2.  **Rendering:** Render the scene from the current camera viewpoint. This rendering includes both the base shape textures and the editable textures of the ellipsoids. The final pixel color $C$ is computed by compositing these layers, weighted by the discriminative mask $M$.
3.  **Normal Map Generation:** Render the normal maps of the current 3DGS scene. These normal maps serve as the shape encoding for the T2I model, providing geometric constraints to the diffusion prior.
4.  **Loss Computation:**
    *   **SDS Loss:** Compute the Score Distillation Sampling (SDS) loss [89] using the T2I model. The input to the T2I model is the rendered image (or a latent representation thereof) combined with the normal maps as conditioning. The loss guides the optimization towards the text prompt semantics.
    *   **Regularization:** Incorporate additional regularization terms to improve editing quality. These typically include constraints on Gaussian opacity, scale, and position to prevent artifacts and maintain structural integrity.
5.  **Optimization:** Update the parameters of the editable ellipsoids (position, scale, rotation, opacity, SH coefficients) using gradient descent on the combined loss. The base shape parameters are typically frozen or heavily regularized to preserve background integrity.
6.  **Iteration:** Repeat steps 2-5 for a fixed number of iterations or until convergence.

**Formulas:**
*   **SDS Loss:** $\mathcal{L}_{SDS} = \mathbb{E}_{t, \epsilon} \left[ w(t) (\epsilon_\theta(z_t; y, t) - \epsilon) \right]$, where $z_t$ is the noisy latent, $y$ includes the rendered image and normal maps, and $\epsilon_\theta$ is the denoiser.
*   **Composite Image:** $I = M \cdot I_{editable} + (1 - M) \cdot I_{base}$.

**Parameters:**
*   Learning rate: Not given in the sources.
*   Number of iterations: Not given in the sources.
*   SDS weight $w(t)$: Standard cosine schedule or constant, as per [89].

**Edge Cases and Pitfalls:**
*   **Precision:** Normal map rendering must be accurate to provide meaningful shape encoding.
*   **Boundaries:** The discriminative mask $M$ must be smooth or carefully handled to avoid visible seams between edited and base regions.
*   **Degenerate Input:** If the focal region is too small, the edit may not be perceptible; if too large, it may overwrite significant parts of the scene.

## Subtopics

### RealmDreamer
RealmDreamer is a variant that utilizes diffusion priors from inpainting and depth estimation models to generate different viewpoints of a scene iteratively. Unlike Focaldreamer which focuses on editing a single scene with a mask, RealmDreamer uses these priors to synthesize consistent multi-view data for scene composition or generation. It is chosen when iterative viewpoint generation is required for scene-level consistency rather than localized editing. See [RealmDreamer](realmdreamer.md) for details.

## Operating characteristics
*   **Time Complexity:** Depends on the number of optimization iterations and the cost of rendering normal maps and images per iteration. Typically comparable to other 3DGS editing methods using SDS.
*   **Memory Complexity:** Standard 3DGS memory plus overhead for storing normal maps and intermediate gradients.
*   **Strengths:** Precise focal editing; preserves background integrity via discriminative masking; leverages normal maps for better geometric control.
*   **Limitations:** Requires careful mask definition; quality depends on the T2I model's ability to interpret normal maps; may introduce artifacts if regularization is insufficient.
*   **When to use:** When localized, text-guided editing of a specific region in a 3D scene is required.
*   **When not to use:** When global scene generation or editing is needed; when normal map rendering is not feasible.

## Background
Focaldreamer, also known as GSEditor-DDS and GSEditor-iN2N, is part of the latest series of methods [21,24-26, 156] that employ SDS loss within 3DGS to facilitate creative editing. It builds on the idea of using 3DGS for precise and flexible modifications. The method was introduced in the context of 3D editing surveys, highlighting the use of text prompts and discriminative masks for localized changes. It is related to other editing methods like TIP-Editor and CustomNeRF, but distinguishes itself by its specific use of normal maps and ellipsoids.

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
