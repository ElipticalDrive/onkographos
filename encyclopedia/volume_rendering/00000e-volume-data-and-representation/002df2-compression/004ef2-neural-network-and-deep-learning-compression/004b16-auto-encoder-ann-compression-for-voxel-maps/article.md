# Auto-Encoder (ANN) Compression for Voxel Maps

Also called: AE Compression, Neural Network Compression

## Overview
Auto-Encoder (ANN) compression for voxel maps uses a non-linear neural network to learn a compressed latent representation of voxel data, aiming to reduce storage and transmission costs while preserving structural integrity for downstream tasks like mapping or pose estimation [S1]. Unlike linear methods such as PCA, this approach models complex, non-linear correlations within the voxel space, though empirical results indicate it often yields higher reconstruction error (MSE) than PCA for the same latent dimension, while potentially offering superior performance in metric tasks like Absolute Trajectory Error (ATE) [S1].

## How it works
**Inputs and Outputs:**
- Input: A batch of voxel map blocks or descriptors (e.g., TSDF values or feature vectors).
- Output: A compressed latent code (dimension $d$) and, upon decoding, a reconstructed voxel block.

**Data Structures:**
The network is a fully connected feed-forward autoencoder.
- **Encoder:** Maps input dimension $N$ to latent dimension $d$.
- **Decoder:** Maps latent dimension $d$ back to $N$.
- **Architecture:** The specific implementation cited uses a symmetric structure with 5 layers:
  - Layer 1: $N \to 4096$
  - Layer 2: $4096 \to 512$
  - Layer 3 (Bottleneck): $512 \to d$
  - Layer 4: $d \to 512$
  - Layer 5: $512 \to N$
- **Activation:** Sigmoid function applied to hidden layers [S1].

**Training Procedure:**
1. **Data Preparation:** Split the dataset into batches. The cited implementation uses 400 batches of 500 samples each.
2. **Split:** 300 batches for training, 50 for testing, 50 for validation [S1].
3. **Optimization:** Use batch gradient descent to minimize the Mean Squared Error (MSE) between the input voxel data and the reconstructed output [S1].
4. **Stopping Criterion:** Training stops when the change in reconstruction error on the validation set falls below a threshold [S1].
5. **Implementation:** Matrix-vector operations are performed on the GPU using cuBLAS and Thrust libraries [S1].

**Reconstruction:**
Given a latent code $z$, the decoder network $D(z)$ produces the reconstructed voxel block $\hat{x}$. The loss function $L$ is:
$$ L = \frac{1}{M} \sum_{i=1}^{M} ||x_i - D(E(x_i))||^2 $$
where $x_i$ is the input block, $E$ is the encoder, $D$ is the decoder, and $M$ is the batch size [S1].

**Parameters:**
- **Latent Dimension ($d$):** Tested values include 32, 64, and 128 [S1].
- **Hidden Layer Sizes:** 4096 and 512 nodes [S1].
- **Activation:** Sigmoid [S1].

**Edge Cases and Pitfalls:**
- **Reconstruction Fidelity:** ANN encoders generally produce worse MSE than PCA encoders for the same latent dimension [S1].
- **Convergence:** The choice of stopping criterion is critical; validation error must be monitored to prevent overfitting [S1].
- **Hardware Dependency:** The cited implementation relies on GPU acceleration (cuBLAS/Thrust); CPU-only implementation may be prohibitively slow for large batches [S1].

## Subtopics

### [Mask-based Color Compression Routing](19109-mask-based-color-compression-routing.md)
A module that deduces binary masks to determine whether Gaussian color attributes should be directly quantized or processed through an autoencoder, balancing size and fidelity while preserving structural integrity.

## Operating characteristics
- **Time Complexity:** Training requires iterative gradient descent over batches. Inference (encoding/decoding) is dominated by matrix multiplications, optimized via GPU libraries (cuBLAS/Thrust) [S1].
- **Memory Complexity:** The model stores weights for the 5-layer network. For the cited architecture, this is significant but fixed regardless of the number of voxel blocks. The latent code size is $d$ per block [S1].
- **Measured Performance:**
  - **MSE:** NN 128 achieved $45.66 \pm 4.25$ MSE, compared to PCA 128's $26.88 \pm 2.57$ [S1].
  - **ATE (Pose Estimation):** NN 128 achieved a mean ATE of $0.13 \pm 0.20$ m, which is better than PCA 128's $0.56 \pm 0.68$ m [S1].
- **Strengths:** Better performance in downstream metric tasks (ATE) despite higher reconstruction error; non-linear modeling capability [S1].
- **Limitations:** Higher reconstruction error (MSE) compared to linear PCA; complex training process; requires GPU for efficient implementation [S1].
- **When to Use:** When downstream tasks (e.g., SLAM, pose estimation) benefit from non-linear feature preservation and metric accuracy is prioritized over raw pixel-wise reconstruction fidelity [S1].
- **When Not to Use:** When strict reconstruction fidelity (low MSE) is required, or when computational resources for training and GPU inference are limited [S1].

## Background
This method was proposed in the context of compressed voxel-based mapping for robotics, specifically for dense TSDF (Truncated Signed Distance Field) space mapping [S1]. It was introduced as an alternative to Principal Component Analysis (PCA) compression, exploring whether non-linear neural networks could better capture the statistical structure of voxel data for autonomous navigation and mapping tasks [S1].

## Sources
[S1] Compressed Voxel-Based Mapping Using Unsupervised Learning, https://doi.org/10.3390/robotics6030015, fetched 2026-10-26
