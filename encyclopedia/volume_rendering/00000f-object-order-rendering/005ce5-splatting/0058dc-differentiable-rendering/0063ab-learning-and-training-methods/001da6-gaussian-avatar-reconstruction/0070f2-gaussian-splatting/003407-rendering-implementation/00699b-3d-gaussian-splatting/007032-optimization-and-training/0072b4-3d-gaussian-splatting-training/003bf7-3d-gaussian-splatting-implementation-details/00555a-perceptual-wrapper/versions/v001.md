# Perceptual Wrapper

Also called: Perceptual Wrapper for 3DGS, Perceptual Wrapper with Common Randomness

## Overview
The Perceptual Wrapper (PW) is a plug-and-play 2D module that enhances the rendered outputs of existing 3D Gaussian Splatting (3DGS) representations by synthesizing high-frequency texture details in a content- and view-dependent manner [S1, unofficial]. It addresses the limitation of standard 3DGS in rendering sharp textures, particularly under memory-constrained or rate-distortion-optimized (RDO) pipelines where blurriness is exacerbated [S1, unofficial]. Unlike methods that modify the 3D primitive densification or loss functions directly (e.g., Perceptual-GS), PW operates as a post-processing synthesis network in the image domain, conditioned on pseudo-random Gaussian noise and Plücker embeddings, and is trained using Wasserstein Distortion (WD) loss [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A base rendered image $I_{base}^{v_j} \in \mathbb{R}^{3 \times H \times W}$ from view $v_j$, a multi-scale pseudo-random Gaussian noise pyramid $C$, and per-pixel 6D Plücker embeddings $P^{v_j} \in \mathbb{R}^{6 \times H \times W}$ [S1, unofficial].
*   **Output:** A residual texture detail map $R^{v_j} \in \mathbb{R}^{3 \times H \times W}$, which is added to the base image to produce the final output $I_{final}^{v_j}$ [S1, unofficial].

**Data Structures:**
1.  **Base Rendered Image ($I_{base}^{v_j}$):** The standard 3DGS rendering result for view $v_j$.
2.  **Noise Pyramid ($C$):** An 8-level spatial pyramid $C = \{C_i\}_{i=0}^7$.
    *   Level $C_i \in \mathbb{R}^{\frac{H}{2^i} \times \frac{W}{2^i}}$.
    *   $C_0$ has dimensions $H \times W$.
    *   Values are sampled independently from a standard normal distribution $\mathcal{N}(0, 1)$.
    *   The same fixed random seed is used for all views and between training/decoding (common randomness) [S1, unofficial].
3.  **Plücker Embeddings ($P^{v_j}$):** A 6-channel tensor where each pixel $(x,y)$ contains the vector $P(x,y) = (d, o \times d)$ [S1, unofficial].

**Synthesis Network Architecture:**
The network $f_\theta$ is lightweight, similar to the COOL-CHIC family [S1, unofficial]:
1.  Concatenate inputs: $I_{base}^{v_j}$, $C$ (upsampled to $H \times W$ via bilinear interpolation), and $P^{v_j}$.
2.  Two $1 \times 1$ convolutional layers.
3.  Two $3 \times 3$ residual convolutional layers.
4.  Output: $R^{v_j}$ [S1, unofficial].

**Numbered Steps:**

1.  **Compute Plücker Embeddings:** For each pixel $(x,y)$ in the target view $v_j$, calculate the normalized viewing direction $d$ and the moment vector $o \times d$:
    $$
    d = \frac{R K^{-1}[x,y,1]^T}{\|R K^{-1}[x,y, 1]^T\|} \in \mathbb{R}^3
    $$
    where $K \in \mathbb{R}^{3 \times 3}$ is the camera intrinsic matrix, $R \in \mathbb{R}^{3 \times 3}$ is the camera-to-world rotation matrix, and $o \in \mathbb{R}^3$ is the camera center in world coordinates [S1, unofficial].

2.  **Generate Noise Pyramid:** Using a fixed random seed, generate the 8-level noise pyramid $C$. Values are drawn from $\mathcal{N}(0, 1)$. This noise is view-consistent and shared across encoding/decoding [S1, unofficial].

3.  **Render Base Image:** Render the scene using the current 3DGS representation to obtain $I_{base}^{v_j}$ [S1, unofficial].

4.  **Upsample Noise:** Bilinearly upsample each level $C_i$ of the noise pyramid to the resolution $H \times W$ [S1, unofficial].

5.  **Synthesis Forward Pass:** Process the concatenated inputs through the synthesis network $f_\theta$ to generate the residual:
    $$
    R^{v_j} = f_\theta(\{I_{base}^{v_j}, C, P^{v_j}\}) \in \mathbb{R}^{3 \times H \times W}
    $$
    [S1, unofficial].

6.  **Reconstruct Final Image:** Add the residual to the base image:
    $$
    I_{final}^{v_j} = I_{base}^{v_j} + R^{v_j}
    $$
    [S1, unofficial].

**Training Procedure:**
The training is a two-stage, per-scene optimization process [S1, unofficial]:

1.  **Stage 1 (Warm-up, Iterations 0–14,999):**
    *   Optimize the underlying 3DGS representation only.
    *   Loss: $\mathcal{L} = \mathcal{L}_{3DGS-D} + \mathcal{L}_{Reg} + \lambda \mathcal{L}_{rate}$ (if applicable).
    *   The synthesis network is inactive.
    *   Duration: 15,000 iterations. This ensures stable densification and pruning of Gaussian primitives before introducing the wrapper [S1, unofficial].

2.  **Stage 2 (Joint Optimization, Iteration 15,000+):**
    *   Activate the synthesis network.
    *   Replace $\mathcal{L}_{3DGS-D}$ with Wasserstein Distortion $\mathcal{L}_{WD}$.
    *   Jointly optimize the synthesis network parameters $\theta$ and the 3DGS representation.
    *   Loss:
        $$
        \mathcal{L} = \mathcal{L}_{WD}(I_{final}^{v_j}, I_{gt}^{v_j}) + \mathcal{L}_{Reg} + \lambda \mathcal{L}_{rate}
        $$
    *   $\mathcal{L}_{WD}$ is evaluated between the reconstructed image $I_{final}^{v_j}$ and ground truth $I_{gt}^{v_j}$ across training views [S1, unofficial].

**Parameters:**
*   **Stage 1 Iterations:** 15,000 [S1, unofficial].
*   **Noise Distribution:** Standard Normal $\mathcal{N}(0, 1)$ [S1, unofficial].
*   **Pyramid Levels:** 8 ($i=0$ to $7$) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Optimization Instability:** Directly optimizing the full pipeline with WD loss from the start can destabilize 3DGS training, especially if Gaussian densification/pruning are still evolving [S1, unofficial]. The two-stage approach mitigates this.
*   **View Consistency:** The noise must be view-consistent (fixed across views) to ensure coherent texture synthesis for novel views [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The synthesis network is lightweight (two $1\times1$ and two $3\times3$ conv layers), incurring negligible overhead to rendering speed [S1, unofficial].
*   **Memory Complexity:** Adds minimal memory overhead for the synthesis network weights and the noise pyramid (which is generated on-the-fly from a seed, requiring no storage of noise parameters) [S1, unofficial].
*   **Performance:** On the Mip-NeRF 360 dataset, the lowest-rate point achieved by PW attained an Elo score on par with the highest-rate point of baseline methods (CAT-3DGS Pro* and HAC++), corresponding to a 6x reduction in bit-rate at equivalent perceptual quality [S1, unofficial].
*   **Strengths:** Plug-and-play compatibility with any existing 3DGS representation; effective at mitigating blurriness in highly textured regions under bandwidth constraints; view-dependent texture synthesis [S1, unofficial].
*   **Limitations:** Requires a two-stage training process; relies on a fixed random seed for noise generation [S1, unofficial].
*   **When to use:** When perceptual quality is prioritized over strict pixel-wise fidelity, or when operating under strict storage/transmission bandwidth constraints [S1, unofficial].
*   **When not to use:** When strict photometric accuracy (e.g., PSNR) is the primary metric, as WD loss optimizes for perceptual similarity rather than pixel-wise error [S1, unofficial].

## Background
The Perceptual Wrapper was introduced by H.-B. Yang et al. in the paper "Seeing What Matters: Perceptual Wrapper with Common Randomness for 3D Gaussian Splatting" [S1, unofficial]. It addresses the limitations of previous perceptual methods like Perceptual-GS, which used sensitivity-guided densification and conventional L1/SSIM losses [S1, unofficial]. PW leverages insights from image and video compression communities regarding objective quality metrics that correlate with human perception, specifically using Wasserstein Distortion [S1, unofficial]. It was implemented in PyTorch and tested on NVIDIA Tesla V100 GPUs, evaluated on Mip-NeRF 360, Tanks & Temples, and Deep Blending datasets [S1, unofficial].

## Sources
[S1] Seeing What Matters: Perceptual Wrapper with Common Randomness for 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2606.11782, fetched 2026-10-29 (unofficial)
