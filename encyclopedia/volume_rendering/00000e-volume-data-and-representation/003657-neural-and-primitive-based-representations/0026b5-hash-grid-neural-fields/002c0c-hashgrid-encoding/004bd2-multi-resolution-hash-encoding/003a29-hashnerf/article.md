# HashNeRF

Also called: Instant Neural Graphics Primitives, Instant Neural Graphics Primitives with a Multiresolution Hash Encoding

## Overview
HashNeRF is a method for optimizing Neural Radiance Fields (NeRF) that drastically reduces training time by replacing positional encoding with a trainable multiresolution hash encoding [S1, unofficial]. It maps 3D coordinates to a high-dimensional feature space using multiple hash tables and linear interpolation, followed by a shallow neural network to predict density and color [S1, unofficial]. It sets itself apart from standard NeRF by using a generic, trainable encoding that allows the model to converge in fewer iterations and train significantly faster on modern GPUs [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D spatial coordinate $\mathbf{x} = (x, y, z)$ and a viewing direction $\mathbf{d}$.
*   **Output:** Volume density $\sigma$ and view-dependent color $\mathbf{c}$.

**Data Structures:**
The method relies on $L$ hash tables $H_0, \dots, H_{L-1}$, where each table $H_l$ corresponds to a specific resolution level $l$. Each table has a fixed size $T_l$ and maps hashed coordinate indices to feature vectors of dimension $F$ [S1, unofficial]. The shallow neural network consists of two parts: Density MLP layers (outputting log-space density) and Color MLP layers (adding color variation information) [S1, unofficial].

**Algorithm Steps:**
1.  **Coordinate Hashing:** For each resolution level $l$, the input coordinate $\mathbf{x}$ is hashed to an index $i_l$. This involves scaling $\mathbf{x}$ to the resolution of level $l$, computing grid coordinates, and applying a hash function to map these to the table size $T_l$ [S1, unofficial].
2.  **Feature Fetching:** The feature vector $\mathbf{f}_l$ is retrieved from hash table $H_l$ at index $i_l$ [S1, unofficial].
3.  **Interpolation:** Linear interpolation is performed on the features from the surrounding grid vertices (corners of the voxel containing $\mathbf{x}$) to obtain the interpolated feature vector $\mathbf{v}_l$ for level $l$ [S1, unofficial].
4.  **Concatenation:** The interpolated feature vectors from all $L$ levels are concatenated to form the combined feature vector $\mathbf{V} = [\mathbf{v}_0; \dots; \mathbf{v}_{L-1}]$ [S1, unofficial].
5.  **Decoding:** $\mathbf{V}$ is passed through shallow neural network heads (MLPs) to predict volume density $\sigma$ and view-dependent color $\mathbf{c}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hash_neRF_query(x, d, hash_tables, mlp_density, mlp_color):
    """
    Query the HashNeRF model for density and color at coordinate x and direction d.
    
    Args:
        x: 3D coordinate (x, y, z)
        d: Viewing direction vector
        hash_tables: List of L hash tables H_0 ... H_{L-1}
        mlp_density: MLP for density prediction
        mlp_color: MLP for color prediction
        
    Returns:
        sigma: Volume density
        c: View-dependent color
    """
    features = []
    
    # Step 1-3: Hash, Fetch, and Interpolate for each resolution level
    for l in range(len(hash_tables)):
        # Scale coordinate to level l resolution
        x_scaled = x * (2**l)
        
        # Compute grid coordinates (floor)
        x_floor = floor(x_scaled)
        
        # Compute fractional part for interpolation
        frac = x_scaled - x_floor
        
        # Get the 8 corner indices for trilinear interpolation
        # Note: The parent article describes linear interpolation; 
        # standard 3D implementation requires trilinear interpolation.
        # We assume standard trilinear interpolation over the voxel corners.
        corner_indices = get_corner_indices(x_floor)
        
        # Fetch features from the hash table
        corner_features = []
        for idx in corner_indices:
            # Hash index to table size T_l
            hash_idx = hash_function(idx, table_size=hash_tables[l].size)
            # Retrieve feature vector
            f = hash_tables[l][hash_idx]
            corner_features.append(f)
            
        # Trilinear interpolation of corner features
        v_l = trilinear_interpolate(corner_features, frac)
        features.append(v_l)
        
    # Step 4: Concatenate features
    V = concatenate(features)
    
    # Step 5: Decode Density
    # The density MLP outputs log-space density
    sigma_log = mlp_density(V)
    sigma = exp(sigma_log)
    
    # Step 5: Decode Color
    # Concatenate V and viewing direction d for color prediction
    # Note: Standard NeRF concatenates direction to the feature vector
    # before the color MLP. The source mentions "Color MLP layers - 
    # the former outputs the log-space density, while the color layers 
    # add color variation information".
    # We assume standard NeRF color input: [V; d]
    color_input = concatenate([V, d])
    c = mlp_color(color_input)
    
    return sigma, c
```

**Parameters:**
*   **Hardware:** NVIDIA GeForce RTX 3090 Ti GPUs [S1, unofficial].
*   **Training Iterations:** Evaluated at 500 and 1,000 iterations [S1, unofficial].
*   **Dataset:** 100 training, 100 validation, 200 test images (RGB + depth) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** Hash collisions are a known phenomenon. In Instant-NGP, the hash table directly stores features, and collisions are handled by overwriting or specific hashing strategies [S1, unofficial]. In other implementations like MERF, collisions are resolved using a 2-layer MLP with 64 hidden units [S2].
*   **Vanishing Gradients:** The default architecture includes residual connections between the image generation and color generation pipelines to ensure communication and avoid vanishing gradients [S1, unofficial].

## Subtopics

### [HashNeRF Architectural Variants](8688-hashnerf-architectural-variants.md)
This variant details four specific architectural modifications to the HashNeRF color and density networks, proposed to evaluate the impact of residual connections, convolutional layers, and network depth on reconstruction quality and training time. These variants address the limitations of the standard shallow multi-layer perceptron (MLP) decoder by exploring alternative feature aggregation and information preservation strategies. They are distinct from the base HashNeRF method [8678-hashnerf.md] by altering the internal connectivity and layer types of the feature-decoding stage.

### HollowNeRF
HollowNeRF is a NeRF compression method that automatically sparsifies the feature grid during training by using a trainable 3D saliency mask to guide efficient feature pruning, redistributing hash collisions to improve rendering quality with fewer parameters [S2]. It uses 31% of the parameters of Instant-NGP while delivering comparable rendering quality [S2]. It achieves up to 1dB PSNR accuracy gain using only 56% of the parameters compared to Instant-NGP [S2].

**Steps:**
1.  Given input coordinate $\mathbf{x}=(x,y,z)$, fetch feature $\mathbf{f}$ from multi-level hashgrid following Instant-NGP [S2].
2.  Predict saliency weight $p$ by processing information from trainable 3D saliency grid $G$ through trilinear interpolation and a sigmoid function [S2].
3.  Scale feature $\mathbf{f}$ by $p$ to get weighted feature $\mathbf{v} = p\mathbf{f}$ [S2].
4.  Feed weighted feature $\mathbf{v}$ into MLP to decode density $\sigma$ and color [S2].
5.  Ensure $\mathbf{v}=0$ maps to $\sigma=0$ using a zero-skipping gate [S2].
6.  Use density and color for volume rendering, compare to ground truth for loss $L$ [S2].
7.  During training, use ADMM pruner to sparsify saliency value distribution by enforcing sparsity constraint when optimizing loss $L$ [S2].

**Parameters:**
*   $T$: Resolution of the trainable 3D saliency grid $G$, typically much smaller than the resolution of the dense feature grid [S2].

## Operating characteristics
HashNeRF trains faster and converges in fewer iterations than NeRF because the encoding is generic, trainable, and the network is shallow with fewer parameters [S1, unofficial]. The shallow network architecture reduces the computational load per iteration, while the hash encoding allows for efficient memory access and parallelization on GPUs [S1, unofficial]. The method is particularly effective for real-time applications and scenarios requiring rapid iteration, such as Gigapixel approximation, generating signed distance functions, and Neural Radiance Caching [S1, unofficial].

## Background
HashNeRF, formally known as "Instant Neural Graphics Primitives with a Multiresolution Hash Encoding," was proposed by Thomas Müller, Alex Evans, Christoph Schied, and Alexander Keller [S1, unofficial]. It was published in ACM Transactions on Graphics in July 2022 [S1, unofficial]. The method builds on the original NeRF framework [S1, unofficial] but introduces a radical approach to reducing training times by leveraging multiresolution hash encoding [S1, unofficial].

## Sources
[S1] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S2] HollowNeRF: Pruning Hashgrid-Based NeRFs with Trainable Collision Mitigation, https://doi.org/10.1109/iccv51070.2023.00322, fetched 2026-10-26
