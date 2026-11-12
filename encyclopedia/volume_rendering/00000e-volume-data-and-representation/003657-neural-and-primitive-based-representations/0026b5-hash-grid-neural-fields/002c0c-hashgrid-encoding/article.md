# HashGrid Encoding

Also called: Hash Grid, Hash Encodings, GridEncoding, Instant NGP encoding, Instant Neural Graphics Primitives Encoding

## Overview
HashGrid Encoding maps continuous spatial coordinates to high-dimensional feature vectors using a hierarchy of sparse hash tables at multiple resolutions [S3, unofficial][S13]. It solves the computational bottleneck of vanilla Neural Radiance Fields (NeRF), which rely on deep Multi-Layer Perceptrons (MLPs) that are slow to train and render [S3, S10, unofficial]. By replacing heavy MLP inference with fast hash table lookups and trilinear interpolation, it accelerates training by 1000–10,000× and rendering by >3000× compared to vanilla NeRF, while maintaining comparable visual quality [S3, S10, unofficial]. It sets itself apart from frequency-based encodings (like Fourier features) by learning spatially adaptive features directly in the grid, allowing it to capture both global shape and fine local textures more effectively [S7, unofficial]. Unlike tree-based encodings (e.g., ACORN), it avoids serial pointer-chasing and execution divergence, mapping well to GPUs [S3, unofficial][S13].

## How it works
**Inputs and Outputs:**
*   **Input:** A tensor of input coordinates $\mathbf{x} \in \mathbb{R}^{B \times D}$, where $B$ is the batch size and $D$ is the spatial dimensionality (typically 3 for 3D space, or 2 for 2D images) [S9, S12, unofficial]. Values are expected to be normalized to the range $[0, 1]$ [S9, S12, unofficial].
*   **Output:** A tensor of encoded features $\mathbf{y} \in \mathbb{R}^{B \times (L \cdot C)}$, where $L$ is the number of resolution levels and $C$ is the number of feature channels per level [S9, S12, unofficial]. Optionally, the gradient of the output with respect to the input, $\frac{\partial \mathbf{y}}{\partial \mathbf{x}} \in \mathbb{R}^{B \times L \times D \times C}$, is computed for backpropagation [S9, S12, unofficial].

**Data Structures:**
The encoding consists of $L$ hash tables (grids). Each level $l \in \{0, \dots, L-1\}$ has a hash table of size $T_l$ (number of entries) and stores feature vectors of dimension $C$ [S3, S12, unofficial].
*   **Embeddings:** A flattened 1D array `embeddings` of shape $[\sum_{l=0}^{L-1} T_l, C]$ [S9, S12, unofficial]. Entries are typically stored at half precision (2 bytes) for memory efficiency, with a master copy in full precision for stable mixed-precision updates [S3, unofficial][S13].
*   **Offsets:** An integer array `offsets` of size $L+1$, where `offsets[l]` is the starting index in `embeddings` for level $l$ [S9, S12, unofficial]. The hash table size for level $l$ is $T_l = \text{offsets}[l+1] - \text{offsets}[l]$ [S12, unofficial].
*   **Parameters:**
    *   $L$: Number of resolution levels. Typical value: 16 [S3, unofficial][S13].
    *   $C$: Number of feature channels per level. Supported values: 1, 2, 4, 8. $C=2$ provides the best cost-quality trade-off on modern GPUs [S3, S9, S12, unofficial].
    *   $H$: Base resolution (grid size at level 0) [S9, S12, unofficial].
    *   $S$: Scaling factor exponent. Controls exponential resolution growth [S9, S12, unofficial].
    *   $T$: Max entries per level (hash table size). Tunable hyper-parameter trading parameters for quality. Typical range: $[2^{14}, 2^{24}]$ [S3, unofficial][S13].
    *   $N_{\min}$: Coarsest resolution. Typical value: 16 [S13].
    *   $N_{\max}$: Finest resolution. Tunable to match finest detail in training data. Typical range: $[512, 524288]$ [S13].

**Algorithm:**
The encoding is computed level by level to optimize GPU cache usage [S3, unofficial][S13]. For each level $l$, the input coordinates are scaled to the resolution of that level, mapped to integer grid coordinates, hashed to table indices, and trilinearly interpolated.

1.  **Calculate Resolution and Scale:**
    For each level $l$, compute the scale factor and the integer grid resolution:
    $$ \text{scale}_l = 2^{l \cdot S} \cdot H - 1 $$
    $$ \text{resolution}_l = \lceil \text{scale}_l \rceil + 1 $$
    [S4, S9, S12, unofficial][S13]

2.  **Normalize and Scale Coordinates:**
    For each input coordinate $\mathbf{x}_b$ (batch element $b$), normalize to $[0, 1]$ if not already, then scale to grid space:
    $$ \mathbf{pos}_b = \mathbf{x}_b \cdot \text{scale}_l + \mathbf{offset} $$
    where $\mathbf{offset}$ is $0.0$ if `align_corners` is true, and $0.5$ if false (center-aligned sampling) [S4, S9, S12, unofficial].

3.  **Compute Integer Grid Coordinates:**
    Compute the integer grid position and the fractional part for interpolation:
    $$ \mathbf{pos\_grid}_b = \lfloor \mathbf{pos}_b \rfloor $$
    $$ \mathbf{frac}_b = \mathbf{pos}_b - \mathbf{pos\_grid}_b $$
    [S4, S9, S12, unofficial]

4.  **Hash Lookup:**
    For each of the $2^D$ corner vertices of the voxel defined by $\mathbf{pos\_grid}_b$:
    *   Determine the local grid coordinate $\mathbf{pos\_grid\_local}$ for the corner. For dimension $d$, if the corner bit is 0, use $\mathbf{pos\_grid}_b[d]$; if 1, use $\mathbf{pos\_grid}_b[d] + 1$ [S4, S12, unofficial].
    *   Compute the hash index using a spatial hash function. A common implementation uses XOR with large primes:
        $$ \text{hash\_val} = \bigoplus_{d=0}^{D-1} (\mathbf{pos\_grid\_local}[d] \times \pi_d) $$
        where $\pi_d$ are unique large primes (e.g., $\pi_0=1, \pi_1=2654435761, \pi_2=805459861$ for $D=3$) [S4, S9, S12, unofficial][S13].
    *   Map the hash value to the table index:
        $$ \text{index} = (\text{hash\_val} \pmod{T_l}) \cdot C + c $$
        for each channel $c \in \{0, \dots, C-1\}$ [S4, S9, S12, unofficial].
    *   *Note:* If the stride (calculated as $\sum_{d=0}^{D-1} \mathbf{pos\_grid\_local}[d] \times \prod_{k=0}^{d-1} (\text{resolution}_l+1)$) exceeds $T_l$, the hash function is used; otherwise, linear indexing may be used [S4, S12, unofficial].

5.  **Trilinear Interpolation:**
    Compute the interpolation weight $w$ for the corner based on the fractional coordinates:
    $$ w = \prod_{d=0}^{D-1} \begin{cases} \mathbf{frac}_b[d] & \text{if corner bit } d \text{ is 1} \\ 1 - \mathbf{frac}_b[d] & \text{if corner bit } d \text{ is 0} \end{cases} $$
    [S4, S9, S12, unofficial][S13]
    Accumulate the weighted feature vectors from the hash table into the output feature vector for level $l$:
    $$ \mathbf{y}_{b,l} += w \cdot \text{embeddings}[\text{index}] $$
    [S4, S9, S12, unofficial]

6.  **Concatenation:**
    Concatenate the feature vectors from all $L$ levels to form the final encoded input $\mathbf{y}_b \in \mathbb{R}^{L \cdot C}$ [S3, unofficial][S13].

**Backpropagation:**
Gradients are computed with respect to the embeddings and the input coordinates $\mathbf{x}$.
*   **Embedding Gradients:** For each corner, the gradient of the loss $\mathcal{L}$ with respect to the embedding is accumulated using atomic addition:
    $$ \frac{\partial \mathcal{L}}{\partial \text{embeddings}[\text{index}]} += w \cdot \frac{\partial \mathcal{L}}{\partial \mathbf{y}_{b,l}} $$
    [S4, S9, S12, unofficial]
    *Optimization:* Use `__half2` vectorization for $C=2$ to speed up atomic additions [S4, S9, S12, unofficial]. Skip Adam steps for entries with zero gradient to save ~10% performance [S3, unofficial].
*   **Input Gradients:** Compute $\frac{\partial \mathbf{y}}{\partial \mathbf{x}}$ by summing contributions from all levels. For each dimension $d$, compute the partial derivative by considering the left and right neighbors along $d$ and using the derivative of the interpolation function (e.g., $1.0$ for linear, $6x(1-x)$ for smoothstep) [S9, S12, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def hash_grid_encode(inputs, offsets, embeddings, L, C, H, S, align_corners=True):
    """
    inputs: [B, D] float tensor, values in [0, 1]
    offsets: [L+1] int array, cumulative hash table sizes
    embeddings: [sum(offsets), C] float tensor
    L: int, number of levels
    C: int, channels per level
    H: float, base resolution
    S: float, scaling factor exponent
    align_corners: bool, if True offset is 0.0, else 0.5
    """
    B, D = inputs.shape
    outputs = torch.zeros(L, B, C, dtype=inputs.dtype, device=inputs.device)
    dy_dx = torch.zeros(B, L, D, C, dtype=inputs.dtype, device=inputs.device)
    
    # Primes for hashing (example for D=3)
    primes = [1, 2654435761, 805459861]
    
    for l in range(L):
        # 1. Calculate scale and resolution
        scale = math.exp2(l * S) * H - 1.0
        resolution = math.ceil(scale) + 1
        hashmap_size = offsets[l+1] - offsets[l]
        
        # 2. Scale coordinates
        offset_val = 0.0 if align_corners else 0.5
        pos = inputs * scale + offset_val
        
        # 3. Compute integer grid coords and fractional parts
        pos_grid = torch.floor(pos).long()
        frac = pos - pos_grid.float()
        
        # 4. Iterate over 2^D corners
        level_output = torch.zeros(B, C, dtype=inputs.dtype, device=inputs.device)
        level_grad_input = torch.zeros(B, D, C, dtype=inputs.dtype, device=inputs.device)
        
        for idx in range(1 << D):
            # Determine local grid coords and weight
            w = 1.0
            pos_grid_local = torch.zeros(D, dtype=torch.long, device=inputs.device)
            
            for d in range(D):
                if (idx >> d) & 1:
                    w *= frac[:, d]
                    pos_grid_local[d] = pos_grid[:, d] + 1
                else:
                    w *= 1.0 - frac[:, d]
                    pos_grid_local[d] = pos_grid[:, d]
            
            # 5. Hash lookup
            # Compute hash value
            hash_val = torch.zeros(B, dtype=torch.int32, device=inputs.device)
            for d in range(D):
                hash_val = torch.bitwise_xor(hash_val, (pos_grid_local[d].int() * primes[d]))
            
            # Map to index
            indices = (hash_val % hashmap_size) * C
            
            # Fetch embeddings and accumulate
            # Note: In practice, this is done in a fused CUDA kernel for performance
            # Here we simulate the accumulation for clarity
            for c in range(C):
                emb = embeddings[indices + c]
                level_output[:, c] += w * emb[:, c]
                
                # Gradient w.r.t embeddings would be accumulated here via atomicAdd in CUDA
                # For dy_dx, we need the derivative w.r.t input
                # This is simplified; full dy_dx requires partial derivatives along each dim
                pass
        
        # 6. Compute dy_dx (simplified linear interpolation derivative)
        # For each dimension d, compute gradient by looking at left/right neighbors
        for d in range(D):
            # Left neighbor index
            pos_grid_left = pos_grid.clone()
            pos_grid_left[:, d] -= 1
            # Right neighbor index
            pos_grid_right = pos_grid.clone()
            pos_grid_right[:, d] += 1
            
            # Compute hashes for left and right
            # ... (similar to above) ...
            
            # Compute weight product for other dimensions
            # w_other = product of frac or 1-frac for dims != d
            # ...
            
            # Gradient contribution
            # dy_dx[:, l, d, :] = scale * w_other * (emb_right - emb_left)
            pass
            
        outputs[l] = level_output
        dy_dx[:, l, :, :] = level_grad_input
        
    return outputs, dy_dx
```

## Subtopics

### [Progressive Training](9142-progressive-training.md)
Progressive training is a strategy for optimizing multi-resolution hash encodings that gradually activates higher resolution levels over the course of training. It solves the problem of training instability inherent in hash-based neural fields: using only low-resolution grids leads to underfitting (smooth geometry lacking details), while using all high-resolution grids from the start causes overfitting to specific camera views, resulting in noise, artifacts, and view inconsistency. By starting with coarse levels and incrementally introducing finer details, the method ensures view consistency and efficient convergence. It differs from [Instant-NGP](6604-instant-ngp.md) primarily in its explicit multi-rate-distortion supervision and residual grid freezing mechanisms, and from [Dense Grid Encoding](9335-dense-grid-encoding.md) by leveraging spatial hashing for memory efficiency.

### [Dense Grid Encoding](9335-dense-grid-encoding.md)
Dense Grid Encoding is a parametric input encoding method that maps input coordinates to multi-dimensional feature vectors by storing trainable parameters in a dense, regular grid structure. It solves the problem of efficiently representing high-frequency scene details in neural radiance fields (NeRF) by trading a larger memory footprint for reduced computational cost during training. Unlike fixed frequency encodings, it allows the network to learn spatially varying features, but unlike sparse or hash-based variants, it allocates parameters uniformly across the entire domain, which can be wasteful in empty space.

### [ACORN](9336-acorn.md)
ACORN is a parametric feature encoding method that uses a tree subdivision of the input domain to output dense feature grids in leaf nodes, which are then linearly interpolated. It yields a greater degree of adaptivity compared with previous parametric encodings, albeit at greater computational cost.

### [Multi-resolution Hash Encoding](10200-multi-resolution-hash-encoding.md)
Multi-resolution Hash Encoding (MHE) is a coordinate-based feature encoding method that maps 3D spatial coordinates to learnable feature vectors using a hierarchy of sparse hash tables at increasing resolutions. It solves the computational bottleneck of vanilla Neural Radiance Fields (NeRF), which rely on deep Multi-Layer Perceptrons (MLPs) that are slow to train and render. By replacing the heavy MLP inference with fast hash table lookups and trilinear interpolation, MHE accelerates training by 1000–10,000× and rendering by >3000× compared to vanilla NeRF, while maintaining comparable visual quality. It sets itself apart from frequency-based encodings (like Fourier features) by learning spatially adaptive features directly in the grid, allowing it to capture both global shape and fine local textures more effectively.

### [RHINO](10245-rhino.md)
RHINO is a regularization framework for hash-based Implicit Neural Representations (INRs) that restores interpolation quality by introducing an additional input coordinate with an analytical and continuous transformation $T$ to the network, compensating for the broken gradient flow between input coordinates and hash-keys. It solves the problem of discontinuous and noisy interpolation in unsampled regions caused by the non-analytical mapping from spatial coordinates to hash keys. Unlike [Instant-NGP](6604-instant-ngp.md) which relies solely on hash-based indexing, RHINO rebuilds the connection between the input space and the network output, ensuring that interpolation follows spectral bias.

### [HollowNeRF Saliency Grid Weighting](10745-hollownerf-saliency-grid-weighting.md)
HollowNeRF Saliency Grid Weighting is a collision mitigation technique for hash grid-based neural radiance fields that learns a 3D probability distribution of feature presence to weight feature vectors fetched from a multi-level hashgrid. Unlike Instant-NGP, which distributes hash collisions evenly across voxels, this method learns to "shuffle" collisions by assigning higher saliency weights to important features, allowing them to dominate shared hash buckets during gradient updates while pruning less significant features.

## Operating characteristics
**Time and Memory Complexity:**
*   **Forward Pass:** $O(B \cdot L \cdot 2^D \cdot C)$ operations per batch element [S9, unofficial].
*   **Backward Pass:** $O(B \cdot L \cdot 2^D \cdot N_C)$ with atomic addition overhead, where $N_C = \min(2, C)$ [S9, S12, unofficial].
*   **Memory:** $O(\sum_{l=0}^{L-1} T_l \cdot C)$ for the embedding table, where $T_l$ is the hash table size for level $l$ [S12, unofficial][S13]. The total number of trainable parameters is $O(T \cdot L \cdot F)$, where $T$ is the max entries per level and $F$ is the feature dimension [S13].

**Measured Performance:**
*   Training speed is over 8× faster than frequency encoding with similar parameter counts, achieving state-of-the-art quality in seconds [S3, unofficial][S13].
*   Rendering performance is in the tens of milliseconds at 1920×1080 resolution [S3, unofficial][S13].
*   Performance drops significantly if the hash table size exceeds the L2 cache limit (e.g., $2 \cdot T \cdot F > 6 \text{ MB}$ on RTX 3090) [S3, unofficial][S13].

**Strengths and Limitations:**
*   **Strengths:** High training and rendering speed; memory efficient for sparse scenes; adaptive to training data distribution; no structural updates needed during training [S3, unofficial][S13].
*   **Limitations:** Hash collisions can cause artifacts (e.g., surface roughness in SDFs) if the table size is too small [S3, S7, unofficial][S13]; performance sensitive to hash table size relative to GPU cache; input coordinates must be in $[0, 1]$ [S9, S12, unofficial].

**When to Use:**
*   Use for real-time or near-real-time neural graphics primitives (NeRF, SDF, radiance caching) where training speed and rendering latency are critical [S3, unofficial][S13].
*   Avoid if exact collision-free interpolation is required without relying on gradient averaging, or if the scene is extremely dense and uniform (dense grids may be more efficient) [S3, unofficial][S13].

## Background
HashGrid Encoding was introduced by Thomas Müller, Alex Evans, Christoph Schied, and Alexander Keller in the paper "Instant Neural Graphics Primitives with a Multiresolution Hash Encoding" (2022) [S3, unofficial]. It was developed to address the high computational cost of training and evaluating neural graphics primitives, which traditionally relied on deep MLPs or complex tree-based structures [S3, unofficial]. The method combines multi-resolution decomposition with spatial hashing to reduce parameter count while maintaining reconstruction quality [S3, unofficial][S13]. It has become a standard component in many modern neural rendering frameworks, including Instant-NGP [S3, unofficial][S13].

## Sources
[S1] One Is All: Bridging the Gap between Neural Radiance Fields Architectures with Progressive Volume Distillation, https://doi.org/10.1609/aaai.v37i1.25135, fetched 2026-10-26
[S2] Optimizing Real-Time Neural Radiance Fields (NeRF) for Dynamic 3D Scene Rendering on Consumer GPUs, https://doi.org/10.5281/zenodo.21889839, fetched 2026-10-26 (unofficial)
[S3] Nvidia – learning representations of positions in a 3D scene [pdf], https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.pdf, fetched 2026-10-26 (unofficial)
[S4] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S5] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S6] Progressive Volume Distillation with Active Learning for Efficient NeRF Architecture Conversion, https://doi.org/10.1007/s11263-026-02815-1, fetched 2026-10-26
[S7] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S8] (PDF) Volume Feature Rendering for Fast Neural Radiance Field ..., https://www.researchgate.net/publication/371136390_Volume_Feature_Rendering_for_Fast_Neural_Radiance_Field_Reconstruction, fetched 2026-10-27 (unofficial)
[S9] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
[S10] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
[S11] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
[S12] DESY-FS-PETRA/HoloINR, https://github.com/DESY-FS-PETRA/HoloINR, fetched 2026-10-27 (unofficial)
[S13] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
