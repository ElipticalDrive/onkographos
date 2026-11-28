# ViT-NeRF

Also called: Global-Local Feature NeRF, Hybrid ViT-CNN NeRF, Vision Transformer for NeRF (ViT-NeRF)

## Overview
ViT-NeRF is a view synthesis method that combines a Vision Transformer (ViT) to encode global shape information and a 2D CNN to extract local appearance features, conditioning volume rendering on hybrid per-pixel features [S1]. It addresses the limitation of pure global feature methods (which suffer from inconsistent appearance near input views) and pure local feature methods (which lack long-range semantic consistency) by fusing both representations [S1]. Unlike CodeNeRF and DISN, it does not require a canonical coordinate system to utilize global features, and compared to SRN and PixelNeRF, it produces sharper edges and clearer structures, particularly on invisible parts like chair legs [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single input image $I_s \in \mathbb{R}^{H \times W \times 3}$, where $H$ and $W$ are height and width [S1].
*   **Output:** A rendered 2D image $\hat{C}(r)$ for a target viewpoint defined by ray origin $o$ and direction $d$ [S1].

**Data Structures:**
*   **Image Tokens:** Sequence of flattened 2D patches $P \in \mathbb{R}^{N \times P^2 \times 3}$, where $N = \frac{HW}{P^2}$ is the number of patches and $P$ is the patch size [S1].
*   **Latent Features:** Set of latent features $f$ extracted by the ViT encoder [S1].
*   **Multi-level Feature Maps:** A set of global feature maps $W^0_G, ..., W^J_G$ at different resolutions [S1].
*   **Local Feature Map:** $W_L \in \mathbb{R}^{\frac{H}{2} \times \frac{W}{2} \times D_L}$ extracted by the CNN [S1].
*   **Hybrid Feature Map:** $W$, the fused result of global and local features [S1].
*   **Per-pixel Feature:** $W(\pi(x))$, sampled from $W$ at the 2D projection $\pi(x)$ of a 3D point $x$ [S1].

**Numbered Steps:**

1.  **Tokenization:** Reshape input image $I_s$ into a sequence of flattened 2D patches. Linearly project each patch to an image token $P_l$ [S1].
2.  **Global Encoding:** Input image tokens and learnable positional embeddings $e$ into the Transformer Encoder to extract global latent features $f$ [S1].
3.  **Class Token Dropping:** Define operation $O: \mathbb{R}^{(N+1) \times D} \rightarrow \mathbb{R}^{N \times D}$ to drop the class token from $f$. The class token is useful for self-attention but has no physical meaning when unflattened [S1].
4.  **Unflattening:** Define operation $U: \mathbb{R}^{N \times D} \rightarrow \mathbb{R}^{\frac{H}{P} \times \frac{W}{P} \times D}$ to unflatten the remaining tokens into feature patches $P_f = \{P^0_f, ..., P^J_f\}$, where $P^j_f \in \mathbb{R}^{\frac{H}{P} \times \frac{W}{P} \times D}$ [S1].
5.  **Global Feature Map Generation:** For each level $j \in \{0, ..., J\}$, generate global feature maps $W^j_G$ using a convolutional decoder $D$:
    $$W^j_G = (D \circ U \circ O)(f^j)$$
    The decoder $D: \mathbb{R}^{\frac{H}{P} \times \frac{W}{P} \times D} \rightarrow \mathbb{R}^{H_j \times W_j \times D_j}$ performs: (i) $1 \times 1$ convolution, (ii) resampling via strided or transposed convolution to size $H_j \times W_j$, and (iii) $3 \times 3$ convolution to $D_j$ channels [S1].
6.  **Local Feature Extraction:** Extract local features $W_L$ using a 2D CNN module $G_L$:
    $$W_L = G_L(I_s)$$
    $G_L$ consists of three ResBlocks. $W_L \in \mathbb{R}^{\frac{H}{2} \times \frac{W}{2} \times D_L}$ [S1].
7.  **Hybrid Feature Fusion:** Fuse multi-level global features $W^0_G, ..., W^J_G$ and local feature $W_L$ using a convolutional layer $G$ to produce the hybrid feature map $W$:
    $$W = G(W^0_G, W^1_G, ..., W^J_G; W_L)$$
    [S1].
8.  **Ray Sampling:** For a target view, sample a ray $r(t) = o + td$ bounded by near plane $t_{near}$ and far plane $t_{far}$. Pick equally distributed samples $x_c$ along the ray [S1].
9.  **Feature Sampling:** For each 3D sample location $x_c$, compute its 2D projection $\pi(x)$ on the input image plane and sample the hybrid feature $W(\pi(x))$ [S1].
10. **Positional Encoding:** Encode the 3D location $x_c$ using positional encoding $\gamma(x_c)$ with $M=10$ frequency bases:
    $$\gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), ..., \sin(2^{M-1} \pi p), \cos(2^{M-1} \pi p))$$
    [S1].
11. **MLP Prediction:** Input $\gamma(x_c)$, viewing direction $d_c$, and per-pixel feature $W(\pi(x))$ into an MLP to output density $\sigma$ and color $c$:
    $$(\sigma, c) = \text{MLP}(\gamma(x_c); d_c; W(\pi(x)))$$
    [S1].
12. **Volume Rendering:** Render the target view using standard volume rendering integral:
    $$\hat{C}(r) = \int_{t_n}^{t_f} T(t)\sigma(t)c(t)dt$$
    where $T(t) = \exp(-\int_{t_n}^{t} \sigma(r(s))ds)$ is the accumulated transmittance. Approximate the integral with quadrature [S1].

**Parameters:**
*   **Frequency bases ($M$):** 10 [S1].
*   **Learning Rate:** $10^{-4}$ for the NeRF MLP; $10^{-5}$ for the ViT and CNN modules [S1].
*   **Batch Size:** 8 instances, with 512 rays per instance [S1].
*   **ViT Initialization:** Initialized from pretrained weights of reference [55] and fine-tuned [S1].
*   **Warm-up:** Linear warm-up from 0 learning rate for the first 10k steps [S1].

**Pitfalls:**
*   Using only global ViT features compromises rendering quality for views close to the input view (inconsistent color/appearance). This is solved by adding the local CNN branch [S1].
*   The class token in ViT has no physical meaning when unflattened into spatial feature maps and must be dropped before decoding [S1].

## Operating characteristics
*   **Hardware:** Evaluated on 16 NVIDIA A100 GPUs [S1].
*   **Training Time:** Converges at 500K iterations [S1].
*   **Strengths:** Produces sharper edges and clearer structures than SRN and PixelNeRF, avoiding blurriness on invisible parts like chair legs [S1]. Combines the advantage of 2D-based methods (local detail) and 1D-based methods (global information) [S1].
*   **Limitations:** Volume-based methods are computationally intensive and require more memory when increasing grid size [S1].

## Background
ViT-NeRF was introduced in the paper "Vision Transformer for NeRF-Based View Synthesis from a Single Input Image" [S1]. It builds on the success of Vision Transformers in learning long-range dependencies between features, applying this to the NeRF framework to improve reconstruction quality in unseen regions [S1].

## Sources
[S1] Vision Transformer for NeRF-Based View Synthesis from a Single Input Image, https://doi.org/10.1109/wacv56688.2023.00087, fetched 2026-10-27
