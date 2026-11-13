# Deep Implicit Volume Compression

Also called: Autoencoder-based Implicit Volume Compression, Block-based neural TSDF compression, Deep Implicit Volume Compression Network, End-to-end TSDF compression, Implicit Volume Compression

## Overview
Deep Implicit Volume Compression is a block-based neural network architecture trained end-to-end to compress Truncated Signed Distance Fields (TSDF) stored in 3D voxel grids, achieving state-of-the-art rate-distortion trade-offs [S1]. It differs from standard convolutional autoencoders by employing a dual-output decoder head, lossless compression of topological signs via conditional entropy modeling, and a topology-aware distortion loss, ensuring the reconstructed mesh remains homeomorphic to the ground truth [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A TSDF volume $V$ represented as a 3D voxel grid. The method processes non-overlapping blocks $\{x_m\}$ of size $k \times k \times k$ that contain a zero crossing (occupied blocks) [S1].
*   **Output:** Compressed bitstreams for latent codes ($\hat{z}_{bits}$) and signs ($s_{bits}$), along with block indices for spatial reconstruction [S1].

**Data Structures:**
*   **Block ($x$):** A $k \times k \times k$ voxel grid. In the reference implementation, $k=8$, voxel size is 5mm, resulting in a physical block size of $40mm^3$ [S1].
*   **Latent Code ($\hat{z}$):** Continuous values output by the encoder, quantized and compressed [S1].
*   **Sign Configuration ($s$):** A discrete grid of values $\{-1, +1\}$ indicating the sign of the TSDF at each voxel. Remapped to $\{0, 1\}$ for loss calculation [S1].
*   **Block Indices:** Integer triples $(x, y, z)$ identifying the position of each occupied block in the global volume [S1].

**Network Architecture:**
*   **Encoder ($E$):** Consists of three 3D convolutional layers [S1].
*   **Decoder ($D$):** Consists of three 3D transposed convolutional layers (stride 2) followed by two parallel 3D convolutional heads (stride 1). One head predicts TSDF magnitudes, the other predicts sign probabilities [S1].

**Parameters:**
*   **Block Size:** $8^3$ voxels [S1].
*   **Voxel Size:** 5mm [S1].
*   **Model Size Target:** < 2MB (specifically < 1.8MB for the 3-layer variant) [S1].
*   **Rate Parameter ($\lambda$):** Weighting factor for the rate-distortion trade-off, varied during inference to generate rate-distortion curves [S1].

**Algorithm Steps:**

1.  **Block Extraction:** Extract all non-overlapping $k \times k \times k$ blocks $\{x_m\}$ from the TSDF volume $V$ that contain a zero crossing [S1].
2.  **Encoding:** Compute the latent representation $\hat{z}$ using the encoder $E$ with parameters $\theta_e$:
    $$ \hat{z} = E(x; \theta_e) $$
    During training, uniform noise $\epsilon \sim U[-0.5, 0.5]$ is added to $\hat{z}$ to simulate quantization. At test time, $\hat{z}$ is quantized [S1].
3.  **Sign Probability Estimation:** Compute the conditional probability distribution over the TSDF signs $s$ given the latent code $\hat{z}$ using the sign head of the decoder:
    $$ p_{s|\hat{z}}(s|\hat{z}; \theta_s) $$
    Here, $\theta_s$ are the learnable parameters of the sign head [S1].
4.  **Entropy Coding (Sender):**
    *   Compute the bitstream $\hat{z}_{bits}$ by losslessly coding $\hat{z}$ using an entropy coder and the learned prior distribution $p_{\hat{z}}(\hat{z}; \phi)$ [S1].
    *   Compute the bitstream $s_{bits}$ by losslessly coding the ground truth signs $s$ using an entropy coder and the conditional distribution $p_{s|\hat{z}}(s|\hat{z}; \theta_s)$ [S1].
    *   Compute block indices. Sort blocks in ascending order and apply delta encoding to the index vector to make it entropy encoder friendly [S1].
5.  **Transmission:** Transmit $\hat{z}_{bits}$, $s_{bits}$, and the encoded block indices to the receiver [S1].
6.  **Decoding (Receiver):**
    *   Recover $\hat{z}$ using entropy decoding with the shared prior $p_{\hat{z}}$ [S1].
    *   Re-compute $p_{s|\hat{z}}$ to recover the losslessly coded ground truth signs $s$ [S1].
    *   Recover the lossy TSDF values $\hat{x}$ using the decoder $D$ and the signs $s$:
        $$ \hat{x} = s \odot |D(\hat{z}; \theta_d)| $$
        where $\odot$ is element-wise product, $|\cdot|$ is element-wise absolute value, and $\theta_d$ are decoder parameters [S1].
7.  **Volume Reconstruction:** Stitch the reconstructed blocks $\hat{x}$ together using the transmitted indices to form the global TSDF volume [S1].
8.  **Mesh Extraction:** Extract a triangular mesh from the reconstructed TSDF volume using Marching Cubes. Since signs are transmitted losslessly, the mesh topology is guaranteed to match the uncompressed TSDF [S1].

**Training Objective:**
The model parameters $\Theta = \{\theta_e, \theta_s, \theta_d, \phi\}$ are optimized by minimizing:
$$ \arg \min_{\Theta} D_{\hat{x}}(x, \hat{x}; \theta_e, \theta_d) + \lambda [R_{\hat{z}}(\hat{z}; \phi) + R_s(s; \theta_s)] $$
where:
*   **Distortion Loss ($D_{\hat{x}}$):**
    $$ D_{\hat{x}} = \frac{1}{B} \sum_{n=1}^{B} \sum_{d \in \{x,y,z\}} \| m_d \cdot (\hat{x}_n - x_n) \|_2^2 $$
    $B$ is the number of blocks. $m_d$ is a mask of "important" voxels for dimension $d$. A voxel is marked as important if it has at least one neighbor with an opposing sign (indicating proximity to the surface). Voxels with neighbors of opposing signs in multiple dimensions appear in multiple masks, increasing their weight [S1].
*   **Latent Rate Loss ($R_{\hat{z}}$):** An estimate of the differential entropy of the noisy latent code, minimizing the bitrate of $\hat{z}$ [S1].
*   **Sign Rate Loss ($R_s$):** The cross-entropy between the ground truth signs $s$ (remapped to $\{0,1\}$) and the predicted conditional probabilities $p_{s|\hat{z}}$, minimizing the bitrate of $s$ [S1].

**Edge Cases and Pitfalls:**
*   **Topological Errors:** Without lossless sign compression or topology masking, compression errors in TSDF magnitudes can cause topological errors in the reconstructed surface [S1].
*   **Error Bound:** Lossless sign transmission guarantees the reconstructed surface is homeomorphic to the ground truth, bounding the maximum reconstruction error by the voxel size (5mm) [S1].
*   **Computational Feasibility:** The dense TSDF volume data is high-dimensional (e.g., $240 \times 240 \times 400$ voxels per frame), making joint compression of entire sequences infeasible; block-based processing is required [S1].

## Subtopics

### Lossless TSDF Sign Compression
A specific component of the method where the discrete sign configuration $s$ is compressed losslessly using an entropy coder conditioned on the latent code $\hat{z}$ [S1]. It uses the conditional distribution $p_{s|\hat{z}}$ rather than a prior $p_s$ because $s$ is dependent on $\hat{z}$ by design, resulting in significantly lower entropy (48x improvement over naive models) [S1]. This ensures the reconstructed surface topology matches the ground truth, bounding reconstruction error by the voxel size [S1]. Without this, topological errors arise from compression errors in magnitudes [S1].

## Operating characteristics
*   **Time Complexity:** Computational feasibility requires processing each frame independently in a block-based manner due to the high dimensionality of dense TSDF volumes [S1].
*   **Memory Complexity:** The model size is targeted to be roughly 2MB (specifically < 1.8MB for the 3-layer variant), suitable for mobile deployment [S1].
*   **Measured Performance:** On 4D performance capture datasets (Tang et al. [59] and Guo et al. [26]), the method reduces bitrate by 66% for the same distortion, or reduces distortion by 50% for the same bitrate, compared to state-of-the-art methods [S1]. It achieves bitrates that are 33% of prior art [59] [S1].
*   **Sign Compression:** Raw sign data is 155.1KB/volume. The learned data-dependent model reduces this to 2.9KB/volume (48x improvement over naive probabilistic models) [S1].
*   **Strengths:** Sub-voxel precision by compressing TSDF directly rather than voxel occupancy; guaranteed topological correctness; low model size.
*   **Limitations:** Requires block-based processing which may introduce boundary artifacts if not handled carefully; dependent on the accuracy of the learned probability models.
*   **Usage:** Evaluated on RGBD multi-view sequences at 30Hz [S1].

## Background
The method was introduced by Danhang Tang, Saurabh Singh, Philip A. Chou, Christian Häne, Mingsong Dou, Sean Fanello, Jonathan Taylor, Philip Davidson, Onur G. Guleryuz, Yinda Zhang, Shahram Izadi, Andrea Tagliasacchi, Sofien Bouaziz, and Cem Keskin from Google in May 2020 [S1]. It addresses the need for efficient compression of implicit volume representations (TSDFs) for real-time dense surface mapping in AR devices and free-viewpoint videos, where volumetric data is high-dimensional and computationally expensive to transmit and store [S1]. It builds on [Convolutional Autoencoder for Volume Compression](5718-convolutional-autoencoder-for-volume-compression.md) by adding specific modifications for implicit volumes, including sign handling and topology-aware losses.

## Sources
[S1] Deep Implicit Volume Compression, https://doi.org/10.1109/cvpr42600.2020.00137, fetched 2026-10-26
