# Deep Implicit Volume Compression

Also called: Autoencoder-based Implicit Volume Compression, Block-based neural TSDF compression, Deep Implicit Volume Compression Network, End-to-end TSDF compression, Implicit Volume Compression

## Overview
Deep Implicit Volume Compression is a block-based neural network architecture trained end-to-end to compress Truncated Signed Distance Fields (TSDF) stored in 3D voxel grids, achieving state-of-the-art rate-distortion trade-offs [S1]. It solves the problem of topological errors in reconstructed surfaces caused by quantization noise in magnitude values by losslessly compressing the sign configuration separately from the geometric magnitude [S1]. Unlike standard convolutional autoencoders that minimize only reconstruction error, this method employs dual-output heads and conditional entropy coding to minimize bit rate while guaranteeing that the reconstructed surface is homeomorphic to the ground truth [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of TSDF volumes $V = \{V_t\}_{t=1}^T$, where each volume is a 3D grid of voxels. For a single block, the input is a $k \times k \times k$ voxel grid $x$ containing a zero crossing [S1].
*   **Output:** Compressed bitstreams for the latent code ($\hat{z}_{bits}$) and signs ($s_{bits}$), and block indices for volume stitching. The receiver reconstructs the TSDF block $\hat{x}$ [S1].

**Data Structures:**
*   **TSDF Block ($x$):** A 3D array of size $k \times k \times k$. Values are real numbers representing signed distance.
*   **Sign Configuration ($s$):** A 3D array of size $k \times k \times k$ with discrete values $\{-1, +1\}$. Remapped to $\{0, 1\}$ for loss calculation [S1].
*   **Latent Code ($\hat{z}$):** Continuous values output by the encoder, quantized for transmission [S1].
*   **Block Indices:** Triplets $(x, y, z)$ identifying the position of each occupied block in the global volume [S1].

**Network Architecture:**
*   **Encoder ($E$):** Consists of three 3D convolutional layers [S1].
*   **Decoder ($D$):** Consists of three 3D transposed convolutional layers, each with a stride of 2, followed by two final convolutional heads with a stride of 1 [S1].
    *   **Head 1 (Magnitude):** Outputs the absolute TSDF values $|D(\hat{z})|$ [S1].
    *   **Head 2 (Signs):** Outputs the conditional probability distribution $p_{s|\hat{z}}$ for the signs [S1].

**Parameters:**
*   **Voxel Size:** 5 mm [S1].
*   **Block Size ($k$):** $8^3$ voxels (physical size $40 \text{mm}^3$) [S1].
*   **Model Size:** Target < 2 MB (actual < 1.8 MB for 3 layers) [S1].
*   **Rate Parameter ($\lambda$):** Weighting factor for the rate-distortion trade-off, varied during inference [S1].

**Numbered Steps:**

1.  **Block Extraction:** From the TSDF volume $V$, extract all non-overlapping blocks $\{x_m\}$ of size $k \times k \times k$ that contain a zero crossing. These are "occupied blocks" [S1].
2.  **Latent Encoding:** Compute the latent representation $\hat{z}$ using the encoder $E$ with parameters $\theta_e$:
    $$ \hat{z} = E(x; \theta_e) $$
    During training, uniform noise $\epsilon \sim U[-0.5, 0.5]$ is added to $\hat{z}$ to simulate quantization: $\hat{z} = E(x; \theta_e) + \epsilon$ [S1]. At test time, $\hat{z}$ is quantized [S1].
3.  **Sign Probability Estimation:** Compute the conditional probability distribution over the TSDF signs $s$ given the latent code $\hat{z}$ using the sign head of the decoder:
    $$ p_{s|\hat{z}}(s|\hat{z}; \theta_s) = D_s(\hat{z}) $$
    where $D_s$ is the sign prediction head and $\theta_s$ are its learnable parameters [S1].
4.  **Entropy Coding (Sender):**
    *   Compute the bitstream $\hat{z}_{bits}$ by losslessly coding $\hat{z}$ using the learned prior distribution $p_{\hat{z}}(\hat{z}; \phi)$ and an entropy coder [S1].
    *   Compute the bitstream $s_{bits}$ by losslessly coding the ground truth signs $s$ using the conditional distribution $p_{s|\hat{z}}(s|\hat{z}; \theta_s)$ and an entropy coder [S1].
5.  **Index Encoding:** Sort the occupied blocks in ascending order. Apply delta encoding to the vector of block indices to make it entropy-coder friendly [S1].
6.  **Transmission:** Transmit $\hat{z}_{bits}$, $s_{bits}$, and the encoded block indices [S1].
7.  **Reconstruction (Receiver):**
    *   Recover $\hat{z}$ using entropy decoding with the shared prior $p_{\hat{z}}$ [S1].
    *   Re-compute $p_{s|\hat{z}}$ to recover the losslessly coded ground truth signs $s$ [S1].
    *   Reconstruct the TSDF block $\hat{x}$ using the decoder $D$ and the signs $s$:
        $$ \hat{x} = s \odot |D(\hat{z}; \theta_d)| $$
        where $\odot$ is element-wise product, $|\cdot|$ is element-wise absolute value, and $\theta_d$ are decoder parameters [S1].
8.  **Volume Stitching:** Place reconstructed blocks $\hat{x}$ into the global volume grid using the transmitted indices [S1].
9.  **Mesh Extraction:** Extract a triangular mesh from the reconstructed TSDF volume using Marching Cubes. Since signs are transmitted losslessly, the mesh topology is guaranteed to match the uncompressed TSDF [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def extract_occupied_blocks(tsdf_volume, block_size=8):
    """
    Extracts non-overlapping blocks containing a zero crossing.
    Returns a list of tuples: (block_data, global_index_xyz)
    """
    blocks = []
    # Iterate over the volume with stride block_size
    for x in range(0, tsdf_volume.shape[0], block_size):
        for y in range(0, tsdf_volume.shape[1], block_size):
            for z in range(0, tsdf_volume.shape[2], block_size):
                # Extract block
                block = tsdf_volume[x:x+block_size, y:y+block_size, z:z+block_size]
                # Check for zero crossing (sign change)
                if has_zero_crossing(block):
                    blocks.append((block, (x, y, z)))
    return blocks

def has_zero_crossing(block):
    """Checks if a block contains voxels with opposing signs."""
    signs = np.sign(block)
    # A zero crossing exists if there are both positive and negative values
    return np.any(signs > 0) and np.any(signs < 0)

def encode_block(block, encoder, decoder, prior_dist, sign_head):
    """
    Sender-side encoding of a single block.
    """
    # 1. Encode to latent
    z = encoder(block)
    
    # 2. Quantize latent (simulate quantization for entropy coding)
    # In training, noise is added. In inference, we quantize.
    z_quantized = np.round(z) # Simplified quantization
    
    # 3. Compute conditional sign probabilities
    p_s_given_z = sign_head(z_quantized)
    
    # 4. Get ground truth signs
    s = np.sign(block)
    s_binary = (s + 1) / 2 # Map {-1, 1} to {0, 1}
    
    # 5. Entropy code latent and signs
    # Using an arithmetic coder or similar entropy coder
    z_bits = entropy_encode(z_quantized, prior_dist)
    s_bits = entropy_encode(s_binary, p_s_given_z)
    
    return z_bits, s_bits

def decode_block(z_bits, s_bits, decoder, prior_dist, sign_head):
    """
    Receiver-side decoding of a single block.
    """
    # 1. Decode latent
    z_quantized = entropy_decode(z_bits, prior_dist)
    
    # 2. Decode signs
    p_s_given_z = sign_head(z_quantized)
    s_binary = entropy_decode(s_bits, p_s_given_z)
    s = 2 * s_binary - 1 # Map {0, 1} to {-1, 1}
    
    # 3. Reconstruct TSDF
    # Decoder outputs magnitude |D(z)|
    magnitude = np.abs(decoder(z_quantized))
    x_hat = s * magnitude
    
    return x_hat

def compute_loss(block, x_hat, s, decoder, prior_dist):
    """
    Computes the training loss for a block.
    """
    # 1. Distortion Loss with Topology Masking
    # Create masks for important voxels (neighbors with opposing signs)
    # This is a simplified representation of the mask generation
    m_x, m_y, m_z = create_topology_masks(s)
    
    distortion = 0
    for d in [m_x, m_y, m_z]:
        distortion += np.sum((d * (x_hat - block))**2)
    
    B = 1 # Number of blocks in this batch (simplified)
    D_x_hat = (1/B) * distortion
    
    # 2. Sign Rate Loss (Cross Entropy)
    p_s_given_z = decoder.sign_head(decoder.encoder(block)) # Forward pass to get probs
    # Remap s to {0, 1} for cross entropy
    s_binary = (s + 1) / 2
    R_s = cross_entropy(s_binary, p_s_given_z)
    
    # 3. Latent Rate Loss (Estimate of differential entropy)
    # Add uniform noise during training to simulate quantization
    z_noisy = decoder.encoder(block) + np.random.uniform(-0.5, 0.5, size=decoder.encoder.output_shape)
    R_z = estimate_entropy(z_noisy, prior_dist)
    
    # Total Loss
    lambda_param = 0.1 # Example weighting factor
    total_loss = D_x_hat + lambda_param * (R_z + R_s)
    
    return total_loss

def create_topology_masks(s):
    """
    Creates masks for voxels that have neighbors with opposing signs.
    """
    # Implementation details for mask generation would depend on specific neighbor checks
    # This is a placeholder for the logic described in [S1]
    pass

def morton_encode_3d(x, y, z, num_bits=10):
    """
    Encodes 3D coordinates into a Morton code.
    Prioritizes y-axis over x and z.
    """
    code = 0
    for b in range(num_bits):
        code += (4 * ((y >> b) & 1) + 2 * ((x >> b) & 1) + ((z >> b) & 1)) * (2 ** (3 * b))
    return code

def morton_decode_3d(code, num_bits=10):
    """
    Decodes a Morton code back to 3D coordinates.
    """
    x, y, z = 0, 0, 0
    for b in range(num_bits):
        shift = 3 * b
        z |= ((code >> shift) & 1) << b
        x |= ((code >> (shift + 1)) & 1) << b
        y |= ((code >> (shift + 2)) & 1) << b
    return x, y, z
```

**Edge Cases and Pitfalls:**
*   **Topological Errors:** Without lossless sign transmission, compression error in TSDF magnitudes can cause topological errors in the reconstructed surface [S1].
*   **Reconstruction Error Bound:** With lossless sign transmission, the reconstruction error is bounded by the voxel size (5 mm) [S1].
*   **Masking Importance:** Without topology masking in the distortion loss, the model wastes complexity on voxels far from the surface, leading to significant deviation from ground truth [S1].
*   **Conditional vs Prior:** Using the prior $p_s(s)$ for signs instead of the conditional $p_{s|\hat{z}}(s|\hat{z})$ results in higher entropy and less efficient compression [S1].

## Subtopics

### [Lossless TSDF Sign Compression](7794-lossless-tsdf-sign-compression.md)
Lossless TSDF Sign Compression is a method for compressing implicit volume representations by separately encoding the topological structure (signs) and geometric magnitude (values) of a Truncated Signed Distance Field (TSDF). It solves the problem of topological errors in reconstructed surfaces caused by quantization noise in the magnitude values. Unlike methods that compress the full TSDF field uniformly, this approach ensures that the zero-crossing surface topology remains identical to the ground truth by losslessly encoding the sign configuration, thereby bounding the reconstruction error by the voxel size.

## Operating characteristics
*   **Time Complexity:** Processing is block-based, making it computationally feasible for high-dimensional volumes (e.g., $240 \times 240 \times 400$ voxels) by processing each frame independently [S1].
*   **Memory Complexity:** Model size is < 1.8 MB for 3 layers, suitable for mobile deployment [S1].
*   **Measured Performance:**
    *   Reduces bitrate by 66% for the same distortion, or reduces distortion by 50% for the same bitrate, compared to state-of-the-art [S1].
    *   Achieves bitrates that are 33% of prior art [59] [S1].
    *   Sign compression rate: Raw sign data is 155.1 KB/volume. Learned model reduces this to 2.9 KB/volume (48x improvement over naive probabilistic model) [S1].
    *   Compresses TSDF directly rather than voxel occupancy, yielding sub-voxel precision and bitrates at least three times better than MPEG V-PCC [S1].
*   **Strengths:** Guarantees topological correctness; high compression efficiency; sub-voxel precision.
*   **Limitations:** Requires training; block-based processing may introduce boundary artifacts if not handled carefully (though stitching is supported); dependent on the quality of the learned distributions.
*   **When to Use:** Real-time dense surface mapping, AR devices, free-viewpoint videos, 4D performance capture [S1].
*   **When Not to Use:** When topological fidelity is not required and simpler methods suffice; when computational resources for training are unavailable.

## Background
*   **Origin:** Proposed by Danhang Tang, Saurabh Singh, Philip A. Chou, Christian Häne, Mingsong Dou, Sean Fanello, Jonathan Taylor, Philip Davidson, Onur G. Guleryuz, Yinda Zhang, Shahram Izadi, Andrea Tagliasacchi, Sofien Bouaziz, and Cem Keskin from Google [S1].
*   **Date:** May 2020 [S1].
*   **Paper:** "Deep Implicit Volume Compression" [S1].
*   **History:** Builds on recent advances in learned compression and 3D/4D reconstruction pipelines. It addresses limitations of existing methods like Draco (requires decimation) and Tang et al. [59] (block artifacts) [S1].
*   **Usage:** Evaluated on Tang et al. [59] and Guo et al. [26] datasets, using RGBD multi-view sequences at 30Hz [S1].

## Sources
[S1] Deep Implicit Volume Compression, https://doi.org/10.1109/cvpr42600.2020.00137, fetched 2026-10-26
