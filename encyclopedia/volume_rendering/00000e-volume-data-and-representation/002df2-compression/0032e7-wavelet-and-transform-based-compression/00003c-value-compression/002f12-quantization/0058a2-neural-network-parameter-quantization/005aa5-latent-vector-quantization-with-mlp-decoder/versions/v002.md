# Latent Vector Quantization with MLP Decoder

Also called: Quantized Latent Encoding

## Overview
Latent Vector Quantization with MLP Decoder is a compression technique for 3D Gaussian attributes where per-point data is encoded into a discrete latent vector and reconstructed via a Multi-Layer Perceptron (MLP) [S1]. It differs from direct quantization or complex probabilistic models by using a lightweight, fixed decoder to map integer indices to continuous attributes, avoiding the runtime GPU memory overhead and training speed penalties associated with learnable probability models or Gumbel annealing [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous attribute vectors $\boldsymbol{a} \in \mathbb{R}^k$ for a set of points. The attribute set is $A = \{p, s, r, c, o\}$, representing position, scale, rotation, color, and opacity [S1].
*   **Output:** A quantized latent vector $\boldsymbol{q} \in \mathbb{Z}^l$ of integer values, and the MLP decoder parameters $D$ [S1].

**Data Structures:**
*   **Latent Vector ($\boldsymbol{q}$):** A vector of dimension $l$ containing integer values [S1].
*   **Decoder ($D$):** An MLP function mapping $\mathbb{Z}^l \rightarrow \mathbb{R}^k$ that reconstructs attributes from the latent indices [S1].

**Steps:**
1.  **Initialization:** Initialize the MLP decoder $D$ and the latent vectors $\boldsymbol{q}$ for each point, while maintaining continuous approximations $\widehat{\boldsymbol{q}}$ for training [S1].
2.  **Forward Pass (Reconstruction):** During training, the continuous approximation $\widehat{\boldsymbol{q}}$ is passed through the Straight-Through Estimator (STE) to obtain the discrete index $\boldsymbol{q}$ [S1]. The decoder $D$ processes this to produce the attribute $\boldsymbol{a}$:
    $$ \boldsymbol{a} = D(STE(\widehat{\boldsymbol{q}})) $$
    where $STE(\widehat{\boldsymbol{q}})$ rounds $\widehat{\boldsymbol{q}}$ to the nearest integer [S1].
3.  **Backpropagation:** Gradients are computed with respect to the continuous approximation $\widehat{\boldsymbol{q}}$ [S1]. The STE allows gradients to pass through the rounding operation directly to $\widehat{\boldsymbol{q}}$, enabling end-to-end training similar to standard 3D Gaussian Splatting procedures [S1].
4.  **Post-Training Compression:** After training, $\widehat{\boldsymbol{q}}$ is rounded to the nearest integer $\boldsymbol{q}$ [S1]. Entropy coding is applied to the resulting discrete latents for efficient storage alongside the decoder $D$ [S1].

**Parameters:**
*   **Latent Dimension ($l$):** Not given in the sources [S1].
*   **Quantized Attributes:** Color ($c$) and Rotation ($r$) are quantized to reduce memory footprint [S1]. Opacity ($o$) is quantized to improve optimization [S1].
*   **Non-Quantized Attributes:** Position ($p$), Scale ($s$), and Base Band Color SH coefficients are **not** quantized [S1]. They are sensitive to initialization, and quantizing them causes large performance drops [S1].

**Pitfalls:**
*   **Performance Drop:** Quantizing position, scale, or base band SH coefficients leads to significant quality degradation due to sensitivity to initialization [S1].
*   **Opacity Saturation:** Without opacity quantization, large positive gradients saturate opacity to 1, creating rasterization artifacts that are not removed by pruning [S1].

## Subtopics

### Opacity Quantization
Opacity quantization serves as a soft regularizer during optimization [S1]. By requiring multiple gradient updates to move an opacity value from one quantization bin to the next, it prevents the saturation of opacity values to 1 caused by large positive gradients, thereby reducing artifacts in rendered views [S1].

## Operating characteristics
*   **Memory:** Reduces memory footprint for color, rotation, and opacity attributes by storing integer latents instead of full floating-point vectors [S1].
*   **Efficiency:** Avoids the large overhead in runtime GPU memory and training speed associated with complex decoders, learnable probability models, or Gumbel annealing [S1].
*   **Quality:** Maintains reconstruction quality comparable to standard methods while reducing artifacts caused by opacity saturation [S1].
*   **Limitations:** Not all attributes can be quantized; position, scale, and base band SH coefficients must remain unquantized to maintain performance [S1].

## Background
This method is introduced in the EAGLES (Efficient Accelerated 3D Gaussians with Lightweight EncodingS) paper [S1]. It addresses the high memory requirements of 3D Gaussian Splatting by proposing lightweight quantization for per-point attributes, specifically targeting color, rotation, and opacity [S1].

## Sources
[S1] EAGLES: Efficient Accelerated 3D Gaussians with Lightweight EncodingS, https://doi.org/10.1007/978-3-031-73036-8_4, fetched 2026-10-27
