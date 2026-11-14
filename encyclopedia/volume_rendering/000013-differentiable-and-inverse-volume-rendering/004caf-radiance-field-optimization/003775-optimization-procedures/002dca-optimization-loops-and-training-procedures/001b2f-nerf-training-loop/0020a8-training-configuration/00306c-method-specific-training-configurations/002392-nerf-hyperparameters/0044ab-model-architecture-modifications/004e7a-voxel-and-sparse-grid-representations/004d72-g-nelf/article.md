# G-NeLF

Also called: Grid-based Neural Light Field, Memory- and Data-Efficient Hybrid Neural Light Field

## Overview
G-NeLF is a grid-based Neural Light Field (NeLF) method for novel view synthesis that predicts ray color directly without point-wise volume density modeling [S1, unofficial]. It represents a ray as a sequence of spatial-aware features extracted from a hash-based multi-resolution tri-plane, which are decoded by an LSTM network to predict color [S1, unofficial]. Unlike standard NeRF methods that model density and color per point, G-NeLF bypasses explicit volumetric integration, allowing it to achieve comparable rendering quality to state-of-the-art grid-based NeRFs (e.g., Instant-NGP, TensoRF) with approximately one-tenth the parameters [S1, unofficial]. It is specifically designed to be trained on 100 or fewer original views without requiring pseudo-view synthesis [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), a set of training images.
*   **Outputs:** Predicted ray color $(r, g, b)$.

**Data Structures:**
1.  **Hash-Based Multi-Resolution Tri-Plane:**
    *   Consists of three orthogonal planes: $\pi_{xy}, \pi_{xz}, \pi_{yz}$ [S1, unofficial].
    *   Each plane has $N$ resolutions [S1, unofficial].
    *   For each resolution $i$ and plane $\pi$, there is a hash table $hash_{\pi,i}$ [S1, unofficial].
    *   **Hash Table Layout:** Maps integer coordinate pairs to feature vectors. The hash index is calculated as:
        $$ \text{Index} = (x^a \oplus y^b) \mod T $$
        where $x, y$ are the projected 2D integer coordinates, $\oplus$ is bitwise XOR, $T$ is the hash table size, and $a, b$ are constants [S1, unofficial]. The source specifies $a=1$ and $b \in \{2, 654,435, 761\}$ [S1, unofficial].
    *   **Feature Vector:** Each hash table entry stores a feature vector of dimension 2 [S1, unofficial].
    *   **Point Feature Vector ($F$):** For a 3D point $\mathbf{x}$, the feature vector is the concatenation of features retrieved from all resolutions of all three planes.
        $$ F = \text{Concat}([F_{\pi_{xy}}, F_{\pi_{xz}}, F_{\pi_{yz}}]) $$
        where $F_{\pi}$ is the concatenation of features from all $N$ resolutions of plane $\pi$ [S1, unofficial].
        Total dimension of $F$: $3 \times N \times 2$ [S1, unofficial].

2.  **Ray Feature Sequence:**
    *   A ray is represented by a sequence of $K$ feature vectors $\{F_1, \dots, F_K\}$, where each $F_i$ corresponds to the $i$-th sampled point along the ray [S1, unofficial].

3.  **LSTM Decoder:**
    *   A 2-layer (or 3-layer for large models) LSTM network [S1, unofficial].
    *   Hidden state $H_i$ and Cell state $C_i$ [S1, unofficial].

**Parameters:**
*   **Hash Table Size ($T$):** $2^{14}$ for small/medium models; $2^{16}$ for large models [S1, unofficial].
*   **Grid Resolutions:**
    *   Small/Medium: $N=8$ levels, ranging from 16 to 1024 [S1, unofficial].
    *   Large: $N=16$ levels, ranging from 16 to 2048 [S1, unofficial].
*   **Feature Dimension:** 2 per level [S1, unofficial].
*   **LSTM Hidden Size:** 32 (small), 128 (medium), 128 (large) [S1, unofficial].
*   **Sequence Length ($K$):** 256 for 360° scenes; 128 for forward-facing scenes (using NDC coordinates) [S1, unofficial].
*   **Model Size:** 0.95 MB (474,432 parameters) for the small configuration [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Emission:** For each pixel in the target image, emit a ray based on the camera model [S1, unofficial].
2.  **Sampling:** Sequentially sample $K$ points $\{\mathbf{x}_1, \dots, \mathbf{x}_K\}$ along the ray within the scene's bounding box [S1, unofficial].
3.  **Feature Extraction (for each point $\mathbf{x}_i$):**
    *   Project $\mathbf{x}_i$ onto the three orthogonal planes to get 2D coordinates $\mathbf{x}_{\pi}$ for $\pi \in \{\pi_{xy}, \pi_{xz}, \pi_{yz}\}$ [S1, unofficial].
    *   For each plane $\pi$ and resolution $i \in \{1, \dots, N\}$:
        *   Calculate grid coordinates using integer division by grid size $s_i$: $u = \lfloor x_{\pi,0} / s_i \rfloor$, $v = \lfloor x_{\pi,1} / s_i \rfloor$ [S1, unofficial].
        *   Retrieve features from the four corner vertices of the grid cell using the hash table:
            $$ t_l = hash_{\pi,i}([u, v]) $$
            $$ t_r = hash_{\pi,i}([u, v+1]) $$
            $$ b_l = hash_{\pi,i}([u+1, v]) $$
            $$ b_r = hash_{\pi,i}([u+1, v+1]) $$
        *   Compute the weighted sum of these four features based on the distance between $\mathbf{x}_i$ and the vertices (trilinear interpolation) [S1, unofficial].
        *   Append the result to the plane's feature list [S1, unofficial].
    *   Concatenate features from all resolutions for each plane [S1, unofficial].
    *   Concatenate the three planes to form the point's feature vector $F_i$ [S1, unofficial].
4.  **Direction Encoding:** Encode the ray's 3D normalized direction vector $\mathbf{d}$ using Spherical Harmonics [S1, unofficial].
5.  **LSTM Decoding:**
    *   Initialize hidden state $H_0$ and cell state $C_0$ to zero [G].
    *   For $i = 1$ to $K$:
        $$ H_i, C_i = \text{LSTM}(\text{Concat}(F_i, \mathbf{d}), H_{i-1}, C_{i-1}) $$
        where $\text{Concat}(F_i, \mathbf{d})$ combines the point feature and encoded direction [S1, unofficial].
6.  **Color Prediction:**
    $$ (r, g, b) = \text{MLP}(H_K) $$
    where $\text{MLP}$ is a 2-layer network and $H_K$ is the final hidden state [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def get_hash_index(x, y, a, b, T):
    """Calculate hash index for a 2D coordinate."""
    # a=1, b varies (e.g., 2, 654435, 761)
    return (x**a ^ y**b) % T

def retrieve_feature(hash_table, x_coord, y_coord, grid_size):
    """
    Retrieve and interpolate feature for a 2D point from a hash table.
    hash_table: dict mapping index -> feature_vector (dim 2)
    x_coord, y_coord: float 2D coordinates
    grid_size: int size of the grid at this resolution
    """
    # Integer grid coordinates
    u = int(x_coord // grid_size)
    v = int(y_coord // grid_size)
    
    # Fractional parts for interpolation
    fx = (x_coord // grid_size) - u
    fy = (y_coord // grid_size) - v
    
    # Corner indices
    idx_tl = get_hash_index(u, v, 1, 2, len(hash_table)) # Using b=2 as example
    idx_tr = get_hash_index(u, v + 1, 1, 2, len(hash_table))
    idx_bl = get_hash_index(u + 1, v, 1, 2, len(hash_table))
    idx_br = get_hash_index(u + 1, v + 1, 1, 2, len(hash_table))
    
    # Retrieve features (handle hash collisions by taking the value at the index)
    f_tl = hash_table.get(idx_tl, np.zeros(2))
    f_tr = hash_table.get(idx_tr, np.zeros(2))
    f_bl = hash_table.get(idx_bl, np.zeros(2))
    f_br = hash_table.get(idx_br, np.zeros(2))
    
    # Trilinear interpolation (bilinear in 2D)
    # Top edge
    f_top = (1 - fx) * f_tl + fx * f_tr
    # Bottom edge
    f_bot = (1 - fx) * f_bl + fx * f_br
    # Final interpolation
    f_interp = (1 - fy) * f_top + fy * f_bot
    
    return f_interp

def get_point_feature(x_3d, tri_plane_hash_tables, resolutions):
    """
    Get feature vector F for a 3D point x_3d.
    tri_plane_hash_tables: dict of dicts, e.g., {'xy': {res: hash_table, ...}, ...}
    resolutions: list of grid sizes for each resolution level
    """
    F_planes = []
    
    # Planes: xy, xz, yz
    # Note: Projection logic depends on coordinate system. 
    # Assuming standard projection: xy->(x,y), xz->(x,z), yz->(y,z)
    plane_coords = {
        'xy': (x_3d[0], x_3d[1]),
        'xz': (x_3d[0], x_3d[2]),
        'yz': (x_3d[1], x_3d[2])
    }
    
    for plane_name in ['xy', 'xz', 'yz']:
        F_resolutions = []
        coords_2d = plane_coords[plane_name]
        
        for i, grid_size in enumerate(resolutions):
            # Retrieve from hash table for this plane and resolution
            hash_table = tri_plane_hash_tables[plane_name][i]
            f_res = retrieve_feature(hash_table, coords_2d[0], coords_2d[1], grid_size)
            F_resolutions.append(f_res)
            
        # Concatenate features from all resolutions for this plane
        F_plane = np.concatenate(F_resolutions)
        F_planes.append(F_plane)
        
    # Concatenate features from all three planes
    F = np.concatenate(F_planes)
    return F

def spherical_harmonics_encode(direction, degree=3):
    """
    Encode direction vector using Spherical Harmonics.
    Simplified placeholder for standard SH encoding.
    """
    # Standard SH encoding for degree 3 results in 16 coefficients
    # This is a standard math operation, not a library call
    theta = np.arccos(np.clip(direction[2], -1.0, 1.0))
    phi = np.arctan2(direction[1], direction[0])
    
    # Coefficients for degree 0 to 3
    sh = np.zeros(16)
    sh[0] = 0.282095
    sh[1] = 0.488603 * direction[1]
    sh[2] = 0.488603 * direction[2]
    sh[3] = 0.488603 * direction[0]
    sh[4] = 1.092548 * direction[0] * direction[1]
    sh[5] = -1.092548 * direction[1] * direction[2]
    sh[6] = 0.315392 * (3 * direction[2]**2 - 1)
    sh[7] = -1.092548 * direction[0] * direction[2]
    sh[8] = 0.546274 * (direction[0]**2 - direction[1]**2)
    sh[9] = 0.590592 * direction[1] * (3 * direction[0]**2 - direction[1]**2)
    sh[10] = -2.890611 * direction[0] * direction[1] * direction[2]
    sh[11] = 0.457046 * direction[2] * (5 * direction[2]**2 - 3)
    sh[12] = -2.890611 * direction[1] * (direction[0]**2 - direction[1]**2)
    sh[13] = 1.445305 * direction[0] * (5 * direction[2]**2 - 1)
    sh[14] = -1.445305 * direction[1] * (5 * direction[2]**2 - 1)
    sh[15] = 0.373176 * direction[2] * (7 * direction[2]**2 - 3)
    
    return sh

def lstm_step(input_vec, h_prev, c_prev, weights):
    """
    Single LSTM step.
    input_vec: concatenated feature and direction
    h_prev, c_prev: previous hidden and cell states
    weights: LSTM parameters (W_i, W_f, W_c, W_o, b_i, b_f, b_c, b_o)
    """
    # Standard LSTM equations
    i = sigmoid(input_vec @ weights['W_i'] + h_prev @ weights['U_i'] + weights['b_i'])
    f = sigmoid(input_vec @ weights['W_f'] + h_prev @ weights['U_f'] + weights['b_f'])
    c_tilde = tanh(input_vec @ weights['W_c'] + h_prev @ weights['U_c'] + weights['b_c'])
    o = sigmoid(input_vec @ weights['W_o'] + h_prev @ weights['U_o'] + weights['b_o'])
    
    c_curr = f * c_prev + i * c_tilde
    h_curr = o * tanh(c_curr)
    
    return h_curr, c_curr

def g_nelf_render(ray_origin, ray_direction, tri_plane_hash_tables, resolutions, lstm_weights, mlp_weights):
    """
    Render a single ray using G-NeLF.
    """
    K = 256 # For 360 scenes
    # Sample points along the ray
    # Assuming a bounding box or fixed range. 
    # In practice, this requires scene bounds. 
    # Here we assume uniform sampling over a normalized range [0, 1] for simplicity
    # or NDC coordinates for forward-facing.
    t = np.linspace(0, 1, K) 
    points = ray_origin + ray_direction * t[:, np.newaxis]
    
    # Encode direction
    d_encoded = spherical_harmonics_encode(ray_direction / np.linalg.norm(ray_direction))
    
    # Initialize LSTM states
    h = np.zeros(lstm_weights['U_i'].shape[0])
    c = np.zeros(lstm_weights['U_i'].shape[0])
    
    # Process feature sequence
    for i in range(K):
        # Get feature for point i
        F_i = get_point_feature(points[i], tri_plane_hash_tables, resolutions)
        
        # Concatenate feature and direction
        input_vec = np.concatenate([F_i, d_encoded])
        
        # LSTM step
        h, c = lstm_step(input_vec, h, c, lstm_weights)
        
    # Final MLP prediction
    color = h @ mlp_weights['W_out'] + mlp_weights['b_out']
    return color
```

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** At the highest resolution, the hash table maps approximately 100,000 3D positions to a single entry [S1, unofficial]. This is not a bug but a feature: points far from the visible surface have low volume rendering weights and thus do not significantly influence the color. The training process allows the hash entry to adaptively capture the appearance of the dominant visible points [S1, unofficial].
*   **Sequential Order:** The method relies on the temporal order of ray propagation (near to far). Using MLPs or Transformers (which are permutation-invariant or anti-order) degrades performance because they ignore this sequential information [S1, unofficial].
*   **Coordinate Systems:** For forward-facing scenes, coordinates are converted to NDC (Normalized Device Coordinates) and the sequence length is reduced to 128 [S1, unofficial].

## Operating characteristics
*   **Model Size:** Extremely compact. 0.95 MB (474,432 parameters) for the small configuration [S1, unofficial]. This is approximately one-tenth the size of Instant-NGP (64.1 MB) and TensoRF (71.8 MB) [S1, unofficial].
*   **Training Data:** Requires only 100 or fewer original training views. Does not require synthesizing pseudo-views, saving significant storage and compute compared to methods like R2L [S1, unofficial].
*   **Performance:**
    *   Surpasses R2L by 0.04 dB PSNR on Realistic Synthetic 360° dataset with the same small model size [S1, unofficial].
    *   Surpasses VQRF by 0.79 dB PSNR at comparable model size (1.41 MB vs 1.43 MB) [S1, unofficial].
    *   Achieves comparable rendering quality to standard grid-based NeRF methods (Instant-NGP, TensoRF) [S1, unofficial].
*   **Hardware:** Experiments conducted on a single RTX 3090 GPU using float16 precision [S1, unofficial].
*   **Strengths:** High memory efficiency, fast training due to small model size and lack of pseudo-view generation, good performance on limited data.
*   **Limitations:** Does not model volume density explicitly, which may limit its applicability in scenarios requiring explicit volumetric properties (e.g., light transport simulation, occlusion handling beyond color prediction). Performance depends heavily on the LSTM's ability to capture long-range dependencies in the feature sequence.

## Background
G-NeLF was introduced in the paper "G-NeLF: Memory- and Data-Efficient Hybrid Neural Light Field for Novel View Synthesis" [S1, unofficial]. It addresses the limitations of previous Neural Light Field (NeLF) methods, which often required training a NeRF first and synthesizing thousands of pseudo-views to train the NeLF, leading to high storage and computational costs [S1, unofficial]. It also improves upon the rendering quality of earlier NeLF approaches, which were generally inferior to NeRF methods [S1, unofficial]. The method builds on the concept of hash-based multi-resolution grids (popularized by Instant-NGP) but adapts them for a ray-wise feature sequence representation decoded by an LSTM, leveraging the sequential nature of ray propagation [S1, unofficial].

## Sources
[S1] G-NeLF: Memory- and Data-Efficient Hybrid Neural Light Field for Novel View Synthesis, https://arxiv.org/abs/2409.05617v1, fetched 2026-10-27 (unofficial)
