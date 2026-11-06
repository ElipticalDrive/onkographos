# Deep Learning and Neural Architectures

Also called: Deep Learning-based Tomographic Reconstruction, Neural Inverse Rendering

## Overview
This family encompasses methods that replace or augment traditional algebraic regularizers with neural networks to solve the inverse problem of tomographic reconstruction. It distinguishes itself from classical iterative methods (like SIRT or ART) by leveraging the expressive power of deep learning to enforce priors, either implicitly through network architecture (Deep Image Prior) or explicitly through learned features (Transformers), often enabling reconstruction from sparse or noisy data where traditional methods fail.

## Shared principles
All methods in this family operate within the differentiable inverse rendering framework defined by [Tomographic reconstruction as inverse rendering](45-tomographic-reconstruction-as-inverse-rendering.md). They minimize the loss $\mathcal{L}(\mathcal{P}(x), p)$ where $x$ is the volume estimate parameterized by a neural network or latent variables, and $\mathcal{P}$ is the differentiable forward projector. The gradient $\frac{\partial \mathcal{L}}{\partial x}$ is computed via back-projection $\mathcal{P}^T$, allowing end-to-end optimization of network weights or latent codes using gradient-based optimizers like Adam [S1, unofficial][S2].

## Subtopics

### [Deep Image Prior Reconstruction](1221-deep-image-prior-reconstruction.md)
An iterative method that uses a neural network as an implicit regularizer. It jointly optimizes a latent variable $z$ and the network weights $\theta$ to fit the sinogram, distinguishing itself by requiring no training data, relying solely on the network's inductive bias [1221-deep-image-prior-reconstruction.md].

### [DuDoTrans](1409-dudotrans.md)
A dual-domain transformer architecture for sparse-view CT. It replaces standard regularization with Swin Transformer blocks that process spatial and frequency-domain features, distinguishing itself by handling the ill-posed nature of undersampled data through hierarchical attention [1409-dudotrans.md].

### [End-to-End Reconstruction Model Architecture](7504-end-to-end-reconstruction-model-architecture.md)
A simultaneous reconstruction and denoising architecture using two U-net instances ($g_1$ and $g_2$). It distinguishes itself by integrating reconstruction and denoising into a single differentiable pipeline [7504-end-to-end-reconstruction-model-architecture.md].

### Generator Architecture
A specific neural generator design $G(r, z)$ mapping sinogram $r$ and latent $z$ to an image. It uses Embedding Conv 2D and U-net blocks, with $z$ integrated via a 'Modified Block' [S1, unofficial].

## Background
The application of deep learning to tomographic reconstruction emerged as a response to the limitations of filtered back-projection (FBP) and algebraic methods in sparse-view and low-dose scenarios. Early approaches used supervised learning with paired data, while later works like Deep Image Prior introduced unsupervised, optimization-based approaches that leverage the network structure as a prior [S1, unofficial]. The field has since expanded to include transformer-based models (e.g., DuDoTrans) to capture long-range dependencies in projection data [1409-dudotrans.md].

## Sources
[S1] Generative Tomography Reconstruction, https://core.ac.uk/works/91148923, fetched 2026-10-31 (unofficial)
