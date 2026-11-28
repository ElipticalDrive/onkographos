# Spatial Feature Encoding

Also called: Spatial Feature Encoder, SF Encoder, Feature Registration (Freg)

## Overview
Spatial Feature Encoding maps input view semantics to per-pixel feature vectors, which are subsequently projected into 3D space for volume rendering or registration tasks [S1, S3]. It differs from raw pixel usage by learning robust, lighting-invariant patterns and differs from 3D-native encodings by operating in 2D image space before reprojection [S1].

## Shared principles
The family operates on the principle of encoding 2D image data (RGB or RGBD) into a dense feature map $F$ or $C$ of dimensions $H \times W \times D$, where $H$ and $W$ are image height and width, and $D$ is the feature dimension [S1, S3].

**Input Augmentation:**
For methods incorporating camera awareness, the input tensor is augmented by stacking the camera position vector $\mathbf{c} \in \mathbb{R}^3$ with the RGB channels [S1]. For each pixel $p$ at coordinates $(u, v)$, the input feature vector $\mathbf{x}_{u,v}$ is constructed as:
$$ \mathbf{x}_{u,v} = [\text{RGB}_{u,v}, \mathbf{c}] \in \mathbb{R}^{3+3} $$
The resulting input tensor has dimensions $H \times W \times 6$ [S1]. Camera rotation and intrinsics are explicitly omitted as they do not noticeably improve performance [S1].

**Encoding Architecture:**
The core mechanism involves passing the augmented input through a neural network encoder to produce the feature map [S1, S3]. Common architectural components include:
1.  **Convolutional Layers:** Used for hierarchical feature extraction [S1].
2.  **Skip Connections:** Applied between encoder and decoder stages to preserve high-frequency local features [S1].
3.  **Global Feature Learning:** Fully connected (FC) layers may be integrated in the bottleneck to learn global features [S1].

**Output:**
The output is a feature map $F \in \mathbb{R}^{H \times W \times D}$ [S1]. This map is later used for depth estimation, view synthesis, or inverse projection into 3D world coordinates using known camera intrinsics [S1, S3].

## Subtopics

### [CNN-based 2D Feature Extraction](8649-cnn-based-2d-feature-extraction.md)
This method uses a convolutional neural network (CNN) to extract robust, per-pixel feature vectors from input RGB images, replacing raw RGB values which are sensitive to lighting and noise. It differs from simple raw pixel usage by learning general patterns and explicitly incorporating camera position awareness, while differing from other spatial encodings by focusing on 2D image-space feature maps that are later reprojected to 3D. It builds on [Spatial Feature Encoding](16657-spatial-feature-encoding.md).

### Spatial Feature Encoder
**Definition:** Maps scene semantics of an input view $I_i$ to per-pixel feature vectors $F_i$ [S1].
**Parameters:**
*   **Output Dimension:** 61 dimensions per vector [S1].
*   **Total Dimension:** After concatenation with RGB channels, the total dimension is 64 [S1].
*   **Architecture:** Built on BigGAN architecture [S1].
**History:** Built on BigGAN architecture [S1].

### SF Encoder
**Steps:**
1.  Transpose input tensor from $[batch\_size, num\_points, input\_dim]$ to $[batch\_size, input\_dim, num\_points]$ [S2, unofficial].
2.  Pass transposed input through sequential convolutional layers with BatchNorm and ReLU activations [S2, unofficial].
3.  Apply max pooling over the point dimension ($dim=2$) to aggregate local features into a global feature vector of shape $[batch\_size, feature\_dim]$ [S2, unofficial].
4.  If $vae$ is True, compute $\mu$ and $\logvar$ via fully connected layers. Clamp $\logvar$ to $[-6, 3]$. If $deterministic$ is False, sample $z = \mu + std * noise$ where $std = \exp(0.5 * \logvar) + 1e-8$. Return $z, \mu, \logvar$ [S2, unofficial].
5.  If $vae$ is False, pass global feature through two fully connected layers with ReLU and linear output respectively [S2, unofficial].
**Parameters:**
*   $input\_dim$: number of input channels per point, default 3 [S2, unofficial].
*   $feature\_dim$: dimensionality of the global feature vector, default 512 [S2, unofficial].
*   $vae$: whether to use variational autoencoder reparameterization, default True [S2, unofficial].
*   $deterministic$: whether to skip stochastic sampling and use only $\mu$, default False [S2, unofficial].
**Layout:** Four 1D convolutional layers with kernel size 1: $Conv1d(input\_dim, 64, 1)$, $Conv1d(64, 128, 1)$, $Conv1d(128, 256, 1)$, $Conv1d(256, feature\_dim, 1)$. Each followed by $BatchNorm1d$ and ReLU activation [S2, unofficial].

### Feature Registration (Freg)
**Definition:** Encodes an input RGBD image $I_t$ into a same-sized feature map $C_t$ of dimensions $H \times W \times D$ using an encoder network with parameters $\theta_{enc}$ [S3].
**Formula:**
$$ m_t^{l}, n_t^{l} = F_{reg}(I_t, p_t; \theta_{enc}) $$
where $m_t^l$ is the local map feature and $n_t^l$ is the local mask count [S3].

## Background
Spatial Feature Encoding is a foundational component in neural rendering pipelines that bridge 2D observations and 3D representations. The concept of mapping view semantics to feature vectors for forward warping and depth estimation was introduced in the context of real-time novel view synthesis [S1]. The use of BigGAN-based architectures for this specific encoding task highlights the adaptation of generative models for feature extraction [S1]. Later developments integrated these encodings into registration processes for visual navigation, where RGBD inputs are encoded and inverse-projected to 3D world positions [S3]. Variations such as the SF Encoder demonstrate the application of point-cloud-centric encodings (using 1D convolutions and VAEs) for spatial feature representation [S2, unofficial].

## Sources
[S1] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
[S2] cilix-ai/gs-embedding, https://github.com/cilix-ai/gs-embedding, fetched 2026-10-27 (unofficial)
[S3] Renderable Neural Radiance Map for Visual Navigation, https://doi.org/10.1109/cvpr52729.2023.00878, fetched 2026-10-26
