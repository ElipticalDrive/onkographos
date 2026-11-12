# Large Reconstruction Model (LRM)
Also called: Adobe 2023, LRM (Large Reconstruction Model)

## Overview
LRM is a feedforward neural network that performs single-image to 3D reconstruction by predicting a triplane neural representation in one transformer pass [S1, unofficial]. It solves the problem of slow per-scene optimization by replacing iterative fitting with a direct inference step [S2, unofficial]. It is distinguished from its siblings by outputting a triplane representation rather than explicit Gaussians (like GS-LRM) or meshes (like InstantMesh) [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: A single RGB image $I \in \mathbb{R}^{H \times W \times 3}$.
- Output: A triplane representation $\mathcal{T} \in \mathbb{R}^{3 \times C \times H_t \times W_t}$, where 3 denotes the XY, YZ, and XZ planes, $C$ is the channel dimension, and $H_t, W_t$ are the triplane resolution.

**Data structures:**
- **Triplane**: A tensor of shape $(3, C, H_t, W_t)$. Each plane corresponds to a 2D slice of the 3D space.
- **Query**: A 3D coordinate $\mathbf{x} \in \mathbb{R}^3$ normalized to $[-1, 1]^3$.

**Steps:**
1. **Image Encoding**: The input image $I$ is processed by a Vision Transformer (ViT) backbone to produce image tokens [S1, unofficial].
2. **Transformer Processing**: The image tokens are concatenated with learnable positional embeddings and processed by a transformer encoder/decoder to model spatial relationships [S1, unofficial].
3. **Triplane Decoding**: The transformer output is decoded into the triplane representation $\mathcal{T}$ [S1, unofficial].
4. **Neural Rendering (Implicit)**: To render a novel view, 3D points $\mathbf{x}$ are sampled along rays. For each point, coordinates are projected onto the three planes of $\mathcal{T}$ to extract features via bilinear interpolation. These features are aggregated and passed through a small MLP to predict color and density [S1, unofficial].
5. **Differentiable Rendering**: Standard volume rendering is applied to the predicted densities and colors to produce the output image [S1, unofficial].

**Training:**
- The model is trained end-to-end using photometric losses (e.g., LPIPS, SSIM) between rendered novel views and ground truth images [S1, unofficial].
- Training data: Objaverse and Objaverse-XL datasets [S1, unofficial].

**Parameters:**
- Triplane resolution $H_t, W_t$: Not given in the sources.
- Channel dimension $C$: Not given in the sources.

**Edge cases and pitfalls:**
- The output is a neural representation, not explicit geometry. Extracting meshes requires marching cubes on the density field [S1, unofficial].
- Single-image input may lead to ambiguities in unseen views, mitigated by training on large-scale datasets [S2, unofficial].

## Subtopics

### [Large Gaussian Reconstruction Model (GRM)](5961-large-gaussian-reconstruction-model-grm.md)
Predicts 3D Gaussians directly instead of a triplane, supporting both single and multi-view inputs.

## Operating characteristics
- **Time complexity**: $O(1)$ with respect to scene complexity per inference pass, as it is a feedforward model [S1, unofficial].
- **Memory complexity**: Depends on the triplane size and transformer depth. Not given in the sources.
- **Performance**: Real-time inference on a single GPU [S1, unofficial].
- **Strengths**: Fast reconstruction without iterative optimization [S2, unofficial].
- **Limitations**: Output is a neural representation, requiring additional steps for mesh extraction [S1, unofficial].

## Background
- **Origin**: LRM was introduced by Adobe in 2023 [S1, unofficial].
- **Paper**: "LRM: Large Reconstruction Model for Single Image to 3D" [S2, unofficial].
- **Development**: It represents a shift from per-scene optimization to feedforward prediction, building on the success of transformers in 2D vision [S1, unofficial].

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
