# ViT-NeRF

Also called: Global-Local Feature NeRF, Hybrid ViT-CNN NeRF, Vision Transformer for NeRF (ViT-NeRF)

## Overview
ViT-NeRF is a view synthesis method that combines a Vision Transformer (ViT) to encode global shape information and a 2D CNN to extract local appearance features, conditioning volume rendering on hybrid per-pixel features [S1]. It addresses the limitation of pure global feature methods, which compromise rendering quality for views close to the input view, by fusing long-range dependencies with local appearance consistency [S1]. Unlike CodeNeRF and DISN, it does not require a canonical coordinate system to utilize global features, and compared to SRN and PixelNeRF, it produces sharper edges and clearer structures on invisible parts [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single source image $I_s \in \mathbb{R}^{H \times W \times 3}$, where $H$ and $W$ are height and width [S1].
*   **Output:** A rendered 2D image for a target viewpoint defined by ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1].

**Data Structures:**
*   **Image Tokens:** Sequence of flattened 2D patches $P \in \mathbb{R}^{N \times P^2 \times 3}$, where $N = \frac{HW}{P^2}$ is the number of patches and $P$ is the patch size [S1].
*   **Latent Features:** Set of latent features $\mathbf{f}$ extracted by the ViT encoder [S1].
*   **Global Feature Maps:** Multi-level feature maps $W^0_G, ..., W^J_G$, where $W^j_G \in \mathbb{R}^{H_j \times W_j \times D_j}$ [S1].
*   **Local Feature Map:** $W_L \in \mathbb{R}^{\frac{H}{2} \times \frac{W}{2} \times D_L}$ [S1].
*   **Hybrid Feature Map:** $W \in \mathbb{R}^{H \times W \times D}$, the fused result [S1].
*   **Positional Encoding:** $\gamma(\mathbf{p}) \in \mathbb{R}^{2M}$, where $M=10$ [S1].

**Numbered Steps:**

1.  **Global Feature Extraction:**
    a.  Reshape $I_s$ into a sequence of flattened 2D patches $P$.
    b.  Linearly project each patch to an image token.
    c.  Input image tokens and learnable positional embeddings into the Transformer Encoder to extract latent features $\mathbf{f}$ [S1].
    d.  Drop the class token from $\mathbf{f}$. The class token is useful during self-attention but has no physical meaning when unflattened [S1].
    e.  Unflatten the remaining tokens into feature patches $\mathbf{P}_f$ [S1].
    f.  Generate multi-level global feature maps $W^j_G$ using a convolutional decoder $D$ for each level $j \in \{0, ..., J\}$:
        $$W^j_G = (D \circ U \circ O)(\mathbf{f}_j)$$
        where $O: \mathbb{R}^{(N+1) \times D} \rightarrow \mathbb{R}^{N \times D}$ drops the class token, $U: \mathbb{R}^{N \times D} \rightarrow \mathbb{R}^{\frac{H}{P} \times \frac{W}{P} \times D}$ unflattens, and $D$ consists of a $1 \times 1$ convolution, strided/transposed convolution for resizing, and a $3 \times 3$ convolution [S1].

2.  **Local Feature Extraction:**
    a.  Pass $I_s$ through a 2D CNN module $G_L$ consisting of three ResBlocks to extract local image features:
        $$W_L = G_L(I_s)$$
        where $G_L: \mathbb{R}^{H \times W \times 3} \rightarrow \mathbb{R}^{\frac{H}{2} \times \frac{W}{2} \times D_L}$ [S1].

3.  **Feature Fusion:**
    a.  Fuse the global maps $W^0_G, ..., W^J_G$ and the local map $W_L$ using a convolutional layer $G$ to produce the hybrid feature map $W$:
        $$W = G(W^0_G, W^1_G, ..., W^J_G; W_L)$$
        [S1].

4.  **Volumetric Rendering:**
    a.  For a target ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ bounded by $t_{near}$ and $t_{far}$, sample equally distributed 3D locations $\mathbf{x}_c$ along the ray [S1].
    b.  Project $\mathbf{x}_c$ to the 2D image plane to find the corresponding pixel $\pi(\mathbf{x}_c)$ in the source image.
    c.  Extract the per-pixel hybrid feature $\mathbf{w} = W(\pi(\mathbf{x}_c))$ [S1].
    d.  Encode the 3D location $\mathbf{x}_c$ using positional encoding $\gamma(\mathbf{x}_c)$ with $M=10$ frequency bases:
        $$\gamma(\mathbf{p}) = (\sin(2^0 \pi \mathbf{p}), \cos(2^0 \pi \mathbf{p}), ..., \sin(2^{M-1} \pi \mathbf{p}), \cos(2^{M-1} \pi \mathbf{p}))$$
        [S1].
    e.  Input $\gamma(\mathbf{x}_c)$, viewing direction $\mathbf{d}_c$, and feature $\mathbf{w}$ into an MLP to output density $\sigma$ and color $\mathbf{c}$:
        $$(\sigma, \mathbf{c}) = \text{MLP}(\gamma(\mathbf{x}_c); \mathbf{d}_c; \mathbf{w})$$
        [S1].
    f.  Render the image using standard volume rendering:
        $$\hat{C}(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \sigma(t) \mathbf{c}(t) dt$$
        where $T(t) = \exp(-\int_{t_n}^{t} \sigma(\mathbf{r}(s)) ds)$ is the accumulated transmittance [S1].

**Parameters:**
*   **Frequency bases ($M$):** 10 [S1].
*   **Learning Rate:** $10^{-4}$ for the NeRF MLP; $10^{-5}$ for the ViT and CNN modules [S1].
*   **Batch Size:** 8 instances, with 512 rays per instance [S1].
*   **Training Iterations:** Converges at 500K iterations [S1].
*   **Warm-up:** Linear warm-up from 0 learning rate for the first 10k steps [S1].
*   **ViT Initialization:** Initialized from pretrained weights of reference [55] and fine-tuned [S1].
*   **Local CNN:** $G_L$ consists of three ResBlocks [S1].

**Edge Cases and Pitfalls:**
*   **Class Token:** The class token in ViT must be dropped before decoding into spatial feature maps, as it has no physical meaning in that context [S1].
*   **Global-Only Limitation:** Using only global ViT features compromises rendering quality for views close to the input view (inconsistent color/appearance). This is solved by adding the local CNN branch [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** On ShapeNet Chairs, ViT-NeRF achieves PSNR 24.48, SSIM 0.93, LPIPS 0.077. On ShapeNet Cars, it achieves PSNR 22.88, SSIM 0.91, LPIPS 0.084 [S1].
*   **Hardware:** Trained on 16 NVIDIA A100 GPUs [S1].
*   **Strengths:** Produces sharper edges and clearer structures than SRN and PixelNeRF, especially on invisible parts like chair legs [S1]. Combines the advantage of 2D-based methods (local appearance) and 1D-based methods (global information) [S1].
*   **Limitations:** Volume-based methods are more computationally intensive and require more memory when increasing the grid size [S1].

## Background
ViT-NeRF was introduced in "Vision Transformer for NeRF-Based View Synthesis from a Single Input Image" [S1]. It builds on the success of Vision Transformers in learning long-range dependencies between features [S1]. Unlike CodeNeRF and DISN, it does not require a canonical coordinate system to utilize global features [S1].

## Sources
[S1] Vision Transformer for NeRF-Based View Synthesis from a Single Input Image, https://doi.org/10.1109/wacv56688.2023.00087, fetched 2026-10-27
