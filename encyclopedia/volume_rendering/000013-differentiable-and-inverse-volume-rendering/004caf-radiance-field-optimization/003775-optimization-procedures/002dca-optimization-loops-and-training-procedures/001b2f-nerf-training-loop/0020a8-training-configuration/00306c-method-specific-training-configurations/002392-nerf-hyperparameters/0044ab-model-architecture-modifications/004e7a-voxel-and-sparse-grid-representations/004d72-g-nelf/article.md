# G-NeLF

Also called: Grid-based Neural Light Field, Memory- and Data-Efficient Hybrid Neural Light Field

## Overview
G-NeLF is a grid-based Neural Light Field (NeLF) method for novel view synthesis that predicts ray color directly without point-wise volume density modeling [S1, unofficial]. It represents a ray as a sequence of spatial-aware features extracted from a hash-based multi-resolution tri-plane, which are decoded by an LSTM network to predict color [S1, unofficial]. Unlike standard NeRF, which models density and color at sampled points, G-NeLF bypasses volume rendering integration, relying on the sequential processing of features along the ray path to capture view-dependent appearance [S1, unofficial]. It is distinguished from sibling NeLF methods like R2L by its use of a compact hash-based tri-plane grid and an LSTM decoder that leverages the temporal order of ray propagation, achieving superior performance with significantly fewer parameters (e.g., 0.95 MB) and requiring only original training views without synthesizing pseudo-views [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera pose (origin $\mathbf{o}$ and direction $\mathbf{d}$), bounding box of the scene.
*   **Output:** Predicted ray color $(r, g, b)$.

**Data Structures:**
1.  **Hash Table:** A mapping structure storing learnable feature vectors.
    *   **Size:** $T$ entries. $T = 2^{14}$ for small/medium models, $T = 2^{16}$ for large models [S1, unofficial].
    *   **Layout:** Maps integer coordinate pairs to feature vectors. The index is calculated as:
        $$ \text{Index} = (x_a \oplus y_b) \pmod T $$
        where $\oplus$ is the XOR operation, and $a=1, b=2,654,435,761$ [S1, unofficial].
2.  **Tri-Plane:** Three orthogonal feature planes $\{\pi_{xy}, \pi_{xz}, \pi_{yz}\}$ [S1, unofficial].
    *   **Resolutions:** Each plane has $N$ resolution levels.
        *   Small/Medium: $N=8$ levels, resolutions from 16 to 1024 [S1, unofficial].
        *   Large: $N=16$ levels, resolutions from 16 to 2048 [S1, unofficial].
    *   **Feature Dimension:** Each resolution level stores features of dimension 2 [S1, unofficial].
3.  **Point Feature Vector ($\mathbf{F}$):**
    *   Constructed by concatenating features from all $N$ resolutions across all 3 planes.
    *   **Total Size:** $3 \times N \times 2$ dimensions [S1, unofficial].
4.  **Ray Feature Sequence:**
    *   A sequence $\{\mathbf{F}_1, \dots, \mathbf{F}_K\}$ of length $K$.
    *   $K=256$ for 360° scenes, $K=128$ for forward-facing scenes (using NDC coordinates) [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Emission:** Emit rays based on the camera model for each pixel in the target image [S1, unofficial].
2.  **Sampling:** Sequentially sample $K$ points $\{\mathbf{x}_1, \dots, \mathbf{x}_K\}$ along the ray within the scene's bounding box [S1, unofficial].
3.  **Direction Encoding:** Encode the ray's 3D normalized direction vector $\mathbf{d}$ using Spherical Harmonics (SH) [S1, unofficial].
4.  **Feature Extraction (for each point $\mathbf{x}_k$):**
    *   Project $\mathbf{x}_k$ onto the three orthogonal planes $\pi \in \{\pi_{xy}, \pi_{xz}, \pi_{yz}\}$ to obtain 2D coordinates $\mathbf{x}_\pi$ [S1, unofficial].
    *   For each resolution level $i \in \{1, \dots, N\}$ with grid size $s_i$:
        *   Calculate integer grid coordinates for the four corners surrounding $\mathbf{x}_\pi$:
            $$ \text{top-left} = (\lfloor x_\pi[0]/s_i \rfloor, \lfloor x_\pi[1]/s_i \rfloor) $$
            $$ \text{top-right} = (\lfloor x_\pi[0]/s_i \rfloor, \lfloor x_\pi[1]/s_i \rfloor + 1) $$
            $$ \text{bottom-left} = (\lfloor x_\pi[0]/s_i \rfloor + 1, \lfloor x_\pi[1]/s_i \rfloor) $$
            $$ \text{bottom-right} = (\lfloor x_\pi[0]/s_i \rfloor + 1, \lfloor x_\pi[1]/s_i \rfloor + 1) $$
        *   Retrieve feature vectors from the hash table for each corner using the index formula [S1, unofficial].
        *   Compute the weighted sum of these four corner features based on the distance between $\mathbf{x}_k$ and the vertices (bilinear interpolation) [S1, unofficial].
        *   Append the result to the plane's feature list [S1, unofficial].
    *   Concatenate the feature lists from all $N$ resolutions for each plane [S1, unofficial].
    *   Concatenate the features from the three planes to form the point feature vector $\mathbf{F}_k$ [S1, unofficial].
5.  **Sequence Assembly:** Assemble $\mathbf{F}_1, \dots, \mathbf{F}_K$ into the ray feature sequence [S1, unofficial].
6.  **Color Decoding:**
    *   Initialize LSTM hidden state $\mathbf{H}_0$ and cell state $\mathbf{C}_0$ to zero [G].
    *   For $i = 1$ to $K$:
        $$ (\mathbf{H}_i, \mathbf{C}_i) = \text{LSTM}(\text{Concat}(\mathbf{F}_i, \mathbf{d}), \mathbf{H}_{i-1}, \mathbf{C}_{i-1}) $$
        where $\mathbf{d}$ is the SH-encoded direction [S1, unofficial].
    *   Predict color using a 2-layer MLP on the final hidden state:
        $$ (r, g, b) = \text{MLP}(\mathbf{H}_K) $$
        [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def get_hash_index(x, y, table_size):
    """Calculate hash index using XOR and modulo."""
    # Constants from source [S1, unofficial]
    a = 1
    b = 2654435761
    return (x * a ^ y * b) % table_size

def get_point_features(point_3d, tri_plane_hash_tables, resolutions):
    """
    Extract feature vector for a 3D point from the tri-plane.
    
    Args:
        point_3d: tuple (x, y, z)
        tri_plane_hash_tables: dict mapping plane_name -> list of hash tables (one per resolution)
        resolutions: list of grid sizes (s_i) for each resolution level
        
    Returns:
        F: concatenated feature vector
    """
    F = []
    planes = ['xy', 'xz', 'yz']
    
    # Map 3D point to 2D coordinates for each plane
    # xy: (x, y), xz: (x, z), yz: (y, z)
    plane_coords = {
        'xy': (point_3d[0], point_3d[1]),
        'xz': (point_3d[0], point_3d[2]),
        'yz': (point_3d[1], point_3d[2])
    }
    
    for plane in planes:
        F_plane = []
        coords_2d = plane_coords[plane]
        
        for i, s_i in enumerate(resolutions):
            # Get hash table for this resolution and plane
            hash_table = tri_plane_hash_tables[plane][i]
            
            # Calculate integer grid coordinates
            x_int = int(math.floor(coords_2d[0] / s_i))
            y_int = int(math.floor(coords_2d[1] / s_i))
            
            # Define 4 corners
            corners = [
                (x_int, y_int),           # top-left
                (x_int, y_int + 1),       # top-right
                (x_int + 1, y_int),       # bottom-left
                (x_int + 1, y_int + 1)    # bottom-right
            ]
            
            # Retrieve features from hash table
            corner_features = []
            for cx, cy in corners:
                idx = get_hash_index(cx, cy, len(hash_table))
                # Retrieve feature vector (dimension 2)
                feat = hash_table[idx] 
                corner_features.append(feat)
            
            # Weighted sum (bilinear interpolation)
            # Note: Source [S1, unofficial] mentions "weighted-sum feature based on the distance"
            # Standard bilinear interpolation weights are used here [G]
            x_frac = (coords_2d[0] / s_i) - x_int
            y_frac = (coords_2d[1] / s_i) - y_int
            
            # corner_features order: tl, tr, bl, br
            tl, tr, bl, br = corner_features
            
            # Interpolate
            top = tl * (1 - x_frac) + tr * x_frac
            bottom = bl * (1 - x_frac) + br * x_frac
            weighted_feat = top * (1 - y_frac) + bottom * y_frac
            
            F_plane.append(weighted_feat)
        
        # Concatenate features from all resolutions for this plane
        F_plane_concat = concat_all(F_plane)
        F.append(F_plane_concat)
        
    # Concatenate features from all 3 planes
    return concat_all(F)

def decode_ray_color(feature_sequence, direction_encoded, lstm_model, mlp_model):
    """
    Decode ray color using LSTM and MLP.
    
    Args:
        feature_sequence: list of feature vectors [F1, ..., FK]
        direction_encoded: SH-encoded direction vector d
        lstm_model: 2-layer LSTM network
        mlp_model: 2-layer MLP network
        
    Returns:
        color: (r, g, b)
    """
    # Initialize states [G]
    H = [0.0, 0.0, 0.0, 0.0] # Example hidden state init, actual size depends on model
    C = [0.0, 0.0, 0.0, 0.0] # Example cell state init
    
    for F_i in feature_sequence:
        # Concatenate feature and direction
        input_vec = concat(F_i, direction_encoded)
        
        # LSTM step
        H, C = lstm_step(input_vec, H, C)
        
    # Final prediction
    color = mlp_predict(H)
    return color
```

**Parameters:**
*   **Model Size:** 0.95 MB (small), 1.41 MB (medium/large) [S1, unofficial].
*   **Hash Table Size:** $2^{14}$ (small/medium), $2^{16}$ (large) [S1, unofficial].
*   **Resolutions:** 8 levels (16–1024) for small/medium; 16 levels (16–2048) for large [S1, unofficial].
*   **Feature Dimension:** 2 per level [S1, unofficial].
*   **LSTM Hidden Size:** 32 (small), 128 (medium), 128 (large, 3-layer) [S1, unofficial].
*   **Sequence Length ($K$):** 256 (360°), 128 (forward-facing) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** At the highest resolution, approximately 100,000 3D positions map to a single hash entry [S1, unofficial]. This is not detrimental because points far from the surface have low influence on the final color in volume rendering contexts, allowing the hash entry to adaptively capture appearance features of the dominant nearby points [S1, unofficial].
*   **Decoder Choice:** Using an MLP ignores the temporal order of the feature sequence, while a Transformer's attention mechanism is permutation-invariant and anti-order, losing sequential ray propagation information [S1, unofficial]. LSTM is chosen to preserve this order [S1, unofficial].

## Subtopics

### R2L
A previous NeLF method that uses direct concatenation of frequency-encoded coordinates as input [S1, unofficial]. G-NeLF surpasses R2L by using a feature sequence and LSTM, achieving higher PSNR with fewer parameters and no pseudo-view synthesis [S1, unofficial].

### Instant-NGP
A grid-based NeRF method using multi-resolution hash grids [S1, unofficial]. G-NeLF uses ~1/10th the parameters of Instant-NGP while achieving comparable or superior performance, leveraging the NeLF paradigm to avoid density modeling overhead [S1, unofficial].

### VQRF
A NeRF model compression method using vector quantization [S1, unofficial]. G-NeLF achieves +0.79 dB PSNR at a comparable model size (1.41 MB vs 1.43 MB) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering is $O(K \cdot N)$ per ray, where $K$ is the number of sampled points and $N$ is the number of resolutions. The LSTM decoding is $O(K \cdot H)$, where $H$ is the hidden size [S1, unofficial].
*   **Memory Complexity:** Dominated by the hash tables. Total parameters are ~0.95–1.41 MB, significantly lower than Instant-NGP (64.1 MB) and TensoRF (71.8 MB) [S1, unofficial].
*   **Performance:** On Realistic Synthetic 360° dataset at 400x400, G-NeLF outperforms R2L by 0.04 dB (small model) and 2.48 dB (large model) [S1, unofficial]. It achieves comparable PSNR to standard NeRF methods [S1, unofficial].
*   **Training Data:** Requires only original training views (e.g., 100 views), avoiding the need to synthesize >10K pseudo-views required by earlier NeLF methods [S1, unofficial].
*   **Strengths:** Extremely compact model size, efficient training with limited data, high rendering quality for a NeLF method [S1, unofficial].
*   **Limitations:** Does not explicitly model geometry (density), which may limit applications requiring explicit mesh extraction or volume-based effects [S1, unofficial].

## Background
G-NeLF was proposed to address the training difficulties and high storage overhead of Neural Light Field (NeLF) methods. Traditional NeLF methods required training a NeRF model first and synthesizing over 10,000 views to train the NeLF, which was resource-intensive [S1, unofficial]. G-NeLF introduces a hybrid approach using a spatial-aware feature sequence from a hash-based multi-resolution tri-plane and an LSTM decoder to predict ray color directly, enabling training with as few as 100 original views [S1, unofficial]. The method leverages the temporal order of ray propagation, which MLPs and Transformers fail to capture effectively [S1, unofficial].

## Sources
[S1] G-NeLF: Memory- and Data-Efficient Hybrid Neural Light Field for Novel View Synthesis, https://arxiv.org/abs/2409.05617v1, fetched 2026-10-27 (unofficial)
