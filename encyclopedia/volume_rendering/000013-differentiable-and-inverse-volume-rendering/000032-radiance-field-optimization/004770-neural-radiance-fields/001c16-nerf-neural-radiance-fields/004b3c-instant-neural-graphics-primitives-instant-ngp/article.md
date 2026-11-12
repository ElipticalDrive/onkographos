# Instant Neural Graphics Primitives (Instant NGP)

Also called: MLP with Multiresolution Hash Encoding, multiresolution learnable hashgrids

## Overview
Instant Neural Graphics Primitives (Instant NGP) is a method that greatly improves NeRF model training and inference speed using a learned parametric multi-resolution hash encoding and advanced ray marching techniques [S1]. It replaces the large, deep MLP of Vanilla NeRF with a compact structure using multiresolution learnable hashgrids to explicitly encode spatial information, reducing computation cost and improving training stability [S2, unofficial]. Unlike Vanilla NeRF, which updates all weights globally, Instant NGP updates only embeddings associated with the neighborhood of a sampled point, providing a local constraint that accelerates convergence [S2, unofficial].

## How it works
Instant NGP replaces the positional encoding and deep MLP of the original NeRF with a multi-resolution hash encoding scheme and a shallow decoder [S2, unofficial]. The method operates as follows:

**Inputs and Outputs:**
*   **Inputs:** A set of training images with known camera poses; a scene bounding box.
*   **Outputs:** A trained hash grid structure and a shallow MLP decoder capable of predicting volume density ($\sigma$) and view-dependent color ($\mathbf{c}$) for any 3D point and viewing direction.

**Data Structures:**
1.  **Multi-resolution Hash Grids:** A hierarchy of $L$ 3D grids $G^i$ (where $i=0 \dots L-1$) with increasing resolutions [S2, S4, unofficial].
    *   Each grid $G^i$ has a resolution $R_i$ (number of vertices per axis).
    *   Instead of storing a unique feature vector per vertex, each grid uses a fixed-size hash table $H^i$ of size $T_i$ [S2, unofficial].
    *   A hash function $h_i: \mathbb{Z}^3 \rightarrow \{0, \dots, T_i-1\}$ maps vertex indices to the hash table, allowing embeddings to be shared across vertices [S2, unofficial].
    *   Each entry in $H^i$ stores a trainable embedding vector $\mathbf{e} \in \mathbb{R}^d$ (where $d$ is the feature dimension, typically 2 or 4) [S4, unofficial].
2.  **Shallow MLP (Decoder):** A small multi-layer perceptron that takes the concatenated features from the hash grids and outputs density and color [S2, unofficial].

**Numbered Steps:**
1.  **Initialization:** Initialize $L$ hash grids with resolutions $R_0, \dots, R_{L-1}$ and hash table sizes $T_0, \dots, T_{L-1}$. Initialize the shallow MLP weights.
2.  **Ray Sampling:** For each pixel, cast a ray through the scene. Sample points $\mathbf{x}_j$ along the ray within the scene bounding box.
3.  **Feature Retrieval (Hash Encoding):** For each sampled point $\mathbf{x}_j$ and each resolution level $i$:
    *   Determine the 8 surrounding vertices of $\mathbf{x}_j$ in grid $G^i$.
    *   Compute the hash indices for these 8 vertices using $h_i$.
    *   Retrieve the 8 embedding vectors from hash table $H^i$ using these indices.
    *   Perform trilinear interpolation of these 8 embeddings based on $\mathbf{x}_j$'s fractional position within the voxel to obtain the feature vector $\mathbf{f}_i$ for level $i$ [S2, unofficial].
4.  **Feature Concatenation:** Concatenate the feature vectors from all $L$ levels: $\mathbf{F} = [\mathbf{f}_0; \mathbf{f}_1; \dots; \mathbf{f}_{L-1}]$ [S2, unofficial].
5.  **Decoding:** Pass $\mathbf{F}$ and the viewing direction $\mathbf{d}$ into the shallow MLP to predict density $\sigma$ and color $\mathbf{c}$ [S2, unofficial].
6.  **Volume Rendering:** Use the predicted $\sigma$ and $\mathbf{c}$ with standard volume rendering integration (e.g., differentiable volume rendering) to compute the pixel color [S1].
7.  **Optimization:** Compute the loss between rendered and ground-truth images. Update the hash table embeddings and MLP weights via gradient descent [S2, unofficial].

**Formulas:**
*   **Trilinear Interpolation:** For a point $\mathbf{x}$ in grid $G^i$, let $\mathbf{v}_k$ be the 8 surrounding vertices and $\mathbf{e}_k$ be their retrieved embeddings. The interpolated feature $\mathbf{f}_i$ is:
    $$ \mathbf{f}_i = \sum_{k=0}^{7} w_k \mathbf{e}_k $$
    where $w_k$ are the trilinear interpolation weights determined by the distance of $\mathbf{x}$ to $\mathbf{v}_k$ [S2, unofficial].
*   **Hash Mapping:** The index in the hash table for vertex $\mathbf{v}$ at resolution $i$ is:
    $$ \text{index} = h_i(\mathbf{v}) \pmod{T_i} $$
    where $h_i$ is a hash function (e.g., a simple integer hash) [S2, unofficial].

**Parameters:**
*   **Number of resolutions ($L$):** Typically 10–16 [S4, unofficial].
*   **Hash table size ($T_i$):** Grows with resolution. Not given in the sources.
*   **Feature dimension ($d$):** Typically 2 or 4 per level [S4, unofficial].
*   **MLP Architecture:** Shallow, e.g., 2–3 hidden layers [S2, unofficial].
*   **Ray Marching:** Advanced techniques include exponential stepping, empty space skipping, and sample compaction [S1].

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** Since hash tables are finite, collisions occur. The hash function and table size must be chosen to minimize detrimental collisions [S2, unofficial].
*   **Precision:** Trilinear interpolation requires precise floating-point arithmetic for fractional coordinates [S2, unofficial].
*   **Boundary Conditions:** Points outside the bounding box must be handled (e.g., treated as empty or background) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hash_encode(point, grids, hash_tables, hash_functions):
    """
    point: (3,) float32, query point in 3D space
    grids: list of resolution sizes (R_0, ..., R_{L-1})
    hash_tables: list of hash tables, each of size T_i
    hash_functions: list of hash functions h_0, ..., h_{L-1}
    Returns: concatenated feature vector
    """
    features = []
    for i, R in enumerate(grids):
        # Find the 8 surrounding vertices in grid i
        # Assuming grid spans [0, 1]^3 for simplicity, scale point if needed
        vertex_coords = floor(point * R) # (3,) int
        # Get fractional part for interpolation weights
        frac = point * R - vertex_coords # (3,) float
        
        # Generate 8 vertex indices and weights
        # This is a simplified representation of trilinear interpolation
        verts = []
        weights = []
        for k in range(8):
            # k represents the corner offset (0 or 1 in each dimension)
            offset = [(k >> d) & 1 for d in range(3)]
            v = vertex_coords + offset
            # Compute weight for this corner
            w = 1.0
            for d in range(3):
                if offset[d] == 0:
                    w *= (1.0 - frac[d])
                else:
                    w *= frac[d]
            verts.append(v)
            weights.append(w)
        
        # Retrieve embeddings via hash
        embeddings = []
        for v in verts:
            # Hash the vertex coordinates to get table index
            idx = hash_functions[i](v[0], v[1], v[2]) % len(hash_tables[i])
            emb = hash_tables[i][idx]
            embeddings.append(emb)
        
        # Trilinear interpolation
        f_i = sum(w * e for w, e in zip(weights, embeddings))
        features.append(f_i)
    
    return concat(features)

def forward(point, direction, grids, hash_tables, hash_functions, mlp):
    features = hash_encode(point, grids, hash_tables, hash_functions)
    sigma, color = mlp(features, direction)
    return sigma, color
```

## Subtopics

### fV-SRN
An interactive rendering method for compressed Implicit Neural Representations (INRs) that uses a volumetric latent grid and a lightweight MLP to achieve real-time frame rates by fusing network evaluation into shared memory [S5, unofficial]. It differs from Neurcomp by eliminating global memory access via shared memory fusion, achieving significantly higher speed [S5, unofficial]. It constructs a volumetric latent grid (typically $32^3$ resolution, 16 features per vertex) paired with a lightweight MLP (four layers, 32 channels) [S5, unofficial]. A custom GPU TensorCore kernel fuses the entire network evaluation into 48 kB of on-chip shared memory, eliminating global memory access during inference [S5, unofficial]. It predicts density, gradient, and curvature in a single forward pass to avoid finite-difference gradient computation [S5, unofficial]. Rendering speedup is 150–200× over Neurcomp, and shaded rendering speedup is 12× [S5, unofficial]. Performance is 5–10 FPS at $512^2$ resolution on volumes up to $1024^3$ [S5, unofficial]. Training requires the full volume in GPU memory, limiting use to offline workflows or requiring sufficient VRAM [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is drastically reduced compared to Vanilla NeRF. Convergence is reduced from several hours to a few minutes per scene [S2, unofficial]. Instant NGP can be easily trained in less than 10 minutes [S3]. On a CUDA RTX 3090, training the Lego model (800x800 resolution, 20,000 iterations) took 170 seconds [S4, unofficial].
*   **Memory Complexity:** Memory consumption is drastically reduced compared to unique feature vectors per vertex due to embedding sharing via hashing [S2, unofficial].
*   **Performance:** Enables real-time rendering [S4, unofficial].
*   **Strengths:** Significant improvements in training time and rendering speed; local constraint improves convergence stability; continuous representation avoids blocky artifacts [S2, unofficial].
*   **Limitations:** Hash collisions may affect quality if table sizes are too small; requires careful tuning of hash table sizes and resolutions [S2, unofficial].
*   **When to use:** When fast training and real-time inference are required for NeRF-like scenes [S4, unofficial].
*   **When not to use:** When extremely high precision is required and hash collisions are detrimental, or when memory is extremely constrained and hash table sizes cannot be sufficiently large [S2, unofficial].

## Background
Instant Neural Graphics Primitives (Instant NGP) was proposed in January 2022 [S1]. It was developed by NVIDIA researchers to address the slow training and inference speeds of Neural Radiance Fields (NeRF) [S4, unofficial]. The method introduced a learned parametric multi-resolution hash encoding and advanced ray marching techniques, significantly accelerating training and inference while enhancing scene reconstruction accuracy [S1]. It has become a baseline for many subsequent NeRF variants due to its efficiency and quality [S3].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
[S3] ScanNeRF: a Scalable Benchmark for Neural Radiance Fields, https://doi.org/10.1109/wacv56688.2023.00088, fetched 2026-10-26
[S4] Taichi NeRF (Part 1): Develop and Deploy Instant NGP without writing CUDA | Taichi Docs, https://docs.taichi-lang.org/blog/taichi-instant-ngp, fetched 2026-10-29 (unofficial)
[S5] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
