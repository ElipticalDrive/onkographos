# Score Distillation Loss Modifications
## Overview
This family comprises modifications to the Score Distillation Sampling (SDS) objective used in diffusion-guided radiance field optimization. While standard SDS predicts noise at a single timestep to guide optimization, these modifications introduce mechanisms such as multi-step interval supervision, inpainting conditioning, background semantic prompting, and latent-space re-noising to address specific artifacts like floating objects, black holes, and low-resolution details. They set themselves apart from the parent [Diffusion-Based Guidance for Radiance Field Optimization](8117-diffusion-based-guidance-for-radiance-field-optimization.md) by altering the loss computation or the conditioning inputs to the diffusion model rather than changing the core gradient backpropagation mechanism.
## Shared principles
All methods in this family operate within the differentiable rendering loop where a radiance field $\Theta$ is optimized to minimize a loss derived from a frozen diffusion model. The core mechanism involves:
1.  Rendering an image $I$ (or latent $z_0$) from the current parameters $\Theta$.
2.  Adding noise $\epsilon$ to obtain a noisy sample $x_t$ (or $z_t$) at timestep $t$.
3.  Querying the frozen diffusion model $\epsilon_\theta$ to predict the noise $\hat{\epsilon}$.
4.  Computing a gradient signal based on the residual $\hat{\epsilon} - \epsilon$ and backpropagating it through the differentiable renderer to update $\Theta$.

The shared mathematical foundation is the SDS gradient:
$$ \nabla_\Theta L = \mathbb{E}_{t,\epsilon} \left[ \omega(t) \cdot (\hat{\epsilon}_\theta(x_t, \cdot) - \epsilon) \frac{\partial x_t}{\partial \Theta} \right] $$
where $\omega(t)$ is a weighting function dependent on the diffusion schedule. The modifications in this family alter the inputs to $\hat{\epsilon}_\theta$ (e.g., masks, background prompts), the set of timesteps $t$ considered (intervals, schedules), or the space in which the optimization occurs (latent residuals).
## Subtopics

### [Interval Score Matching](8995-interval-score-matching.md)
Interval Score Matching (ISM) modifies the standard single-step SDS by incorporating supervision from multiple interval steps of the diffusion process [8995-interval-score-matching.md]. It replaces the standard DDPM forward process with DDIM inversion to determine noise levels for these intervals, mitigating high reconstruction error inherent in single-step prediction [8995-interval-score-matching.md]. It is chosen when single-step SDS yields poor geometric consistency or high error. Link: [Interval Score Matching](8995-interval-score-matching.md).

### [Inpainting SDS Loss](11185-inpainting-sds-loss.md)
Inpainting SDS Loss (iSDS) conditions the diffusion model on a binary mask $M$ and a masked version of the rendered image $I_M$ to generate content that harmonizes with existing scene context [11185-inpainting-sds-loss.md]. Unlike vanilla SDS, which often produces floating objects, iSDS ensures proper pose and geometry integration by providing visual context to the inpainting model [11185-inpainting-sds-loss.md]. It is primarily used for inserting new objects into existing NeRFs or inpainting missing regions [11185-inpainting-sds-loss.md]. Link: [Inpainting SDS Loss](11185-inpainting-sds-loss.md).

### [Mask+SDS with Background Prompt](14382-mask-sds-with-background-prompt.md)
Mask+SDS with Background Prompt addresses the "black hole" artifact in NeRF editing by explicitly providing a background description $P_{bg}$ alongside the object prompt $P_{obj}$ [14382-mask-sds-with-background-prompt.md]. It concatenates these into a single prompt (e.g., "object in background") to guide the diffusion model's text encoder, ensuring the surrounding environment is semantically consistent [14382-mask-sds-with-background-prompt.md]. It sets itself apart from standard Mask+SDS by adding semantic context and from Mask+SDS+CLIP by relying on the diffusion model rather than a separate CLIP loss for background filling [14382-mask-sds-with-background-prompt.md]. Link: [Mask+SDS with Background Prompt](14382-mask-sds-with-background-prompt.md).

### [Renoised Score Distillation (RSD)](16424-renoised-score-distillation-rsd.md)
Renoised Score Distillation (RSD) is designed for 2D image resolution and super-resolution within iterative 3D synchronization [16424-renoised-score-distillation-rsd.md]. It optimizes learnable latent residuals $h_\theta$ by combining ancestral sampling and SDS, guiding optimization towards intermediate denoised latents $z_{t-1}$ over a linearly decreasing time schedule [16424-renoised-score-distillation-rsd.md]. Unlike standard SDS (limited details) or ancestral sampling (structural inconsistency), RSD generates sharp, high-resolution details consistent with low-resolution conditioning [16424-renoised-score-distillation-rsd.md]. Link: [Renoised Score Distillation (RSD)](16424-renoised-score-distillation-rsd.md).

### Masked-NeRF + DreamFusion
A baseline method using per-scene text prompts to guide masked region generation via SDS loss [S1]. It improves quality by gradually and uniformly decreasing the maximum noise step $t_{max}$ to the minimum noise step $t_{min}$ during fitting, leveraging high noise levels for global structure and low noise levels for detail refinement [S1].

## Background
Score Distillation Sampling (SDS) was introduced in DreamFusion [S1, implied context] as a method to optimize 3D representations using 2D diffusion priors. The modifications in this family emerged to address specific limitations of DreamFusion, such as the "slimy" appearance, floating objects, and lack of background coherence. Inpainting SDS and Mask+SDS variants were developed to enable controllable editing and object insertion in NeRFs [S1]. Renoised Score Distillation was introduced to handle super-resolution tasks where standard SDS fails to preserve high-frequency details [S1]. Interval Score Matching was proposed to reduce reconstruction error by utilizing multiple diffusion steps [S1].
## Sources
[S1] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
