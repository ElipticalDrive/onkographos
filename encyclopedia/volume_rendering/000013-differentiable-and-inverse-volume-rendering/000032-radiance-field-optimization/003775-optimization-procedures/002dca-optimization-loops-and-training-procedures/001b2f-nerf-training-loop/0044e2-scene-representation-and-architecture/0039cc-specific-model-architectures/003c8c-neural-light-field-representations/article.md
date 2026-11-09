# Neural Light Field Representations

Also called: NeLF

## Overview
Neural Light Field (NeLF) representations replace the volumetric ray-marching paradigm of Neural Radiance Fields (NeRF) with a direct mapping from camera ray parameters to pixel color, enabling orders-of-magnitude faster inference by eliminating iterative sampling [S1]. Unlike sibling architectures that model density and view-dependent color via 5D fields, NeLF distills a pre-trained NeRF teacher into a deep residual student network that predicts RGB values directly from a 4D ray representation, bypassing the need for volume integration [S1].

## Shared principles
The family operates on the principle of **Light Field Distillation**, where a computationally expensive volumetric renderer (the teacher) is replaced by a fast, feed-forward neural network (the student) that approximates the final pixel color.

**Input Representation:**
The student network $G_\phi$ does not take a single 3D point and direction. Instead, it takes a ray parameterized by origin $\mathbf{o} \in \mathbb{R}^3$ and normalized direction $\mathbf{d} \in \mathbb{R}^3$. To capture local geometry and occlusion cues without explicit ray marching, the input is expanded by sampling $K$ points along the ray segment. These $K$ 3D coordinates are concatenated to form a $3K$-dimensional input vector [S1].

**Network Architecture:**
The student is a deep Multi-Layer Perceptron (MLP) with extensive residual connections. The depth is typically 88 layers, with widths of 256 or 181 units [S1]. Residual blocks are critical for training stability in such deep networks [S1].

**Training Objective:**
The network is trained using Mean Squared Error (MSE) loss between the student's output and the teacher's rendered RGB color. The teacher provides pseudo-labels by rendering rays sampled from the scene's bounding box [S1].

## Subtopics

### [Neural Light Field (NeLF)](9257-neural-light-field-nelf.md)
The primary implementation using R2L (Ray-to-Light) distillation. It uses a 16-point ray representation and hard example mining to handle the complexity of direct color prediction. Choose this when inference speed is critical and a pre-trained NeRF teacher is available.

## Background
The method originates from the work on Neural Light Fields via Ray-to-Light (R2L) distillation, which addresses the high computational cost of volumetric rendering in NeRF [S1]. It was developed to enable real-time novel view synthesis by shifting the computational burden from inference (ray marching) to training (distillation) [S1].

## Sources
[S1] Neural Light Field (NeLF) via R2L Distillation, "Neural Light Fields for Real-Time Novel View Synthesis", 2021.
