# Spatial Feature Encoding


## Overview
Spatial Feature Encoding maps input view semantics to per-pixel feature vectors, which are subsequently projected into 3D space for volume rendering or registration tasks [S1, S3]. It operates in 2D image space before reprojection [S1].

## Shared principles
The family encodes 2D image data (RGB or RGBD) into feature maps or vectors [S1, S3]. The core mechanism involves passing augmented inputs through a neural network encoder to produce the feature representation [S1, S3]. Common architectural components include convolutional layers for hierarchical feature extraction [S1], skip connections between encoder and decoder stages to preserve high-frequency local features [S1], and fully connected layers in the bottleneck to learn global features [S1]. The output consists of per-pixel feature vectors $F_i$, which are concatenated with RGB channels to form a 64-dimensional total in the primary implementation [S1]. For registration tasks, the encoder produces a same-sized feature map $C_t$ of dimensions $H \times W \times D$ [S3].

## Subtopics

### [CNN-based 2D Feature Extraction](8649-cnn-based-2d-feature-extraction.md)
This method extracts robust, generalizable per-pixel features from input RGB images using a CNN encoder. It solves the problem of raw RGB values being sensitive to lighting conditions and environmental noise by learning a compact feature representation that encodes geometric and photometric patterns. It differs from simple raw RGB usage by explicitly incorporating camera position awareness and using skip connections to preserve high-frequency details, distinguishing it from other spatial encoding methods that may rely on positional encoding or hash grids.

### [TriVol Encoding](19086-trivol-encoding.md)
TriVol Encoding is a method for compressing sparse 3D point clouds into a dense, continuous feature representation suitable for neural volume rendering. It solves the memory bottleneck of standard dense voxel grids by grouping voxels along each axis into "slim" volumes, thereby amplifying the receptive field on the grouped axis while maintaining local detail on the other two. Unlike [CNN-based 2D Feature Extraction](8649-cnn-based-2d-feature-extraction.md) which operates on 2D image planes, TriVol operates directly on 3D spatial structures derived from point clouds, enabling efficient high-resolution modeling without proportional increases in computational cost.

### Spatial Feature Encoder
**Definition:** Maps scene semantics of an input view $I_i$ to per-pixel feature vectors $F_i$ [S1].
**Parameters:** Output dimension is 61 dimensions per vector [S1]. After concatenation with RGB channels, the total dimension is 64 [S1]. Architecture is built on BigGAN architecture [S1].
**History:** Built on BigGAN architecture [S1].

### SF Encoder
**Steps:**
1. Transpose input tensor from $[batch\_size, num\_points, input\_dim]$ to $[batch\_size, input\_dim, num\_points]$ [S2, unofficial].
2. Pass transposed input through sequential convolutional layers with BatchNorm and ReLU activations [S2, unofficial].
3. Apply max pooling over the point dimension ($dim=2$) to aggregate local features into a global feature vector of shape $[batch\_size, feature\_dim]$ [S2, unofficial].
4. If $vae$ is True, compute $\mu$ and $\logvar$ via fully connected layers [S2, unofficial]. Clamp $\logvar$ to $[-6, 3]$. If $deterministic$ is False, sample $z = \mu + std * noise$ where $std = \exp(0.5 * \logvar) + 1e-8$ [S2, unofficial]. Return $z, \mu, \logvar$ [S2, unofficial].
5. If $vae$ is False, pass global feature through two fully connected layers with ReLU and linear output respectively [S2, unofficial].
**Parameters:** $input\_dim$: number of input channels per point, default 3 [S2, unofficial]. $feature\_dim$: dimensionality of the global feature vector, default 512 [S2, unofficial]. $vae$: whether to use variational autoencoder reparameterization, default True [S2, unofficial]. $deterministic$: whether to skip stochastic sampling and use only $\mu$, default False [S2, unofficial].
**Layout:** Four 1D convolutional layers with kernel size 1: $Conv1d(input\_dim, 64, 1)$, $Conv1d(64, 128, 1)$, $Conv1d(128, 256, 1)$, $Conv1d(256, feature\_dim, 1)$. Each followed by $BatchNorm1d$ and ReLU activation [S2, unofficial].

### Feature Registration (Freg)
**Definition:** Encodes an input RGBD image $I_t$ into a same-sized feature map $C_t$ of dimensions $H \times W \times D$ using an encoder network with parameters $\theta_{enc}$ [S3].
**Formula:**
$$ m_t^{l}, n_t^{l} = F_{reg}(I_t, p_t; \theta_{enc}) $$
where $m_t^l$ is the local map feature and $n_t^l$ is the local mask count [S3].

## Background
The concept of mapping view semantics to feature vectors for forward warping and depth estimation was introduced in the context of real-time novel view synthesis [S1]. The use of BigGAN-based architectures for this specific encoding task highlights the adaptation of generative models for feature extraction [S1]. Later developments integrated these encodings into registration processes for visual navigation, where RGBD inputs are encoded and inverse-projected to 3D world positions [S3]. Variations such as the SF Encoder demonstrate the application of point-cloud-centric encodings (using 1D convolutions and VAEs) for spatial feature representation [S2, unofficial].

## Sources
[S1] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
[S2] cilix-ai/gs-embedding, https://github.com/cilix-ai/gs-embedding, fetched 2026-10-27 (unofficial)
[S3] Renderable Neural Radiance Map for Visual Navigation, https://doi.org/10.1109/cvpr52729.2023.00878, fetched 2026-10-26
