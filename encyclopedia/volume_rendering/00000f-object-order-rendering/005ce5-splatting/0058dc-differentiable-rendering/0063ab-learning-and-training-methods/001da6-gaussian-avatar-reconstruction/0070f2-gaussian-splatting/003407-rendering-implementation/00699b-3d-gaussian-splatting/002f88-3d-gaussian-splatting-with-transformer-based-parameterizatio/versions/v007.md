# 3D Gaussian Splatting with Transformer-based Parameterization

Also called: GS-LRM

## Overview
This method is a feed-forward neural network approach that reconstructs a 3D Gaussian Splatting scene from a small set of input images by leveraging a Transformer architecture. It solves the problem of sparse-view reconstruction by tokenizing input views and camera parameters, processing them through a Transformer block, and directly decoding the resulting tokens into 3D Gaussian parameters. It sets itself apart from standard 3D Gaussian Splatting [2812-3d-gaussian-splatting.md] by replacing iterative optimization with a direct inference pipeline, and from pixel-aligned methods like MVSplat [S1] by operating in a global token space rather than per-pixel ray alignment.

## How it works
The method functions as a multi-view feed-forward network. It takes a small set of input images and their corresponding camera parameters as input and outputs a set of 3D Gaussians (position, rotation, scale, opacity, and spherical harmonic coefficients for color) that represent the scene.

**Data Structures:**
1.  **Input Tokens:** A sequence of tokens representing the input images and camera poses.
2.  **Transformer Output:** A sequence of latent tokens produced by the Transformer encoder/decoder blocks.
3.  **Gaussian Parameters:** The final output, typically a tensor or list of Gaussian primitives.

**Steps:**
1.  **Tokenization:** The method takes $N$ input images (typically $N=2$ to $4$) and their associated camera parameters. These are converted into a unified token sequence. The images are likely processed through a visual encoder (e.g., ViT) to extract visual features, which are then concatenated with embeddings representing the camera intrinsics and extrinsics [S1].
2.  **Transformer Processing:** The concatenated token sequence is passed through a Transformer block. The Transformer attends to relationships between the different views and camera poses, aggregating multi-view information into the latent tokens [S1].
3.  **Decoding:** The output tokens from the Transformer are decoded into the parameters of the 3D Gaussians. This involves a linear or MLP-based projection from the latent token space to the Gaussian parameter space (position, scale, rotation, opacity, color) [S1].

**Formulas:**
Let $I_1, ..., I_N$ be the input images and $C_1, ..., C_N$ be the camera parameters.
The input is tokenized into a sequence $T_{in}$.
The Transformer computes $T_{out} = \text{Transformer}(T_{in})$.
The Gaussian parameters $G$ are decoded as $G = \text{Decoder}(T_{out})$.

**Parameters:**
-   Number of input views: Typically 2 to 4 [S1].
-   Transformer depth and width: Not given in the sources.

**Edge Cases:**
-   **Scale Ambiguity:** If camera intrinsics are not properly encoded, scale ambiguity may arise. NoPoSplat [S1] addresses this by introducing intrinsics token embeddings, but the base GS-LRM method assumes known poses [S1].
-   **Sparse Views:** Performance may degrade if the number of input views is too small or if overlap is minimal, though the Transformer's attention mechanism helps mitigate this compared to pixel-aligned methods [S1].

## Subtopics

### [VolSplat](7674-volsplat.md)
Replaces pixel-aligned prediction with a voxel-aligned strategy, aggregating multi-view features into a shared 3D voxel grid before regressing Gaussian parameters.

### [Sequential Gaussian Attribute Decoding](16432-sequential-gaussian-attribute-decoding.md)
Decodes Gaussian attributes (position, rotation, scale, opacity, color) in a fixed sequential order from tri-plane features to reduce degrees of freedom.

### [Gaussian Decoding from Scene Tokens](17342-gaussian-decoding-from-scene-tokens.md)
Maps learned scene tokens into freely placed 3D Gaussians, decoupling placement from fixed camera rays for flexible scene representation.

## Operating characteristics
-   **Time Complexity:** Inference time is constant with respect to scene complexity once the model is trained, depending primarily on the number of input views and the Transformer size. It is significantly faster than iterative optimization methods like standard 3D Gaussian Splatting.
-   **Memory Complexity:** Memory usage is dominated by the Transformer model weights and the intermediate feature maps. The output Gaussian count is determined by the decoding head and is not explicitly constrained by the input resolution in the same way as pixel-aligned methods.
-   **Strengths:** Fast inference (feed-forward); handles sparse views better than pixel-aligned methods due to global context aggregation; no need for per-scene optimization.
-   **Limitations:** Quality may be lower than optimized 3D Gaussian Splatting for dense views; requires training a large neural network; potential for floaters or alignment errors if the Transformer fails to resolve multi-view consistency [S1].
-   **When to use:** When fast reconstruction from a few images is required, or when iterative optimization is computationally prohibitive.
-   **When not to use:** When high-fidelity reconstruction from many views is needed and optimization time is acceptable.

## Background
The method is referred to as GS-LRM in the literature [S1]. It builds upon earlier feed-forward networks like MVSplat [S1] and PixelSplat. GS-LRM specifically tokenizes images and camera parameters, processes them with a Transformer, and decodes them into Gaussian parameters, representing a shift from pixel-aligned to token-aligned Gaussian prediction [S1]. NoPoSplat [S1] is a related variant that removes the need for known camera poses.

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
