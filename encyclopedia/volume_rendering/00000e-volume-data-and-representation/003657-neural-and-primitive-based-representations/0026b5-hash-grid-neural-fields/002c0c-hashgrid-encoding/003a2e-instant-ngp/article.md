# Instant-NGP

Also called: Instant Neural Graphics Primitives

## Overview
Instant-NGP replaces the slow multi-layer perceptron (MLP) of standard Neural Radiance Fields (NeRF) with a multi-resolution hash-encoded feature grid to accelerate training and rendering while maintaining comparable visual quality [S1, S4]. It achieves a 1000–10000× training speedup by leveraging highly optimized GPU hash table lookups, allowing convergence in 2–5 minutes on a single GPU [S1]. Unlike frequency-based encodings that prioritize low-frequency shape capture, hash encoding facilitates both overall shape and local textures, preserving sharpness and fine details [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial coordinates $\mathbf{x}$ and view direction $\mathbf{d}$.
*   **Output:** Volume density $\sigma$ and RGB color $\mathbf{c}$.

**Data Structures:**
1.  **Multi-resolution Hash Grid (MHG):** A set of $L$ hash tables, one for each resolution level $l \in \{0, \dots, L-1\}$.
    *   Each table maps 3D grid coordinates to feature vectors.
    *   Entry precision: Half-precision (2 bytes) for storage, with a master full-precision copy for stable mixed-precision updates [S4].
    *   Feature dimensions per lookup: $F=2$ [S4].
    *   Hash table size threshold: Performance remains constant for $T \le 2^{19}$ entries; beyond this, performance drops due to L2 cache limits (e.g., 6 MB on RTX 3090) where $2 \cdot T \cdot F > 6 \cdot 2^{20}$ bytes [S4].
2.  **Encoder MLP:** A small network processing the concatenated features.
    *   Architecture: 4-layer MLP with dimensions $32 \times 64$, $64 \times 64$, $64 \times 64$, $64 \times 8$ [S5].
    *   Input: 32-dimensional vector (concatenated multi-resolution hash features).
    *   Output: 8D feature vector $M_i$ and opacity $\alpha_i$ [S5].
    *   Activation: GELU [S2].
3.  **Decoder MLP:** A small network predicting color.
    *   Architecture: 3-layer MLP with dimensions $17 \times 16$, $16 \times 16$, $16 \times 3$ [S5].
    *   Input: 17D vector (8D spatial features $M_i$ + 9D view direction encoded via Spherical Harmonics) [S5].
    *   Output: 3D RGB color $\mathbf{c}_i$ [S5].
4.  **Density Mapping Layer:**
    *   For NeRF synthetic datasets: A tiny network with one hidden layer and 64 neurons [S2].
    *   For real-world 360 datasets: A linear transform from the feature vector to density [S2].

**Algorithm Steps:**
1.  **Hash Encoding:** For each input position $\mathbf{x}$, compute the hash index for each resolution level $l$.
    *   The 3D space is subdivided at multiple resolutions.
    *   Grid coordinates are hashed to a 1D table. If the number of voxels exceeds $2^{19}$, positions are hashed to a 1D table [S2].
    *   Retrieve feature vectors from all $L$ levels.
2.  **Feature Concatenation:** Concatenate the feature vectors from all levels into a single input vector for the Encoder MLP.
3.  **Encoding:** Pass the concatenated features through the Encoder MLP to obtain the 8D feature vector $M_i$ and opacity $\alpha_i$ [S5].
    $$ E(\text{hash}(\mathbf{p}_i); \theta_E) \rightarrow M_i, \alpha_i $$
4.  **Directional Encoding:** Encode the view direction $\mathbf{d}$ using Spherical Harmonics (degree 4, resulting in 9 coefficients) [S5].
5.  **Decoding:** Concatenate $M_i$ and the SH-encoded direction to form a 17D vector. Pass it through the Decoder MLP to predict RGB color $\mathbf{c}_i$ [S5].
    $$ D(M_i, \text{SH}(\mathbf{d}_i); \theta_D) \rightarrow \mathbf{c}_i $$
6.  **Density Derivation:** Derive density $\sigma$ from the queried feature vector using the density mapping layer [S2].
7.  **Rendering:** Use standard volume rendering (ray marching) with the derived density and color [S1].
8.  **Training Optimization:**
    *   **Evaluation Order:** Evaluate hash tables level by level for a batch of inputs (all inputs for level 0, then level 1, etc.) to optimize GPU cache usage [S4].
    *   **Gradient Skipping:** Skip Adam steps for hash table entries with exactly 0 gradient, saving ~10% performance when gradients are sparse [S4].
    *   **Initialization:** MLP weights initialized via Glorot and Bengio [S4]; Hash table entries initialized from $U(-10^{-4}, 10^{-4})$ [S4].
    *   **Optimizer:** Adam with $\beta_1=0.9, \beta_2=0.99, \epsilon=10^{-15}$ [S4]. Weak L2 regularization ($10^{-6}$) applied to MLP weights, not hash entries [S4].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
L = 16  # Number of hash levels (example)
F = 2   # Feature dimensions per lookup
T_max = 2**19  # Cache threshold

# Data Structures
hash_tables = [EmptyHashTable(size=T_max) for _ in range(L)]
encoder_mlp = MLP(layers=[32, 64, 64, 64, 8], activation=GELU)
decoder_mlp = MLP(layers=[17, 16, 16, 3], activation=ReLU)
density_layer = TinyDensityNetwork()  # Or LinearTransform depending on dataset

# Initialization
for table in hash_tables:
    table.init_uniform(-1e-4, 1e-4)
encoder_mlp.init_glorot()
decoder_mlp.init_glorot()

def hash_encode(positions, levels):
    """
    Encodes positions using multi-resolution hash grid.
    Returns concatenated features for all levels.
    """
    all_features = []
    # Evaluate level by level for cache efficiency [S4]
    for l in range(levels):
        # Compute grid coordinates for level l
        grid_coords = compute_grid_coords(positions, resolution=l)
        # Hash coordinates to table indices
        indices = hash_function(grid_coords)
        # Lookup features from the hash table
        features = hash_tables[l].lookup(indices)
        all_features.append(features)
    # Concatenate features from all levels
    # Input dimension becomes L * F
    return concatenate(all_features, axis=-1)

def encode_and_decode(positions, directions):
    """
    Full forward pass of Instant-NGP.
    """
    # 1. Hash Encoding
    # Input to encoder is 32D (L=16, F=2) [S5]
    encoded_features = hash_encode(positions, L)
    
    # 2. Encoder MLP
    # Output: 8D feature vector + opacity
    feature_vector, opacity = encoder_mlp(encoded_features)
    
    # 3. Directional Encoding
    # Spherical Harmonics degree 4 -> 9 coefficients [S5]
    sh_coeffs = spherical_harmonic_encode(directions, degree=4)
    
    # 4. Decoder Input
    # Concatenate 8D feature + 9D SH = 17D [S5]
    decoder_input = concatenate([feature_vector, sh_coeffs], axis=-1)
    
    # 5. Decoder MLP
    # Output: 3D RGB color
    color = decoder_mlp(decoder_input)
    
    # 6. Density Mapping
    # Derived from feature vector (not explicitly passed to decoder)
    density = density_layer(feature_vector)
    
    return color, density, opacity

def train_step(positions, directions, target_colors, target_opacities):
    """
    Training step with gradient skipping optimization.
    """
    # Forward pass
    colors, densities, opacities = encode_and_decode(positions, directions)
    
    # Compute loss
    loss = compute_loss(colors, target_colors)
    
    # Backward pass
    gradients = compute_gradients(loss)
    
    # Gradient Skipping: Skip updates for hash entries with 0 gradient [S4]
    for l in range(L):
        zero_grad_indices = find_zero_gradients(gradients, hash_tables[l])
        hash_tables[l].skip_update(zero_grad_indices)
        
    # Update parameters
    optimizer.step(encoder_mlp, decoder_mlp, hash_tables, density_layer)
```

## Subtopics

### F-Hash
An extension of multi-resolution hash encoding to four dimensions using Tesseract grids and collision-free minimal perfect hashing [S8, unofficial]. It achieves 10–100× faster convergence than standard hash encoding (e.g., 47.1 dB in 7.4 minutes vs. 23.7 dB in 45.7 minutes on combustion data) with compression ratios of 9.5–55.2× [S8, unofficial].

## Operating characteristics
*   **Training Speed:** 1000–10000× faster than NeRF, converging in 2–5 minutes on a single GPU [S1]. Converges to 95% of final PSNR in 2.4 ± 0.4 minutes on Blender scenes [S1].
*   **Rendering Speed:** 2.4 ± 0.4 FPS on an A100 GPU at 1080p [S1]. This is 12× faster than NeRF (0.2 FPS) but 28× slower than 3D Gaussian Splatting (68.4 FPS) [S1].
*   **Quality:** On NeRF Synthetic (Blender) dataset, PSNR = 32.6 ± 0.4 dB [S1]. On Mip-NeRF 360 unbounded dataset, PSNR = 26.4 ± 0.6 dB [S1]. In medical imaging, it outperforms vanilla NeRF and FDK in SSIM, LPIPS, and GMSD metrics [S3].
*   **Memory:** Hash table size $T$ affects performance. Performance is stable for $T \le 2^{19}$ but drops significantly beyond due to L2 cache limits [S4].
*   **Strengths:** Extremely fast training, good preservation of fine details and sharpness compared to frequency-based methods [S3].
*   **Limitations:** Moderate surface accuracy due to faster density representation rather than explicit surface geometry [S1]. Performance sensitive to hash table size relative to GPU cache [S4].

## Background
Instant-NGP was introduced by Müller et al. in 2022 [S4]. It was developed to address the slow training and rendering speeds of NeRF by replacing the MLP with a multi-resolution hash-encoded feature grid [S1]. The method was implemented in CUDA and integrated with the tiny-cuda-nn framework to optimize inference and backpropagation [S4]. It has been widely adopted for rapid prototyping and applications requiring immediate visual feedback [S1].

## Sources
[S1] Neural Rendering Techniques, https://doi.org/10.5281/zenodo.19614729, fetched 2026-10-26
[S2] Volume Feature Rendering for Fast Neural Radiance Field Reconstruction, https://doi.org/10.52202/075280-2853, fetched 2026-10-26
[S3] The Adaption of Recent New Concepts in Neural Radiance Fields and Their Role for High-Fidelity Volume Reconstruction in Medical Images., https://doi.org/10.3390/s24185923, fetched 2026-10-26
[S4] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
[S5] UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene, https://doi.org/10.48550/arXiv.2310.13263, fetched 2026-10-29
[S6] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S7] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S8] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
