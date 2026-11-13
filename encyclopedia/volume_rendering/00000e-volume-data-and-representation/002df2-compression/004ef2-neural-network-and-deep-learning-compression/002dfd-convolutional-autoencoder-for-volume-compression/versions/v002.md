# Convolutional Autoencoder for Volume Compression

Also called: Autoencoder-based volume rendering, CNN Autoencoder, Compressed Volume Rendering using Deep Learning, Data-driven compressed volume rendering, Deep Learning Compression

## Overview
This method obtains a data-driven compact representation of multivariate time-varying volumetric data using a deep convolutional autoencoder network, enabling interactive rendering and navigation by decompressing the representation on the GPU [S1, unofficial]. It sets itself apart from traditional transform-based siblings like [Wavelet and Transform-based Compression](6879-wavelet-and-transform-based-compression.md) by learning hierarchical features directly from the data distribution rather than relying on fixed mathematical bases, resulting in significantly lower reconstruction error and fewer visual artifacts at equivalent storage ratios [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Multivariate time-varying volumetric data represented as a multidimensional array of size $X \times Y \times Z \times K \times T$, where $X, Y, Z$ are spatial dimensions, $K$ is the number of multivariate fields (channels), and $T$ is the number of time steps [S1, unofficial].
*   **Output:** A compressed representation (latent code) for each time step, and a reconstructed volume slice during rendering [S1, unofficial].

**Data Structures and Layout:**
The network architecture is a stacked convolutional autoencoder. The encoder path compresses the input, and the decoder path reconstructs it. The specific layer configuration is defined below [S1, unofficial].

*   **Encoder Path:**
    1.  Convolutional Layer: Input Channels 1 $\to$ Output Channels 64, Filter Size $3 \times 3$, Stride 1.
    2.  Convolutional Layer: Input Channels 64 $\to$ Output Channels 64, Filter Size $3 \times 3$, Stride 1.
    3.  Max Pooling Layer: Input Channels 64, Kernel $2 \times 2$, Stride 2.
    4.  Convolutional Layer: Input Channels 64 $\to$ Output Channels 64, Filter Size $3 \times 3$, Stride 1.
    5.  Convolutional Layer: Input Channels 64 $\to$ Output Channels 32, Filter Size $3 \times 3$, Stride 1.
    6.  Max Pooling Layer: Input Channels 32, Kernel $2 \times 2$, Stride 2.
    7.  Convolutional Layer: Input Channels 32 $\to$ Output Channels 16, Filter Size $3 \times 3$, Stride 1.
    8.  Convolutional Layer: Input Channels 16 $\to$ Output Channels 8, Filter Size $3 \times 3$, Stride 1.
    9.  Max Pooling Layer: Input Channels 8, Kernel $2 \times 2$, Stride 2.
    10. Convolutional Layer (Encoder Output): Input Channels 8 $\to$ Output Channels 4, Filter Size $3 \times 3$, Stride 1. This output is the compressed representation.

*   **Decoder Path:**
    1.  Convolutional Layer: Input Channels 4 $\to$ Output Channels 8, Filter Size $3 \times 3$, Stride 1.
    2.  Convolutional Layer: Input Channels 8 $\to$ Output Channels 16, Filter Size $3 \times 3$, Stride 1.
    3.  Convolutional Layer: Input Channels 16 $\to$ Output Channels 32, Filter Size $3 \times 3$, Stride 1.
    4.  Upsampling Layer: Input Channels 32, Kernel $2 \times 2$, Stride 2.
    5.  Convolutional Layer: Input Channels 32 $\to$ Output Channels 64, Filter Size $3 \times 3$, Stride 1.
    6.  Convolutional Layer: Input Channels 64 $\to$ Output Channels 64, Filter Size $3 \times 3$, Stride 1.
    7.  Upsampling Layer: Input Channels 64, Kernel $2 \times 2$, Stride 2.
    8.  Convolutional Layer: Input Channels 64 $\to$ Output Channels 64, Filter Size $3 \times 3$, Stride 1.
    9.  Convolutional Layer: Input Channels 64 $\to$ Output Channels 64, Filter Size $3 \times 3$, Stride 1.
    10. Upsampling Layer: Input Channels 64, Kernel $2 \times 2$, Stride 2.
    11. Convolutional Layer (Decoder Output): Input Channels 64 $\to$ Output Channels 1, Filter Size $3 \times 3$, Stride 1.

**Numbered Steps:**

1.  **Preprocessing:** Normalize the input volume data between 0 and 1 for each multivariate dimension across all time steps. Arrange the data as multivariate slices along the z-axis so that the multivariate field acts as a channel to the network [S1, unofficial].
2.  **Training:**
    *   Randomly sample multivariate slices from all time steps.
    *   Feed slices through the autoencoder.
    *   Update weights using the ADADELTA solver with momentum 0.95 to minimize cross-entropy loss between the input slice and the reconstructed slice.
    *   Training stops when the mean squared error (MSE) between the input volume and the reconstructed volume stops decreasing [S1, unofficial].
3.  **Splitting:** Separate the trained autoencoder into distinct encoding and decoding networks [S1, unofficial].
4.  **Compression:** The encoder outputs a compressed representation computed as a nonlinear combination of input data and learned hierarchical features. This representation is stored on GPU memory [S1, unofficial].
5.  **Decompression (On-the-fly):** During rendering, feed the compressed slice through the decoder network (Convolutional and Upsampling layers) to reconstruct the original resolution slice on the fly [S1, unofficial].
6.  **Rendering:**
    *   For a required time step, decompress subsampled slices along the z-axis to create a coarser volume.
    *   Render this coarser volume.
    *   Continuously refine by decompressing remaining slices while rendering.
    *   Switch to finer volume rendering once decompression is complete [S1, unofficial].
    *   Apply a user-defined transfer function to map voxel intensity (0-255) to color and opacity using linear interpolation between adjacent control points [S1, unofficial].

**Formulas:**
*   **Convolution Operation:** The $i$-th channel $V_i$ of the output volume is calculated as [S1, unofficial]:
    $$V_i = \sum_{j=1}^p W_{j,q} * U_j + b_i$$
    Where $W_{j,q}$ are trainable filters, $U_j$ are input channels, and $b_i$ is the trainable bias.
*   **Activation Functions:**
    *   Hidden layers use Rectified Linear Units (ReLU): $f(v_i) = \max(0, v_i)$ [S1, unofficial].
    *   Output layer uses Sigmoid to map output to range [0, 1]:
        $$f(v_i) = \frac{1}{1 + e^{-v_i}}$$
        [S1, unofficial]

**Parameters:**
*   **Filter Size:** $3 \times 3$ for all convolutional layers [S1, unofficial].
*   **Optimizer:** ADADELTA with momentum 0.95 [S1, unofficial].
*   **Loss Function:** Cross-entropy between input and reconstructed slice [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Constraints:** The compressed representation must be small enough to reside in GPU memory. The method enforces this by design, requiring only additional memory for one reconstructed slice at full resolution during decompression [S1, unofficial].
*   **Training Convergence:** The number of iterations depends on dataset complexity. Training must monitor MSE to prevent underfitting or overfitting [S1, unofficial].

## Subtopics

### [Deep Implicit Volume Compression](6807-deep-implicit-volume-compression.md)
Deep Implicit Volume Compression is a block-based neural network architecture trained end-to-end to compress Truncated Signed Distance Fields (TSDF) stored in 3D voxel grids, achieving state-of-the-art rate-distortion trade-offs. It differs from standard convolutional autoencoders by employing a dual-output decoder head, lossless compression of topological signs via conditional entropy modeling, and a topology-aware distortion loss, ensuring the reconstructed mesh remains homeomorphic to the ground truth.

### 3D Shape Autoencoder
An autoencoder trained to learn a prior of 3D shapes using 3D convolutions [S2, unofficial].
*   **Architecture:**
    *   **Encoder:** Series of 3D convolutions with channels {64, 128, 256, 512}, kernel sizes {5, 5, 2, 2}, and strides {2, 2, 2, 2}.
    *   **Middle:** Fully-connected layer mapping last conv output to a 200-dimensional vector, followed by a sigmoid activation.
    *   **Decoder Input:** Fully-connected layer mapping 200-dim vector to a $(4 \times 4 \times 512)$-dimensional vector, reshaped to tensor size $4 \times 4 \times 512$.
    *   **Decoder:** Series of 3D up-convolutions with channels {256, 128, 64, 32, 1}, kernel sizes {4, 4, 4, 4, 4}, and strides {2, 2, 2, 2, 1}.
*   **Activation:** ELU for all layers except the last convolution layer in the encoder and decoder, which uses Sigmoid [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Decompression is performed on the fly per slice using basic convolution and upsampling operations, enabling interactive speeds [S1, unofficial].
*   **Memory Complexity:** Stores only the compressed representation for all time steps on GPU memory. Decompression requires additional memory for only one reconstructed slice at full resolution [S1, unofficial].
*   **Measured Performance:**
    *   **Dataset 1** (52x52x37, 88 timesteps, 2.56GB): CNN Autoencoder MSE=0.000061, PSNR=42.125. Best Wavelet (Discrete Mayer) MSE=0.0015, PSNR=28.239 [S1, unofficial].
    *   **Dataset 2** (100x100x100, 20 timesteps, 5.81GB): CNN Autoencoder MSE=0.0049, PSNR=23.098. Best Wavelet (Biorthogonal 9/3) MSE=0.0219, PSNR=16.588 [S1, unofficial].
    *   **Compression Ratio:** 14.28:1 (compressed representation takes 7% of total memory) [S1, unofficial].
*   **Strengths:** Significantly lower MSE and higher PSNR compared to wavelet compression at the same storage usage. Provides close approximation with no visible artifacts, whereas wavelets result in coarse reconstruction with artifacts [S1, unofficial].
*   **Limitations:** Requires offline training. Performance depends on the complexity of the dataset and desired quality [S1, unofficial].

## Background
This method was proposed to address the challenge of visualizing large amounts of multivariate time-varying volumetric data generated by scientific simulations, which are essential for understanding underlying processes but difficult to store and transmit [S1, unofficial]. It builds on the principle that deep neural networks correspond to hierarchical Tucker decomposition, offering richer representational capacity than traditional Tensor Approximation [S1, unofficial]. The approach enables the compact representation to fit on GPU memory, facilitating interactive rendering and navigation [S1, unofficial].

## Sources
[S1] Compressed volume rendering using deep learning, https://www.cs.umd.edu/sites/default/files/scholarly_papers/Jain.pdf, fetched 2026-10-26 (unofficial)
[S2] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
