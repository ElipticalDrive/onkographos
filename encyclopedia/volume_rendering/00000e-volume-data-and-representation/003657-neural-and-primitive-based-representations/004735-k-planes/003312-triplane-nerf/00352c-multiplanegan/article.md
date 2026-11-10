# MultiPlaneGAN

Also called: MultiPlane 3D Representation

## Overview
MultiPlaneGAN is a GAN-based 3D generation framework that synthesizes 3D objects by replacing the tri-plane representation of EG3D with a MultiPlane 3D representation consisting of 32 planes posed on a sphere [S1]. It differs from its parent [TriPlane NeRF](6932-triplane-nerf.md) and the baseline EG3D by using a spherical distribution of feature planes rather than axis-aligned orthogonal planes, while retaining the EG3D decoder, super-resolution module, and discriminator architecture [S1].

## How it works
**Inputs and Outputs:**
The method takes a latent code from a StyleGAN2 mapping network as input and outputs a synthesized 2D image of a 3D object from a given camera pose [S1].

**Data Structures:**
1.  **MultiPlane Representation:** Consists of 32 planes, each with 3 channels [S1]. The planes are positioned on a sphere using an icosphere distribution [S1].
2.  **Feature Decoder:** A lightweight module that aggregates information from the 32 planes [S1].
3.  **EG3D Backbone:** Generates features using a 2D convolutional StyleGAN2 backbone [S1].
4.  **Super-Resolution Module:** Upsamples and refines raw neurally rendered images [S1].
5.  **Discriminator:** A pose-conditioned StyleGAN2 discriminator with dual discrimination [S1].

**Algorithm Steps:**
1.  **Plane Initialization:** Initialize 32 feature planes, each with 3 channels, positioned on a unit sphere using an icosphere distribution [S1].
2.  **Feature Generation:** Use a 2D convolutional StyleGAN2 backbone to generate a latent code [S1].
3.  **Aggregation:** The MultiPlane decoder aggregates information from the 32 planes to produce input for the EG3D super-resolution module [S1].
4.  **Neural Rendering:** The EG3D neural renderer aggregates features from the planes and predicts 32-channel feature images from a given camera pose [S1].
5.  **Super-Resolution:** The super-resolution module upsamples and refines the raw neurally rendered images [S1].
6.  **Discrimination:** The generated images are critiqued by the pose-conditioned StyleGAN2 discriminator [S1].
7.  **Training:** The entire pipeline is trained end-to-end from random initialization using a non-saturating GAN loss function with R1 regularization, following the StyleGAN2 training scheme [S1].
8.  **Two-Stage Strategy:** Training begins with a reduced neural rendering resolution of $64^2$ pixels, followed by a short fine-tuning period at full resolution of $128^2$ pixels [S1].
9.  **Regularization:** Additional regularization is applied to encourage smoothness of the density field to reduce artifacts in 3D shapes [S1].

**Parameters:**
*   Number of planes: 32 [S1].
*   Channels per plane: 3 [S1].
*   Reduced neural rendering resolution: $64 \times 64$ pixels [S1].
*   Full neural rendering resolution: $128 \times 128$ pixels [S1].

**Pitfalls:**
Artifacts in 3D shapes can occur; these are addressed by adding regularization to encourage smoothness of the density field [S1].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
The main handicap of MultiPlaneGAN is using 2D image-based representations [S1]. It is used for synthesizing 3D objects (e.g., ShapeNet Cars) and generalizing to unseen objects [S1].

**When to Use:**
Use when GAN-based synthesis of 3D objects is required and a spherical distribution of feature planes is preferred over axis-aligned planes.

## Background
MultiPlaneGAN is based on the EG3D GAN [S1]. EG3D uses a classical 2D generator to produce the tri-plane representation and 2D discriminator [S1]. MultiPlaneGAN replaces the tri-plane with a MultiPlane decoder to show such properties [S1]. The MultiPlane decoder can be used as a part of a larger architecture [S1].

## Sources
[S1] MultiPlaneNeRF: Neural radiance field with non-trainable representation, https://doi.org/10.1016/j.eswa.2025.127350, fetched 2026-10-29
