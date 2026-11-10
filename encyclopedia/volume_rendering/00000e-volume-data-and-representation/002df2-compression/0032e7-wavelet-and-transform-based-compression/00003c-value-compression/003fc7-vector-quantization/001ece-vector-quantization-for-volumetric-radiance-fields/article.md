# Vector Quantization for Volumetric Radiance Fields

Also called: Codebook Encoding, Codebook Lookup Decompression, VQ Decompression, Vector Quantization (VQ), Vector Quantization (VQ) Decompression, Vector Quantization (VQ) Random Access

## Overview
Vector Quantization for Volumetric Radiance Fields (VQRF) is a compression strategy that encodes voxel color features into a compact codebook, substituting multiple voxel features with a single code vector to reduce storage from individual voxel storage to codebook storage plus mapping indices [S1]. It achieves this by combining adaptive voxel pruning with learnable vector quantization, outperforming uncompressed baselines and other size-focused methods in PSNR-size trade-offs [S1]. Unlike general dictionary-based VQ which supports per-voxel random access via GPU memory indirection [S2][S3], VQRF specifically targets the redundancy in volumetric radiance fields (such as DVGO, Plenoxels, and TensoRF) to achieve approximately 1 MB storage footprints [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained volumetric radiance field model (e.g., DVGO, Plenoxels, TensoRF) containing a density grid and color feature voxels.
*   **Output:** A compressed representation consisting of a codebook, voxel-to-codebook mapping indices, a mask, quantized density/non-VQ features, and entropy-encoded bitstream.

**Data Structures:**
The final storage consists of six components [S1]:
1.  **2-bit Mask:** Identifies voxels as pruned, VQ-compressed, or non-VQ (unaltered).
2.  **Codebook:** A matrix of size $K \times C$ storing code vectors.
3.  **Mapping Indices:** An array of size $N$ storing indices into the codebook for VQ voxels.
4.  **Quantized Density Grid:** 8-bit uniform quantization of density values.
5.  **Quantized Non-VQ Voxels:** 8-bit uniform quantization of color features for voxels saved without VQ.
6.  **Small Network Weights:** If applicable (e.g., for TensoRF or DVGO variants using a small network).

**Parameters:**
*   $N$: Number of voxels.
*   $C$: Channel dimension of color features.
*   $K$: Codebook size. Default: 4096 [S1].
*   $\beta_k$: Keep quantile. Fraction of most important voxels saved without VQ. Value: 0.6 for DVGO/Plenoxels, 0.7 for TensoRF [S1].
*   $\lambda_d$: Decay factor for moving average in code update. Default: 0.8 [S1].
*   $J$: Number of expired codes per step. Default: 10 [S1].
*   $\theta_k$: Keeping threshold for importance scores.

**Algorithm Steps:**

1.  **Importance Scoring & Pruning:**
    Calculate an importance score $I_j$ for each voxel $v_j$. Prune voxels based on a prune quantile $\beta_p = 0.001$ [S1].

2.  **Codebook Initialization:**
    Initialize codebook $B = \{b_1, ..., b_K\}$ using weighted clustering. Minimize the weighted within-cluster sum of squares [S1]:
    $$ \arg \min_{\mathbf{B}} \sum_{k=1}^K \sum_{\mathbf{v}_j \in \mathcal{R}(\mathbf{b}_k)} \|\mathbf{u}_j - \mathbf{b}_k\|_2^2 \cdot I_j $$
    where $\mathbf{u}_j$ is the color feature, $I_j$ is the importance score, and $\mathcal{R}(b_k)$ is the set of voxels assigned to code $b_k$.
    *   *Implementation Note:* Use iterative optimization. Randomly select a batch of voxels. Calculate Euclidean distance to all code vectors. Assign voxel to nearest code. Update code vector using moving average [S1].

3.  **Code Update Rule:**
    During initialization, update code vector $b_k$ for assigned voxels [S1]:
    $$ \mathbf{b}_k := \lambda_d \mathbf{b}_k + (1 - \lambda_d) \sum_{\mathbf{v}_j \in \mathcal{R}(\mathbf{b}_k)} I_j \mathbf{u}_j $$

4.  **Code Expiration (Handling Inactive Codes):**
    Track capacity $s_k = \sum_{\mathbf{v}_j \in \mathcal{R}(b_k)} I_j$. Rank codes by capacity. Expire $J$ codes with lowest capacity. Reinitialize them with features of the top $J$ most important voxels in the current batch [S1].

5.  **Voxel Selection (Non-VQ Voxels):**
    Determine the keeping threshold $\theta_k$ using the inverse cumulative distribution function $F^{-1}$ of importance scores [S1]:
    $$ \theta_k = Q(\beta_k) = F^{-1}(\beta_k) $$
    Voxels with importance score $I_j > \theta_k$ are marked as **non-VQ voxels** and stored directly. Voxels with $I_j \le \theta_k$ are assigned to the codebook [S1].

6.  **Joint Finetuning:**
    Fix the voxel-to-codebook mapping. Tune four components: code vectors, density grid, non-VQ voxels, and small network (if applicable) [S1].
    *   Update code vectors by synchronizing weights across assigned voxels every $i$ iterations [S1].
    *   *Finetuning Steps:* 10,000 steps for VQ-DVGO (8196 rays/batch); 25,600 steps for VQ-Plenoxels (batch size 5000) [S1].

7.  **Post-Processing:**
    *   Apply 8-bit uniform weight quantization to density voxels and non-VQ feature voxels (cast full-precision float to unsigned int) [S1].
    *   Do not quantize the codebook [S1].
    *   Encode all components (mask, codebook, indices, quantized grids) with entropy encoding (LZ77) and pack together [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vector_quantization_for_volumetric_radiance_fields(voxels, importance_scores, K=4096, beta_k=0.6, lambda_d=0.8, J=10, finetune_steps=10000):
    """
    Compress volumetric radiance fields using Vector Quantization.
    
    Args:
        voxels: List of voxel objects, each with .color_feature (vector) and .importance_score (float).
        importance_scores: List of importance scores corresponding to voxels.
        K: Codebook size.
        beta_k: Keep quantile for non-VQ voxels.
        lambda_d: Decay factor for code update.
        J: Number of expired codes per step.
        finetune_steps: Number of finetuning iterations.
        
    Returns:
        compressed_data: Dictionary containing codebook, indices, mask, quantized grids.
    """
    N = len(voxels)
    
    # 1. Initialization
    # Initialize codebook B with K random voxels or via k-means
    codebook = initialize_codebook(voxels, K)
    
    # 2. Codebook Initialization via Iterative Optimization
    for iter in range(100): # Approximation steps for initialization
        batch_indices = random_sample(N, batch_size=10000)
        batch_voxels = [voxels[i] for i in batch_indices]
        
        # Assign voxels to nearest code
        assignments = {} # code_index -> list of voxel indices
        for idx, v in enumerate(batch_voxels):
            dists = [euclidean_distance(v.color_feature, c) for c in codebook]
            best_code_idx = argmin(dists)
            if best_code_idx not in assignments:
                assignments[best_code_idx] = []
            assignments[best_code_idx].append(idx)
            
        # Update codes
        for code_idx, voxel_indices in assignments.items():
            if not voxel_indices:
                continue
            weighted_sum = sum(voxels[i].importance_score * voxels[i].color_feature for i in voxel_indices)
            capacity = sum(voxels[i].importance_score for i in voxel_indices)
            # Moving average update
            codebook[code_idx] = lambda_d * codebook[code_idx] + (1 - lambda_d) * weighted_sum
            
        # 3. Code Expiration
        # Track capacity for all codes
        capacities = [0.0] * K
        for code_idx, voxel_indices in assignments.items():
            capacities[code_idx] += sum(voxels[i].importance_score for i in voxel_indices)
            
        # Expire J codes with lowest capacity
        sorted_codes = sorted(range(K), key=lambda k: capacities[k])
        expired_codes = sorted_codes[:J]
        top_important_voxels = sorted(range(N), key=lambda i: importance_scores[i], reverse=True)[:J]
        
        for code_idx, voxel_idx in zip(expired_codes, top_important_voxels):
            codebook[code_idx] = voxels[voxel_idx].color_feature
            
    # 4. Determine Non-VQ Voxels
    # Calculate threshold theta_k
    sorted_scores = sorted(importance_scores)
    # Find index corresponding to beta_k quantile
    threshold_idx = int(beta_k * N)
    theta_k = sorted_scores[threshold_idx] if threshold_idx < N else sorted_scores[-1]
    
    non_vq_indices = []
    vq_indices = []
    
    for i, score in enumerate(importance_scores):
        if score > theta_k:
            non_vq_indices.append(i)
        else:
            vq_indices.append(i)
            
    # 5. Final Assignment for VQ Voxels
    vq_code_indices = []
    for i in vq_indices:
        dists = [euclidean_distance(voxels[i].color_feature, c) for c in codebook]
        vq_code_indices.append(argmin(dists))
        
    # 6. Joint Finetuning
    # Fix mapping. Tune codebook, density, non-VQ features.
    for step in range(finetune_steps):
        # Render batch rays
        # Compute gradients w.r.t codebook, density, non-VQ features
        # Update codebook by synchronizing weights across assigned voxels
        # (Implementation detail: aggregate gradients from all voxels assigned to a code)
        pass 
        
    # 7. Post-Processing
    # Quantize density grid and non-VQ features to 8-bit
    quantized_density = quantize_8bit(density_grid)
    quantized_non_vq_features = quantize_8bit([voxels[i].color_feature for i in non_vq_indices])
    
    # Create mask
    # 0: Pruned, 1: VQ, 2: Non-VQ
    mask = create_mask(N, pruned_indices, vq_indices, non_vq_indices)
    
    # Encode with LZ77
    compressed_data = {
        'codebook': codebook,
        'vq_indices': vq_code_indices,
        'mask': mask,
        'quantized_density': quantized_density,
        'quantized_non_vq_features': quantized_non_vq_features
    }
    
    return entropy_encode(compressed_data)

def euclidean_distance(v1, v2):
    return sum((a - b) ** 2 for a, b in zip(v1, v2)) ** 0.5

def quantize_8bit(values):
    # Cast full-precision float to unsigned int (8-bit uniform)
    # Implementation depends on specific normalization range
    return [int(v * 255) for v in values]
```

## Subtopics

### [VQRF (Vector Quantized Radiance Fields)](1903-vqrf-vector-quantized-radiance-fields.md)
[1903-vqrf-vector-quantized-radiance-fields.md]: A compression framework designed to reduce the storage footprint of volumetric radiance fields (such as DVGO, Plenoxels, and TensoRF) to approximately 1 MB while maintaining high rendering fidelity. It achieves this by combining adaptive voxel pruning with learnable vector quantization, outperforming uncompressed baselines and other size-focused methods in PSNR-size trade-offs.

### [Neural Vector Quantization](10103-neural-vector-quantization.md)
[10103-neural-vector-quantization.md]: A quantization technique that quantizes 3D Gaussian parameters grouped by K-means and compresses distilled SH features using lightweight neural decoders to minimize storage costs.

### Flag-based Classified Hierarchical Vector Quantization
[S3]: A dictionary-based method where each voxel is reconstructed on-the-fly by accessing selected codewords, supporting per-voxel random access.

### Compressed Volume Format with Precomputed Normals
[S5, unofficial]: A compressed volume format that may be rendered directly with minimal impact on rendering time, useful for extracting information from large datasets. It provides fast random access to voxel values by decoding voxels with a simple access to the codebook. The scheme consists of a vector of density values and the precomputed normal fields of voxels in the partitioning subblocks. It quantifies data into a codebook and represents each subblock by an appropriate index. Composing partial images in correct order, a ray casting with parallel projection is accelerated by shading only the vectors in the codebook. Some blockiness and contouring was reported. It achieved a compression ratio of 5:1 for volume data of $128^3$ [S5, unofficial].

### VQ-Based Model Compression
[S6]: Compresses each basic iVR-GS model using vector quantization to enhance scalability for large numbers of transfer functions without sacrificing reconstruction accuracy. It samples transfer functions uniformly over the whole voxel value range to ensure both content-rich and content-poor parts are included during compression evaluation. At a model size of ~280 MB, VQ compression allows nearly 4× the number of TFs compared to non-VQ iVR-GS [S6].

### Energy-based Thresholding for Vector Quantization
[S7, unofficial]: A technique to reduce the bitstream size of Vector Quantization (VQ) by discarding low-energy code vectors. It replaces vectors with energy below a threshold T with a predefined zero-energy vector Z, allowing entropy encoders to compress the stream more effectively. It designs a codebook matrix M with n rows, ensuring one row Z (index z) has zero energy (all elements are zero). For each index i from the encoder, it calculates the squared magnitude $|M_i|^2$ of the code vector $M_i$. If $|M_i|^2 > T$, it transmits index i to the entropy encoder. If $|M_i|^2 \le T$, it replaces index i with z (the index of the zero-energy vector Z) before transmitting to the decoder. The threshold T is individual for each pyramid layer. Larger threshold values are assigned to lower pyramid levels to discard more coefficients. Unlike standard VQ which transmits all indices, this method maps most indices to a single value (z), creating a sparse stream ideal for entropy coding (LZ/Huffman) [S7, unofficial].

## Operating characteristics
*   **Time Complexity:** The computational complexity of reconstructing VQ is linear $O(N)$, where $N$ is the number of blocks [S2][S3].
*   **Memory Complexity:** Storage cost is reduced from $N \times C \times 16$ bits to $16KC + N \log_2(K)$ bits for features, plus density and mask storage [S1].
*   **Compression Rate:** The compression rate $r$ is given by [S1]:
    $$ r = \frac{16NC}{N\log_2(K) + 16KC} $$
    For $K=4096$, VQ-DVGO achieves an average compression ratio of 75× on Synthetic-NeRF with a 0.13 dB PSNR drop. VQ-Plenoxels achieves 20× compression with a 0.2 dB drop. VQ-TensoRF achieves 20× compression, outperforming TensoRF-CP in size/quality trade-offs [S1].
*   **Strengths:** High compression ratios (up to 75×), low PSNR loss, fast random access reconstruction on GPU [S1][S2][S3].
*   **Limitations:** Requires joint finetuning which adds training time (50% of original training time for DVGO, 20% additional for Plenoxels) [S1]. Inactive code issue can degrade representation if not handled [S1].
*   **When to use:** When storage footprint is critical (e.g., web-based rendering, mobile devices) and some training time overhead is acceptable.
*   **When not to use:** When real-time training is required or when the scene has very low redundancy (e.g., TensoRF already has small size, so gains are smaller) [S1].

## Background
Vector Quantization for Volumetric Radiance Fields was introduced in the paper "Compressing Volumetric Radiance Fields to 1 MB" [S1]. It builds on the concept of dictionary-based VQ used in general volume rendering [S2][S3][S4, unofficial], where voxels are reconstructed on-the-fly by accessing codewords. The method specifically adapts this to radiance fields like DVGO [S2], Plenoxels [S1], and TensoRF [S1] to address their large storage requirements. It is used in contexts requiring high-fidelity compression of neural radiance fields, such as web-based visualization and mobile rendering [S1][S5, unofficial].

## Sources
[S1] Compressing Volumetric Radiance Fields to 1 MB, https://doi.org/10.1109/cvpr52729.2023.00411, fetched 2026-10-26
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S4] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
[S5] Volumetric Medical Images Visualization on Mobile Devices, http://hdl.handle.net/2099.1/11308, fetched 2026-10-26 (unofficial)
[S6] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
[S7] Confocal microscopic image sequence compression using vector quantization and 3D pyramids, http://eprints.gla.ac.uk/3795/1/confocal_image_sequence.pdf, fetched 2026-10-30 (unofficial)
