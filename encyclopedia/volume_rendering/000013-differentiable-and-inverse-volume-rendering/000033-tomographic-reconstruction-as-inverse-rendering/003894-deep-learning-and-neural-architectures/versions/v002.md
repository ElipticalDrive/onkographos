# Deep Learning and Neural Architectures

Also called: Deep Learning-based Tomographic Reconstruction, Neural Inverse Rendering

## Overview
This family encompasses methods that utilize neural networks to solve the inverse problem of tomographic reconstruction, either by serving as implicit regularizers, direct mapping functions, or feature extractors within a differentiable rendering loop. Unlike traditional algebraic or analytical methods, these approaches leverage the expressive power of deep learning to handle sparse, noisy, or incomplete projection data. They distinguish themselves from sibling methods like [Neural Adaptive Tomography (NeAT)](1862-neural-adaptive-tomography-neat.md) by focusing on end-to-end differentiable architectures or implicit priors rather than explicit adaptive sampling or specific Gaussian parameterizations.

## Shared principles
All methods in this family rely on the differentiability of the forward projection operator $\mathcal{P}$ and its adjoint (back-projection) $\mathcal{P}^T$ [G]. The core optimization problem is generally formulated as minimizing a loss function $\mathcal{L}$ between the measured sinogram $p$ and the rendered projection $\hat{p} = \mathcal{P}(x)$ of the current volume estimate $x$:

$$
\min_x \mathcal{L}(\mathcal{P}(x), p) + \lambda R(x)
$$

where $R(x)$ is a regularizer (e.g., Total Variation, neural prior) and $\lambda$ is a weighting factor [S1, unofficial][S2].

The gradient of the loss with respect to the volume $x$ is computed via the chain rule:

$$
\frac{\partial \mathcal{L}}{\partial x} = \mathcal{P}^T \left( \frac{\partial \mathcal{L}}{\partial \hat{p}} \right)
$$

The term $\frac{\partial \mathcal{L}}{\partial \hat{p}}$ is the residual gradient in projection space (e.g., $p - \hat{p}$ for MSE), and $\mathcal{P}^T$ is the back-projection operator [G].

Key shared components include:
1.  **Forward Projection:** Ray marching through the volume representation (voxels, octrees, or neural fields) to compute line integrals of density/absorption coefficients [S1, unofficial].
2.  **Backward Projection (Adjoint):** Distributing the error from the 2D detector plane back into the 3D volume. This is mathematically the transpose of the forward operator, ensuring stable convergence in iterative methods [G].
3.  **Optimization Loop:** An iterative process (e.g., SGD, Adam) that updates the volume parameters (voxel values, neural network weights, or Gaussian parameters) to reduce the projection error [S1, unofficial][S2].

## Subtopics

### [Deep Image Prior Reconstruction](1221-deep-image-prior-reconstruction.md)
An iterative inverse rendering method that reconstructs a volume or image from sparse or noisy projection data by using a neural network as an implicit regularizer. It distinguishes itself from traditional iterative algebraic methods by jointly optimizing a latent variable for data consistency and the network weights to fit the sinogram.

### [DuDoTrans](1409-dudotrans.md)
A dual-domain transformer architecture designed for sparse-view CT reconstruction, operating within the differentiable inverse rendering framework [Deep Learning and Neural Architectures](8282-deep-learning-and-neural-architectures.md). It addresses the ill-posed nature of sparse-view tomography by leveraging Swin Transformer blocks with window-based multi-head self-attention (W-MSA) and shifted windows to capture long-range dependencies while maintaining computational efficiency. Unlike standard Multi-Head Self-Attention which computes attention over the entire image ($O(N^2)$ complexity), DuDoTrans restricts attention to local windows, reducing complexity to $O(N \cdot W_s^2)$ where $N$ is the number of tokens and $W_s$ is the window size.

### [End-to-End Reconstruction Model Architecture](7504-end-to-end-reconstruction-model-architecture.md)
This method is an end-to-end differentiable architecture for tomographic reconstruction that directly maps a noisy sinogram into a denoised reconstruction. It combines reconstruction and denoising in a single forward pass to mitigate noise amplification caused by the ill-conditioning of the Radon transform. Unlike traditional Filtered Back-Projection (FBP) which uses fixed linear filtration, or iterative methods that separate reconstruction and regularization, this model uses two trainable neural networks connected by a differentiable Radon back-projection operator. It builds on [Deep Learning and Neural Architectures](8282-deep-learning-and-neural-architectures.md) by implementing the shared principle of end-to-end optimization via back-projection gradients.

### Generator Architecture
#### definition
- [S1, unofficial] A generator network $G(r, z)$ that maps a sinogram $r$ and a latent vector $z$ to a reconstructed image, used for optimization in the inverse problem.
#### layout
- [S1, unofficial] Input: Sinogram $r$ and latent vector $z$. Processing: $r$ passes through Embedding Conv 2D and U-net Blocks. $z$ is processed by a 'Modified Block' (described in Section 2.2 of source, not detailed here) and integrated into the network.

### Depth-Encoded Ray Transformer (DERT)
#### definition
- [S2] A transformer architecture that infers volume density by incorporating relative depth differences as position encoding, rather than absolute positions.
#### step
- [S2] 1. Sample M points p_m on ray r with depths t_m. 2. Calculate relative depth encoding E_m for each point using δ_m. 3. Element-wise add E_m to point feature f_m. 4. Pass through self-attention and feed-forward network to predict volume density σ_m.
#### parameter
- [S2] K: Length of the position encoding feature vector.

### MRI-based Acoustic Impedance Estimation
#### definition
- [S3] A supervised learning approach to map MRI intensities to acoustic impedance values Z(x, y, z) using a multi-layer perceptron (MLP) fitted to paired data of MRI intensities and reference impedance values from literature.
#### parameter
- [S3] Reference impedance values: Literature values [4,5,6,14] used for supervised training.
#### pitfall
- [S3] MRI intensities do not correspond directly to physical quantities like density or impedance, making impedance reconstruction from MRI fundamentally ill-posed.

## Background
The application of deep learning to tomographic reconstruction has evolved from post-processing denoising to end-to-end differentiable inverse rendering. Early works focused on using CNNs to clean FBP reconstructions, while recent approaches integrate neural networks directly into the forward model to learn complex priors or implicit representations. The family includes architectures like U-Nets for direct mapping [S1, unofficial], Transformers for capturing long-range dependencies in sparse views [S1, unofficial], and specialized encoders for specific physical modalities like ultrasound [S3] or NeRFs with depth perception [S2].

## Sources
[S1] Generative Tomography Reconstruction, https://core.ac.uk/works/91148923, fetched 2026-10-31 (unofficial)
[S2] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S3] DiffUS: Differentiable Ultrasound Rendering from Volumetric Imaging., https://doi.org/10.1007/978-3-032-06329-8_3, fetched 2026-10-26
