# TransNeRF

Also called: Generalizable Neural Radiance Fields for Novel View Synthesis with Transformer

## Overview
TransNeRF is a generalizable neural radiance field method that learns a generic scene representation from multiple source views without requiring per-scene optimization [S1, unofficial]. It replaces the pooling-based multi-view fusion of previous methods with an encoder-decoder Transformer framework that explores deep relationships among source views and enforces local consistency along rays [S1, unofficial]. Unlike [ViT-NeRF](13878-vit-nerf.md) and [VisionNeRF](13885-visionnerf.md) which focus on single-scene reconstruction with ViT backbones, TransNeRF is designed for cross-scene generalization, rendering novel views for unseen scenes directly from a pre-trained model [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $M$ source views $\{I_m\}_{m=1}^M$ with corresponding camera parameters $\{\Theta_m\}_{m=1}^M$ (poses, intrinsics, scene bounds), and a query 3D point location $(x, y, z)$ with viewing direction $d$.
*   **Outputs:** Differential volumetric density $\sigma$ and directional emitted color $c$ for the query point.

**Data Structures and Layouts:**
*   **Query Embeddings:** Learnable embeddings for the query point.
    *   Density query: $x^0_\sigma$ (initial), $x^L_\sigma$ (final).
    *   Color query: $Y_0$ (initial), $Y_L$ (final).
    *   Color ray query: $z^0_i$ (initial), $z^L_i$ (final).
*   **Source View Embeddings:**
    *   Initial: $x^0_m$ derived from U-Net features and viewing direction $d^m_{src}$.
    *   Final: $x^L_m$ from the Density-ViewDecoder.
*   **Ray Neighbors:** For a query point $q_i$ on a target ray, neighbors are $q_{i-n}, \dots, q_{i-1}, q_{i+1}, \dots, q_{i+n}$.
*   **Positional Encodings:** $E_{pos}$ added to ray-space inputs, computed via sine/cosine functions of different frequencies [S1, unofficial].

**Parameters:**
*   $N_q$: Number of queries stacked in $Q$.
*   $N_{kv}$: Number of key-value pairs stacked in $K$.
*   $d_v$: Dimension of $V$.
*   $H$: Number of attention heads.
*   $d_h$: Feature dimension per head, such that $H \times d_h = d_k$.
*   $n$: Number of neighboring points on each side along the ray.
*   $L$: Number of Transformer blocks.

**Step-by-Step Method:**

1.  **Feature Extraction:** For each source view $m$, extract a feature volume using a pre-trained U-Net. Retrieve spatial-aligned 2D-pixel features for the query 3D point $(x, y, z)$ to form the initial source-view feature $x^0_m$. Record the viewing direction $d^m_{src}$ for the projected pixel [S1, unofficial].

2.  **Density-ViewDecoder (Surrounding-view Space):**
    *   **Input:** $X_0 = [x^0_\sigma; x^0_1; x^0_2; \dots; x^0_M]$.
    *   **Process:** Apply $L$ blocks of self-attention. Each block $l$ computes:
        $$ \tilde{X}_l = \text{Norm}(\text{Density-ViAttn}(X_{l-1}) + X_{l-1}) $$
        $$ X_l = \text{Norm}(\text{FFN}(\tilde{X}_l) + \tilde{X}_l) $$
        Where $\text{Density-ViAttn}(X) = \text{MH-Attn}(X, X, X)$ [S1, unofficial].
    *   **Output:** $X_L = [x^L_\sigma; x^L_1; \dots; x^L_M]$. The latent density representation is derived from $x^L_\sigma$ and the 3D coordinates $(x, y, z)$ [S1, unofficial].

3.  **Density-RayDecoder (Ray-cast Space):**
    *   **Input:** Initial density representations of neighbors $\sigma^0_{i-n}, \dots, \sigma^0_{i+n}$ plus positional encodings: $D_0 = [\sigma^0_{i-n}; \dots; \sigma^0_{i+n}] + E_{pos}$.
    *   **Note:** The initial density $\sigma^0$ for each point is computed by a linear function of the Density-ViewDecoder output: $\sigma^0 = F_C(x^L_\sigma(x, y, z))$ [S1, unofficial].
    *   **Process:** Apply $L$ blocks of self-attention. Each block $l$ computes:
        $$ \tilde{D}_l = \text{Norm}(\text{Density-Attn}(D_{l-1}) + D_{l-1}) $$
        $$ D_l = \text{Norm}(\text{FFN}(\tilde{D}_l) + D_{l-1}) $$
        Where $\text{Density-Attn}(D) = \text{MH-Attn}(D, D, D)$ [S1, unofficial].
    *   **Output:** The final density $\sigma$ for the query point is obtained by projecting the output embedding $\sigma^L_i$ via a linear function [S1, unofficial].

4.  **Color-ViewDecoder (Surrounding-view Space):**
    *   **Input Query:** $Y_0 = F_C(\sigma^L_i) \oplus d_{tgt}$, where $\sigma^L_i$ is the latent density from Density-RayDecoder and $d_{tgt}$ is the target viewing direction [S1, unofficial].
    *   **Input Keys:** $X_{src} = [F_C(x^L_1) \oplus d^1_{src}; \dots; F_C(x^L_M) \oplus d^M_{src}]$.
    *   **Input Values:** $C_{src} = [\gamma(c^1_{src}); \dots; \gamma(c^M_{src})]$, where $\gamma(\cdot)$ is an embedding function and $c^m_{src}$ is the RGB color of the source view [S1, unofficial].
    *   **Process:** Apply $L$ blocks of cross-attention. Each block $l$ computes:
        $$ \tilde{Y}_l = \text{Norm}(\text{Color-ViAttn}(Y_{l-1}, X_{src}, C_{src}) + Y_{l-1}) $$
        $$ Y_l = \text{Norm}(\text{FFN}(\tilde{Y}_l) + \tilde{Y}_l) $$
        Where $\text{Color-ViAttn}(Y, X_{src}, C_{src}) = \text{MH-Attn}(Y, X_{src}, C_{src})$ [S1, unofficial].
    *   **Output:** Latent color representation $Y_L$ [S1, unofficial].

5.  **Color-RayDecoder (Ray-cast Space):**
    *   **Input:** Initial color embeddings of neighbors $z^0_{i-n}, \dots, z^0_{i+n}$ plus positional encodings: $Z_0 = [z^0_{i-n}; \dots; z^0_{i+n}] + E_{pos}$.
    *   **Note:** The latent color representation $Y_L$ from Color-ViewDecoder is assigned to the corresponding $z^0_i$ [S1, unofficial].
    *   **Process:** Apply $L$ blocks of self-attention. Each block $l$ computes:
        $$ \tilde{Z}_l = \text{Norm}(\text{Color-Attn}(Z_{l-1}) + Z_{l-1}) $$
        $$ Z_l = \text{Norm}(\text{FFN}(\tilde{Z}_l) + Z_{l-1}) $$
        Where $\text{Color-Attn}(Z) = \text{MH-Attn}(Z, Z, Z)$ [S1, unofficial].
    *   **Output:** The predicted color $c$ is obtained by projecting the output embedding $z^L_i$ via a linear function [S1, unofficial].

6.  **Volume Rendering:**
    *   Use classical differentiable volume rendering to compute the rendered color $C(r)$ for a camera ray $r(t) = o + td$ with bounds $[t_n, t_f]$:
        $$ C(r) = \int_{t_n}^{t_f} T(t)\sigma(r(t))c(r(t), d)dt $$
        $$ T(t) = \exp\left(-\int_{t_n}^{t} \sigma(r(s))ds\right) $$
    *   The model is trained end-to-end by minimizing the MSE loss between rendered and ground truth colors: $\mathcal{L} = \sum_{r \in R(\Theta)} \|\hat{C}(r) - C(r)\|_2^2$ [S1, unofficial].

**Multi-Head Attention Implementation:**
$$ \text{MH-Attn}(Q, K, V) = \text{Concat}(A_1, \dots, A_H)W $$
$$ A_h = \text{Attention}(Q_h, K_h, V_h) = \text{softmax}\left(\frac{Q_h K_h^T}{\sqrt{d_h}}\right)V_h $$
Where $Q_h = QW^q_h$, $K_h = KW^k_h$, $V_h = VW^v_h$ [S1, unofficial].

**Pitfalls:**
*   Pooling-based fusion models fail to explore complex relationships across multiple views and ignore local consistency of the 5D radiance field [S1, unofficial].
*   The method assumes a pre-trained U-Net is available for feature extraction [S1, unofficial].

## Operating characteristics
*   **Generalization:** TransNeRF generalizes to unseen scenes without per-scene finetuning after training on a large dataset of diverse scenes [S1, unofficial].
*   **Complexity:** The use of Transformer attention mechanisms allows for exploring deep relationships among views, but introduces higher computational complexity compared to simple pooling or CNN-based fusion, particularly in the view-decoder stages where attention is computed over $M$ source views [S1, unofficial].
*   **Strengths:** Captures global semantic context and local geometric consistency simultaneously. Invariant to permutations of source views [S1, unofficial].
*   **Limitations:** Requires a large dataset for training the generalizable prior. Relies on the quality of the pre-trained U-Net encoder [S1, unofficial].

## Background
TransNeRF was proposed to address the limitations of per-scene NeRF optimization and the inability of previous generalizable methods to model complex multi-view relationships. It builds on the success of Transformers in NLP and Vision [S1, unofficial]. The method was introduced in a paper titled "Generalizable Neural Radiance Fields for Novel View Synthesis with Transformer" [S1, unofficial].

## Sources
[S1] Generalizable Neural Radiance Fields for Novel View Synthesis with Transformer, https://doi.org/10.48550/arxiv.2206.05375, fetched 2026-10-27 (unofficial)
