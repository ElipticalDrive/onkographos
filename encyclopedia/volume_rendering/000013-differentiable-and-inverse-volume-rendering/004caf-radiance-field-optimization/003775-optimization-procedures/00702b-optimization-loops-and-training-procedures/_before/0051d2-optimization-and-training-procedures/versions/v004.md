# Optimization and Training Procedures


## Overview
This family covers the algorithmic procedures for optimizing Neural Radiance Fields (NeRF) and related representations, ranging from standard training loops to specialized acceleration and editing techniques. It addresses the problem of efficiently minimizing the photometric loss between rendered and ground-truth images while managing computational constraints. Unlike standard gradient descent on the full network, methods in this family often employ structural modifications to the optimization scope, such as skipping expensive fine-pass evaluations for empty rays or restricting updates to specific network layers to preserve pre-trained geometric features.

## Shared principles
The optimization of neural radiance fields generally involves minimizing a photometric loss function, typically Mean Squared Error (MSE), between the volume-rendered color $\hat{C}$ and the ground-truth image color $C$. The optimization targets the parameters $\theta$ of the implicit representation (e.g., MLP weights, grid features). The standard training loop involves:
1.  Sampling rays from camera poses.
2.  Evaluating the representation at sample points along rays.
3.  Volume rendering the output (density and color) to produce pixel colors.
4.  Computing the loss and backpropagating gradients to update parameters.

## Subtopics

### [Activation-Informed Coarse-to-Fine Rendering](12277-activation-informed-coarse-to-fine-rendering.md)
A rendering acceleration technique that skips the computationally expensive fine-pass evaluation for rays determined to be empty or background. It leverages intermediate activation values from the coarse pass to identify such rays, using a threshold derived from the mean activation across the image. Unlike standard rendering which evaluates both coarse and fine networks for all pixels, this approach selectively bypasses the fine network, trading a negligible amount of quality for significant speedup in scenes with transparent backgrounds.

### [Last Layer Re-training](13532-last-layer-re-training.md)
A technique for instantly propagating a color edit from a single user-provided view to an entire pre-trained Neural Radiance Field (NeRF) by selectively retraining only the final layer of the color Multi-Layer Perceptron (MLP). It addresses the computational cost and view-dependent inconsistency issues of full model retraining, preserving existing geometric and view-dependent information while enabling near real-time editing. Unlike full retraining, which risks altering useful pre-trained features, this method restricts parameter updates to the last layer, ensuring rapid convergence and higher-quality edits.

### Rip-NeRF Optimization
The optimization process for Rip-NeRF involves tuning the tiny MLP weights and the ripmaps $R$. Model weights are initialized following Glorot and Bengio (2010), while ripmaps $R$ start from a uniform distribution over $[-0.01, 0.01]$ [S1]. The AdamW optimizer is used with a base learning rate of $2 \times 10^{-3}$, scaled by 10x for $R$ due to its direct scene representation role [S1]. The learning rate schedule reduces the rate by 0.6x at steps 60K, 90K, 100K, and 108K, over a total of 120K iterations [S1]. The batch size is dynamically adjusted to maintain approximately 256K spheres per batch [S1].

## Background
The standard NeRF architecture [S6, unofficial] consists of two Multi-Layer Perceptrons (MLPs): a coarse network and a fine network. The coarse network takes positional encoded position $\gamma(\mathbf{x})$ and outputs a latent feature vector $\mathbf{h}$ and volume density $\sigma$. The fine network takes $\mathbf{h}$, positional encoded position $\gamma(\mathbf{x})$, and positional encoded view direction $\gamma(\mathbf{d})$ to output color $\mathbf{c}$.

$$
\gamma(\mathbf{x}, d_1, d_2) = [\mathbf{x}, \sin(2^{d_1}\pi\mathbf{x}), \cos(2^{d_1}\pi\mathbf{x}), \dots, \sin(2^{d_2-1}\pi\mathbf{x}), \cos(2^{d_2-1}\pi\mathbf{x})]
$$

Modifications generally fall into three categories:
1.  **Input Expansion:** Adding new conditioning vectors (e.g., appearance embeddings $\mathbf{z}_i$, fluid volume $v$, or previous stage RGB) to the input concatenation.
2.  **Output Expansion:** Adding auxiliary heads to predict values other than $\sigma$ and $\mathbf{c}$, such as semantic labels $y_{aux}$, object codes $o$, or area encoding probabilities $p$. These are rendered using the same transmittance weights $T_i$ and opacity $\alpha_i$ as color:
    $$
    \hat{Y}(r) = \sum_{i=1}^{N} T_i \alpha_i y_{aux}(x_i)
    $$
3.  **Representation Substitution:** Replacing the continuous MLP function with discrete structures (voxel grids, hash tables) or different implicit surfaces (SDFs).

## Sources
[S1] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
