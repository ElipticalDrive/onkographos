# Optimization and Training Procedures


## Overview
This family covers the algorithmic procedures for optimizing neural radiance fields, including training loops, hyperparameter schedules, and post-training parameter modifications. It distinguishes itself from sibling articles by focusing on the *process* of learning and editing (e.g., learning rates, optimizer choices, re-training strategies) rather than the *architecture* of the representation itself.

## Shared principles
Not given in the sources.

## Subtopics

### [Activation-Informed Coarse-to-Fine Rendering](12277-activation-informed-coarse-to-fine-rendering.md)
A method to accelerate NeRF inference by skipping the expensive fine-pass MLP evaluation for rays determined to be empty or background, based on intermediate activation values from the coarse pass [12277-activation-informed-coarse-to-fine-rendering]. It distinguishes itself from siblings like [Last Layer Re-training](13532-last-layer-re-training.md) by operating during the forward pass to prune computation rather than optimizing the model structure post-training [12277-activation-informed-coarse-to-fine-rendering].

### [Last Layer Re-training](13532-last-layer-re-training.md)
A technique for instantly propagating a color edit from a single view to an entire pre-trained Neural Radiance Field (NeRF) by selectively retraining only the last layer of the color Multi-Layer Perceptron (MLP) [13532-last-layer-re-training]. It solves the problem of efficient 3D scene editing by avoiding the computational cost and visual artifacts associated with full model retraining, while preserving existing view-dependent information such as specularities [13532-last-layer-re-training]. Unlike [Activation-Informed Coarse-to-Fine Rendering](12277-activation-informed-coarse-to-fine-rendering.md), which focuses on rendering efficiency during optimization, this method focuses on post-training parameter modification for editing [13532-last-layer-re-training].

#### Rip-NeRF Optimization
**Definition**
The optimization process for Rip-NeRF involves the tiny MLP weights and the ripmaps $\mathbf{R}$ [S1].

**Steps**
The learning rate schedule is reduced by 0.6x at steps 60K, 90K, 100K, and 108K, over a total of 120K iterations [S1]. The batch size is dynamically adjusted to maintain approximately 256K spheres per batch [S1].

**Parameters**
MLP weights are initialized following [Glorot and Bengio 2010] [S1]. Ripmaps $\mathbf{R}$ are initialized from a uniform distribution over $[-0.01, 0.01]$ [S1]. The optimizer is AdamW [S1]. The base learning rate for $\mathbf{R}$ is scaled by 10x due to its direct scene representation role [S1]. The base learning rate is $2 \times 10^{-3}$ [S1].

## Background
Not given in the sources.

## Sources
[S1] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
