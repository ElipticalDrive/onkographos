# InstantNGP Network Architecture

Also called: Instant Neural Graphics Primitives Network, InstantNGP MLP, MonoSDF Network, Network Architecture and Implementation

## Overview
This article describes the specific Multi-Layer Perceptron (MLP) architecture and feature encoding scheme introduced in Instant Neural Graphics Primitives (InstantNGP) [S1, unofficial]. It solves the latency bottleneck of training and querying large neural fields by combining a multi-resolution hash encoding with compact MLPs. It sets itself apart from the parent [NeRF MLP Architecture](910-nerf-mlp-architecture.md) by replacing positional encoding with a learnable, sparse hash grid, and from siblings like [Tiny NeRF](1275-tiny-nerf.md) by using a deeper, wider base network with specific hidden unit counts optimized for GPU memory bandwidth.

## How it works
The InstantNGP architecture consists of three distinct MLPs (Base SDF, Bias, and Color) that share a common input encoding strategy. The method maps 3D spatial coordinates to features, which are then processed by the MLPs to predict network outputs (e.g., density, SDF, color).

### Inputs and Outputs
*   **Inputs:** 3D spatial coordinates $\mathbf{x} \in \mathbb{R}^3$. In view-dependent variants, viewing direction $\mathbf{d} \in \mathbb{R}^3$ is also an input.
*   **Outputs:**
    *   Base SDF Network: Signed Distance Field value $s$ or density $\sigma$.
    *   Bias Network: Bias correction term $b$.
    *   Color Network: View-dependent color $\mathbf{c} \in \mathbb{R}^3$.

### Data Structures: Multi-Resolution Hash Encoding
InstantNGP uses a multi-resolution hash grid to encode input positions. This replaces the fixed-frequency positional encoding used in standard NeRF.

1.  **Grid Levels:** The encoding uses $L$ resolution levels. For each level $l \in \{0, \dots, L-1\}$, a 3D grid of resolution $R_l$ is defined.
2.  **Hash Map:** Each level $l$ has an associated hash map $H_l$ of size $T_l$ (typically $2^{19}$ [S4]). The hash map stores feature vectors of dimension $F$ (typically 2 or 4 per coordinate dimension, so $3 \times F$ total per position).
3.  **Hash Function:** A spatial hash function $h_l(\mathbf{x})$ maps the 3D coordinate $\mathbf{x}$ to an index in $H_l$. The hash function is usually a simple modulo-based hash of the quantized integer coordinates.
4.  **Feature Vector:** For a query $\mathbf{x}$, the feature vector $\mathbf{v}$ is computed by:
    *   For each level $l$, retrieve the feature vector $\mathbf{f}_l$ from $H_l$ at index $h_l(\mathbf{x})$.
    *   Interpolate $\mathbf{f}_l$ from the 8 surrounding grid vertices if $\mathbf{x}$ is not aligned with the grid.
    *   Concatenate the features from all levels: $\mathbf{v} = [\mathbf{f}_0; \mathbf{f}_1; \dots; \mathbf{f}_{L-1}]$.

### Network Layouts
The architecture comprises three MLPs. All use ReLU activations except where noted.

1.  **Base SDF Network:**
    *   Structure: 4-layer MLP.
    *   Hidden Units: 256 per layer [S1, unofficial].
    *   Input: Encoded position vector $\mathbf{v}$.
    *   Output: Scalar $s$ (SDF) or $\sigma$ (density).

2.  **Bias Network:**
    *   Structure: 2-layer MLP.
    *   Hidden Units: Similar scale to Base SDF (implied 256) [S1, unofficial].
    *   Input: Encoded position vector $\mathbf{v}$.
    *   Output: Scalar bias $b$.

3.  **Color Network:**
    *   Structure: 2-layer MLP.
    *   Hidden Units: 128 per layer [S1, unofficial].
    *   Input: Encoded position vector $\mathbf{v}$ concatenated with encoded viewing direction $\mathbf{d}$ (if view-dependent).
    *   Output: Color vector $\mathbf{c} \in \mathbb{R}^3$.

### Numbered Steps for Querying
1.  **Encode Position:** Given query point $\mathbf{x}$, compute the multi-resolution hash encoded feature vector $\mathbf{v}$ by querying the hash grids at all $L$ levels and concatenating the results.
2.  **Base SDF Prediction:** Pass $\mathbf{v}$ through the Base SDF Network (4-layer, 256 hidden units) to obtain the raw SDF value $s_{raw}$.
3.  **Bias Prediction:** Pass $\mathbf{v}$ through the Bias Network (2-layer, 256 hidden units) to obtain the bias term $b$.
4.  **Combine SDF:** Compute the final SDF value $s = s_{raw} + b$.
5.  **Color Prediction (if applicable):**
    *   Encode viewing direction $\mathbf{d}$ using spherical harmonics or positional encoding.
    *   Concatenate $\mathbf{v}$ and encoded $\mathbf{d}$.
    *   Pass through the Color Network (2-layer, 128 hidden units) to obtain color $\mathbf{c}$.

### Parameters
*   **Base SDF Hidden Units:** 256 [S1, unofficial].
*   **Base SDF Layers:** 4 [S1, unofficial].
*   **Color Hidden Units:** 128 [S1, unofficial].
*   **Color Layers:** 2 [S1, unofficial].
*   **Hash Map Size:** $2^{19}$ [S4].
*   **Point Cloud Size (for initialization/guidance):** Downsampled to 2 million points for robustness experiments [S1, unofficial].

### Edge Cases and Pitfalls
*   **Hash Collisions:** With a finite hash map size ($2^{19}$), collisions are inevitable. The hash function and grid resolution must be balanced to minimize artifacts [S4].
*   **Precision:** The implementation often uses half-precision (FP16) for memory efficiency and speed, which may require careful handling of gradient scaling [S3, S4].
*   **Boundary Conditions:** The hash grid is typically defined within a bounding box. Queries outside this box must be handled (e.g., by returning a default value or clamping) [S1, G, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class InstantNGPNetwork:
    def __init__(self, hash_map_size=2**19, num_levels=16, base_hidden=256, color_hidden=128, feature_dim=2):
        """
        Initialize the InstantNGP network architecture.
        
        Args:
            hash_map_size (int): Size of the hash maps for each level.
            num_levels (int): Number of resolution levels in the multi-resolution hash grid.
            base_hidden (int): Number of hidden units in the Base SDF and Bias networks.
            color_hidden (int): Number of hidden units in the Color network.
            feature_dim (int): Dimension of the feature vector per coordinate per level.
        """
        self.hash_map_size = hash_map_size
        self.num_levels = num_levels
        self.base_hidden = base_hidden
        self.color_hidden = color_hidden
        self.feature_dim = feature_dim
        
        # Initialize hash grids (simplified representation)
        # In practice, these are sparse hash maps stored in GPU memory
        self.hash_grids = [np.random.randn(hash_map_size, 3 * feature_dim).astype(np.float32) for _ in range(num_levels)]
        
        # Initialize MLP weights
        # Base SDF: Input -> 256 -> 256 -> 256 -> 256 -> Output (1)
        self.base_sdf_weights = [
            np.random.randn(3 * num_levels * feature_dim, base_hidden).astype(np.float32),
            np.random.randn(base_hidden, base_hidden).astype(np.float32),
            np.random.randn(base_hidden, base_hidden).astype(np.float32),
            np.random.randn(base_hidden, base_hidden).astype(np.float32),
            np.random.randn(base_hidden, 1).astype(np.float32)
        ]
        
        # Bias Network: Input -> 256 -> 256 -> Output (1)
        self.bias_weights = [
            np.random.randn(3 * num_levels * feature_dim, base_hidden).astype(np.float32),
            np.random.randn(base_hidden, base_hidden).astype(np.float32),
            np.random.randn(base_hidden, 1).astype(np.float32)
        ]
        
        # Color Network: Input + Direction -> 128 -> 128 -> Output (3)
        # Assuming direction is encoded to a fixed size, e.g., 27 (3 SH bands)
        self.color_direction_dim = 27 
        self.color_weights = [
            np.random.randn(3 * num_levels * feature_dim + self.color_direction_dim, color_hidden).astype(np.float32),
            np.random.randn(color_hidden, color_hidden).astype(np.float32),
            np.random.randn(color_hidden, 3).astype(np.float32)
        ]

    def _hash(self, x, level):
        """
        Hash a 3D coordinate to an index in the hash map for a given level.
        Simplified implementation.
        """
        # Quantize and hash
        # In practice, this involves scaling x to the grid resolution for level 'level'
        # and then applying a hash function
        return int(np.abs(np.sum(x * 1000)) % self.hash_map_size)

    def _interpolate(self, x, level):
        """
        Interpolate features from the hash grid for a given level.
        Simplified implementation assuming direct lookup for non-grid-aligned points.
        """
        idx = self._hash(x, level)
        return self.hash_grids[level][idx]

    def _encode(self, x):
        """
        Encode 3D position x using multi-resolution hash grid.
        """
        features = []
        for level in range(self.num_levels):
            # Get feature vector for this level
            # Note: In a real implementation, this would involve trilinear interpolation
            # from the 8 surrounding grid vertices. Here we assume direct lookup for simplicity.
            feat = self._interpolate(x, level)
            features.append(feat)
        return np.concatenate(features)

    def _relu(self, x):
        return np.maximum(0, x)

    def _mlp_forward(self, x, weights):
        """
        Forward pass through an MLP.
        """
        for i, w in enumerate(weights):
            x = np.dot(x, w)
            if i < len(weights) - 1:
                x = self._relu(x)
        return x

    def forward(self, x, d=None):
        """
        Forward pass through the InstantNGP network.
        
        Args:
            x (np.ndarray): 3D position [x, y, z].
            d (np.ndarray, optional): Viewing direction [dx, dy, dz].
            
        Returns:
            tuple: (sdf_value, color)
        """
        # 1. Encode position
        encoded_x = self._encode(x)
        
        # 2. Base SDF Network
        sdf_raw = self._mlp_forward(encoded_x, self.base_sdf_weights)
        
        # 3. Bias Network
        bias = self._mlp_forward(encoded_x, self.bias_weights)
        
        # 4. Combine SDF
        sdf = sdf_raw + bias
        
        # 5. Color Network (if direction is provided)
        color = None
        if d is not None:
            # Encode direction (simplified: assume pre-encoded to fixed size)
            encoded_d = np.zeros(self.color_direction_dim) # Placeholder for SH encoding
            input_color = np.concatenate([encoded_x, encoded_d])
            color = self._mlp_forward(input_color, self.color_weights)
            
        return sdf, color
```

## Subtopics

### iNGP Texture Storage
iNGP Texture Storage is a variant of the InstantNGP feature storage method that uses textures instead of hash tables to improve memory locality and leverage hardware interpolation [S2]. It sets itself apart from the standard hash-based approach by storing presummed features $g'$ at resolution level $L$, where $g'(v) = \sum_{l=0}^{L} g(v, l)$ for each texel in $L$ [S2]. This allows fetching anti-aliased features by interpolating between two levels implied by pixel area $p(x)$, reducing the number of texture fetches per MLP evaluation to 8, from 24 in the original hash-based approach (3+3×7) [S2]. It is chosen when rendering performance is critical and the scene can be represented with explicit 3D grids or triplanes [S2].

### Tiny MLP for Density and Color Estimation
The Tiny MLP is a compact network variant used in Rip-NeRF that nonlinearly maps the Ripmap-Encoding feature vector $f$ and view direction $d$ to density $\tau$ and color $c$ [S3]. It sets itself apart from the standard InstantNGP MLP by being significantly smaller (width 128, input dimension 160) and optimized for fused and half-precision operations using tiny-cuda-nn [S3]. The first two layers process $f$ to yield $\tau$ and a 15-dimensional geometric feature $f_{geo}$, while the final three layers combine $f_{geo}$ with spherical harmonics-encoded $d$ to estimate $c$ [S3]. It is chosen when computational efficiency and low latency are prioritized over the capacity of larger MLPs [S3].

### Flash Cache Implementation
The Flash Cache Implementation is a variant of the InstantNGP architecture designed for inverse rendering, using a fast cache NGP to reduce bias in radiance cache calculations [S4]. It sets itself apart by using a specific NGP configuration with a maximum grid resolution of 256 and a 4-layer 128-hidden-unit MLP to predict $S=8$ sample-point distances [S4]. The hash maps for all NGPs in this implementation have a size of $2^{19}$ [S4]. It is chosen when performing inverse rendering tasks that require efficient sampling and caching of radiance information [S4].

## Operating characteristics
*   **Time Complexity:** The hash encoding is $O(L)$ where $L$ is the number of levels, with constant-time hash lookups. The MLP forward pass is $O(N \cdot H)$ where $N$ is the number of layers and $H$ is the hidden size.
*   **Memory Complexity:** The hash grid memory is $O(L \cdot T \cdot F)$ where $T$ is the hash map size and $F$ is the feature dimension. For $T=2^{19}$ and $F=2$, this is manageable on modern GPUs.
*   **Performance:** InstantNGP achieves real-time rendering speeds on standard GPUs due to the efficient hash encoding and compact MLPs [S1, unofficial].
*   **Strengths:** High training and inference speed, compact memory footprint, good generalization for small to medium scenes.
*   **Limitations:** Hash collisions can cause artifacts, limited capacity for very large or complex scenes compared to voxel-based methods, and potential precision issues with half-precision arithmetic.
*   **When to use:** Real-time neural rendering, small to medium scenes, resource-constrained environments.
*   **When not to use:** Very large open-world scenes, high-fidelity offline rendering where capacity is more important than speed.

## Background
InstantNGP was introduced by Müller et al. in 2022 [S1, unofficial]. It was developed to address the slow training and inference times of previous NeRF methods. The method combines multi-resolution hash encoding with compact MLPs to achieve real-time performance. It has become a standard baseline for many subsequent neural rendering methods, including MonoSDF [S1, unofficial] and others that build upon its architecture.

## Sources
[S1] Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering, https://doi.org/10.48550/arXiv.2310.07997, fetched 2026-10-30 (unofficial)
[S2] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
[S3] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
[S4] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
