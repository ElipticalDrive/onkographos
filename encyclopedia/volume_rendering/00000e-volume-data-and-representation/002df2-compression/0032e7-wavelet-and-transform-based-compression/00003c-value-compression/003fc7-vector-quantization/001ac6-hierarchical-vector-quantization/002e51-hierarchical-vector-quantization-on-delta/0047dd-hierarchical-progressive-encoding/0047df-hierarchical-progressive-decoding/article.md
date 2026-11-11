# Hierarchical Progressive Decoding

Also called: Adaptive Rendering, HPC Decoding

## Overview
Hierarchical Progressive Decoding is the decoding phase of the Neural Residual Radiance Fields (ReRF) codec, designed to recover feature grids from a compressed bitstream for the real-time rendering of dynamic free-viewpoint videos (FVV) [S1]. It differs from static volume rendering by utilizing temporal redundancy through motion-compensated prediction and residual encoding, allowing adaptive streaming based on available bandwidth [S1]. It builds on [Hierarchical Progressive Encoding](12107-hierarchical-progressive-encoding.md) by interpreting the hierarchical residual structure to reconstruct multi-resolution feature grids.

## How it works
**Inputs and Outputs:**
*   **Input:** Compressed bitstream $B_t$ for frame $t$; quantization parameter $q$; entropy decoder $D$; previous frame's reconstructed feature grid buffer $\hat{F}_{l, t-1}$ for all levels $l \in [1, L]$; global MLP $\Phi$.
*   **Output:** Reconstructed feature grids $\hat{F}_{l, t}$ for all levels $l \in [1, L]$; rendered color $\hat{c}_l(r)$ for each resolution level $l$.

**Data Structures:**
*   **Bitstream $B_t$:** Contains entropy-coded residuals for the current frame.
*   **Feature Grid $\hat{F}_{l, t}$:** A 3D grid of feature vectors at resolution level $l$ for frame $t$.
*   **Residual $\hat{R}_{l, t}$:** The decoded difference between the current frame's features and the previous frame's features at level $l$.
*   **Buffer:** Stores $\hat{F}_{l, t-1}$ for temporal prediction.

**Steps:**

1.  **Entropy Decoding:** Decode the bitstream $B_t$ using the entropy decoder $D$ to recover the quantized residuals. The decoder must reverse the quantization applied during encoding. The quantization involved multiplying the minimum residual value by $q$ and converting to non-negative values. The recovery formula is [S1]:
    $$ \hat{R}_{l, t} = \frac{D(B_{l, t}) + q \cdot \min(R_{l, t})}{q} $$
    where $D(B_{l, t})$ is the decoded integer value, $q$ is the quantization parameter, and $\min(R_{l, t})$ is the minimum residual value transmitted or known from context to shift the data range back to its original scale. Note: The source notation $Q(q \cdot \min(R_t))$ in Eq. (6) implies the quantization operation; the inverse operation recovers the residual.

2.  **Feature Reconstruction:** Reconstruct the feature grid $\hat{F}_{l, t}$ at level $l$ by adding the recovered residual $\hat{R}_{l, t}$ to the previous frame's reconstructed feature grid $\hat{F}_{l, t-1}$ at the same level. This is performed recursively for all levels $l \in [1, L]$ [S1]:
    $$ \hat{F}_{l, t} = \hat{F}_{l, t-1} + \hat{R}_{l, t} $$
    For the first frame of a Group of Features (GoF), $\hat{F}_{l, t-1}$ is initialized to zero or the full reference grid if specified by the encoder.

3.  **Adaptive Selection:** Select a subset of resolution levels $l \in [1, L_{max}]$ based on available bandwidth or computational constraints. Lower levels $l$ provide coarser details, while higher levels provide finer details.

4.  **Feature to Radiance Conversion:** Pass the reconstructed feature grids $\hat{F}_{l, t}$ through the global MLP $\Phi$ to obtain color $c_{l, i}$ and density $\sigma_{l, i}$ for each sampled point $i$ along a ray $r$.

5.  **Volume Rendering:** Perform standard volume rendering to obtain the predicted color $\hat{c}_l(r)$ at resolution $l$ for each pixel [S1]:
    $$ \hat{c}_l(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_{l, i} \delta_i)) c_{l, i} $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_{l, j} \delta_j)$ is the transmittance, $\delta_i$ is the distance between adjacent samples, and $N$ is the number of samples along the ray.

**Parameters:**
*   $L$: Total number of resolution levels. Not given in the sources.
*   $q$: Quantization parameter trading off reconstruction quality and model storage. Not given in the sources.
*   $N$: Number of samples per ray. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Drift:** Errors in $\hat{R}_{l, t}$ accumulate over time if the previous frame $\hat{F}_{l, t-1}$ is incorrect. The GoF structure limits this drift to the group size.
*   **Boundary Conditions:** The first frame of a GoF requires a full reference grid. If not provided, initialization must be handled (e.g., zero). [G]
*   **Quantization Precision:** The division by $q$ in the decoding step can introduce floating-point precision errors. [G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_progressive_decode(bitstream_B_t, q, entropy_decoder_D, 
                                    prev_features_F_hat, global_MLP_Phi, 
                                    num_levels_L, ray_samples):
    """
    Decodes feature grids and renders a frame using hierarchical progressive decoding.
    
    Args:
        bitstream_B_t: Compressed bitstream for frame t.
        q: Quantization parameter.
        entropy_decoder_D: Function to decode entropy-coded data.
        prev_features_F_hat: Dictionary of feature grids {l: F_hat_l_t_minus_1} for all levels.
        global_MLP_Phi: Neural network mapping features to color and density.
        num_levels_L: Total number of resolution levels.
        ray_samples: List of 3D points along rays for volume rendering.
        
    Returns:
        rendered_colors: Dictionary of rendered colors {l: c_hat_l} for selected levels.
    """
    current_features_F_hat = {}
    rendered_colors = {}
    
    # Step 1 & 2: Decode and Reconstruct Feature Grids
    for l in range(1, num_levels_L + 1):
        # Extract bitstream segment for level l
        B_l_t = extract_bitstream_segment(bitstream_B_t, l)
        
        # Decode residuals
        # Note: min_R_l_t must be known from context or transmitted separately
        # Assuming min_R_l_t is available or zero for first frame
        min_R_l_t = get_min_residual(l) # [G] Placeholder for context-dependent value
        quantized_residual = entropy_decoder_D(B_l_t)
        
        # Reverse quantization
        R_hat_l_t = (quantized_residual + q * min_R_l_t) / q
        
        # Reconstruct feature grid
        F_hat_l_t_minus_1 = prev_features_F_hat.get(l, zeros_like_grid()) # [G] Init for first frame
        F_hat_l_t = F_hat_l_t_minus_1 + R_hat_l_t
        current_features_F_hat[l] = F_hat_l_t
        
    # Step 3: Adaptive Selection (Simplified: use all levels for now)
    selected_levels = range(1, num_levels_L + 1)
    
    # Step 4 & 5: Render
    for l in selected_levels:
        F_hat_l_t = current_features_F_hat[l]
        color_sum = 0.0
        transmittance = 1.0
        
        for point in ray_samples:
            # Get features at point
            features = sample_feature_grid(F_hat_l_t, point)
            
            # Convert to color and density
            c_l_i, sigma_l_i = global_MLP_Phi(features)
            
            # Volume rendering step
            delta_i = get_sample_distance(point) # [G] Placeholder
            alpha = 1.0 - exp(-sigma_l_i * delta_i)
            color_sum += transmittance * alpha * c_l_i
            transmittance *= (1.0 - alpha)
            
        rendered_colors[l] = color_sum
        
    return rendered_colors, current_features_F_hat
```

## Subtopics

### [ReRF Player Reconstruction](12534-rerf-player-reconstruction.md)
ReRF Player Reconstruction is the decoding phase of the Neural Residual Radiance Fields (ReRF) codec, designed to recover feature grids from a compressed bitstream for the real-time rendering of dynamic free-viewpoint videos (FVV). It differs from static volume rendering by utilizing temporal redundancy through motion-compensated prediction and residual encoding, allowing adaptive streaming based on available bandwidth.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of levels $L$ and the number of samples $N$ per ray. Decoding is $O(L \cdot N)$, rendering is $O(L \cdot N)$.
*   **Memory Complexity:** $O(L \cdot V)$ where $V$ is the volume of the feature grid. Requires storing the previous frame's feature grids for temporal prediction.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with bandwidth by selecting fewer levels $L$. Higher levels provide finer detail but require more data.
*   **Strengths:** Adaptive quality based on bandwidth; efficient temporal compression via residuals.
*   **Limitations:** Drift over long sequences if not reset; dependency on previous frame accuracy.
*   **When to use:** Dynamic volumetric video streaming with variable bandwidth.
*   **When not to use:** Static volumes or scenarios requiring absolute precision without drift tolerance.

## Background
Hierarchical Progressive Decoding is part of the HPC (Hierarchical Progressive Coding) framework for volumetric video, introduced in the paper "HPC: Hierarchical Progressive Coding Framework for Volumetric Video" [S1]. It addresses the challenge of streaming dynamic free-viewpoint videos by leveraging temporal redundancy and multi-resolution representations.

## Sources
[S1] HPC: Hierarchical Progressive Coding Framework for Volumetric Video, https://doi.org/10.1145/3664647.3681107, fetched 2026-10-28
