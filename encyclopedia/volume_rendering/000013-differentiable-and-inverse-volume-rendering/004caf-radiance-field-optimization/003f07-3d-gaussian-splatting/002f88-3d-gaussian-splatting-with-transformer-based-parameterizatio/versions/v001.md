# 3D Gaussian Splatting with Transformer-based Parameterization

Also called: GS-LRM

## Overview
This method enables feed-forward reconstruction of 3D Gaussian primitives from a small set of input images by leveraging a Transformer architecture to predict Gaussian parameters directly [S1]. It distinguishes itself from iterative optimization-based 3D Gaussian Splatting by eliminating the need for per-scene training, instead learning a direct mapping from image space to Gaussian space [S1]. Unlike pixel-aligned feed-forward methods such as MVSplat, which rely on cost volumes and U-Net decoders, this approach tokenizes images and camera parameters to predict Gaussians aligned along camera-to-pixel rays, offering a more direct parameterization [S1].

## How it works
The method operates as a feed-forward neural network that takes a small set of input views and outputs a set of 3D Gaussian primitives.

**Inputs and Outputs:**
*   **Input:** A set of $N$ input images (typically $N=2$ to $4$) and their corresponding camera parameters (extrinsics and intrinsics).
*   **Output:** A set of 3D Gaussian primitives, each defined by position, covariance, opacity, and spherical harmonics coefficients for color.

**Data Structures:**
The input is processed as a sequence of tokens. Each token represents an image patch or the entire image along with associated camera embedding tokens. The Transformer encoder processes these concatenated tokens. The decoder maps the resulting output tokens to the Gaussian parameter space.

**Steps:**
1.  **Tokenization:** The input images are divided into patches or processed as global tokens. Camera parameters (extrinsics and intrinsics) are encoded into token embeddings. These are concatenated to form the input sequence for the Transformer [S1].
2.  **Transformer Encoding:** The concatenated tokens are passed through a Transformer block. The self-attention mechanism allows the model to aggregate multi-view information and resolve geometric consistency [S1].
3.  **Decoding:** The output tokens from the Transformer are decoded into Gaussian parameters. This involves predicting the mean position, covariance matrix, opacity, and color coefficients for each Gaussian primitive [S1].
4.  **Rendering:** The predicted Gaussians are rendered using differentiable rasterization to produce the output image, which can be used for training the feed-forward network via a loss function comparing the rendered image to the ground truth input views [S1].

**Formulas:**
Let $I_i$ be the $i$-th input image and $C_i$ be its camera parameters. The input sequence $X$ is formed by concatenating tokens from $I_i$ and embeddings of $C_i$:
$$ X = \text{Concat}(\text{Tokens}(I_1), \text{Embed}(C_1), \dots, \text{Tokens}(I_N), \text{Embed}(C_N)) $$
The Transformer encoder $E$ processes $X$:
$$ Z = E(X) $$
The decoder $D$ maps $Z$ to Gaussian parameters $\Theta$:
$$ \Theta = D(Z) $$
The rendered image $I_{render}$ is obtained by rasterizing $\Theta$:
$$ I_{render} = \text{Rasterize}(\Theta) $$

**Parameters:**
*   Number of input images: Typically 2 to 4 [S1].
*   Transformer depth and width: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Scale Ambiguity:** In scenarios with minimal view overlap, scale ambiguity can arise. This is addressed in related methods like NoPoSplat by using a canonical coordinate system and intrinsics token embeddings, but the specific handling in GS-LRM is not detailed in the sources [S1].
*   **Multi-view Alignment:** The method aims to resolve multi-view alignment issues inherent in pixel-aligned methods by using the Transformer's global attention [S1].

## Subtopics

### [VolSplat](7674-volsplat.md)
VolSplat is a feed-forward method for 3D Gaussian Splatting (3DGS) that regresses Gaussian primitives directly from 3D features using a voxel-aligned prediction strategy. It solves the limitations of pixel-aligned feed-forward methods, specifically the struggle to resolve multi-view alignment and the inability to adaptively control Gaussian density according to scene complexity. By unprojecting 2D features into a 3D voxel grid, it achieves adaptive density control and robust multi-view consistency.

## Operating characteristics
*   **Time Complexity:** Feed-forward inference is significantly faster than iterative optimization-based 3DGS, as it does not require per-scene training [S1].
*   **Memory Complexity:** Depends on the Transformer size and the number of predicted Gaussians. Not given in the sources.
*   **Strengths:** Real-time reconstruction from few views; no per-scene optimization needed [S1].
*   **Limitations:** Quality may be lower than iterative methods, especially in complex scenes or with very sparse views. Not given in the sources.
*   **When to use:** When fast reconstruction from a few images is required.
*   **When not to use:** When high-fidelity reconstruction is critical and computational time for optimization is available.

## Background
The method is part of the broader trend of feed-forward neural radiance fields and Gaussian splatting, aiming to replace iterative optimization with direct regression. It is closely related to MVSplat and GS-LRM, which also use Transformers for multi-view stereo and Gaussian prediction [S1]. The specific formulation of tokenizing images and camera parameters for Gaussian prediction is attributed to GS-LRM [S1].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
