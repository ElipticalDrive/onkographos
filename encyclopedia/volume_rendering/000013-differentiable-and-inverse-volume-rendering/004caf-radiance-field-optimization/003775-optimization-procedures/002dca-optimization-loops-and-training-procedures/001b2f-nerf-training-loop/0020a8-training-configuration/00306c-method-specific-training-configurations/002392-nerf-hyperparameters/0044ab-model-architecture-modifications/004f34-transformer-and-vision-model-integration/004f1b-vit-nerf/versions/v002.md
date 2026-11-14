# ViT-NeRF

Also called: Global-Local Feature NeRF, Hybrid ViT-CNN NeRF, Vision Transformer for NeRF (ViT-NeRF)

## Overview
ViT-NeRF is a view synthesis method that conditions volume rendering on hybrid per-pixel features extracted from a single input image [S1]. It combines a Vision Transformer (ViT) to encode global shape information with a 2D CNN to extract local appearance details, addressing the blurriness and color inconsistency that occur when using only global features near the input view [S1]. Unlike methods requiring a canonical coordinate system, ViT-NeRF operates directly on image-based features, producing sharper edges and clearer structures in unseen regions compared to siblings like PixelNeRF [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single input image $I_s \in \mathbb{R}^{H \times W \times 3}$, where $H$ and $W$ are height and width [S1].
*   **Output:** A rendered 2D image for a target viewpoint [S1].

**Data Structures:**
*   **Image Tokens:** Sequence of flattened 2D patches $P \in \mathbb{R}^{N \times P^2 \times 3}$, where $N = \frac{HW}{P^2}$ is the number of patches and $P$ is the patch size [S1].
*   **Latent Features:** Set of latent features $f$ extracted by the ViT encoder [S1].
*   **Multi-level Feature Maps:** Global feature maps $W^0_G, ..., W^J_G$ at various resolutions [S1].
*   **Local Feature Map:** $W_L \in \mathbb{R}^{\frac{H}{2} \times \frac{W}{2} \times D_L}$ extracted by the CNN [S1].
*   **Hybrid Feature Map:** $W$, the fused result of global and local features [S1].

**Numbered Steps:**

1.  **Global Feature Extraction:**
    *   Reshape $I_s$ into a sequence of flattened 2D patches [S1].
    *   Linearly project each patch to an image token $P_l$ [S1].
    *   Pass image tokens and learnable positional embeddings $e$ through a Transformer Encoder to extract global information as latent features $f$ [S1].

2.  **Feature Map Construction:**
    *   Drop the class token from the latent features. The class token is useful for self-attention but lacks physical meaning when unflattened into spatial maps [S1]. Define operation $O: \mathbb{R}^{(N+1) \times D} \rightarrow \mathbb{R}^{N \times D}$ [S1].
    *   Unflatten the remaining tokens into feature patches $P_f = \{P^0_f, ..., P^J_f\}$, where $P^j_f \in \mathbb{R}^{\frac{H}{2^j} \times \frac{W}{2^j} \times D}$ [S1]. Define unflatten operation $U: \mathbb{R}^{N \times D} \rightarrow \mathbb{R}^{\frac{H}{2^j} \times \frac{W}{2^j} \times D}$ [S1].
    *   Generate multi-level global feature maps $W^j_G$ using a convolutional decoder $D$ for each level $j \in \{0, ..., J\}$ [S1]:
        $$W^j_G = (D \circ U \circ O)(f^j)$$
        The decoder $D: \mathbb{R}^{\frac{H}{2^j} \times \frac{W}{2^j} \times D} \rightarrow \mathbb{R}^{\frac{H}{2^j} \times \frac{W}{2^j} \times D_j}$ consists of: (i) a $1 \times 1$ convolution, (ii) resampling via strided or transposed convolution to size $\frac{H}{2^j} \times \frac{W}{2^j}$, and (iii) a $3 \times 3$ convolution to produce $D_j$ channels [S1].

3.  **Local Feature Extraction:**
    *   Extract local appearance features using a 2D CNN module $G_L$ [S1]:
        $$W_L = G_L(I_s), \quad G_L: \mathbb{R}^{H \times W \times 3} \rightarrow \mathbb{R}^{\frac{H}{2} \times \frac{W}{2} \times D_L}$$
        $G_L$ consists of three ResBlocks [S1]. This step prevents the blurriness and color inconsistency associated with using only global ViT features for views close to the input [S1].

4.  **Feature Fusion:**
    *   Fuse the multi-level global features $W^0_G, ..., W^J_G$ and the local feature map $W_L$ using a convolutional layer $G$ to produce the hybrid feature map $W$ [S1]:
        $$W = G(W^0_G, W^1_G, ..., W^J_G; W_L)$$
        [S1]

5.  **Conditioned Rendering:**
    *   Sample a ray $r(t) = o + td$ from the target viewpoint, bounded by $t_{near}$ and $t_{far}$ [S1].
    *   Pick equally distributed samples along the ray. For a 3D sample location $x_c$ in camera coordinates, extract the per-pixel feature $W(\pi(x_c))$ where $\pi(x_c)$ is the projection of $x_c$ onto the input image plane [S1].
    *   Encode $x_c$ with positional encoding $\gamma(x_c)$ using $M=10$ frequency bases [S1]:
        $$\gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), ..., \sin(2^{M-1} \pi p), \cos(2^{M-1} \pi p))$$
        [S1]
    *   Input the viewing direction $d_c$, encoded position $\gamma(x_c)$, and feature $W(\pi(x_c))$ into a NeRF MLP to predict density $\sigma$ and color $\mathbf{c}$ [S1]:
        $$(\sigma, \mathbf{c}) = \text{MLP}(\gamma(x_c); d_c; W(\pi(x_c)))$$
        [S1]
    *   Render the image via volume rendering [S1]:
        $$\hat{C}(r) = \int_{t_n}^{t_f} T(t)\sigma(t)\mathbf{c}(t) dt, \quad T(t) = \exp\left(-\int_{t_n}^{t} \sigma(r(s)) ds\right)$$
        Approximate the integral with quadrature [S1].

**Parameters:**
*   **Frequency bases ($M$):** 10 [S1].
*   **Learning Rate:** $10^{-4}$ for the NeRF MLP; $10^{-5}$ for the ViT and CNN modules [S1].
*   **Batch Size:** 8 instances, with 512 rays per instance [S1].
*   **ViT Initialization:** Pretrained weights from reference [55] (ViT-Base), fine-tuned during training [S1].
*   **Warm-up:** Linear warm-up from 0 learning rate for the first 10k steps [S1].

**Edge Cases and Pitfalls:**
*   **Class Token:** The ViT class token must be dropped before unflattening; retaining it introduces artifacts as it lacks spatial meaning [S1].
*   **Local vs. Global:** Relying solely on global ViT features degrades rendering quality for views close to the input view (inconsistent color/appearance). The local CNN branch is required to fix this [S1].

## Operating characteristics
*   **Hardware:** Evaluated on 16 NVIDIA A100 GPUs [S1].
*   **Training Time:** Converges at 500K iterations [S1].
*   **Strengths:** Produces sharper edges and clearer structures than SRN and PixelNeRF, particularly on invisible parts like chair legs [S1]. Does not require a canonical coordinate system, unlike CodeNeRF and DISN [S1].
*   **Limitations:** Volume-based methods are computationally intensive and require more memory as grid size increases [S1].

## Background
ViT-NeRF was introduced in "Vision Transformer for NeRF-Based View Synthesis from a Single Input Image" (WACV 2023) [S1]. It builds on the success of Vision Transformers in capturing long-range dependencies, combining them with 2D CNNs to leverage both global context and local appearance details for novel view synthesis from a single image [S1].

## Sources
[S1] Vision Transformer for NeRF-Based View Synthesis from a Single Input Image, https://doi.org/10.1109/wacv56688.2023.00087, fetched 2026-10-27
