# 3D Gaussian Splatting with Transformer-based Parameterization

Also called: GS-LRM

## Overview
This method reconstructs a 3D Gaussian Splatting scene from a small set of input images using a Transformer architecture [S1]. It tokenizes input views and camera parameters, processes them through a Transformer block, and directly decodes the resulting tokens into 3D Gaussian parameters [S1].

## How it works
**Inputs:** $N$ input images ($I_1, \dots, I_N$) and their corresponding camera parameters ($C_1, \dots, C_N$), where $N$ is typically 2 to 4 [S1].
**Outputs:** A set of 3D Gaussian parameters (position, rotation, scale, opacity, and color) [S1].
**Data Structures:**
1. **Input Tokens:** A concatenated sequence of image features and camera parameter embeddings [S1].
2. **Latent Tokens:** The output sequence from the Transformer block [S1].
3. **Gaussian Parameters:** The decoded primitive attributes [S1].
**Steps:**
1. **Tokenization:** The $N$ input images and their camera parameters are tokenized and concatenated into a single sequence $T_{\text{in}}$ [S1].
2. **Transformer Processing:** The sequence $T_{\text{in}}$ is passed through a Transformer block to produce latent tokens $T_{\text{out}}$ [S1].
3. **Decoding:** The latent tokens $T_{\text{out}}$ are decoded into the final 3D Gaussian parameters $G$ [S1].
**Formulas:**
$$T_{\text{in}} = \text{Concat}(\text{Tokenize}(I_1, \dots, I_N), \text{Tokenize}(C_1, \dots, C_N))$$
$$T_{\text{out}} = \text{Transformer}(T_{\text{in}})$$
$$G = \text{Decoder}(T_{\text{out}})$$
where $T_{\text{in}}$ is the concatenated input token sequence, $\text{Tokenize}$ converts images and camera parameters into embeddings, $\text{Transformer}$ is the processing block, $T_{\text{out}}$ is the latent token sequence, $\text{Decoder}$ maps latent tokens to Gaussian attributes, and $G$ is the output parameter set [S1].
**Parameters:**
- Number of input views: Typically 2 to 4 [S1].
- Transformer depth and width: Not given in the sources.
**Edge Cases:**
- Scale Ambiguity: If camera intrinsics are not properly encoded, scale ambiguity may arise; NoPoSplat addresses this by introducing intrinsics token embeddings, while the base method assumes known poses [S1].

## Subtopics

### [VolSplat](7674-volsplat.md)
Replaces pixel-aligned prediction with a voxel-aligned strategy, aggregating multi-view features into a shared 3D voxel grid before regressing Gaussian parameters.

### [Sequential Gaussian Attribute Decoding](16432-sequential-gaussian-attribute-decoding.md)
Decodes Gaussian attributes (position, rotation, scale, opacity, color) in a fixed sequential order from tri-plane features to reduce degrees of freedom.

### [Gaussian Decoding from Scene Tokens](17342-gaussian-decoding-from-scene-tokens.md)
Maps learned scene tokens into freely placed 3D Gaussians, decoupling placement from fixed camera rays for flexible scene representation.

## Operating characteristics
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.
- Strengths: Not given in the sources.
- Limitations: Not given in the sources.
- When to use: Not given in the sources.
- When not to use: Not given in the sources.

## Background
The method is referred to as GS-LRM in the literature [S1]. It builds upon earlier feed-forward networks like MVSplat and PixelSplat [S1]. GS-LRM specifically tokenizes images and camera parameters, processes them with a Transformer, and decodes them into Gaussian parameters, representing a shift from pixel-aligned to token-aligned Gaussian prediction [S1]. NoPoSplat is a related variant that removes the need for known camera poses [S1].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
