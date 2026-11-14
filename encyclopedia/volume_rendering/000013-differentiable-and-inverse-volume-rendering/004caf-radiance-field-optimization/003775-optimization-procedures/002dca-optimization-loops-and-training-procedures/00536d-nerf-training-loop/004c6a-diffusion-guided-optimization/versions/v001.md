# Diffusion-guided Optimization

Also called: Diffusion-guided NeRF, Score Distillation Sampling (SDS) based optimization, Diffusion Prior Regularization

## Overview
Diffusion-guided optimization is a family of methods that train Neural Radiance Fields (NeRFs) by using a pretrained 2D diffusion model as a perceptual prior or loss function, rather than relying solely on ground-truth pixel comparisons. This approach solves the problem of underconstrained optimization in scenarios with sparse input views or the complete absence of 3D training data, enabling text-to-3D generation and robust few-view reconstruction. It differs from standard NeRF training by replacing the photometric loss with a gradient derived from the diffusion model's score function, allowing the synthesis of novel 3D objects from text prompts [S2] or the regularization of geometry in sparse settings [S1].

## Shared principles
All methods in this family share the core mechanism of backpropagating gradients from a 2D diffusion model through the differentiable NeRF rendering process. The optimization objective minimizes a loss $L$ that combines rendering fidelity with diffusion consistency:
$$ L = \lambda_{\text{rend}} L_{\text{rend}} + \lambda_{\text{diff}} L_{\text{diff}} $$
where $L_{\text{diff}}$ is typically an approximation of the diffusion model's negative log-likelihood (NLL) or score distillation loss [S1, S2]. The diffusion model acts as a regularizer, penalizing renderings that are inconsistent with the learned 2D data distribution. The specific conditioning of the diffusion model varies: it may be conditioned on text prompts [S2], observed input images and poses [S1], or both. The optimization loop iteratively updates NeRF parameters (or scene codes) using an optimizer like Adam, leveraging the gradient of the diffusion loss with respect to the NeRF output [S1, S2].

## Subtopics

### [DreamFusion](12087-dreamfusion.md)
A text-to-3D method that optimizes a NeRF to maximize the score of rendered images under a text-conditioned diffusion model. It differs from siblings by requiring no 3D data and using text prompts for view control, enabling synthesis of novel objects.

### [Single-Stage Diffusion NeRF Training](12803-single-stage-diffusion-nerf-training.md)
A method that jointly optimizes scene codes, a diffusion prior, and a NeRF decoder in a single loop. It differs by preventing bias in scene codes that occurs in two-stage methods, using empirical weighting to balance rendering and diffusion losses.

### [ReconFusion](13211-reconfusion.md)
A few-view reconstruction method that regularizes NeRF optimization with a diffusion prior conditioned on input images. It differs from DreamFusion by using image-conditioned diffusion to preserve appearance of well-sampled regions and outperforms SDS in sparse settings.

## Background
The concept originated with DreamFusion [S2], which introduced Score Distillation Sampling (SDS) for text-to-3D generation, demonstrating that 2D diffusion models could guide 3D optimization without 3D data. Subsequent work, such as ReconFusion [S1], adapted this for few-view reconstruction by conditioning the diffusion prior on observed views to mitigate artifacts like "fog" and "floater" artifacts common in sparse NeRF training. Single-stage methods [S1] further refined the approach by integrating the diffusion prior directly into the training loop to avoid biases inherent in decoupled optimization stages. These methods are primarily used in novel view synthesis, 3D generation from text, and robust reconstruction from sparse inputs.

## Sources
[S1] Single-Stage Diffusion NeRF Training, ReconFusion, and related notes.
[S2] DreamFusion and Score Distillation Sampling notes.
