# 3D Gaussian Splatting with Transformer-based Parameterization


## Overview
This method reconstructs 3D Gaussian primitives from a small set of input images by tokenizing the images and their camera parameters, concatenating them, and passing them through a Transformer block whose output tokens are decoded into Gaussian parameters [S1]. It builds on [3D Gaussian Splatting](1795-3d-gaussian-splatting.md) by replacing iterative optimization with a direct feed-forward Transformer parameterization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** $N$ input images and their corresponding camera parameters, where $N=2$ to $4$ [S1].
*   **Output:** Gaussian parameters decoded from the Transformer's output tokens [S1].

**Data Structures:**
The input is processed as a concatenated sequence of image tokens and camera parameter embeddings [S1].

**Steps:**
1.  Tokenize the $N$ input images and encode their camera parameters into embeddings.
2.  Concatenate the image tokens and camera embeddings into a single sequence.
3.  Pass the concatenated sequence through a Transformer block.
4.  Decode the output tokens into Gaussian parameters.

**Formulas:**
Let $I_i$ be the $i$-th input image and $C_i$ be its camera parameters. The input sequence $X$ is formed by concatenating tokens from $I_i$ and embeddings of $C_i$:
$$ X = \text{Concat}(\text{Tokens}(I_1), \text{Embed}(C_1), \dots, \text{Tokens}(I_N), \text{Embed}(C_N)) $$
The Transformer encoder $E$ processes $X$:
$$ Z = E(X) $$
The decoder $D$ maps $Z$ to Gaussian parameters $\Theta$:
$$ \Theta = D(Z) $$

**Parameters:**
*   Number of input images: Typically 2 to 4 [S1].
*   Transformer depth and width: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Scale Ambiguity:** In scenarios with minimal view overlap, scale ambiguity can arise; related methods address this with canonical coordinate systems and intrinsics token embeddings, but specific handling for this method is not detailed in the sources [S1].
*   **Parameter Dimensions:** The exact dimensions of the predicted Gaussian parameters (e.g., mean, covariance, opacity, color coefficients) are not given in the sources.

## Subtopics

### [VolSplat](7674-volsplat.md)
VolSplat is a multi-view feed-forward framework that replaces pixel-aligned Gaussian prediction with a voxel-aligned prediction strategy. It aggregates multi-view features into a shared 3D voxel grid before regressing Gaussian parameters, enabling adaptive density control based on volumetric scene complexity. Choose it when volumetric scene complexity requires explicit 3D feature aggregation rather than direct ray-aligned prediction.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Direct feed-forward parameterization from a few views [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method corresponds to the GS-LRM architecture, which tokenizes two to four images and their camera parameters, concatenates them, and passes them through a Transformer block to decode Gaussian parameters [S1]. It is discussed alongside related feed-forward multi-view methods such as MVSplat, NoPoSplat, and CoR-GS [S1].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
