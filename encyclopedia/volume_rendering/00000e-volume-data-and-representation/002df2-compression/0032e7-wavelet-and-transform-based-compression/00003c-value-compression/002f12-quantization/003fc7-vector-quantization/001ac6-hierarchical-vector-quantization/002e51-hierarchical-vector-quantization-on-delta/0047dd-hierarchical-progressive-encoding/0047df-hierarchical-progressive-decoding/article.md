# Hierarchical Progressive Decoding

Also called: Adaptive Rendering, HPC Decoding

## Overview
Hierarchical Progressive Decoding is the decoding phase of the Neural Residual Radiance Fields (ReRF) codec, designed to recover feature grids from a compressed bitstream for the real-time rendering of dynamic free-viewpoint videos (FVV) [S1]. It differs from static volume rendering by utilizing temporal redundancy through motion-compensated prediction and residual encoding, allowing adaptive streaming based on available bandwidth [S1]. It builds on [Hierarchical Progressive Encoding](12107-hierarchical-progressive-encoding.md) by interpreting the hierarchical residual structure to reconstruct multi-resolution feature grids [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Compressed bitstream $B_t$ for frame $t$; quantization parameter $q$; entropy decoder $D$; previous frame's reconstructed feature grid buffer $\hat{F}_{l, t-1}$ for all levels $l \in [1, L]$; global MLP $\Phi$ [S1].
*   **Output:** Reconstructed feature grids $\hat{F}_{l, t}$ for all levels $l \in [1, L]$; rendered color $\hat{c}_l(r)$ for each resolution level $l$ [S1].

**Data Structures:**
*   **Bitstream $B_t$:** Contains entropy-coded residuals for the current frame [S1].
*   **Feature Grid $\hat{F}_{l, t}$:** A 3D grid of feature vectors at resolution level $l$ for frame $t$ [S1].
*   **Residual $\hat{R}_{l, t}$:** The decoded difference between the current frame's features and the previous frame's features at level $l$ [S1].
*   **Buffer:** Stores $\hat{F}_{l, t-1}$ for temporal prediction [S1].

**Steps:**

1.  **Entropy Decoding & Dequantization:** Decode the bitstream $B_t$ using the entropy decoder $D$ to recover the quantized residuals [S1]. The decoder reverses the quantization applied during encoding, which involved multiplying the minimum residual value by $q$ and converting to non-negative values [S1]. The recovery formula is:
    $$ \hat{R}_{l, t} = \frac{D(B_{l, t}) + q \cdot \min(R_{l, t})}{q} $$
    where $D(B_{l, t})$ is the decoded integer value, $q$ is the quantization parameter, and $\min(R_{l, t})$ is the minimum residual value transmitted or known from context to shift the data range back to its original scale [S1].

2.  **Feature Reconstruction:** Reconstruct the feature grid $\hat{F}_{l, t}$ at level $l$ by adding the recovered residual $\hat{R}_{l, t}$ to the previous frame's reconstructed feature grid $\hat{F}_{l, t-1}$ at the same level [S1]:
    $$ \hat{F}_{l, t} = \hat{F}_{l, t-1} + \hat{R}_{l, t} $$
    This is performed recursively for all levels $l \in [1, L]$ [S1]. For the first frame of a Group of Features (GoF), $\hat{F}_{l, t-1}$ is initialized to zero or the full reference grid if specified by the encoder [S1].

3.  **Adaptive Selection:** Select a subset of resolution levels $l \in [1, L]$ based on available bandwidth or computational constraints [S1]. Lower levels $l$ provide coarser details, while higher levels provide finer details [S1].

4.  **Feature to Radiance Conversion:** Pass the reconstructed feature grids $\hat{F}_{l, t}$ through the global MLP $\Phi$ to obtain color $c_{l, i}$ and density $\sigma_{l, i}$ for each sampled point $i$ along a ray $r$ [S1].

5.  **Volume Rendering:** Perform standard volume rendering to obtain the predicted color $\hat{c}_l(r)$ at resolution $l$ for each pixel [S1]:
    $$ \hat{c}_l(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_{l, i} \delta_i)) c_{l, i} $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_{l, j} \delta_j)$ is the transmittance, $\delta_i$ is the distance between adjacent samples, and $N$ is the number of samples along the ray [S1].

**Parameters:**
*   $L$: Total number of resolution levels. Not given in the sources.
*   $q$: Quantization parameter trading off reconstruction quality and model storage. Not given in the sources.
*   $N$: Number of samples per ray. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Drift:** Errors in $\hat{R}_{l, t}$ accumulate over time if the previous frame $\hat{F}_{l, t-1}$ is incorrect [S1]. The GoF structure limits this drift to the group size [S1].
*   **Boundary Conditions:** The first frame of a GoF requires a full reference grid [S1]. If not provided, initialization must be handled (e.g., zero) [G].
*   **Quantization Precision:** The division by $q$ in the decoding step can introduce floating-point precision errors [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def hierarchical_progressive_decode(bitstream_B_t, q, entropy_decoder_D,
                                    prev_features_F_hat, global_MLP_Phi,
                                    num_levels_L, ray_samples, min_residuals):
    """
    Decodes feature grids and renders a frame using hierarchical progressive decoding.
    """
    current_features_F_hat = {}
    rendered_colors = {}

    # Step 1 & 2: Decode, Dequantize, and Reconstruct Feature Grids
    for l in range(1, num_levels_L + 1):
        # Extract bitstream segment for level l
        B_l_t = bitstream_B_t[l]
        
        # Decode residuals
        quantized_residual = entropy_decoder_D(B_l_t)
        
        # Reverse quantization
        min_R = min_residuals[l]
        R_hat_l_t = (quantized_residual + q * min_R) / q
        
        # Reconstruct feature grid
        F_hat_l_t_minus_1 = prev_features_F_hat.get(l, None)
        if F_hat_l_t_minus_1 is None:
            # Initialize to zero grid for first frame of GoF
            F_hat_l_t_minus_1 = [0.0] * len(ray_samples) # [G] Standard initialization for missing reference
        F_hat_l_t = [x + y for x, y in zip(F_hat_l_t_minus_1, R_hat_l_t)]
        current_features_F_hat[l] = F_hat_l_t

    # Step 3: Adaptive Selection (Simplified: use all levels)
    selected_levels = range(1, num_levels_L + 1)

    # Step 4 & 5: Render
    for l in selected_levels:
        F_hat_l_t = current_features_F_hat[l]
        color_sum = 0.0
        transmittance = 1.0

        for i, point in enumerate(ray_samples):
            # Get features at point
            features = F_hat_l_t[i]
            
            # Convert to color and density
            c_l_i, sigma_l_i = global_MLP_Phi(features)
            
            # Volume rendering step
            delta_i = 0.1 # [G] Standard step: distance between adjacent samples
            alpha = 1.0 - math.exp(-sigma_l_i * delta_i)
            color_sum += transmittance * alpha * c_l_i
            transmittance *= (1.0 - alpha)

        rendered_colors[l] = color_sum

    return rendered_colors, current_features_F_hat
```

## Subtopics

### [ReRF Player Reconstruction](12534-rerf-player-reconstruction.md)
ReRF Player Reconstruction is the decoding phase of the Neural Residual Radiance Fields (ReRF) codec, designed to recover feature grids from a compressed bitstream for the real-time rendering of dynamic free-viewpoint videos (FVV). It differs from static volume rendering by utilizing temporal redundancy through motion-compensated prediction and residual encoding, allowing adaptive streaming based on available bandwidth.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of levels $L$ and the number of samples $N$ per ray [G]. Decoding is $O(L \cdot N)$, rendering is $O(L \cdot N)$ [G].
*   **Memory Complexity:** $O(L \cdot V)$ where $V$ is the volume of the feature grid [G]. Requires storing the previous frame's feature grids for temporal prediction [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with bandwidth by selecting fewer levels $L$ [S1]. Higher levels provide finer detail but require more data [S1].
*   **Strengths:** Adaptive quality based on bandwidth; efficient temporal compression via residuals [S1].
*   **Limitations:** Drift over long sequences if not reset; dependency on previous frame accuracy [S1].
*   **When to use:** Dynamic volumetric video streaming with variable bandwidth [S1].
*   **When not to use:** Static volumes or scenarios requiring absolute precision without drift tolerance [G].

## Background
Hierarchical Progressive Decoding is part of the HPC (Hierarchical Progressive Coding) framework for volumetric video, introduced in the paper "HPC: Hierarchical Progressive Coding Framework for Volumetric Video" [S1]. It addresses the challenge of streaming dynamic free-viewpoint videos by leveraging temporal redundancy and multi-resolution representations [S1].

## Sources
[S1] HPC: Hierarchical Progressive Coding Framework for Volumetric Video, https://doi.org/10.1145/3664647.3681107, fetched 2026-10-28
