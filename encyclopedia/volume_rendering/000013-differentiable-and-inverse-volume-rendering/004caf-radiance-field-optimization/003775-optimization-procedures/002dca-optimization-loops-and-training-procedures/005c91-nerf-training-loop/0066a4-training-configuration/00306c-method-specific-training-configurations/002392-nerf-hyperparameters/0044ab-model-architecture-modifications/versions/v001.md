# Model Architecture Modifications


## Overview
Model architecture modifications in NeRF training refer to structural changes in the neural network components or the introduction of auxiliary networks to address specific rendering artifacts, such as overfitting in sparse views or the lack of depth-of-field (DoF) modeling. These modifications differ from hyperparameter tuning or regularization by altering the computational graph or the set of learnable parameters to better represent the underlying scene physics or geometry.

## Shared principles
The family of model architecture modifications generally involves extending the standard NeRF MLP structure. Common mechanisms include:
1.  **Auxiliary Networks:** Training additional neural networks (e.g., smoothing or Lambertian models) alongside the main radiance field to provide inductive biases or consistency constraints [S1].
2.  **Learnable Physical Parameters:** Introducing learnable parameters that represent physical camera properties (e.g., aperture, focus distance) to enable differentiable optical effects [S1].
3.  **Input/Output Modifications:** Altering the input encoding (e.g., reduced frequency bands for specific augmentations) or output structures (e.g., separate depth and color heads) to enforce specific priors [S1].

## Subtopics

### [Simple-NeRF](7371-simple-nerf.md)
Simple-NeRF is a regularization method for sparse input views that trains two augmented NeRF models (Smoothing and Lambertian) in tandem with a main NeRF to reduce overfitting and distortions, using depth supervision and consistency losses [7371-simple-nerf]. It differs from alternatives like RegNeRF, which uses a depth smoothness prior and fuses objects at different depths, by mitigating both floaters and incorrect depth issues through in-situ depth priors [7371-simple-nerf].

### [DoF-NeRF](9432-dof-nerf.md)
DoF-NeRF is a neural rendering framework that extends Neural Radiance Fields (NeRF) to simulate the depth-of-field (DoF) effect of a physical lens aperture, allowing it to operate on views with different focus configurations and manipulate DoF effects by adjusting virtual aperture and focus parameters [9432-dof-nerf]. Unlike vanilla NeRF, which assumes a pinhole camera model and all-in-focus inputs, DoF-NeRF explicitly models the aperture and focus distance, enabling the synthesis of clear novel views from shallow DoF inputs and the generation of all-in-focus scenes from blurred inputs [9432-dof-nerf]. It achieves this by combining optical models with classical volume rendering, specifically incorporating the Circle of Confusion (CoC) to approximate DoF under arbitrary aperture size and focus distance [9432-dof-nerf].

### Rip-NeRF Optimization
#### definition
- [S1] The optimization process for Rip-NeRF, involving the tiny MLP weights and the ripmaps R.
#### step
- [S1] Learning rate schedule: Reduced by 0.6x at steps 60K, 90K, 100K, and 108K, over a total of 120K iterations.
- [S1] Batch size adjustment: Dynamically adjusted to maintain approximately 256K spheres per batch.
#### parameter
- [S1] MLP weights initialization: Glorot and Bengio (2010).
- [S1] Ripmap R initialization: Uniform distribution over [-0.01, 0.01].
- [S1] Optimizer: AdamW.
- [S1] Learning rate scaling: Base learning rate for R is scaled by 10x due to its direct scene representation role.
- [S1] Base learning rate: 2 × 10^-3.

## Background
The concept of modifying NeRF architecture to improve training stability and physical accuracy has evolved from simple MLP adjustments to complex multi-network systems. Simple-NeRF was introduced to address the "floaters" and depth ambiguity issues prevalent in sparse-view NeRFs by leveraging auxiliary models [7371-simple-nerf]. DoF-NeRF addressed the limitation of the pinhole camera assumption by integrating optical blur models directly into the rendering pipeline [9432-dof-nerf]. Rip-NeRF, as described in [S1], utilizes ripmaps and tiny MLPs for anti-aliasing, requiring specific optimization configurations for its unique parameters.

## Sources
[S1] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
