# Neural NeRF Compression (Per-Scene, Encoder-Free)

Also called: Per-scene NeRF compression

## Overview
A method for compressing grid-based NeRF models (specifically TensoRF-VM) by optimizing latent codes directly for individual scenes without an encoder, using a lightweight decoder and a masked entropy model to minimize rate-distortion loss [S1, unofficial]. The problem addressed is the high storage cost of explicit grid NeRF representations (feature planes), which is critical for real-world applications like VR/AR [S1, unofficial]. It differs from previous works (Li et al., 2023a;b; Deng & Tartaglione, 2023) which use voxel pruning and/or vector quantization, by taking into account varying importance levels of voxel grid locations via importance-weighted objectives and masking [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained TensoRF-VM model with frozen feature planes $\{P_i\}_{i=1}^3$, where each $P_i \in \mathbb{R}^{C_i \times H_i \times W_i}$ represents channel-wise concatenations of density and appearance planes [S1, unofficial].
*   **Output:** A compressed bitstring representing the latent codes, along with the decoder $D$ and entropy model $P$ parameters [S1, unofficial].

**Data Structures:**
*   **Feature Planes:** $\{P_i\}_{i=1}^3$. For the Lego scene (Synthetic-NeRF), these take 67.61 MB, while other components (MLP, mask, vector) take 1.21 MB [S1, unofficial].
*   **Latent Codes:** $\{Z_i\}_{i=1}^3$. Initialized as tensors of zeros. These may have lower resolution than $P_i$ [S1, unofficial].
*   **Quantized Latent Codes:** $\{\hat{Z}_i\}_{i=1}^3$. Integers resulting from quantization [S1, unofficial].
*   **Decoder $D$:** A two-layer transposed convolutional neural network with SELU activation [S1, unofficial].
*   **Entropy Model $P$:** A masked entropy model that accounts for spatial inhomogeneity [S1, unofficial].

**Steps:**
1.  **Initialization:** Initialize latent codes $\{Z_i\}_{i=1}^3$ as tensors of zeros [S1, unofficial].
2.  **Joint Optimization:** Optimize $\{Z_i\}_{i=1}^3$, decoder $D$, and entropy model $P$ end-to-end to minimize the rate-distortion loss [S1, unofficial].
    *   **Quantization:** During training, continuous latent codes $Z$ are quantized to integers $\hat{Z}$ using a quantizer $Q$. In practice, uniform noise is injected to approximate quantization [S1, unofficial].
    *   **Reconstruction:** The decoder reconstructs feature planes $\hat{P} = D(\hat{Z})$ [S1, unofficial].
    *   **Rendering:** Use $\hat{P}$ to query sampling point features and render scene color $\hat{C}(r)$ [S1, unofficial].
    *   **Loss Calculation:** Compute distortion loss $d(X, \hat{X})$ and rate loss $-\lambda \log_2 P(\hat{Z})$ [S1, unofficial].
3.  **Compression:** Compress the quantized latent codes $\hat{Z}$ losslessly using the entropy model $P$ to generate a bitstring [S1, unofficial].
4.  **Decoding/Rendering:** At rendering time, decompress the bitstring using $P$ to recover $\hat{Z}$, pass $\hat{Z}$ through $D$ to reconstruct $\hat{P}$, and use $\hat{P}$ to render the scene [S1, unofficial].

**Formulas:**
The estimated color $\hat{C}(r)$ for a ray $r$ is calculated as [S1, unofficial]:
$$ \hat{C}(r) = \sum_{i=1}^{N} T_i \cdot \alpha_i \cdot c_i $$
where:
*   $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the probability of light being absorbed at the $i$-th point, dependent on volume density $\sigma_i$ and distance $\delta_i$ between adjacent sampled points [S1, unofficial].
*   $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the accumulated transmittance [S1, unofficial].

The total loss function is [S1, unofficial]:
$$ L(E, D, P) = \mathbb{E}_{X \sim p(X)} [d(X, D(Q(E(X)))) - \lambda \log_2 P(Q(E(X)))] $$
where $d(\cdot, \cdot)$ is a distortion loss and $\lambda$ balances distortion and rate [S1, unofficial].

**Parameters:**
*   $\lambda$: Balances distortion loss and rate loss [S1, unofficial].

**Pitfalls:**
*   **Amortization Gap:** Using an encoder for amortized inference degrades compression performance; thus, the encoder is removed and latent codes are optimized directly [S1, unofficial].
*   **Training Data Scarcity:** Lack of training data with many i.i.d. scenes prevents standard auto-encoder training on large datasets, necessitating per-scene optimization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Per-scene optimization requires iterative gradient descent to optimize latent codes, decoder, and entropy model jointly. This is slower than encoder-based inference but optimized for storage [S1, unofficial].
*   **Memory Complexity:** The compressed model includes the bitstring, decoder $D$, entropy model $P$, and non-compressed components (rendering MLP, vector components). For Lego, feature planes dominate storage (67.61 MB vs 1.21 MB for others) [S1, unofficial].
*   **Strengths:** Surpasses existing works in grid-based NeRF compression efficacy and reconstruction quality [S1, unofficial]. Handles spatial inhomogeneity via masked entropy model [S1, unofficial].
*   **Limitations:** Requires per-scene optimization, which is computationally intensive during the compression phase. Not suitable for real-time encoding of new scenes [S1, unofficial].
*   **When to use:** When storage/transmission constraints are critical (e.g., VR/AR) and per-scene optimization is feasible [S1, unofficial].
*   **When not to use:** When rapid encoding of many scenes is required [S1, unofficial].

## Background
Originated by Tuan Pham and Stephan Mandt in 2024 [S1, unofficial]. The paper "Neural NeRF Compression" was presented at the 41st International Conference on Machine Learning, Vienna, Austria, PMLR 235, 2024 [S1, unofficial]. It builds on non-linear transform coding techniques from neural image compression (Ballé et al., 2020; Yang et al., 2023b) [S1, unofficial]. It addresses the storage overhead of explicit grid NeRFs, which have become popular for their speed and quality (e.g., TensoRF-VM, Plenoxels, DVGO) [S1, unofficial].

## Sources
[S1] Neural NeRF Compression, https://doi.org/10.48550/arxiv.2406.08943, fetched 2026-10-26 (unofficial)
