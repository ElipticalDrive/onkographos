# Sparse View and Few-Shot Optimization

Also called: Few-shot NeRF, Sparse View NeRF

## Overview
Sparse View and Few-Shot Optimization addresses the degradation of Neural Radiance Fields (NeRF) when trained on a drastically reduced number of input viewpoints, typically $|S| < 20$ [S1, unofficial]. In this regime, standard NeRF exhibits high divergent behaviors, including geometry breakdown, overfitting to input views, and the generation of artifacts that cloud empty space [S1, unofficial]. This family of methods mitigates these issues by introducing in-situ priors, deformable sampling strategies, or consistency regularization to stabilize training and suppress artifacts, distinguishing itself from methods that rely on external large-scale datasets or pre-trained depth estimators.

## Shared principles
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

## Subtopics

### [Simple-NeRF](7371-simple-nerf.md)
Simple-NeRF is a regularization method for sparse-input Neural Radiance Fields that mitigates overfitting, floaters, and shape-radiance ambiguity by training two simpler "augmented" NeRF models in tandem with a main NeRF. Unlike methods requiring pre-training on large datasets or external depth estimators, Simple-NeRF learns in-situ depth priors by restricting the capacity of the augmented models (via reduced positional encoding frequencies and disabled view-dependent radiance) to encourage simpler solutions. It employs a reliability mask to supervise the main NeRF only where augmented depth estimates are consistent with observed images, and enforces coarse-fine consistency to stabilize hierarchical sampling. See [Simple-NeRF](7371-simple-nerf.md).

### [WaH-NeRF](11548-wah-nerf.md)
WaH-NeRF is a training framework for Neural Radiance Fields (NeRF) from sparse inputs that mitigates rendering confusion by addressing two root causes: sample-position confusion ('WHERE to Sample?') and prediction confusion ('HOW to Predict?'). It differs from its siblings and nearest alternatives, such as [Simple-NeRF](7371-simple-nerf.md) or [RegNeRF](11995-nerf-in-the-wild-nerf-w.md), by analyzing the sample-render process and mitigating confusion before volume rendering, rather than relying on regularization for rendered color and depth images after volume rendering. It builds on [mip-NeRF](11313-model-architecture-modifications.md) as its baseline architecture. See [WaH-NeRF](11548-wah-nerf.md).

### Few-shot NeRF
#### definition
- [S1, unofficial] NeRF represents a scene as a continuous function $F_\theta(\cdot)$ implemented by a neural network with parameters $\theta$, taking 3D coordinates $\mathbf{x}$ and viewing direction $\mathbf{d}$ with positional encoding $\gamma(\cdot)$ to output color $\mathbf{c}$ and density $\sigma$.
#### parameter
- [S1, unofficial] Few-shot NeRF setting: number of input viewpoints $|S|$ is drastically reduced to a few viewpoints, specifically $|S| < 20$.

## Background
The term "Few-shot NeRF" was introduced to describe scenarios where the number of input viewpoints is drastically reduced from the standard setting (typically >100 views) to fewer than 20 views [S1, unofficial]. In this regime, NeRF shows high divergent behaviors such as geometry breakdown, overfitting to input viewpoints, and generation of artifacts that cloud the empty space between the camera and object, causing performance to drop sharply [S1, unofficial]. Existing frameworks attempted to overcome this by applying regularization techniques at unknown viewpoints to constrain NeRF with additional 3D priors, but showed limited performance [S1, unofficial]. Recent approaches like Simple-NeRF and WaH-NeRF aim to directly remove ambiguity present in monocular depth estimation (MDE) or sample-position confusion by finetuning with canonical geometry captured by NeRF [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
