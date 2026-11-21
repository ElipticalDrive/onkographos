# Latent Code Conditioning

Also called: Global Conditioning, Instance-Specific Conditioning, Latent Vector Conditioning

## Overview
Latent Code Conditioning is a family of techniques for injecting instance-specific information into a Neural Field decoder by associating a unique latent vector $z_i$ with each data sample (e.g., a specific 3D volume, shape, or dynamic sequence) [S1, unofficial][S2]. It solves the problem of representing multiple distinct instances using a shared neural field architecture, allowing the decoder to generalize across a dataset while preserving instance-specific details [S1, unofficial][S2]. It is distinguished from sibling methods by its specific mechanism of conditioning: [Concatenation Conditioning](12582-concatenation-conditioning.md) mixes the code into the input features, [HyperNetwork Conditioning](12585-hypernetwork-conditioning.md) generates decoder weights from the code, and [Autoencoder/Autodecoder Conditioning](12588-autoencoder-autodecoder-conditioning.md) optimizes the code as a learnable parameter alongside the network weights [S1, unofficial][S2].

## Shared principles
All methods in this family rely on the existence of a shared neural field decoder $f_\theta$ (typically an MLP) and a set of instance-specific latent codes $\{z_i\}$. The core mechanism involves mapping the latent code $z_i \in \mathbb{R}^D$ into the computation graph of the decoder to influence the output $y_i$ for a query point $x$ [S1, unofficial][S2].

The latent code $z_i$ serves as a global summary of the instance's properties (geometry, appearance, or transfer function parameters) [S1, unofficial][S2]. The dimension $D$ is a hyperparameter chosen based on the complexity of the variations between instances [S1, unofficial][S2]. Common values cited are $D=512$ for volumetric applications [S1, unofficial] and $D=256$ for shape space learning [S1, unofficial].

The conditioning is typically applied at one of two points in the decoder:
1.  **Input Conditioning:** The latent code is concatenated with the spatial coordinates $x$ (and potentially other explicit descriptors $d_x$) before entering the first layer of the decoder [S1, unofficial][S2, S3].
2.  **Weight Conditioning:** The latent code is passed through a secondary network (HyperNetwork) to generate the weights $\theta_i$ of the decoder, such that $y_i = f_{\theta_i}(x)$ where $\theta_i = h_\phi(z_i)$ [S1, unofficial].

In the context of dynamic scenes, separate latent codes may be used for shape and appearance, concatenated at different stages of the network (e.g., shape code at input, appearance code at bottleneck) [S2].

## Subtopics

### [Concatenation Conditioning](12582-concatenation-conditioning.md)
Concatenation Conditioning injects the latent code $z_i$ by appending it to the input features or descriptors before evaluation. It is chosen when a fixed decoder architecture is preferred and instance variations can be captured by shifting the input distribution. It differs from HyperNetwork Conditioning by modifying inputs rather than weights, and from Autoencoder Conditioning by typically using a separate encoder-decoder pipeline rather than joint optimization.

### [HyperNetwork Conditioning](12585-hypernetwork-conditioning.md)
HyperNetwork Conditioning uses a secondary network (HyperNetwork) to predict the decoder weights $\theta_i$ from the latent code $z_i$. It is chosen when high representational power is needed with a simpler primary decoder structure, or when instance-specific weight configurations are required. It differs from Concatenation Conditioning by creating a distinct neural field for each instance rather than sharing a single decoder.

### [Autoencoder/Autodecoder Conditioning](12588-autoencoder-autodecoder-conditioning.md)
Autoencoder/Autodecoder Conditioning represents a collection of instances using a shared decoder $f_\theta$ and unique latent codes $z_i$, optimized either via an encoder network or directly as learnable parameters (autodecoder). It is chosen for unsupervised multi-instance representation learning. It differs from siblings by its specific optimization strategy: $z_i$ is treated as an independent trainable variable optimized via gradient descent for each instance, rather than being computed by a separate encoder or injected via hypernetwork weights.

### [Global Conditioning](12589-global-conditioning.md)
Global conditioning encodes the entire information of a data sample into a single latent vector $z$, which is then used to condition a shared neural field decoder. It serves as the foundational mechanism for instance association, distinguishing itself from local conditioning methods that distribute latent codes across spatial or temporal coordinates. It is typically implemented via a mapping network that transforms the raw latent code into an intermediate non-linear latent space.

### Local Conditioning
Local conditioning spreads information about a data sample over multiple latent codes assigned to specific subsets of spatio-temporal coordinates (regions or time steps) [S1, unofficial]. This allows for instance-specific details to be encoded locally rather than globally. A hybrid approach may use one global code for shared basic information and several local codes for instance-specific details [S1, unofficial].

### Dynamic Object Radiance Field Conditioning
Dynamic Object Radiance Field Conditioning uses separate shape and appearance latent vectors to condition a radiance field $\psi$ for dynamic objects [S2]. The shape code is concatenated with the input coordinate $x$ at the network input, while the appearance code is concatenated with the direction $d$ at the bottleneck after density prediction [S2]. Sequence and object appearance latent vectors are concatenated to propagate sequence appearance to individual objects [S2].

### Image Encoding
Image Encoding passes an appearance source image $A$ through a network to generate a latent code, which is then appended to the input voxels [S3]. This method captures the overall color of the object but fails to assign it correctly to individual parts, leading to blurry or misaligned textures compared to more advanced neural rendering methods [S3].

### Edit Code Mapping
Edit Code Mapping maps each edited 2D image $S_i^j$ into a latent feature vector $z_i^j \in \mathbb{R}^{64}$ named edit code to characterize diverse edits and solve the inconsistency problem in 3D scenes caused by 2D image-to-image translation [S4]. The process involves: 1. Extracting features from the edited image using an off-the-shelf VAE encoder $\mathcal{E}$ from Stable Diffusion [30]. 2. Applying a tiny MLP network $\mathcal{M}$ to produce the edit code $z_i^j$. 3. Keeping the pre-trained encoder fixed and only optimizing the parameters of the tiny MLP network during training [S4]. The mapping is formulated as $\mathbf {z}_i^j = \mathcal {D}(\mathbf {S}_i^j) = \mathcal {M}(\mathcal {E}(\mathbf {S}_i^j))$, where $\mathcal{D}$ represents the mapping process, $\mathcal{E}$ is the fixed encoder, and $\mathcal{M}$ is the learnable tiny MLP [S4].

## Background
The concept of using latent codes to condition neural fields was introduced in the context of global conditioning, where one latent vector is provided per data sample to encode the whole information about the data sample [S1, unofficial]. This practice allows for the representation of multiple instances using a single shared network architecture [S1, unofficial]. The term "global conditioning" was used to distinguish this from "local conditioning," where information is spread over multiple latent codes assigned to specific spatio-temporal coordinates [S1, unofficial].

In dynamic scenes, the need for conditioning arose to account for varying transient geometry and appearance across sequences, leading to the use of separate shape and appearance latent vectors [S2]. In the context of neural voxel rendering, image encoding was used to pass appearance source images through a network to generate latent codes for voxel input [S3]. More recently, edit code mapping was proposed to handle 3D scene inconsistencies in NeRF-to-NeRF translation by mapping edited 2D images to latent feature vectors [S4].

## Sources
[S1] Introduction to Neural Fields - inovex GmbH, https://www.inovex.de/de/blog/introduction-to-neural-fields/, fetched 2026-10-29 (unofficial)
[S2] Multi-Level Neural Scene Graphs for Dynamic Urban Environments, https://doi.org/10.1109/cvpr52733.2024.01996, fetched 2026-10-29
[S3] Neural Voxel Renderer: Learning an Accurate and Controllable Rendering Tool, https://doi.org/10.1109/cvpr42600.2020.00546, fetched 2026-10-26
[S4] GenN2N: Generative NeRF2NeRF Translation, https://doi.org/10.1109/CVPR52733.2024.00488, fetched 2026-10-29
