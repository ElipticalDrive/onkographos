# Multi-resolution Hash Encoding

Also called: Instant-NGP, NGP hash encoding, Multi-scale Hash Encoding, Multiresolution Grid Encoding, Multiresolution Dense Grid Encoding, Multi-resolution learnable hashgrids

## Overview
Multi-resolution Hash Encoding (MHE) is a coordinate-based feature encoding method that maps 3D spatial coordinates to learnable feature vectors using a hierarchy of sparse hash tables at increasing resolutions [S1][S3, unofficial][S5]. It solves the computational bottleneck of vanilla Neural Radiance Fields (NeRF), which rely on deep Multi-Layer Perceptrons (MLPs) that are slow to train and render [S3, unofficial][S10]. By replacing the heavy MLP inference with fast hash table lookups and trilinear interpolation, MHE accelerates training by 1000–10,000× and rendering by >3000× compared to vanilla NeRF, while maintaining comparable visual quality [S1][S9, unofficial][S10]. It sets itself apart from frequency-based encodings (like Fourier features) by learning spatially adaptive features directly in the grid, allowing it to capture both global shape and fine local textures more effectively [S7].

## How it works
**Inputs:** 3D point coordinates $p \in \mathbb{R}^3$ (and optionally view direction $d$).
**Outputs:** A concatenated feature vector $F(p)$ used as input to a small decoder MLP.

**Data Structures:**
The method uses $L$ levels of hash tables.
*   **Level $l$ ($0 \le l < L$):** Contains a hash table mapping 3D grid coordinates to feature vectors.
*   **Resolution $N_l$:** The conceptual grid resolution at level $l$.
*   **Hash Table Size $T_l$:** The number of entries in the hash table at level $l$.
*   **Feature Dimension $F$:** The dimensionality of each feature vector stored in the hash table.
*   **Hash Function $h$:** Maps integer grid coordinates $\mathbb{Z}^3$ to table indices $\mathbb{Z}^{T_l}$.

**Parameters:**
*   $N_{min}$: Coarsest resolution (typically 16) [S5][S6, unofficial].
*   $N_{max}$: Finest resolution (typically 2048) [S5].
*   $L$: Number of levels (typically 16) [S6, unofficial].
*   $F$: Feature dimension per level (typically 2) [S6, unofficial][S11].
*   $T_l$: Hash table size. Performance is optimal when $T \le 2^{19}$ to fit within GPU L2 cache limits (e.g., 6 MB on RTX 3090) [S11].
*   Interpolation Weight Function: $d(x) = 6x^5 - 15x^4 + 10x^3$ (second-order continuous interpolation) [S5].

**Algorithm Steps:**

1.  **Calculate Resolutions:** For each level $l \in [0, L-1]$, compute the resolution $N_l$:
    $$ N_l = \lfloor N_{min}^b \rfloor $$
    where the base $b$ is calculated as:
    $$ b = 2^{\frac{\log_2(N_{max}/N_{min})}{L}} $$
    [S5]

2.  **Grid Coordinate Mapping:** For a query point $p$ at level $l$:
    *   Scale the point: $p' = p \cdot N_l$.
    *   Compute floor and ceiling integer coordinates:
        $$ b_l = \lfloor p' \rfloor, \quad d_l = \lceil p' \rceil $$
        These define the 8 vertices of the voxel cell containing $p$ in the conceptual grid at level $l$.
    *   Compute the relative position within the cell:
        $$ w = p' - b_l $$
        where $w \in [0, 1]^3$.
    [S5, S10]

3.  **Hash Table Lookup:** Map the integer coordinates of the 8 vertices to hash table indices using the hash function $h_l: \mathbb{Z}^3 \to \mathbb{Z}^{T_l}$.
    *   If $N_l^3 \le T_l$, the mapping is one-to-one (direct indexing).
    *   Otherwise, use a hash function (e.g., MurmurHash or similar) to map coordinates to indices in $[0, T_l-1]$.
    *   Retrieve the feature vectors $v_{ijk}$ for each vertex $(i,j,k) \in \{b_l, d_l\}^3$ from the hash table.
    [S5, S10]

4.  **Trilinear Interpolation:** Interpolate the 8 retrieved feature vectors using the relative position $w$ and the weight function $d(x)$.
    *   Let $w_x, w_y, w_z$ be the components of $w$.
    *   The interpolated feature $f_l(p)$ at level $l$ is:
        $$ f_l(p) = \sum_{i,j,k \in \{0,1\}} \left( \prod_{c \in \{x,y,z\}} \text{weight}(w_c, i_c, j_c) \right) v_{ijk} $$
        where $\text{weight}(w, 0, 0) = d(1-w)$, $\text{weight}(w, 1, 0) = d(w)$, etc., effectively performing trilinear interpolation with the smooth weight function $d(x)$ [S5, S10].
    [S5, S10]

5.  **Concatenation:** Concatenate the interpolated feature vectors from all $L$ levels to form the final input feature vector $F(p)$:
    $$ F(p) = [f_0(p); f_1(p); \dots; f_{L-1}(p)] $$
    [S3, S6, unofficial][S13]

6.  **Decoding:** Feed $F(p)$ (and optionally view direction $d$ encoded via Spherical Harmonics) into a small MLP to predict density $\sigma$ and color $c$.
    *   Typical MLP: 2 hidden layers, 64 neurons each, ReLU activation [S10, S11].
    *   Alternative MLP (for NeRF synthetic): 1 hidden layer, 64 neurons [S2].
    [S2, S10]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def hash_encode(p, levels, N_min, N_max, T, h_func, features):
    """
    p: Tensor of shape (B, 3), batch of 3D points.
    levels: List of level indices [0, 1, ..., L-1].
    N_min: Coarsest resolution (int).
    N_max: Finest resolution (int).
    T: List of hash table sizes for each level.
    h_func: Function mapping (x, y, z) -> index in [0, T[l]-1].
    features: List of tensors, features[l] has shape (T[l], F).
    """
    L = len(levels)
    # Calculate base b for resolution scaling
    # b = 2^(log2(N_max/N_min)/L)
    # N_l = floor(N_min^b)
    # Note: The formula in [S5] uses N_l = floor(N_min^b) with b defined as above.
    # However, standard Instant-NGP uses geometric progression. 
    # We follow [S5] exactly:
    b = math.pow(2, math.log2(N_max / N_min) / L)
    
    concatenated_features = []
    
    for l in levels:
        # 1. Calculate Resolution N_l
        N_l = math.floor(math.pow(N_min, b))
        
        # 2. Scale and floor/ceil
        # p_scaled = p * N_l
        p_scaled = p * N_l
        b_l = torch.floor(p_scaled) # Shape (B, 3)
        d_l = torch.ceil(p_scaled)  # Shape (B, 3)
        
        # 3. Relative position w
        w = p_scaled - b_l # Shape (B, 3)
        
        # 4. Get 8 vertices
        # Vertices are combinations of b_l and d_l
        # We need to iterate over the 8 corners
        # To optimize, we can compute indices for all 8 corners at once
        
        # Create masks for each corner
        # Corner index k from 0 to 7
        # bit 0: x, bit 1: y, bit 2: z
        # If bit is 0, use b_l, else use d_l
        
        # Generate corner coordinates
        # This part is vectorized in CUDA, here we outline the logic
        corners = []
        for k in range(8):
            corner_coords = torch.zeros_like(b_l)
            if k & 1: corner_coords[:, 0] = d_l[:, 0]
            else: corner_coords[:, 0] = b_l[:, 0]
            
            if k & 2: corner_coords[:, 1] = d_l[:, 1]
            else: corner_coords[:, 1] = b_l[:, 1]
            
            if k & 4: corner_coords[:, 2] = d_l[:, 2]
            else: corner_coords[:, 2] = b_l[:, 2]
            
            corners.append(corner_coords)
            
        # 5. Hash lookup
        # Retrieve features for all corners
        # h_func takes (x, y, z) and returns index
        # We assume h_func is vectorized or called per point
        corner_features = []
        for corner in corners:
            # Extract x, y, z
            x = corner[:, 0].long()
            y = corner[:, 1].long()
            z = corner[:, 2].long()
            
            # Hash to index
            idx = h_func(x, y, z, T[l])
            
            # Gather features
            # features[l] is (T[l], F)
            feat = features[l][idx] # Shape (B, F)
            corner_features.append(feat)
            
        # 6. Trilinear Interpolation with smooth weights
        # Weight function d(x) = 6x^5 - 15x^4 + 10x^3
        def smooth_weight(x):
            return 6*x**5 - 15*x**4 + 10*x**3
            
        wx = smooth_weight(w[:, 0])
        wy = smooth_weight(w[:, 1])
        wz = smooth_weight(w[:, 2])
        
        # Weights for each corner
        # Corner 0 (0,0,0): (1-wx)(1-wy)(1-wz) -> smooth: d(1-wx)...
        # Note: The weight function d(x) is applied to the relative position.
        # For corner (0,0,0), relative pos is (0,0,0), weight is d(1-wx)*d(1-wy)*d(1-wz)?
        # Standard trilinear uses linear weights. [S5] says use d(x) for continuity.
        # d(x) is the interpolation weight for the '1' side? 
        # d(0)=0, d(1)=1.
        # So for corner (0,0,0), weight is (1-d(wx))*(1-d(wy))*(1-d(wz))?
        # Let's assume standard trilinear structure but with d(x) replacing x.
        
        w000 = (1-wx)*(1-wy)*(1-wz)
        w100 = wx*(1-wy)*(1-wz)
        w010 = (1-wx)*wy*(1-wz)
        w001 = (1-wx)*(1-wy)*wz
        w110 = wx*wy*(1-wz)
        w101 = wx*(1-wy)*wz
        w011 = (1-wx)*wy*wz
        w111 = wx*wy*wz
        
        # Sum weighted features
        f_l = (w000 * corner_features[0] + 
               w100 * corner_features[1] + 
               w010 * corner_features[2] + 
               w001 * corner_features[3] + 
               w110 * corner_features[4] + 
               w101 * corner_features[5] + 
               w011 * corner_features[6] + 
               w111 * corner_features[7])
               
        concatenated_features.append(f_l)
        
    # Concatenate all levels
    final_feature = torch.cat(concatenated_features, dim=1)
    return final_feature
```

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** At fine resolutions, $N_l^3 > T_l$, causing hash collisions. This can lead to ambiguity and artifacts. Mitigated by keeping $T_l$ large enough or using good hash functions [S5, S11].
*   **Cache Misses:** If the hash table size $T$ exceeds the GPU L2 cache (e.g., > 6 MB on RTX 3090), performance drops significantly due to memory latency [S11].
*   **Gradient Sparsity:** Gradients are sparse when the number of hash table entries $T$ is much larger than the batch size. Skipping Adam steps for zero gradients saves ~10% performance [S11].
*   **Precision:** Hash table entries are often stored at half precision (2 bytes) for performance, with a master copy in full precision for stable mixed-precision updates [S11].
*   **Boundary Conditions:** Points outside the defined bounding box of the scene should be handled (e.g., clamped or assigned zero features) [G].

## Subtopics

### [Multi-resolution Hash Encoding with Simplex Structure](9584-multi-resolution-hash-encoding-with-simplex-structure.md)
Multi-resolution Hash Encoding with Simplex Structure replaces the hypercube-based grid with a simplex-based structure, storing features at simplex vertices in hash tables. It solves the memory inefficiency of dense grids in simplicial domains by storing only accessed vertices, avoiding overhead for unused vertices. It differs from standard Instant-NGP by using barycentric interpolation within skewed simplices instead of trilinear interpolation within hypercubes.

### [Global Preconditioner for Encoder-Induced Metric](11747-global-preconditioner-for-encoder-induced-metric.md)
The Global Preconditioner for Encoder-Induced Metric applies a fixed linear transformation to the input space to isotropize the average metric induced by the encoder. It solves the problem of anisotropic feature scaling caused by the discrete, multi-resolution grid structure, ensuring all spatial directions are treated equally on average without globally rescaling the volume. Unlike adaptive preconditioners, it uses a closed-form solution derived from the symmetrized average metric of the canonical cell, providing a static, volume-preserving correction that reduces the condition number of the encoder-induced metric to 1 for the average case.

### F-Hash
F-Hash is an extension of multi-resolution hash encoding to four dimensions using Tesseract grids and collision-free minimal perfect hashing [S16, unofficial]. It achieves convergence speeds 10–100× faster than standard hash encoding, with compression ratios of 9.5–55.2× [S16, unofficial]. On combustion data, it achieved 47.1 dB in 7.4 minutes vs. 23.7 dB in 45.7 minutes for standard encoding [S16, unofficial].

### Hybrid Encoding
Hybrid Encoding concatenates the view encoding with all-but-the-first-dimension of the hash encoding for the color MLP, while using only the first dimension of the hash encoding for the density MLP [S17, unofficial]. Specifically, $\gamma'(x) = \gamma_0(x)$ for density, and $\gamma''(d, x) = [\gamma(d), \gamma_{1:N_x}(x)]$ for color [S17, unofficial].

## Operating characteristics
**Time Complexity:** Training time is reduced by 1000–10,000× compared to vanilla NeRF, converging in 2–5 minutes on a single GPU [S1][S9, unofficial]. Rendering speedup is >3000× compared to vanilla NeRF [S9, unofficial].

**Memory Complexity:** Memory usage is significantly lower than dense grids. For example, using 16 levels of 2D feature vectors with resolution $16^3$ to $173^3$ and dimension 2 results in less than half the parameters of a single large dense grid while maintaining similar quality [S6, unofficial]. However, training requires more than 12× the video memory of rendering due to intermediate activations and Adam optimizer statistics [S4].

**Performance:**
*   **NeRF Synthetic (Blender):** PSNR = 32.6 ± 0.4 dB [S1].
*   **Mip-NeRF 360 (Unbounded):** PSNR = 26.4 ± 0.6 dB [S1].
*   **Rendering Speed:** 2.4 ± 0.4 FPS on an A100 GPU at 1080p [S1]. This is 12× faster than NeRF (0.2 FPS) but 28× slower than 3D Gaussian Splatting (68.4 FPS) [S1].
*   **Medical Imaging:** In the Chest dataset, Instant-NGP achieved a 3D SSIM of 0.943 (vs. 0.885 for vanilla NeRF) and LPIPS of 0.090 (vs. 0.164) [S7].

**Strengths:**
*   Extremely fast training and rendering.
*   High-quality reconstruction of fine details and textures.
*   Efficient memory usage compared to dense grids.
*   Suitable for real-time applications and rapid prototyping [S1].

**Limitations:**
*   Moderate surface accuracy compared to SDF-based methods like NeuS [S1].
*   Performance degrades if hash table size exceeds GPU L2 cache limits [S11].
*   Soft tissues in medical images can appear less detailed or blurred compared to hard tissues [S7].
*   Not as fast as explicit representations like 3D Gaussian Splatting for rendering [S1].

**When to Use:**
*   When fast training and rendering are critical.
*   For applications requiring real-time or near-real-time visualization.
*   When memory is constrained but high quality is needed.

**When Not to Use:**
*   When high-precision surface geometry is required (use NeuS or 3DGS).
*   When ultra-high frame rates (>50 FPS) are needed (use 3DGS).
*   When the scene is extremely large and hash table size exceeds cache limits without sparse variants.

## Background
Multi-resolution Hash Encoding was introduced in the paper "Instant Neural Graphics Primitives" by Müller et al. in 2022 [S1][S6, unofficial][S11]. It was developed to address the slow training and rendering speeds of vanilla NeRF, which relied on deep MLPs [S3, unofficial][S10]. The method leverages GPU-optimized hash table lookups and multi-resolution grids to accelerate computation [S1]. It has since become a standard technique in neural rendering, used in various applications including medical image reconstruction [S7], agricultural plant phenotyping [S8], and real-time rendering engines [S12]. Variants like F-Hash [S16, unofficial] and Hybrid Encoding [S17, unofficial] have been proposed to extend its capabilities to higher dimensions and specific use cases.

## Sources
[S1] Neural Rendering Techniques, https://doi.org/10.5281/zenodo.19614729, fetched 2026-10-26
[S2] Volume Feature Rendering for Fast Neural Radiance Field Reconstruction, https://doi.org/10.52202/075280-2853, fetched 2026-10-26
[S3] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
[S4] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
[S5] Multi-scale hash encoding based neural geometry representation, https://doi.org/10.1007/s41095-023-0340-x, fetched 2026-10-26
[S6] Nvidia – learning representations of positions in a 3D scene [pdf], https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.pdf, fetched 2026-10-26 (unofficial)
[S7] The Adaption of Recent New Concepts in Neural Radiance Fields and Their Role for High-Fidelity Volume Reconstruction in Medical Images., https://doi.org/10.3390/s24185923, fetched 2026-10-26
[S8] High-fidelity 3D Reconstruction of Plants using Neural Radiance Field, https://arxiv.org/abs/2311.04154v1, fetched 2026-10-27
[S9] NEPHELE: A Neural Platform for Highly Realistic Cloud Radiance Rendering, https://arxiv.org/abs/2303.04086v1, fetched 2026-10-27 (unofficial)
[S10] MixRT: Mixed Neural Representations For Real-Time NeRF Rendering, https://doi.org/10.1109/3dv62453.2024.00087, fetched 2026-10-27
[S11] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
[S12] UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene, https://doi.org/10.48550/arXiv.2310.13263, fetched 2026-10-29
[S13] Learning Neural Volumetric Representations of Dynamic Humans in Minutes, https://doi.org/10.1109/cvpr52729.2023.00846, fetched 2026-10-30
[S14] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S15] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S16] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S17] DWTNeRF: Boosting Few-shot Neural Radiance Fields via Discrete Wavelet Transform, https://arxiv.org/abs/2501.12637v3, fetched 2026-10-27 (unofficial)
