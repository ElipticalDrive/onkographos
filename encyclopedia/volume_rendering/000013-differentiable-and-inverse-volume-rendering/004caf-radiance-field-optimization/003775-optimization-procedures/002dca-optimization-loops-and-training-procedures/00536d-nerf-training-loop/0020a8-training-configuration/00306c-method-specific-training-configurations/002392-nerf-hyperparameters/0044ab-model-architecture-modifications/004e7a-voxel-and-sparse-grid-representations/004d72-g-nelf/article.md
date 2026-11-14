# G-NeLF

Also called: Grid-based Neural Light Field, Memory- and Data-Efficient Hybrid Neural Light Field

## Overview
G-NeLF is a grid-based Neural Light Field (NeLF) method for novel view synthesis that predicts ray color directly without point-wise volume density modeling [S1, unofficial]. It represents a ray as a sequence of spatial-aware features extracted from a hash-based multi-resolution tri-plane, which are decoded by an LSTM network to predict color [S1, unofficial]. Unlike standard NeRF methods that model density and color per point, G-NeLF bypasses the volume rendering integral, relying on the sequential nature of ray propagation to infer appearance [S1, unofficial]. It is distinguished from its sibling [Voxel-based Radiance Fields](11470-voxel-based-radiance-fields.md) by its use of a recurrent decoder (LSTM) on a feature sequence rather than a static MLP on density/color pairs, and from [Neural Multi-View Image Reconstruction with Hash Encoding](13013-neural-multi-view-image-reconstruction-with-hash-encoding.md) by its specific tri-plane projection and ray-wise sequential processing [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), bounding box of the scene, and trained model parameters (hash tables, LSTM weights).
*   **Output:** Predicted color $\hat{C}(r) = (r, g, b)$ for the ray $r$.

**Data Structures:**
1.  **Hash Table:** A sparse mapping from integer coordinate pairs to feature vectors.
    *   **Layout:** Array of size $T$ (typically $2^{14}$ or $2^{16}$).
    *   **Content:** Each entry stores a feature vector of dimension $D_{feat}$ (typically 2).
    *   **Hash Function:** For a 2D projected coordinate $(x, y)$ at resolution level $i$ with grid size $s_i$, the index is calculated as:
        $$ \text{Index} = (x \oplus y) \mod T $$
        where $\oplus$ is the bitwise XOR operation [S1, unofficial]. Note: The source specifies constants $a=1, b=2,654,435,761$ for the hash function in Instant-NGP context, but for G-NeLF's tri-plane, it simplifies to the XOR of the integer-divided coordinates [S1, unofficial].
2.  **Tri-Plane:** Three orthogonal feature planes $\pi_{xy}, \pi_{xz}, \pi_{yz}$.
    *   **Layout:** Each plane contains $N$ resolution levels.
    *   **Resolutions:**
        *   Small/Medium: $N=8$ levels, grid sizes $s_i$ ranging from 16 to 1024.
        *   Large: $N=16$ levels, grid sizes $s_i$ ranging from 16 to 2048.
3.  **Point Feature Vector ($F$):**
    *   **Layout:** Concatenation of features from all 3 planes and all $N$ resolutions.
    *   **Size:** $3 \times N \times 2$ dimensions.

**Numbered Steps:**

1.  **Ray Emission:** For each pixel in the target image, emit a ray $r$ defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ based on the camera model.
2.  **Bounding Box Intersection:** Determine the intersection of the ray with the scene's bounding box to define the sampling interval $[t_{min}, t_{max}]$.
3.  **Point Sampling:** Sequentially sample $K$ points $\{\mathbf{x}_1, \dots, \mathbf{x}_K\}$ along the ray within the bounding box.
    *   **Parameters:** $K=256$ for 360° scenes; $K=128$ for forward-facing scenes (using NDC coordinates) [S1, unofficial].
4.  **Feature Extraction (for each point $\mathbf{x}_k$):**
    a.  **Projection:** Project $\mathbf{x}_k$ onto the three orthogonal planes $\pi_{xy}, \pi_{xz}, \pi_{yz}$ to obtain 2D coordinates $\mathbf{x}_{\pi}$.
    b.  **Hash Retrieval:** For each plane $\pi$ and each resolution level $i \in \{1, \dots, N\}$:
        i.  Calculate integer grid indices: $u = \lfloor x_{\pi,0} / s_i \rfloor$, $v = \lfloor x_{\pi,1} / s_i \rfloor$.
        ii. Compute hash indices for the four corners:
            $$ h_{tl} = \text{hash}(\lfloor u \rfloor, \lfloor v \rfloor) $$
            $$ h_{tr} = \text{hash}(\lfloor u \rfloor, \lfloor v+1 \rfloor) $$
            $$ h_{bl} = \text{hash}(\lfloor u+1 \rfloor, \lfloor v \rfloor) $$
            $$ h_{br} = \text{hash}(\lfloor u+1 \rfloor, \lfloor v+1 \rfloor) $$
        iii. Retrieve feature vectors $f_{tl}, f_{tr}, f_{bl}, f_{br}$ from the hash table at these indices.
        iv. Perform trilinear interpolation (bilinear in 2D) to get the feature vector $f_{\pi,i}$ for this resolution.
    c.  **Concatenation:** Concatenate the interpolated features from all $N$ resolutions for plane $\pi$ to form $F_{\pi}$.
    d.  **Final Point Feature:** Concatenate $F_{\pi_{xy}}, F_{\pi_{xz}}, F_{\pi_{yz}}$ to form the point feature vector $F_k$.
5.  **Direction Encoding:** Encode the normalized ray direction $\mathbf{d}$ using Spherical Harmonics (SH) to produce vector $\mathbf{d}_{enc}$ [S1, unofficial].
6.  **LSTM Decoding:** Process the sequence $\{F_1, \dots, F_K\}$ sequentially using a 2-layer LSTM network.
    *   **Input to LSTM at step $i$:** Concatenation of the point feature $F_i$ and the encoded direction $\mathbf{d}_{enc}$.
    *   **State Update:**
        $$ h_i, c_i = \text{LSTM}(\text{Concat}(F_i, \mathbf{d}_{enc}), h_{i-1}, c_{i-1}) $$
        where $h_i$ is the hidden state and $c_i$ is the cell state. Initialize $h_0, c_0$ to zero vectors [S1, unofficial].
7.  **Color Prediction:** Pass the final hidden state $h_K$ through a 2-layer Multi-Layer Perceptron (MLP) to predict the ray color:
    $$ \hat{C}(r) = \text{MLP}(h_K) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def hash_index(x, y, table_size):
    """
    Computes the hash index for a 2D coordinate.
    Uses XOR operation as specified in source [S1, unofficial].
    """
    return (x ^ y) % table_size

def get_point_features(x, tri_planes, resolutions, hash_tables):
    """
    Extracts the feature vector F for a 3D point x from the tri-plane.
    
    Args:
        x: 3D coordinate (x, y, z)
        tri_planes: List of 3 planes ['xy', 'xz', 'yz']
        resolutions: List of grid sizes [s_1, ..., s_N]
        hash_tables: List of 3 lists, each containing N hash tables (arrays)
        
    Returns:
        F: Concatenated feature vector
    """
    F_all = []
    
    for plane in tri_planes:
        F_plane = []
        
        # Project 3D point to 2D based on plane
        if plane == 'xy':
            x_2d, y_2d = x[0], x[1]
        elif plane == 'xz':
            x_2d, y_2d = x[0], x[2]
        elif plane == 'yz':
            x_2d, y_2d = x[1], x[2]
            
        for i, s in enumerate(resolutions):
            # Integer division for grid indices
            u = math.floor(x_2d / s)
            v = math.floor(y_2d / s)
            
            # Get hash tables for this plane and resolution
            h_table = hash_tables[tri_planes.index(plane)][i]
            table_size = len(h_table)
            
            # Calculate corner indices
            idx_tl = hash_index(u, v, table_size)
            idx_tr = hash_index(u, v + 1, table_size)
            idx_bl = hash_index(u + 1, v, table_size)
            idx_br = hash_index(u + 1, v + 1, table_size)
            
            # Retrieve features
            f_tl = h_table[idx_tl]
            f_tr = h_table[idx_tr]
            f_bl = h_table[idx_bl]
            f_br = h_table[idx_br]
            
            # Bilinear interpolation
            # Fractional parts
            fx = x_2d / s - u
            fy = y_2d / s - v
            
            # Interpolate top and bottom
            f_top = f_tl * (1 - fx) * (1 - fy) + f_tr * fx * (1 - fy)
            f_bot = f_bl * (1 - fx) * fy + f_br * fx * fy
            
            # Final interpolation
            f_interp = f_top * (1 - fy) + f_bot * fy
            
            F_plane.append(f_interp)
            
        # Concatenate features from all resolutions for this plane
        F_plane_concat = concatenate_along_axis(F_plane, axis=0)
        F_all.append(F_plane_concat)
        
    # Concatenate features from all 3 planes
    F = concatenate_along_axis(F_all, axis=0)
    return F

def decode_ray_color(feature_sequence, direction, lstm_model, mlp_model):
    """
    Decodes the ray color using LSTM and MLP.
    
    Args:
        feature_sequence: List of feature vectors [F_1, ..., F_K]
        direction: Normalized 3D direction vector d
        lstm_model: 2-layer LSTM network
        mlp_model: 2-layer MLP network
        
    Returns:
        color: Predicted RGB color
    """
    # Encode direction using Spherical Harmonics
    d_enc = encode_spherical_harmonics(direction)
    
    # Initialize LSTM states
    h_prev = zero_vector(lstm_model.hidden_size)
    c_prev = zero_vector(lstm_model.hidden_size)
    
    # Process sequence
    for F_k in feature_sequence:
        # Concatenate point feature and direction
        input_vec = concatenate([F_k, d_enc])
        
        # LSTM step
        h_curr, c_curr = lstm_model.step(input_vec, h_prev, c_prev)
        
        # Update states
        h_prev = h_curr
        c_prev = c_curr
        
    # Final prediction
    color = mlp_model(h_prev)
    return color

def render_ray(ray_origin, ray_direction, model_params):
    """
    Main rendering function for a single ray.
    """
    # 1. Sample points
    K = model_params.K # 256 for 360, 128 for forward-facing
    t_min, t_max = intersect_bounding_box(ray_origin, ray_direction, model_params.bbox)
    t_vals = linspace(t_min, t_max, K)
    points = [ray_origin + t * ray_direction for t in t_vals]
    
    # 2. Extract features
    feature_sequence = [get_point_features(p, model_params.tri_planes, 
                                           model_params.resolutions, 
                                           model_params.hash_tables) for p in points]
    
    # 3. Decode
    color = decode_ray_color(feature_sequence, ray_direction, 
                             model_params.lstm, model_params.mlp)
    return color
```

**Parameters:**
*   **Hash Table Size ($T$):** $2^{14}$ for small/medium models; $2^{16}$ for large models [S1, unofficial].
*   **Resolutions ($N$):** 8 levels (16 to 1024) for small/medium; 16 levels (16 to 2048) for large [S1, unofficial].
*   **Feature Dimension per Level:** 2 [S1, unofficial].
*   **LSTM Hidden Size:** 32 (small), 128 (medium), 128 (large, 3 layers) [S1, unofficial].
*   **Sequence Length ($K$):** 256 (360° scenes), 128 (forward-facing scenes) [S1, unofficial].
*   **Model Size:** 0.95 MB (small), 1.41 MB (medium/large) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** At the highest resolution, approximately 100,000 3D positions map to a single hash entry. This is not a critical error because points far from the visible surface have low influence on the final color in standard volume rendering, and in G-NeLF, the dominant appearance features are learned by the positions with higher rendering weights. The features at higher resolutions adaptively concentrate on capturing appearance [S1, unofficial].
*   **Decoder Choice:** Using an MLP directly on concatenated features ignores the temporal order of ray propagation, leading to inefficiency. Using a Transformer is also unsuitable because standard attention mechanisms are permutation-invariant and anti-order, losing the sequential information crucial for ray propagation simulation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Determined by the feature extraction ($O(K \cdot N)$ hash lookups and interpolations) and the LSTM decoding ($O(K \cdot H)$ where $H$ is hidden size).
*   **Memory Complexity:** Dominated by the hash tables. Total parameters are extremely low: 0.95 MB to 1.41 MB, compared to 64.1 MB for Instant-NGP and 71.8 MB for TensoRF [S1, unofficial].
*   **Performance:** On the Realistic Synthetic 360° dataset at 400x400 resolution, G-NeLF surpasses R2L by 0.04 dB PSNR with only 100 training views and 0.95 MB model size. It surpasses VQRF by 0.79 dB PSNR at a comparable model size (1.41 MB vs 1.43 MB) [S1, unofficial]. It achieves comparable rendering quality to standard NeRF methods like Instant-NGP and TensoRF while using approximately one-tenth of the parameters [S1, unofficial].
*   **Strengths:** Extremely compact model size; requires only original training views (no pseudo-view synthesis needed); leverages sequential information for better inference than MLPs.
*   **Limitations:** Does not model explicit density, which may limit its ability to handle complex occlusion or volumetric effects compared to NeRF; performance is tied to the quality of the feature sequence and LSTM capacity.

## Background
G-NeLF was introduced in the paper "G-NeLF: Memory- and Data-Efficient Hybrid Neural Light Field for Novel View Synthesis" [S1, unofficial]. It addresses the limitations of previous NeLF methods, which often required training a NeRF first and synthesizing thousands of views to train the NeLF, or suffered from lower rendering quality compared to NeRF. G-NeLF utilizes a spatial-aware feature sequence derived from a hash-based multi-resolution tri-plane and an LSTM decoder to overcome these difficulties, enabling high-quality novel view synthesis with limited data (100 views or fewer) and minimal storage overhead [S1, unofficial].

## Sources
[S1] G-NeLF: Memory- and Data-Efficient Hybrid Neural Light Field for Novel View Synthesis, https://arxiv.org/abs/2409.05617v1, fetched 2026-10-27 (unofficial)
