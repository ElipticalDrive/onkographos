# Latent Code Conditioning

## Overview
Latent Code Conditioning is a family of techniques for injecting instance-specific information into a Neural Field decoder by associating a unique latent vector $z_i$ with each data sample [S1, unofficial][S2, unofficial]. It solves the problem of representing multiple distinct instances using a shared neural field architecture, allowing the decoder to generalize across a dataset while preserving instance-specific details [S1, unofficial][S2, unofficial]. It is distinguished from sibling methods by its specific mechanism of conditioning: [Concatenation Conditioning](12582-concatenation-conditioning.md) mixes the code into the input features, [HyperNetwork Conditioning](12585-hypernetwork-conditioning.md) generates decoder weights from the code, and [Autoencoder/Autodecoder Conditioning](12588-autoencoder-autodecoder-conditioning.md) optimizes the code as a learnable parameter alongside the network weights [G].

## Shared principles
All methods in this family rely on the existence of a shared neural field decoder $f_\theta$ (typically an MLP) and a set of instance-specific latent codes $\{z_i\}$ [S1, unofficial][S2, unofficial]. The latent code $z_i \in \mathbb{R}^D$ serves as a global summary of the instance's properties, such as geometry, appearance, or transfer function parameters [S1, unofficial][S2, unofficial]. Conditioning is typically applied by either concatenating $z_i$ with spatial coordinates $x$ (and optional explicit descriptors $d_x$) before the decoder's first layer, or by passing $z_i$ through a secondary network to generate instance-specific decoder weights $\theta_i$ [S1, unofficial][S2, unofficial]. In dynamic scenes, separate latent codes may be used for shape and appearance, concatenated at different network stages (e.g., shape code at input, appearance code at bottleneck) [S2, unofficial].

## Subtopics

### [Concatenation Conditioning](12582-concatenation-conditioning.md)
Concatenation Conditioning injects the latent code $z_i$ by appending it to the input features or descriptors before evaluation. It is chosen when a fixed decoder architecture is preferred and instance variations can be captured by shifting the input distribution. It differs from HyperNetwork Conditioning by modifying inputs rather than weights, and from Autoencoder Conditioning by typically using a separate encoder-decoder pipeline rather than joint optimization.

### [HyperNetwork Conditioning](12585-hypernetwork-conditioning.md)
HyperNetwork Conditioning uses a secondary network (HyperNetwork) to predict the decoder weights $\theta_i$ from the latent code $z_i$. It is chosen when high representational power is needed with a simpler primary decoder structure, or when instance-specific weight configurations are required. It differs from Concatenation Conditioning by creating a distinct neural field for each instance rather than sharing a single decoder.

### [Autoencoder/Autodecoder Conditioning](12588-autoencoder-autodecoder-conditioning.md)
Autoencoder/Autodecoder Conditioning represents a collection of instances using a shared decoder $f_\theta$ and unique latent codes $z_i$, optimized either via an encoder network or directly as learnable parameters (autodecoder). It is chosen for unsupervised multi-instance representation learning. It differs from siblings by its specific optimization strategy: $z_i$ is treated as an independent trainable variable optimized via gradient descent for each instance, rather than being computed by a separate encoder or injected via hypernetwork weights.

### [Global Conditioning](12589-global-conditioning.md)
Global conditioning encodes the entire information of a data sample into a single latent vector $z$, which is then used to condition a shared neural field decoder. It serves as the foundational mechanism for instance association, distinguishing itself from local conditioning methods that distribute latent codes across spatial or temporal coordinates. It is typically implemented via a mapping network that transforms the raw latent code into an intermediate non-linear latent space.

### [Conditional Radiance Field Training](15963-conditional-radiance-field-training.md)
Conditional Radiance Field Training models shape and appearance variations across multiple 3D instances by jointly optimizing a single neural radiance field with instance-specific latent codes. It is chosen for efficient multi-instance reconstruction and editing. It differs from standard NeRF by replacing implicit instance identity with explicit, learnable shape and color codes that condition the network's output.

### [Conditional Radiance Field with Shared Branch](15964-conditional-radiance-field-with-shared-branch.md)
The Conditional Radiance Field with Shared Branch is designed for shape and color disentanglement, enabling efficient instance-specific editing by allowing specific submodules to be fine-tuned independently. It is chosen when view synthesis quality and scalability to multiple instances are prioritized. It differs from standard conditional formulations by separating the shape representation into a category-specific shared branch and an instance-specific branch.

### [Conditional Radiance Field](15965-conditional-radiance-field.md)
A Conditional Radiance Field extends the standard NeRF representation to model an entire object class rather than a single static scene, using instance-specific latent vectors for shape and appearance. It is chosen for editing and interpolation across a class of objects. It differs from standard NeRF by decoupling instance-specific details into latent codes $z$ while storing shared class information in the network parameters.

### [Image Encoding](16847-image-encoding.md)
Image Encoding extracts a global latent code from an input appearance source image to condition a neural field decoder. It is chosen when 2D appearance needs to be mapped to a 3D volume without explicit 3D structural priors. It differs from other methods by using a dedicated encoder network to map 2D appearance into a fixed-dimensional vector that is concatenated with spatial coordinates before decoding.

### [Mask Prediction Network](19912-mask-prediction-network.md)
The Mask Prediction Network (MPN) predicts per-point activation masks to selectively enable or disable specific semantic attributes during rendering. It is chosen for disentangling and controlling multiple dynamic or static elements within a single neural field. It differs from sibling conditioning methods by operating as a local gating mechanism that filters contributions based on lifted attributes and canonicalized coordinates.

### Local Conditioning
Local conditioning spreads information about a data sample over multiple latent codes assigned to specific subsets of spatio-temporal coordinates (regions or time steps) [S1, unofficial]. This allows for instance-specific details to be encoded locally rather than globally [S1, unofficial]. A hybrid approach may use one global code for shared basic information and several local codes for instance-specific details [S1, unofficial].

### Dynamic Object Radiance Field Conditioning
Dynamic Object Radiance Field Conditioning uses separate shape and appearance latent vectors to condition a radiance field $\psi$ for dynamic objects [S2, unofficial]. The shape code is concatenated with the input coordinate $x$ at the network input, while the appearance code is concatenated with the direction $d$ at the bottleneck after density prediction [S2, unofficial]. Sequence and object appearance latent vectors are concatenated to propagate sequence appearance to individual objects [S2, unofficial].

### Edit Code Mapping
Edit Code Mapping maps each edited 2D image $S_i^j$ into a latent feature vector $z_i^j \in \mathbb{R}^{64}$ named edit code to characterize diverse edits and solve the inconsistency problem in 3D scenes caused by 2D image-to-image translation [S4]. The process involves: 1. Extracting features from the edited image using an off-the-shelf VAE encoder $\mathcal{E}$ from Stable Diffusion [30] [S4]. 2. Applying a tiny MLP network $\mathcal{M}$ to produce the edit code $z_i^j$ [S4]. 3. Keeping the pre-trained encoder fixed and only optimizing the parameters of the tiny MLP network during training [S4]. The mapping is formulated as $\mathbf {z}_i^j = \mathcal {D}(\mathbf {S}_i^j) = \mathcal {M}(\mathcal {E}(\mathbf {S}_i^j))$, where $\mathcal{D}$ represents the mapping process, $\mathcal{E}$ is the fixed encoder, and $\mathcal{M}$ is the learnable tiny MLP [S4].

### Local Animation Code Method
Local Animation Code Method generates local animation codes from multi-view inputs, mapped to local codes using a 3D CNN, conditioning fine-level radiance fields represented as MLPs [S1, unofficial]. It does not allow direct control over expressions and requires an encoder to generate animation codes from facial keypoints [S1, unofficial].

### Style Interpolation
Style Interpolation is a technique to blend multiple style images by interpolating their latent vectors at test time [S5]. At test time, four different style images are taken, their style latents are extracted, and linearly interpolated between the different styles before passing the interpolated latent to the model [S5]. It demonstrates the model's ability to generate results for a mixture of styles, showing it learns a meaningful mapping from style inputs to predicted RGB values for each 3D Gaussian, providing increased generalisability [S5].

### Canonicalization Network
Canonicalization Network transforms raw 3D coordinates and a per-image latent code into a canonical space using a learned rigid/affine transformation [S6]. It passes encoded $x$ and $\beta$ through fully connected layers with ReLU activation to output rotation $r$ (quaternion, 128 dims) and translation $t$ (128 dims) [S6]. It rigidly transforms each point $x$ with an affine transform using both outputs [S6]. Positional encoding for $x$ uses a windowed positional encoding with 8 components, linearly increasing contribution of components throughout 80k steps [S6]. The last layer is initialized to small values so the network can learn a base structure of the data [S6].

### Latent Interpolation
Latent Interpolation synthesizes images using linearly interpolated latent codes in the $w$ space to validate semantic latent space characteristics [S7].

### MedNeRF
MedNeRF is a conditional NeRF architecture used as a baseline, which suffers from diffuse density fields and unreliable nodule localization due to unconditioned latent codes [S8, unofficial]. The latent code $z$ is sampled from $\mathcal{N}(0, I)$ rather than inverted from patient X-rays, so reconstructed density does not reflect actual anatomy [S8, unofficial]. It uses a scene-agnostic percentile threshold (top 5% of densities) regardless of nodule presence [S8, unofficial]. Method 1 applies direct 95th percentile thresholding to MedNeRF density field, yielding 150% volumetric error [S8, unofficial].

### Density and color field network with per-frame latent code
Density and color field network with per-frame latent code is a neural network architecture for predicting volume density and view-dependent color, modified to encode dynamic human appearance per frame [S9]. It uses a per-frame latent code $\ell_i$ to encode the state of human appearance for frame $i$ [S9]. Layer dimensions (input sizes per block) are 128, 256, 256, 256, 256, 256, 256, 256, 27, 256, 256, 128 [S9].

### Object Cloning via MLP/Latent Codes
Object Cloning via MLP/Latent Codes duplicates the weights of an object's MLPs or its latent codes to result in the cloning of that instance in the scene [S10].

## Background
The concept of using latent codes to condition neural fields was introduced in the context of global conditioning, where one latent vector is provided per data sample to encode the whole information about the data sample [S1, unofficial]. This distinguishes it from local conditioning, where information is spread over multiple latent codes assigned to specific spatio-temporal coordinates [S1, unofficial]. In dynamic scenes, the need for conditioning arose to account for varying transient geometry and appearance across sequences, leading to the use of separate shape and appearance latent vectors [S2, unofficial]. In the context of neural voxel rendering, image encoding was used to pass appearance source images through a network to generate latent codes for voxel input [S3]. More recently, edit code mapping was proposed to handle 3D scene inconsistencies in NeRF-to-NeRF translation by mapping edited 2D images to latent feature vectors [S4].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Introduction to Neural Fields - inovex GmbH, https://www.inovex.de/de/blog/introduction-to-neural-fields/, fetched 2026-10-29 (unofficial)
[S3] Multi-Level Neural Scene Graphs for Dynamic Urban Environments, https://doi.org/10.1109/cvpr52733.2024.01996, fetched 2026-10-29
[S4] GenN2N: Generative NeRF2NeRF Translation, https://doi.org/10.1109/CVPR52733.2024.00488, fetched 2026-10-29
[S5] Gaussian Splatting in Style, https://doi.org/10.1007/978-3-031-85187-2_15, fetched 2026-10-26
[S6] CoNeRF: Controllable Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.01807, fetched 2026-10-26
[S7] GSGAN: Adversarial Learning for Hierarchical Generation of 3D Gaussian Splats, https://doi.org/10.52202/079017-2171, fetched 2026-10-29
[S8] Sparse-View Lung Nodule Volumetry from Digitally Reconstructed Radiographs via AReT: Anatomy-Regularized TensoRF, https://doi.org/10.48550/arxiv.2606.02639, fetched 2026-10-26 (unofficial)
[S9] Animatable Neural Radiance Fields for Modeling Dynamic Human Bodies, https://doi.org/10.1109/iccv48922.2021.01405, fetched 2026-10-26
[S10] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
